# Architecture: Spawn Finalization Subsystem

The spawn finalization subsystem owns the transition from an active spawn to a terminal state (`succeeded`, `failed`, `cancelled`, or `timed_out`). Multiple concurrent writers can attempt finalization — the runner, the reaper, a cancel operation, a launch failure handler. The subsystem's job is to ensure exactly one terminal outcome wins while preserving diagnostic metadata from all writers.

For the spawn lifecycle itself, see
[concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md). For the full
status machine and projection authority model, see
[architecture/state-system.md](state-system.md).

Drain-loop planning and resident completion behavior are documented in
[drain-plans.md](drain-plans.md). This page owns terminal-write authority and
store-level finalization.

## Finalization Authority Hierarchy

The store resolves concurrent finalization races using an authority lattice encoded in `terminal_policy.py`. Two dimensions govern the decision: whether the spawn is already terminal, and whether the incoming origin is authoritative.

**Authoritative origins:** `runner`, `launcher`, `launch_failure`, `cancel`  
**Reconciler origins:** `reconciler` (the reaper)

```
current_status = None             → reject  (spawn row doesn't exist)
current_status is active          → append  (first terminal write — always wins)
current_status is terminal (authoritative origin wrote it):
  incoming is authoritative       → reject  (first authoritative wins)
  incoming is reconciler          → reject
current_status is terminal (reconciler wrote it):
  incoming is authoritative       → replace (runner overrides inference)
  incoming is reconciler          → reject
```

This lattice is implemented as `decide_terminal_write()` in `state/spawn/terminal_policy.py` — a **pure function** with no I/O. It takes `current_status`, `current_terminal_origin`, and `incoming_origin` and returns a `TerminalWriteDecision` with disposition `"append" | "replace" | "reject"`.

---

## Durable Completion Evidence Classification

Explicit cancel intent normally wins finalization: a cancelled spawn should end
as `cancelled` with exit `130`, suppress retries, and release process scopes.
The exception is a **genuine durable completion report** produced before or
despite a late cancel signal. Because durable completion can override cancel
intent, Meridian classifies report text before using it as lifecycle evidence.

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

`spawn_store.finalize_spawn()` runs the complete finalization sequence under the stable per-spawn lock at `locks/spawns/<id>.lock` (v2 format):

1. Read current `state.json` for this spawn
2. Call `decide_terminal_write()` with current state and incoming origin
3. If disposition is `reject`: raise `_FinalizeRejected` (caught and returned as `FinalizeOutcome(wrote=False, ...)`)
4. If disposition is `append` or `replace`: apply the terminal fields to the record and write the updated `state.json` atomically
5. Return `FinalizeOutcome(wrote=True, transitioned=..., snapshot=...)`

The per-spawn lock ensures the read-decide-write sequence is atomic across concurrent processes for one spawn without blocking writes to other spawns. Losers get `wrote=False` and do not write their event — they are genuinely no-op, not just logically overridden.

(Legacy v1 used a global `spawns.jsonl.flock` — a single lock serialized all spawns. V2 replaces this with per-spawn locking, eliminating the global contention bottleneck. See [architecture/state-system.md](state-system.md) for the locked mutation seam.)

`MutationOutcome` fields (returned by `write_state_locked()`):
- `wrote` — whether this call persisted a change
- `snapshot` — the authoritative post-mutation `SpawnRecord`
- `reason` — optional decline reason when `wrote=False`

Mutators return `SpawnRecord | Decline(reason)`; the repository wraps the
result into `MutationOutcome`. The lifecycle layer adds `transitioned` on top
for status-change awareness.

---

## Reducer and Projection (Legacy V1 Reference)

In the v1 JSONL model, the event reducer in `state/spawn/events.py` replayed `spawns.jsonl` to build a `SpawnRecord`. The reducer called `decide_terminal_write()` during projection to determine whether each event's terminal fields should be applied.

In **v2**, state is a single `state.json` per spawn — no replay needed. `decide_terminal_write()` is still called, but at write time under `state.lock`, not at read time during projection. The authority policy is preserved; its application point shifted from read to write.

**Non-terminal metadata merging** (the v1 partial-merge semantics) is not present in v2: the mutator function in `write_state_locked()` takes the full current record and returns an updated record. Accounting fields (tokens, cost, duration) are included in the finalize payload and applied in the mutator when the write is accepted.

---

## CompleteSpawnOutcome: Rich Finalization Return

