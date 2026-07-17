# Test Determinism Infrastructure

Meridian's test suite uses deterministic helpers to replace wall-clock sleeps with controlled fake-time advancement. This eliminates nondeterministic timing flakiness from async and multiprocess tests.

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

### Bounded History-Phase Polling: `wait_for_history_phase()`

After a terminal outcome, publication precedes async telemetry (phase events
written to `history.jsonl`). Tests that assert on phase events must poll with
`wait_for_history_phase()` rather than reading immediately after the outcome
future resolves. The helper polls bounded with the fake clock so it does not
introduce real-time waits.

```python
scenario = PiDrainScenario(...)
# ... drive to terminal outcome ...
await scenario.wait_for_history_phase("cleanup_completed")
```

## Real Sleeps: When They're Acceptable

Real sub-second sleeps are acceptable only when:

1. The sleep IS the behavior under test (e.g., a hook script that actually sleeps, process termination timing)
2. The sleep is in a *safety* timeout that's loose enough to be reliable (30s+) and there's no fake-time alternative

Even in these cases, design tests so that a slow CI run doesn't fail — avoid asserting on exact timing, and keep the approach cross-platform (Unix `signal.pause()` + `SIGTERM` handlers don't work on Windows).

## Related

- [codebase/guide.md](guide.md) — codebase navigation
- [../lessons/pi-rpc-quiescence-impl.md](../lessons/pi-rpc-quiescence-impl.md) — test-design lessons from Pi RPC work
- [../lessons/arch-refactor-pitfalls.md](../lessons/arch-refactor-pitfalls.md) — Windows-specific test hazards
