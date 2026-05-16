# Architecture: Spawn Finalization Subsystem

The spawn finalization subsystem owns the transition from an active spawn to a terminal state (`succeeded`, `failed`, or `cancelled`). Multiple concurrent writers can attempt finalization — the runner, the reaper, a cancel operation, a launch failure handler. The subsystem's job is to ensure exactly one terminal outcome wins while preserving diagnostic metadata from all writers.

This page covers the 2026-05 Phase 1 structural refactor of this subsystem. For background on the spawn lifecycle itself, see [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md). For the full status machine and projection authority model, see [architecture/state-system.md](state-system.md).

---

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

## Store-Level Finalization (Under Per-Spawn Lock)

`spawn_store.finalize_spawn()` runs the complete finalization sequence under the per-spawn `state.lock` (v2 format):

1. Read current `state.json` for this spawn
2. Call `decide_terminal_write()` with current state and incoming origin
3. If disposition is `reject`: raise `_FinalizeRejected` (caught and returned as `FinalizeOutcome(wrote=False, ...)`)
4. If disposition is `append` or `replace`: apply the terminal fields to the record and write the updated `state.json` atomically
5. Return `FinalizeOutcome(wrote=True, transitioned=..., snapshot=...)`

The per-spawn `state.lock` ensures the read-decide-write sequence is atomic across concurrent processes for one spawn without blocking writes to other spawns. Losers get `wrote=False` and do not write their event — they are genuinely no-op, not just logically overridden.

(Legacy v1 used a global `spawns.jsonl.flock` — a single lock serialized all spawns. V2 replaces this with per-spawn locking, eliminating the global contention bottleneck. See [architecture/state-system.md](state-system.md) for the full two-tier write model.)

`FinalizeOutcome` fields:
- `wrote` — whether this call appended an event
- `transitioned` — whether this was a status change (vs metadata-only)
- `snapshot` — the post-write spawn row (or the pre-existing row for rejections)

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
    terminated_after_completion: bool = False
    final_attempt_terminal_observed: bool = False
    retries_attempted: int = 0

    def absorb_attempt(self, attempt: _AttemptRuntime) -> None:
        ...  # merge terminal fields from one attempt
    
    def resolve_terminal_state(
        self, *, received_signal
    ) -> tuple[SpawnStatus, int, str | None]:
        ...  # compute final (status, exit_code, error) from all evidence
```

`resolve_terminal_state()` centralizes the logic that was previously scattered across conditionals in `execute_with_streaming()`:
- Determines `cancelled` from signal + terminal_observed flag combination
- Checks `has_durable_report_completion()` for report-backed success
- Delegates to `resolve_execution_terminal_state()` in `core/spawn_lifecycle.py`

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
    execution_cwd: str
    work_id: str | None
    harness_session_id_observer: Callable[[str], None]
```

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

## Key Files

| File | Role |
|------|------|
| `state/spawn/terminal_policy.py` | Pure function: authority lattice for terminal writes |
| `state/spawn_store.py` | `finalize_spawn()` under per-spawn lock; calls policy; two-tier write dispatch |
| `state/spawn/repository.py` | V2 storage: `read_state`, `write_state`, `write_state_locked`, `scan_spawn_ids` |
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
- [architecture/state-system.md](state-system.md) — JSONL store, atomic writes, flock locking
- [decisions/state.md](../decisions/state.md) — design decisions for this subsystem
- [principles/invariants.md](../principles/invariants.md) — projection authority rule as invariant