`SpawnApplicationService.complete_spawn()` returns a `CompleteSpawnOutcome` that tells callers exactly what happened:

```python
@dataclass(frozen=True)
class CompleteSpawnOutcome:
    wrote: bool               # True when finalize event was appended
    transitioned: bool        # True when this was a state change
    entered_finalizing: bool  # True when this call also set running→finalizing
    already_terminal: bool    # True when spawn was terminal BEFORE this call
    snapshot: SpawnRecord | None  # Post-write row state
    spawn_id: SpawnId
    
    @property
    def accepted(self) -> bool:
        return self.wrote  # first write OR replacement
```

This enables callers to distinguish:
- First terminal write (`wrote=True, already_terminal=False`)
- Authoritative override of reconciler (`wrote=True, already_terminal=True`)
- Race loser (`wrote=False, already_terminal=False`)
- Pre-existing terminal (`wrote=False, already_terminal=True`)

**CR1 fix**: Before the refactor, `complete_spawn()` short-circuited early when the pre-lock snapshot showed the spawn already terminal, bypassing the store's reducer. This meant the in-memory check and the store write weren't atomic — a concurrent writer could have written a reconciler event between the read and the "skip" decision. The fix: `complete_spawn()` always delegates to the store, which calls the policy under flock. The store is the authoritative decision point.

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
    grace_seconds=0.5,
) -> ArbitrationDecision:
```

**Priority order** (first trigger that fires wins):

1. **Terminal frame** — explicit `meridian.spawn.stop` event from the harness (most authoritative)
2. **Budget exceeded** — `budget_task` signals cost/token limit breach
3. **Timeout** — `timeout_task` deadline exceeded
4. **Watchdog** — report watchdog stopped the spawn after grace
5. **Completion** — harness process exited; waits `grace_seconds=0.5s` for a late terminal frame
6. **Signal** — SIGINT/SIGTERM received (lowest precedence)

When completion fires, the arbitrator waits up to 0.5 seconds for a terminal frame before producing a completion decision. This handles the race where the harness emits a terminal frame just before the drain future resolves.

`ArbitrationDecision` fields:
- `trigger` — which `TriggerKind` won
- `terminal_outcome` — the `TerminalEventOutcome` if a terminal frame was received
- `stop_required` — whether the caller must call `manager.stop_spawn()` 
- `synthetic_status/exit_code/error` — values to use when `stop_required` and no terminal frame
- `watchdog_noop` — True when the watchdog resolved without stopping the spawn

Both `run_streaming_spawn()` and `_run_streaming_attempt()` delegate to `arbitrate_terminal()` — there is one arbitration point, not two slightly-different implementations.

---

## StreamingRunConclusion: Attempt Accumulator

`StreamingRunConclusion` in `streaming_runner.py` accumulates execution outcome across retry attempts, replacing six previously scattered mutable sentinel locals.

```python
@dataclass
class StreamingRunConclusion:
    exit_code: int = DEFAULT_INFRA_EXIT_CODE
    failure_reason: str | None = None
    extracted: FinalizeExtraction | None = None
    final_attempt_terminal_observed: bool = False
    cancellation_observed: bool = False
    retries_attempted: int = 0

    def absorb_attempt(self, attempt: _AttemptRuntime) -> None:
        ...  # merge terminal fields from one attempt
    
    def terminal_facts(
        self, *, received_signal
    ) -> ExecutionTerminalFacts:
        ...  # project accumulated evidence into lifecycle terminal facts
