# Architecture: Spawn Finalization Subsystem

The spawn finalization subsystem owns the transition from a running launch to a terminal state (`succeeded`, `failed`, `cancelled`, or `timed_out`). Each launch's terminal is written **once, by the holder of its `launch.lock`**: normally the runner as it exits; for a launch whose runner died, the continue, cancel, repair pass or doctor that claimed the free lock. With one writer there is no race to arbitrate, and the first terminal write wins.

For the spawn lifecycle itself, see
[concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md). For how readers
derive status and who writes a dead launch's outcome, see
[derived status](state-system/derived-status.md). Drain-loop planning and
resident completion are in [drain-plans.md](drain-plans.md). This page owns the
runner's terminal decision and the store-level write.

## One writer, first terminal wins

`spawn_store.finalize_spawn()` declines any write to a row that already has a
terminal, whatever its origin, and, given `expected_n`, any write to a row that
has moved past that launch. `origin` (`runner`, `launcher`, `launch_failure`,
`cancel`, `reconciler`) is a label for diagnostics and telemetry; nothing
compares it.

This replaced an authority lattice in which a `runner`-style origin could
replace a `reconciler` terminal. The lattice existed because the reaper used to
guess that a runner was dead and write over it; once the reaper and every other
dead-launch writer had to claim the free lock first, the runner was provably
gone and could never arrive later, so the replace branch had no caller. v0.18.0
deleted it ([D-continue-reopens](../decisions/spawn-identity.md#d-continue-reopens)).

---

## Durable Completion Evidence Classification

A **genuine durable completion report** decides the outcome in two places. A
live runner lets a durable report outrank a bare cancel request (execution
facts such as a non-zero exit still outrank the report). A launch with no
runner derives `succeeded` from a durable report with or without a cancel
request ([derived status](state-system/derived-status.md#stored-and-derived-status)).
Because a report can override cancel intent, Meridian classifies report text
before using it as lifecycle evidence.

Classification lives in `core/spawn_lifecycle.py` and is reused by extraction
fallbacks and terminal-state resolution. The important classes are:

| Class | Meaning | Lifecycle effect |
| --- | --- | --- |
| `COMPLETION` | Real assistant/user-visible completion content | May resolve to `succeeded` |
| `SYNTHETIC_FAILURE` | Runner-generated failure wrappers such as `# Spawn failed` | Never counts as completion |
| `CONTROL_FRAME` | Structured harness event/control/progress envelopes | Never counts as completion |
| `ABSENT` | Empty or missing report text | No completion evidence |

Harness adapters can emit structured JSON while a cancelled process is
unwinding. Those records are valuable diagnostics, not completion reports. The
classifier rejects these as control frames:

- **Claude**: tool/thinking/user-interrupt/system envelopes, and `result` with
  `is_error=true` such as `aborted_streaming`.
- **Codex**: connection-close errors and `thread/*`, `turn/*`, `item/*`,
  `account/*`, `mcpServer/*`, `remoteControl/*` event namespaces.
- **OpenCode**: `message.*`, `server.*`, `session.*`, and `sync` envelopes.
- **Pi/generic**: lifecycle noise, unsuccessful response/error events,
  `cancelled`/`error` event wrappers, and generated failure markdown.

This rule exists because real cancel regressions showed otherwise: Codex,
Claude, and OpenCode each emitted non-empty structured output during
cancellation, and treating that output as a durable report reclassified the
spawn from `cancelled` to `succeeded`.

---

## Store-Level Finalization (Under Per-Spawn Lock)

`spawn_store.finalize_spawn()` runs inside `write_state_locked()`, under the
per-spawn mutation lock at `locks/spawns/<id>.lock` (separate from the
runner-lifetime `launch.lock`):

1. Re-read `state.json`.
2. Decline if the row is no longer launch `expected_n` (when given) or already
   has a terminal.
3. Otherwise apply the terminal fields (`apply_finalize`) and write atomically.

A declined write emits no event. See [spawn state](state-system/spawn-state.md)
for the locked mutation seam.

`write_state_locked()` returns a discriminated `LockedMutationResult`:
- `Applied(before, after)` — the mutation persisted one atomic transition
- `Declined(snapshot, reason)` — the mutation preserved the current state
- `Missing` — the spawn did not exist when the lock was held

Mutators return `SpawnRecord | Decline(reason)`; the repository wraps the
result into the appropriate variant. The lifecycle layer derives `transitioned`
from comparing `Applied.before.status` and `Applied.after.status`.

Accounting fields (tokens, cost, duration, and a cumulative harness's raw
`harness_cumulative`) travel in the terminal and are applied in the same write.
Usage across launches is on
[spawn launches](state-system/spawn-launches.md#usage-across-launches).

---

## CompleteSpawnOutcome

`SpawnApplicationService.complete_spawn()` returns a `CompleteSpawnOutcome`
(`wrote`, `transitioned`, `already_terminal`, `snapshot`, `spawn_id`). With no
replacement path left, `wrote=True` means this call wrote the first terminal;
`wrote=False` means it wrote nothing, because a terminal was already there
(`already_terminal=True`), the row had moved past `expected_n`, or the row was
missing. `complete_spawn()` always delegates to the store
rather than skipping on a pre-lock read, so the "already terminal" decision is
made under the lock. (The `accepted` property's docstring still says "first OR
replacement"; it is just `wrote`.)

---

## Terminal Arbitration (TerminalArbitrator)

`launch/streaming/terminal_arbitrator.py` provides a single `arbitrate_terminal()` function that races all possible terminal triggers for a streaming spawn:

```python
async def arbitrate_terminal(
    *,
    completion_task,
    terminal_event_future,
    signal_task,
    timeout_task=None,
    budget_task=None,
    watchdog_task=None,
    inactivity_task=None,
) -> ArbitrationDecision:
```

It waits for the first task to finish (`FIRST_COMPLETED`). Several tasks can be
done in the same tick, so it then checks them in priority order:

1. **Terminal frame**: an explicit `meridian.spawn.stop` event from the harness (most authoritative).
2. **Budget exceeded**: `budget_task` signals a cost or token limit breach.
3. **Timeout**: `timeout_task` deadline exceeded. If a terminal frame or completion has already resolved, that wins instead.
4. **Watchdog**: the report watchdog stopped the spawn after the report grace period. It wins only if its result is `True`.
5. **Inactivity**: the inactivity watchdog stopped a stalled spawn. It wins only if its result is `True`.
6. **Completion**: the harness process exited. It uses a terminal frame that has already arrived, if there is one, and does not wait for a later frame.
7. **Signal**: SIGINT/SIGTERM received (lowest precedence). If completion resolved in the same tick, completion wins.

**Watchdogs wake on completion (#605, v0.10.10).** Both watchdogs wait on the
attempt's `completion_event` with a timeout (`_wait_for_event_or_timeout` in
`streaming_runner.py`) instead of sleeping a full poll interval. A normal finish no
longer waits for the report watchdog's next poll. The measured before/after cost
belongs to [per-operation cost](per-operation-cost.md). The report grace wait is
also capped by its remaining deadline.

Because the watchdog now finishes in the same tick as completion, a watchdog
returning `False` (it saw completion and stopped nothing) must not win: it would
have won on every normal finish. `WATCHDOG` and `INACTIVITY` therefore mean "this
task stopped the spawn". The old `watchdog_noop` flag was deleted because it became
unreachable. The lesson is in
[cost cuts that weakened safety checks](../lessons/cost-cuts-and-safety-checks.md#a-faster-wake-changed-arbitration-order-605).

`ArbitrationDecision` fields:
- `trigger`: which `TriggerKind` won.
- `terminal_outcome`: the `TerminalEventOutcome`, if a terminal frame was received.
- `stop_required`: whether the caller must call `manager.stop_spawn()`. It is `False` for `WATCHDOG` and `INACTIVITY`, because those tasks already stopped the spawn.
- `synthetic_status` / `synthetic_exit_code` / `synthetic_error`: the values to use when `stop_required` is set and no terminal frame arrived.

Both `run_streaming_spawn()` and `_run_streaming_attempt()` delegate to `arbitrate_terminal()` — there is one arbitration point, not two slightly-different implementations.

---

## StreamingRunConclusion: One-Turn Accumulator

`StreamingRunConclusion` in `streaming_runner.py` accumulates the one turn's outcome
across startup, drain, extraction, guardrails, identity conclusion, and finalization.
`_run_streaming_attempt()` returns one `_AttemptRuntime`; there is no surrounding
retry classifier or launch loop.

```python
@dataclass
class StreamingRunConclusion:
    exit_code: int = DEFAULT_INFRA_EXIT_CODE
    failure_reason: str | None = None
    extracted: FinalizeExtraction | None = None
    authoritative_terminal_status: TerminalSpawnStatus | None = None
    cancellation_observed: bool = False
    
    def terminal_facts(
        self, *, received_signal
    ) -> ExecutionTerminalFacts:
        ...  # project accumulated evidence into lifecycle terminal facts
```

`terminal_facts()` centralizes the logic that was previously scattered across conditionals in `execute_with_streaming()`:
- Determines `cancellation_observed` from explicit request, signal, or failure reason
- Checks `has_durable_report_completion()` for report-backed success
- Produces `ExecutionTerminalFacts` consumed by `resolve_execution_terminal_outcome()`

`complete_execution` then folds the durable cancel request into those facts and
calls `resolve_execution_terminal_state` in `core/spawn_lifecycle.py`, the one
place a live runner decides report versus cancel; its table is in
`src/meridian/lib/core/.context/CONTEXT.md`. The terminal write itself goes
through `SpawnLifecycleService.finalize`, which defers SIGTERM and SIGHUP
(`mask_termination`) across the store write and the `spawn.finished` hooks, so
a closing terminal can't turn a finished launch into `crashed` and a cancel
racing the finish still gets its hooks.

**Where the mask sits matters.** A deferred signal is re-delivered the moment
its mask exits. A mask inside a callee therefore protects only the callee:
when `finalize` returns, the signal fires, and the caller dies before its own
remaining steps. The streaming runner has three such steps after `finalize`:
marking spawn-owned scopes released, appending `runner_completed` to the
lifecycle log, and unregistering its atexit callback. So it holds its own
mask from `complete_execution` through `runner_completed` and the atexit
unregister (`launch/streaming_runner.py`). The signal is deferred, never swallowed: the
runner still dies by it once the mask exits (exit 143 for SIGTERM). When you add a step that must survive a cancel
racing the finish, put it inside the outermost mask that covers the finish.

---

## PreparedExecutionHandoff: Session Scope Ownership Transfer

`ops/spawn/execute.py` prepares a spawn before it reserves the row
(`_prepare_then_reserve`), so a setup error writes nothing. Execution then runs
in two phases in `ops/spawn/execute_runner.py`:

1. `_prepare_execution_handoff()` — resolve session, build launch context, enter session scope
2. `_invoke_runner()` — delegate to `execute_with_streaming()`

`PreparedExecutionHandoff` is the handoff value between the two phases. It owns the `ExitStack` that holds the session scope context manager:

```python
@dataclass
class PreparedExecutionHandoff:
    resolved_request: SpawnRequest
    launch_context: LaunchContext
    session_context: _SessionExecutionContext
    session_exit_stack: ExitStack   # owns session scope cleanup
    control_root: str       # project config/authority root (where meridian.toml lives)
    task_cwd: str | None    # task working directory when different from control_root; None otherwise
    execution_cwd: str      # legacy alias: actual process cwd (child_cwd)
    work_id: str | None
    harness_session_id_observer: Callable[[str], None]
```

`control_root` and `task_cwd` were split in PR #210. `control_root` anchors config authority (spawn log directories, `--add-dir` roots); `task_cwd` carries the caller's task directory when it differs from the project root. When `task_cwd` is set, `bind_launch_context()` injects `MERIDIAN_TASK_DIR` into the child env and appends a `# Source-edit directory` system prompt block so the agent knows its process cwd is not its task directory. `execution_cwd` is the legacy field name and now points to the actual process working directory (child_cwd). See [architecture/launch-system.md](launch-system.md#control_root--task_cwd-split) and [decisions/launch.md](../decisions/launch.md#d-control-root-task-cwd-split-split-execution_cwd-into-control_root--task_cwd) for rationale.

The ownership transfer pattern in `_prepare_execution_handoff()`:
```python
local_stack = ExitStack()
try:
    ...  # enter session scope into local_stack
    
    # Transfer ownership on success — caller is now responsible for cleanup
    handoff_stack = local_stack
    local_stack = ExitStack()  # replace with empty stack so finally doesn't close it
    return PreparedExecutionHandoff(session_exit_stack=handoff_stack, ...)
except Exception:
    local_stack.close()  # close if we're about to raise — no leak
    raise
```

If preparation succeeds, the caller owns `handoff.session_exit_stack` and must close it after the runner completes (via `_close_execution_handoff()`). If preparation fails, the stack is closed before re-raising, guaranteeing no session scope leak regardless of failure point.

---

## Launch Failure Consolidation (failure_policy.py)

All launch-failure finalization routes through `ops/spawn/failure_policy.py`:

```python
async def finalize_launch_failure(
    runtime_root, project_root, spawn_id, error
) -> CompleteSpawnOutcome:
    # Fixed tuple: status="failed", exit_code=1, origin="launch_failure"
```

Before the refactor, 8 scattered call sites each independently chose `status`, `exit_code`, and `origin` for launch failures. Some used different exit codes or different error strings for the same failure class.

`failure_policy.py` enforces the fixed terminal tuple for all launch-failure scenarios: `status="failed"`, `exit_code=1`, `origin="launch_failure"`. A sync variant `finalize_launch_failure_sync()` serves non-async call sites (background worker cleanup).

---

## Discriminated Lifecycle Facts (PR #423, Typed State Contracts)

Lifecycle evidence that was previously spread across flat top-level fields is now
nested into typed frozen sub-models. The split prevents the reaper from treating a
pre-finalization harness exit as spawn-level truth, and makes it structurally
impossible to construct a terminal spawn without complete terminal facts.

### Legacy exit fields: `runner_exit`, `last_attempt_*`

`runner_exit` (`RunnerExitFacts`: the runner's resolved outcome) and
`last_attempt_exit_code`/`last_attempt_exited_at` (the raw harness exit) are
read-only legacy fields. No new-build code writes them; a reopen clears them
with the other per-launch fields. They are read only to date a derived
terminal's `finished_at` and for display in `spawn show`. Two writes used to
precede the terminal (`record_runner_exit`, then `mark_finalizing`); both were
deleted, and `record_exited` now only emits `spawn.process_exited` telemetry.

### Finalized outcome: `terminal: TerminalFacts | None`

Frozen sub-model (`TerminalFacts`) holding the complete finalized state: `exit_code`,
`finished_at`, `published_at`, `duration_secs`, token/cost metrics,
`error`, `origin`, `harness_cumulative` (the raw running total of a cumulative
harness), `managed_scopes_pending`. Does not carry status; top-level `status` is
the sole status authority. Written by `apply_finalize()`.

### Single status authority

Top-level `status` is the sole status field. `TerminalFacts` carries terminal
evidence but does not repeat status. `StoredSpawnState` uses `extra="forbid"`,
so unknown fields on a v3 row are quarantined rather than silently accepted.
When `status` is terminal, `terminal` must not be `None`; when `status` is
active or `unknown`, `terminal` must be `None`.

Writers stamp `v: 4` when the row has a `launch`, else `v: 3`
([spawn launches](state-system/spawn-launches.md#on-disk)). Known legacy v2 shapes (flat runner-exit/terminal
fields, the interim nested `terminal.status`, retired pre-main fields,
pre-`published_at` history) upgrade in memory through `state/spawn/legacy.py`
before the strict model validates — old history stays readable, and only
conflicting or truly unknown legacy rows quarantine. Reads never rewrite
legacy files. See the legacy-upgrade decision in
[../decisions/state.md](../decisions/state.md).

### Write sequence (crash-only safety invariant)

The runner makes one status write after the harness exits: the terminal, with
report-versus-cancel decided in memory. A crash anywhere before it frees the
runner's `launch.lock`, and readers derive the outcome from the launch's
directory ([derived status](state-system/derived-status.md#stored-and-derived-status)).

### Quarantine for out-of-vocab rows

Out-of-vocabulary persisted rows are quarantined, not coerced. Single reads
raise `SpawnStateQuarantined`; collection reads partition into `SpawnScan`
(valid rows + quarantine reports). Migration and retention fail closed on
quarantine. See `state/spawn/.context/CONTEXT.md` for the full quarantine
contract.

### `runner_created_at_epoch`

`psutil.Process(runner_pid).create_time()` captured when `runner_pid` is written. Status no longer reads it for lock-backed rows. It feeds the exact-birth checks that remain: `legacy_runner_alive` for rows with no lock file, the continue's choice of refusal message, managed-primary cleanup, and cancel's signal targeting, which signals only that exact `(pid, birth)`.

---

## Dead-launch outcome

When a runner dies without writing its terminal, the next holder of its free
lock writes the outcome readers already derive, through
`SpawnApplicationService.finalize_dead_launch`: a continue, cancel, the
background repair pass, or `doctor --kill-orphans`. The rule, the claim and the
callers are on [derived status](state-system/derived-status.md#who-persists-a-dead-launchs-outcome).
Before identity-v2 PR 3 the reaper ran a guessing decision tree here
(`decide_generic_reconciliation`); the
[deleted-mechanisms table](state-system/derived-status.md#deleted-mechanisms-do-not-bring-back)
says why each piece must stay gone.

---

## Key Files

| File | Role |
|------|------|
| `state/spawn_store.py` | `finalize_spawn()` (first terminal wins, `expected_n` guard), `start_spawn`, `reopen_spawn`; single locked mutation dispatch |
| `state/spawn/repository.py` | V2 storage: `read_state`, `write_state_locked`, `scan_spawn_ids`; `StoredSpawnState` quarantine; `Applied`, `Declined`, `Missing` |
| `state/spawn_aggregate.py` | Published-row lifetime: guarded artifact mutation plus deletion under stable outer spawn lock |
| `state/spawn/legacy.py` | One-shot v2→v3 in-memory upgrade at the parse boundary; `RETIRED_LEGACY_FIELDS`; deletable once pre-v3 rows are gone |
| `state/spawn/migration.py` | `ensure_v2_format()`: lazy one-shot migration from JSONL to per-spawn state.json |
| `state/spawn/legacy_events.py` | V1 event types, reducer, parse; still used during migration |
| `core/lifecycle.py` | `SpawnLifecycleService`: `record_launch_facts`, `finalize` (masked store write, then hooks), lifecycle event IDs per launch |
| `core/spawn_service.py` | `CompleteSpawnOutcome`; `complete_spawn()` always delegates to store; `finalize_dead_launch`; the one `cancel` |
| `state/spawn_status.py` | `derive`, `read_derived`, `claim_dead_launch`, `poll_launch`/`wait_until` |
| `ops/spawn/failure_policy.py` | Fixed terminal tuple for all launch failure sites |
| `ops/spawn/execute.py`, `execute_runner.py` | prepare-then-reserve; `PreparedExecutionHandoff`; `ExitStack` transfer |
| `launch/streaming/terminal_arbitrator.py` | Single arbitration point; priority-ordered trigger race |
| `launch/streaming_runner.py` | One-turn startup, drain, watchdog, teardown, `StreamingRunConclusion`, and finalization; delegates to `arbitrate_terminal()` |

---

## Related Pages

- [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — the status machine
- [state-system/derived-status.md](state-system/derived-status.md) — derived status, dead-launch writers, cancel
- [drain-plans.md](drain-plans.md) — streaming drain-plan composition and resident completion.
- [completion-drain-coordination.md](completion-drain-coordination.md) — shared Pi/resident completion mechanism and evidence boundary.
- [architecture/state-system/overview.md](state-system/overview.md) — state store, atomic writes, flock locking, and the published spawn artifact lifetime boundary
- [decisions/state.md](../decisions/state.md) — design decisions for this subsystem
- [Per-operation cost](per-operation-cost.md) — measured finalization floor and
  constant-time acceptance bar
- [Cost cuts that weakened safety checks](../lessons/cost-cuts-and-safety-checks.md)
  — why the watchdog trigger must prove it caused the stop
