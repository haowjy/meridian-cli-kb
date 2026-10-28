# Test Determinism Infrastructure

Meridian's test suite uses deterministic helpers to replace wall-clock sleeps with controlled fake-time advancement, so a test's outcome doesn't depend on how fast the machine runs. Fake time covers only code that reads the patched clock; the sections below cover the other sources of flakiness (file-versus-API windows, processes still writing, interpreter boundaries).

A flaky test is fixed this way or deleted; it is never retried, skipped or given a longer timeout ([D-no-flaky-tests](../decisions/testing.md#d-no-flaky-tests-fix-it-or-delete-it)). Each fake clock in Python comes from `tests.support.fakes.FakeClock`; don't add a local clock stub.

## Principle: Advance Fake Time Past the Boundary

Tests that need to cross a timeout, deadline, or expiry boundary should advance fake time to just past that boundary, then assert the outcome. Adding `asyncio.sleep(0.1)` or `time.sleep(0.5)` just to cross a boundary is a flake waiting to happen — real wall-clock timing varies with machine load, CI contention, and OS scheduling.

**Correct pattern:**

```python
# Advance fake time PAST the expiry window, then assert
await det.clock.sleep(expiry_seconds + 0.1)
assert condition_is_now_true()
```

**Wrong pattern (flake-prone):**

```python
await asyncio.sleep(0.1)  # hoping this crosses the boundary — it might not
assert condition_is_now_true()
```

## Async Determinism: `tests/support/async_determinism.py`

Core harness for async tests that would otherwise need real sleeps.

### `AsyncDeterminism`

Bundles a `FakeClock` with patches for `time.monotonic` and `asyncio.sleep`:

```python
from tests.support.async_determinism import AsyncDeterminism

det = AsyncDeterminism(start=0.0)
det.install(monkeypatch, monotonic_modules=(module_under_test,))
# After install: asyncio.sleep(delay) advances fake clock by delay.
```

**Install targets:**

- `monkeypatch.setattr(asyncio, "sleep", self._fake_sleep)` — all `await asyncio.sleep(d)` calls advance the fake clock
- `monkeypatch.setattr(module.time, "monotonic", self.clock.monotonic)` — for each module in `monotonic_modules`, replaces `time.monotonic()` with the fake clock

**`install_on_running_loop(monkeypatch)`** — patches the already-running event loop's `time()`. Use from inside an async test when the loop was created before `install()`.

### `assert_still_pending(task)`

Yields one event-loop tick, then asserts the `asyncio.Task` hasn't completed. Replaces the flaky pattern of `await asyncio.sleep(0.01); assert not task.done()`, where the sleep might be long enough for the task to actually finish on a fast machine.

```python
await assert_still_pending(task)
# Now advance fake time to cross the boundary
```

### `TaskGate`

Manual barrier for ordering concurrent async work. Waiters block on `await gate.wait_open()` until another coroutine calls `gate.open()`.

```python
gate = TaskGate()
# Worker coroutine:
async def worker():
    await gate.wait_open()  # blocks until released
    ...

# Test:
task = asyncio.create_task(worker())
await assert_still_pending(task)  # worker is blocked
gate.open()  # release
```

### `yield_to_loop()`

Yields one event-loop tick (`asyncio.sleep(0)` via the real asyncio) without advancing fake time. Use when you need to let pending callbacks run without changing the clock.

## Thread Coordination: Observe State, Not Elapsed Time

Threaded tests synchronize on `threading.Event` or another explicit state
transition. Wait for the worker to report that it entered the operation, trigger
the action under test, then wait for the worker's completion event. Assert the
resulting thread and resource state directly; do not infer prompt cancellation
or non-blocking behavior from an elapsed-time threshold. Event wait timeouts are
hang guards, not behavioral assertions.

## Wait on What the API Reports, Not on Its Persistence File

A component that persists state and then adopts it in memory has a window where
the file is ahead of the API. A test that polls the file, sees the new state, and
then calls the API can land in that window. The API then answers from the old
in-memory state, correctly.

#606 was this case. `BashPublication` writes the terminal snapshot to
`bash-records.json` atomically and only then adopts it in memory. That order is
deliberate: a failed write must not leave memory ahead of disk. The
notification-dedup test polled the file for a terminal record, then called
`bash_manage output` once. Inside the window, `output` still saw `running`, so it
correctly wrote no consumed mark, and the test failed. The product was right: the
agent had been told "running", so a later completion notice was legitimate. The
test failed twice in CI on one day and never locally (200/200 plain, 100/100 under
load). An injected 100 ms delay between persist and adopt made it fail 20 of 20
runs, which proved the window.

The fix (PR #608, test-only) retries `output` until the status *it returns* is
terminal, then asserts the mark. Between attempts it yields 10 ms. That is a poll
inside a condition wait, not a fixed sleep: it lets the adoption task run even if
`output` ever stops awaiting I/O. A 2 s deadline reports the attempt count and the
last status. Rejected: reordering the product to adopt before persisting (that
weakens failed-write semantics), and loosening the assertion (the test would stop
guarding against double notification).

#662 was the same window in the other direction: memory ahead of disk. Before a
managed-bash task ping fires, the runtime claims it by setting `ping_sent_at_ms`
in `bash-records.json`; if the ping hook throws, it releases the claim back to
`null`. The ping-retry test made the hook throw, waited until the hook had been
called once, slept 60 ms, then asserted the claim was released on disk. The hook
counts its call before it throws, and `firePing()` publishes the release later,
asynchronously. One CI run
read the file between the two and saw the claim still set; 150 local loops
never did. The fix (#664, test-only) polls the records file until the release
is on disk, then keeps the 60 ms only as the window in which a broken hook would
retry. Deleting the test was rejected: it is the only coverage of a ping retried
after new hooks are bound to the same running shell.

**The rule.** Synchronize on the condition the code under test reads. If the
assertion is about what an API does, wait on that API's own answer; if it is
about what reached disk, wait on the file. A callback, a counter, a log line or a
sibling projection can lead or lag it.

## Wait for the Runner Before Deleting a Spawn Directory

A spawn has three processes that can write into `spawns/pN/`: the harness, the
worker, and the runner that finalizes. `test_continue_ignores_pruned_successor`
(#647) waited for p2's harness and worker, then `rmtree`d `spawns/p2` to simulate
retention. The runner was still finalizing and writing, and the fast gate failed
with `ENOTEMPTY`. The fix waits for the runner to exit too; the file then passed
6× clean at `-n 4`.

**The rule.** Before a test deletes or rewrites a spawn's directory, wait for every
process that writes there to exit, the runner included. "The harness is done" is
not "nothing writes here any more". Since #647 the runner's exit is observable as
`spawns/pN/launch.lock` becoming free.

## Fixtures That Mean "Live" Must Hold the Lock

Under [derived status](../architecture/state-system/derived-status.md) (identity-v2 PR 3),
a stored-active row reads as `running` only while something holds its
`spawns/pN/launch.lock`. A row seeded with no lock file is a pre-PR-2 legacy
row, and `legacy_runner_alive` keeps it alive only for 15 s after `started_at`
when it has no runner PID. A test that seeds such a row and means "a live
spawn" passes only because it reads within that grace. Under xdist load or a
slow first import the row flips to `failed` and the test fails: the timing-flake
class PR 3 exists to remove.

- **A fixture that means "live" holds a `LaunchSlot`** (the runner-side holder
  of the exclusive `launch.lock`), or a subprocess holding the lock, for the
  duration of the test. PR 3 converted idle children, spawn
  tree, API stats, runner-history prune and the Pi drain fixtures.
- **A lockless row is only for testing legacy behaviour.** A test of the rule
  "the reaper never finalizes a lockless row" must use a row *past* the grace;
  inside it, `read_derived` already calls the row alive, so the test passes
  whether or not the reaper skips lockless rows.
- **A fixture that means "dead" uses a free lock**; for a legacy row, the PID
  and birth time of a child the test started and already waited on, so the PID
  is known dead. Never a large constant PID such as `999_999`:
  `pid_max` can exceed it, and cancel will signal whatever process owns it.

## Race Tests Must Go Red Against the Split Implementation

`read_derived` is correct only because it holds the shared lock *across* the row
read. The first version of `test_read_derived_races` ran a real writer against a
real reader and claimed to cover both orderings. The test-architecture review
swapped in the two broken versions:

| Implementation under test | Runs that went red |
|---|---|
| try the lock, release, then read the row | 0 of 20 |
| read the row, then try the lock | 19 of 20 |
| the real `read_derived` | 0 of 20 |

The first window is a few microseconds; across about 36,000 free-running reads
it was hit once. Black-box timing can't red-prove it.

The fix made the interleaving deterministic: in that test only, a wrapper around
`spawn_status.read_state` tells the writer to try `LOCK_EX|LOCK_NB` and publish
an active row before the read proceeds. With the real implementation the shared
lock blocks the writer; with the split one the writer gets in and the reader
reports `crashed`, every time. Coupling the test to one internal call is
justified because the property under test is "held across the read".

**The rule.** A race test is evidence only after it fails against each plausible
wrong implementation. For a read that must hold a lock across it, those are
"probe the lock, release, then read" and "read, then probe the lock". Write
them, run the test against each, and
record the red output with the PR. If timing can't hit the window reliably,
inject the interleaving at the seam the property is about.

## Patches Stop at the Interpreter Boundary

A pytest fixture or `monkeypatch.setattr` changes a module object in the pytest
process only. A test that launches a new Python interpreter (`python -m
meridian`, a driver script, a background runner the CLI starts) gets a fresh
import with the production value. Environment variables do cross, so
`MERIDIAN_HOME` and `_MERIDIAN_RUNTIME_DIR` isolate children; a module constant
with no environment override does not.

The control-socket root (`_POSIX_SOCKET_ROOT` in
`lib/streaming/control_socket.py`, `/tmp/meridian-<uid>` in production) is such
a constant, shared by every process the user runs. The session fixture in
`tests/conftest.py` redirects it to a short private directory, for the pytest
process alone. The launch-lock crash tests (`tests/platform/test_launch_lock.py`)
run the CLI and its runner as separate interpreters through
`tests/support/launch_lock_driver.py` and SIGKILL them on purpose, so their
sockets landed in the real user directory and were never removed. They made up
59 of the 69 control sockets found there on one host.

To isolate such a child, pass the patched value in an environment variable
and have every child interpreter apply the same patch before it runs Meridian:
the launch-lock driver does this in each process it starts, and
`tests/integration/cli/test_spawn_report.py` runs the CLI through a `python -c`
that patches first. The heavier alternative is a test-only `sitecustomize`,
which every child imports at startup (the `--runner-history=off` trap in
`tests/AGENTS.md` works this way); for a single constant, explicit propagation
was preferred over intercepting every child. A deliberately killed process
cleans up nothing, so for these tests isolation is what keeps residue out of
shared user directories.

## Cross-Process Race Harness: `tests/support/process_race.py`

Runs N worker processes under `multiprocessing.get_context("spawn")` with a ready barrier — every child arms before any is released, maximizing real contention.

### `run_spawn_race_or_skip(target, worker_args, *, timeout=120.0) -> list[T]`

```python
from tests.support.process_race import run_spawn_race_or_skip

results = run_spawn_race_or_skip(
    reserve_chat_id,
    [(runtime_root,) for _ in range(8)],
)
assert sorted(results) == [f"c{i}" for i in range(1, 9)]
```

**Parameters:**

- `target` — top-level callable (must be importable for spawn safety); invoked in each child after barrier release
- `worker_args` — one argument tuple per worker; `len(worker_args)` is the worker count
- `timeout` — single overall deadline covering arm, execute, join, and result drain

**Behavior:** Arm barrier ensures every worker is ready before any proceeds. Results are returned ordered by worker index. Raises `pytest.skip` when multiprocessing semaphores are unavailable (e.g., restricted environments). Raises `AssertionError` on timeout or any worker failure.

**`WorkerOutcome`, `SpawnRaceOutcome`, and the base `run_spawn_race` are internal** — callers use only `run_spawn_race_or_skip()`.

## Behavioral vs Safety-Bound Timeouts

Tests use two kinds of timeouts with different purposes and failure modes:

| Kind | Purpose | Example | When it flakes |
|---|---|---|---|
| **Behavioral** | Assert the SUT times out correctly | `pytest.raises(asyncio.TimeoutError)` after advancing fake time past deadline | If the test logic is wrong |
| **Safety-bound** | Prevent a hang from blocking CI forever | `asyncio.wait_for(sut_call, timeout=1.0)` as a hang guard | If real event-loop work exceeds the bound on contended CI |

**Rule:** Keep behavioral timeouts (they assert correct behavior). Remove redundant safety-bound timeouts when the global `pytest-timeout=60` already guards against hangs.

### Windows-Gate Flake Example

A test used `asyncio.wait_for(manager.wait_for_completion(...), timeout=1.0)` as a hang guard. The SUT timing was fake-clock-driven, so the 1.0s real-time bound wasn't asserting anything behavioral — it was only preventing CI hangs. On contended Windows CI, real event-loop work occasionally exceeded 1s, causing intermittent `TimeoutError`.

**Fix:** Remove the tight safety bound and call `await manager.wait_for_completion(...)` directly. The global `pytest-timeout=60` already catches hangs.

Tests that were weakened during de-flake (asserting less to avoid timing flakiness) were restored to full coverage by advancing FakeClock past the boundary: liveness suppression crossing the expiry window, resident rearm crossing the original deadline, lock-blocking with a bounded "stays blocked until release" wait.

## PiDrainScenario Builder: `tests/support/pi.py`

`PiDrainScenario` is the canonical way to write Pi drain and quiescence tests.
It provides a fake `PiRpcConnection`, pre-wired `SpawnManager`, spawn store,
and helpers for emitting events, advancing fake time, and asserting drain
outcomes. Five Pi characterization test files were consolidated behind it in
PR #375 (#372).

Use the builder instead of manually constructing `SpawnManager` +
`PiDrainCoordinator` + fake connections. Direct `manager._sessions` seeding
is a known anti-pattern (tracked in #430).

### Pi Process-Exit Fidelity Helpers

Real `PiRpcConnection` emits an `error/connectionClosed` event with the
process exit code before the event iterator reaches EOF. Test fakes that
model only clean iterator exhaustion or direct `handle_stream_exit(None)`
miss the real precedence path where the generic process-exit failure arrives
before stream exit. Two reusable helpers close this gap:

- `pi_process_exit_event(return_code)` — builds the canonical
  `error/connectionClosed` event with the Pi subprocess exit message.
- `write_pi_bash_record(runtime_root, spawn_id)` — writes managed-bash disk
  evidence representing Pi-private background work that has no Meridian spawn
  row.

Pi exit-classification tests should use these helpers to reproduce the
production event shape. The root cause (#433, pre-existing since #225) was
that fakes modeled EOF but never Pi's non-zero-exit
`error/connectionClosed` event, so the exit predicate and precedence path
were never exercised with realistic evidence.

### Bounded Lifecycle-Phase Polling: `wait_for_phase()`

After a terminal outcome, publication precedes async teardown, so Pi lifecycle
phase events keep arriving after the outcome future resolves. Tests observe the
live events through the manager's event hook (a list they pass in as
`observed`), not by re-reading runner `history.jsonl`: under
`--runner-history=off` no stream exists. Poll with
`tests/support/pi.wait_for_phase(events, phase, count=1)`, which wraps
`wait_until` with a 5 s bound. The durable form of the same facts is the
`pi-lifecycle.json` sidecar, which `spawn show` reads.

```python
observed: list[RawHarnessEvent] = []
# ... build the manager with event_hook=observed.append, drive to terminal outcome ...
await wait_for_phase(observed, "cleanup_completed")
```

## Monkeypatchable Module Finals: A Flake Class

Tests that `monkeypatch.setattr` a module-level timing `Final` (e.g.,
`_FIRST_STDOUT_AFTER_INITIAL_PROMPT_TIMEOUT_SECONDS`) to shorten a timeout race CI
load. The production code reads the Final at runtime; the test overrides it to a
sub-second value so the timeout fires quickly. On a contended CI runner, the real
event can arrive before the shortened timeout expires, flipping the outcome. This
produced at least two CI flake issues (#459, #465) on the same module Finals.

The flake mechanism: `monkeypatch.setattr` replaces the value on the module object,
but the production code reads it at an arbitrary point during execution. If the
timeout is short enough (50-200 ms), real event delivery races the deadline, and the
test outcome depends on CI scheduling.

**Remedy pattern — injected frozen policy:**

1. Replace module-level timing Finals with a frozen `@dataclass` (e.g.,
   `PiRpcTimingPolicy`) carrying all timing constants.
2. Inject the policy at the constructor. Production constructs with defaults;
   tests construct with short but race-free values (1-2 s, not 50 ms).
3. Stamp deadlines at the producing event (e.g., prompt-write success), not at
   the consumer's first iteration.
4. **Delete the old Finals, do not alias them.** `monkeypatch.setattr` on a
   missing attribute raises `AttributeError`, so any straggler test fails
   loudly instead of monkeypatching an inert name and passing silently.

This pattern applies whenever tests need to exercise timeout behavior on a real
subprocess integration test. The injected policy makes the timeout value a
constructor concern, not a module-global mutation.

## Tight Subprocess Timeouts: A Flake Class

Tests that wrap `subprocess.run(["python", "-m", "meridian", ...], timeout=N)`
with a short timeout (e.g., 3 seconds) race CI load. The test expects the
subprocess to complete or fail within the bound, but on a contended runner the
process startup alone can exceed it. The test passes locally and fails
intermittently in CI.

**Known instance (fixed):** `tests/integration/cli/test_spawn_prompt_input.py`
used `timeout=3` on 6 `subprocess.run` calls wrapping `python -m meridian` CLI
invocations. Failed once on a release preflight run (#469) due to CI
contention; PR #471 raised the hang-guard to 20s.

The flake mechanism is the same class as Monkeypatchable Module Finals above:
a timing boundary that's too tight for worst-case CI scheduling. The remedy
differs because the timeout is on a real subprocess, not a monkeypatched
constant:

1. If the bound is only a hang guard, drop it and let the suite's
   pytest-timeout catch hangs. #471 raised this one to 20 s before the
   no-flaky-tests rule; under that rule a raised bound is the wrong fix.
2. If the test asserts timeout behavior (the subprocess should time out),
   use a deterministic signal instead of a wall-clock race.

## Cold Index Builds Inside a Test Deadline: A Flake Class

A test whose first query builds the history index from cold can spend most of its
deadline on that build. Under suite load the build alone exceeds the deadline.
The test passes serially and fails in the full suite.

**Known instances.** There were five, each patched test-side by warming the index:
- **#577 → #624.** Four Pi delivery-contract tests. A cold build took
  1.09–1.94 s under load, inside the 2 s `QUERY_TIMEOUT`, while incremental
  catch-up took 0.11–0.16 s. Red baseline: 4 failures with `-n 8` beside the full
  suite at load 18–21. After warming: 30/30 per file at mean load 30, peak 46.
- **#630 → #637.** `test_reconciled_descendant_evidence_is_transitive_and_reconciled`
  returned `unknown` instead of `blocked`. #616 had fixed the same shape on its
  branch (pass 7). That fix was lost when #616 closed unmerged.
- **#638 → #645.** Pi quiescence disk-race tests. A cold first refresh left the
  unbounded `wait_for_aux_wake()` pending until the xdist worker timeout.
- **#551 → #649.** The retained-child timeout test
  (`test_spawn_manager_child_wave_timeout_publishes_before_descendant_cancellation`).
  `HistoryIndexBusy` left the assessment `unknown` until the hang guard fired.
  1 failure in 4 under load before the fix; 30/30 after.
- **#643 → #650.** The resident guardrail test
  (`test_streaming_runner_failure_policy.py`). The cold build ran inside the test's
  15 s wait. 1 in 10 before; 30/30 after.

**The shared cause was a product bug, and #656 fixed it at the source**
(2026-10-16). Descendant evidence started its 2 s clock and then let `query()`
build the cold index inside it. So under load the product answered `unknown` when
it should have answered `blocked`. Now it builds first and starts the 2 s afterward
([rule](../architecture/state-system/portable-history.md#a-caller-with-a-deadline-builds-the-index-before-starting-its-clock)).
Tests no longer need to warm the index to get a correct descendant-evidence answer.
`test_cold_build_gets_a_separate_descendant_query_deadline`
(`tests/integration/streaming/test_descendant_evidence.py`) pins this without
relying on machine load. It sets the module's `QUERY_TIMEOUT` to 0.2 s, adds a
0.4 s sleep to `HistoryIndex._rebuild_locked`, asserts the slow path ran, and
expects `blocked`. It was red on `main` before #656.

**The existing warm-ups stay.** `warm_history_index()` (`tests/support/history.py`,
moved there by #637) is called at 9 sites in 6 files, all added by the five fixes
above. None of them is needed for correctness anymore. They still keep the cold
build, which is 1–2 s under suite load, out of each test's own wall-clock bounds:
- the guardrail test's 15 s wait;
- the Pi hang guards;
- `wait_for_aux_wake()`, which only the worker timeout bounds (#654 tracks
  bounding it).

#656 kept every warm-up. Nobody has checked whether these tests pass cold under load
without them. Don't delete them as dead code.

**Remedy for a new instance:**
1. If a product caller passes a deadline into an index read, fix the caller to
   build first. That's the bug.
2. If the test has its own wall-clock bound, warm the index with
   `warm_history_index()` before the measured call, so the bound covers only the
   behavior under test.

Don't widen `QUERY_TIMEOUT`; the product's bound is right. A fix that lives only on
an unmerged branch is not a fix on `main`. When a branch closes, file what it fixed
incidentally. Five test-side patches for one cause should have sent someone to look
at the product sooner.

## Real-Time Deadlines in Fake-Clock Tests

A test that advances a fake clock but waits through a helper with a *real*
monotonic deadline is two clocks at once. The fake clock makes the product's
timing deterministic. The real deadline still races the scheduler, so a loaded
suite can time out the wait while the fake-time behavior is correct.

**Known instance (#632, open):**
`tests/integration/idle/test_sidecar.py::test_sidecar_drives_idle_push_warn_compact_and_done`
waited for `state.done["compact"] == "ok"` through `wait_until`
(`tests/support/async_determinism.py`), which then had a 1 s real deadline
(its hang guard is now 30 s). It failed
twice in full-gate runs during PR 1's verification and passed alone in 0.54 s.
Scheduler starvation is the leading hypothesis. A dropped schedule or compaction
event has not been ruled out. Idle compaction has since been deleted; the test
is now `test_sidecar_drives_idle_push_and_warn` and waits the same way for
`state.done["warn"] == "sent"`, so the two-clock hazard remains.

**Remedy:** synchronize on the event the test is waiting for, not a wall-clock
bound. A wait helper has no deadline of its own beyond a generous hang guard:
the Pi tests share one `until()` (`pi_runtime/extensions/shared/test_wait.ts`)
bounded only by vitest's `testTimeout`, which replaced six helpers that each
carried a short deadline (#809). Reproduce under full-gate ordering first, to tell starvation from a lost
event. Don't raise the deadline blind.

## Hung Tests Report Where They Hung

pytest-timeout runs with `timeout_method = "signal"` (`pyproject.toml`). The
`thread` method ends a hung test by calling `os._exit`, so under xdist it shows
as `node down: Not properly terminated` with no stack. One CI run lost four
workers about 65 s after its last progress line, which read as a crash or OOM;
a throwaway reproduction with `--timeout-method=thread` gave the same log
shape from four ordinary timeouts. With `signal` the test fails as
`Failed: Timeout` with a traceback. The cost: a stuck non-daemon thread can
still hang a `-n0` run at interpreter exit, which CI's job `timeout-minutes`
bounds.

## Real Sleeps: When They're Acceptable

Real sub-second sleeps are acceptable only when:

1. The sleep IS the behavior under test (e.g., a hook script that actually sleeps, process termination timing)
2. The sleep is in a *safety* timeout that's loose enough to be reliable (30s+) and there's no fake-time alternative

Even in these cases, design tests so that a slow CI run doesn't fail — avoid asserting on exact timing, and keep the approach cross-platform (Unix `signal.pause()` + `SIGTERM` handlers don't work on Windows).

## Related

- [codebase/guide.md](guide.md) — codebase navigation
- [../lessons/pi-rpc-quiescence-impl.md](../lessons/pi-rpc-quiescence-impl.md) — test-design lessons from Pi RPC work
- [../lessons/arch-refactor-pitfalls.md](../lessons/arch-refactor-pitfalls.md) — Windows-specific test hazards