```

`terminal_facts()` centralizes the logic that was previously scattered across conditionals in `execute_with_streaming()`:
- Determines `cancellation_observed` from explicit request, signal, or failure reason
- Checks `has_durable_report_completion()` for report-backed success
- Produces `ExecutionTerminalFacts` consumed by `resolve_execution_terminal_outcome()`

(The `terminated_after_completion` field and `resolve_terminal_state()` method were
removed in cleanup commit `c04171c2` — the durable-completion-vs-cancel resolution
is now handled by `resolve_completion_cancel_precedence()` in `spawn_lifecycle.py`.)

---

## PreparedExecutionHandoff: Session Scope Ownership Transfer

`execute.py` decomposes foreground and background spawn execution into three phases:

1. `_init_spawn()` — create spawn row, emit start event
2. `_prepare_execution_handoff()` — resolve session, build launch context, enter session scope
3. `_invoke_runner()` — delegate to `execute_with_streaming()`

`PreparedExecutionHandoff` is the handoff value between phases 2 and 3. It owns the `ExitStack` that holds the session scope context manager:

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

`control_root` and `task_cwd` were split in PR #210. `control_root` anchors config authority (spawn log directories, `--add-dir` roots); `task_cwd` carries the caller's task directory when it differs from the project root. When `task_cwd` is set, `bind_launch_context()` injects `MERIDIAN_TASK_DIR` into the child env and appends a `# Source-edit directory` system prompt block so the agent knows its process cwd is not its task directory. `execution_cwd` is the legacy field name and now points to the actual process working directory (child_cwd). See [architecture/launch-system.md](launch-system.md#control_root--task_cwd-split) and [decisions/launch.md](../decisions/launch.md#d-control-root-task-cwd-split) for rationale.

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
nested into typed frozen sub-models. The split prevents the reaper from treating
per-attempt evidence as spawn-level truth, and makes it structurally impossible to
construct a terminal spawn without complete terminal facts.

### Attempt-level fields (flat, overwritten on each retry)

| Field | Type | Semantics |
|-------|------|-----------|
| `last_attempt_exit_code` | `int \| None` | Exit code of the most recently drained harness attempt |
| `last_attempt_exited_at` | `str \| None` | ISO timestamp of the most recently drained attempt |

These carry no spawn-level terminal meaning. A `0` exit code can precede retries
or post-attempt budget failures.

### Runner terminal intent: `runner_exit: RunnerExitFacts | None`

Frozen sub-model (`RunnerExitFacts`) holding the runner's resolved terminal outcome:
`status: TerminalSpawnStatus`, `exit_code: int`, `error: str | None`,
`exited_at: str`. Written exactly once, before `mark_finalizing()`.

Authoritative presence check: `runner_exit is not None`. Backward-compatible
property accessors (`runner_exit_status`, `runner_exit_code`, etc.) delegate to
the sub-model so existing callers compile unchanged.

### Finalized outcome: `terminal: TerminalFacts | None`

Frozen sub-model (`TerminalFacts`) holding the complete finalized state: `status`,
`exit_code`, `finished_at`, `published_at`, `duration_secs`, token/cost metrics,
`error`, `origin`. Written by `apply_finalize()`.

### Enforced-equivalence invariant

A `model_validator(mode="before")` on `SpawnStateFields` enforces:
- Terminal `status` requires non-None `terminal` with `terminal.status == status`
- Active/unknown `status` requires `terminal is None`
- Stale flat lifecycle fields (the pre-#423 schema) are rejected outright

`_RevalidatedFrozenModel.model_copy(update=)` round-trips through
`model_validate` so the invariant survives in-memory copies. This closes the
escape hatch where Pydantic's default `model_copy` would skip validators.

### Write sequence (crash-only safety invariant)

```
1. Resolve terminal_facts from run conclusion
2. lifecycle.record_runner_exit(RunnerExitFacts(...))  ← atomic state write
3. complete_execution(terminal_facts) → mark_finalizing() → finalize(TerminalFacts(...))
```

Crash between 2 and 3: reaper sees `runner_exit` on a `running`/`finalizing`
spawn and uses it. Crash before 2: reaper sees no `runner_exit` and orphan-fails.

All finalization paths must persist `runner_exit` before calling
`complete_execution()`.

### Quarantine for out-of-vocab rows

Out-of-vocabulary persisted rows are quarantined, not coerced. Single reads
raise `SpawnStateQuarantined`; collection reads partition into `SpawnCollection`
(valid rows + quarantine reports). Migration and retention fail closed on
quarantine. See `state/spawn/.context/CONTEXT.md` for the full quarantine
contract.

### `runner_created_at_epoch`

`psutil.Process(runner_pid).create_time()` captured when `runner_pid` is written. Hardens the reaper's PID-liveness check; the prior `started_at` heuristic was fragile under delayed launch.

---

## Reaper `runner_exit` Invariant

The reaper's `decide_generic_reconciliation()` pivots entirely on
`runner_exit is not None` (the `RunnerExitFacts` sub-model). Attempt-level
fields (`last_attempt_exit_code`, `last_attempt_exited_at`) carry no terminal
weight in the reaper's decision.

### When `runner_exit is not None`

Runner resolved its outcome before crashing. Reaper uses
`FinalizeFromRunnerExit(status, exit_code, error)` — passes the runner's
decision directly to `_finalize_and_log()` without re-derivation.

`_in_post_runner_exit_grace()` (keyed on `runner_exit.exited_at`) gives the
runner a brief window to land the terminal write after persisting `runner_exit`
before the reaper steps in.

### When `runner_exit is None` and the runner is dead

Always `FailOrphan`. The reaper does **not** check `durable_report` or
`last_attempt_exit_code`. Both are attempt-level evidence that could be stale.

This closes two false-success channels that existed before this work:
- `process_exit_code == 0` → `FinalizeSucceededFromExit` (attempt exit code misread as spawn terminal exit code)
- Stale `report.md` → `FinalizeSucceededFromReport` when the runner died before resolving its outcome

### `finalizing` without `runner_exit`

Invalid / orphan state. The `finalizing` branch checks `runner_exit` first:
- Present → `FinalizeFromRunnerExit`
- Absent → `FailOrphanFinalization` — does NOT fall back to `durable_report`

Every code path that enters `finalizing` must have already persisted
`runner_exit`. Any path that enters `finalizing` without a prior
`record_runner_exit()` call is a calling-code bug; the reaper treats it as
orphan finalization.

### `FinalizeSucceededFromReport` — preserved paths

Kept for non-retry scenarios where a `report.md` cannot be stale from a prior attempt:
- Pre-worker stalls: still-published rows before detached-worker takeover
- Missing-runner-pid fallback
- `ManagedPrimaryReconciliationStrategy`

These are single-attempt paths where a durable report is reliable terminal evidence.

### Updated decision tree (sketch)

```
decide_generic_reconciliation(record, snapshot, now):
  if status == "finalizing":
    if recent_activity → Skip
    if runner_exit is not None → FinalizeFromRunnerExit
    → FailOrphanFinalization

  if runner_exit is not None:
    if post_runner_exit_grace(runner_exit.exited_at) → Skip
    → FinalizeFromRunnerExit

  # runner_exit is None — no resolved outcome persisted
  if pre_worker_ghost → ...report fallback (single-attempt path)...
  if no runner_pid → ...report fallback (single-attempt path)...
  if runner_pid_alive → Skip
  if recent_activity → Skip
  if startup_grace → Skip
  → FailOrphan("orphan_run", exit_code=last_attempt_exit_code or 1)
```

---

## Key Files

| File | Role |
|------|------|
| `state/spawn/terminal_policy.py` | Pure function: authority lattice for terminal writes |
| `state/spawn_store.py` | `finalize_spawn()` under per-spawn lock; calls policy; single locked mutation dispatch |
| `state/spawn/repository.py` | V2 storage: `read_state`, `write_state_locked`, `scan_spawn_ids`; `StoredSpawnState` quarantine validator; `Decline`, `MutationOutcome` |
| `state/spawn_aggregate.py` | Published-row lifetime: guarded artifact mutation plus deletion under stable outer spawn lock |
| `state/spawn/migration.py` | `ensure_v2_format()`: lazy one-shot migration from JSONL to per-spawn state.json |
| `state/spawn/legacy_events.py` | V1 event types, reducer, parse; still used during migration |
| `core/lifecycle.py` | `FinalizeOutcome` return type; mark_finalizing/finalize wrappers |
| `core/spawn_service.py` | `CompleteSpawnOutcome`; `complete_spawn()` always delegates to store |
| `ops/spawn/failure_policy.py` | Fixed terminal tuple for all launch failure sites |
| `ops/spawn/execute.py` | 3-phase decomposition; `PreparedExecutionHandoff`; `ExitStack` transfer |
| `launch/streaming/terminal_arbitrator.py` | Single arbitration point; priority-ordered trigger race |
| `launch/streaming_runner.py` | `StreamingRunConclusion`; delegates to `arbitrate_terminal()` |

---

## Related Pages

- [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — projection authority model and status machine
- [drain-plans.md](drain-plans.md) — streaming drain-plan composition and resident completion.
- [completion-drain-coordination.md](completion-drain-coordination.md) — shared Pi/resident completion mechanism and evidence boundary.
- [architecture/state-system.md](state-system.md) — JSONL store, atomic writes, flock locking, and the published spawn artifact lifetime boundary
- [decisions/state.md](../decisions/state.md) — design decisions for this subsystem
- [principles/invariants.md](../principles/invariants.md) — projection authority rule as invariant
