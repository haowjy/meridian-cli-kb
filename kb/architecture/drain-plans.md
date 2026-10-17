# Drain plans separate lifecycle policy from generic streaming

`drain_plan_factory.build_drain_plan()` returns the complete `DrainPlan` for one
active spawn. It makes the drain loop's lifecycle policy explicit while leaving
the generic streaming loop harness-agnostic.

## Plan composition

- `coordinator: DrainCoordinator | None` — optional harness-specific completion
  policy object.
- `policy: DrainPolicy | None` — terminal-event behavior (`SingleTurnDrainPolicy`
  by default).
- `raw_terminal_frames_authoritative` and `on_policy_selected` — constant loop
  configuration.
- `aux_wake` / `handle_aux_wake` — non-event wake sources such as Pi disk changes.
- `finalizer` — synchronous finalization effects;
- `teardown` — plan-owned async connection and cleanup-phase policy.

The plain streaming path is intentionally `DrainPlan(coordinator=None)`. There is no
public no-op coordinator. Absence of a coordinator means the generic drain loop handles
events with the selected `DrainPolicy` and finalizes from harness terminal frames or
connection close.

`DrainCoordinator` is narrow: start/stop, timeout selection, event observation,
terminal-event handling, timeout handling, after-event checks, close
classification, and stream-exit handling. The factory composes plain, resident,
and Pi plans; `SpawnManager` supplies generic capabilities and holds no Pi
policy.

Terminal publication is owned by `SpawnManager._publish_terminal()`, an
idempotent barrier called by both the drain loop's natural exit and
`stop_spawn()`. `resolve_terminal_outcome()` resolves competing sources:
success wins; otherwise an authoritative stop outcome wins; otherwise the drain
classification stands. The barrier publishes the outcome, resolves the
completion future, and starts one per-spawn async cleanup task. A stuck
cleanup or connection stop cannot hang publication. Cleanup reports are
telemetry and cannot replace the outcome; the startup reaper reconciles
incomplete cleanup after a crash.

## Resident completion

Codex and OpenCode managed backends can stay resident after a successful turn while
descendant Meridian spawns are still running. `ResidentDrainCoordinator` owns that
model:

1. A successful terminal turn does not immediately finalize if descendant work exists
   or the resident backend requested re-arming.
2. The coordinator emits a turn boundary, marks the backend as `awaiting_done`,
   and polls the shared reconciled transitive descendant tree.
3. `meridian spawn done` and `meridian spawn rearm` are file signals consumed by the
   coordinator. `done` overrides known blockers only. If the fresh assessment is
   `unknown`, `done` waits rather than turning unreadability into success.
   `rearm` opts the backend into explicit residency with a fresh deadline and
   advisory poll messages.
4. When evidence becomes readable, `done` can finalize the pending success.
   Persistent unreadability fails at the completion deadline with
   `resident_evidence_unreadable` and directs the operator to the session log.
5. Advisory follow-up nudges use `ResidentBackendControl.begin_followup_turn()`; drain
   correctness does not depend on the nudge succeeding. Nudges come every 270 s
   (pending #655: capped at half the deadline).

Selection is capability-driven through `connection.resident_backend`, not harness-id
branching. See [concepts/harness-abstraction.md](../concepts/harness-abstraction.md)
for the connection seam.

### Claude native holds

> **Built, not merged:** branch `fix/claude-bg-quiescence` (#655,
> [D-claude-native-hold](../decisions/completion-drain.md#d-claude-native-hold)).
> On `main` a Claude child gets the plain plan and is finalized at its first
> `result`, background work killed, status `succeeded`.

**The signal.** Claude Code (2.1.302) emits
`{"type":"system","subtype":"background_tasks_changed","tasks":[...]}` on
stream-json stdout, a **full snapshot** of Claude-owned background work, every time
it changes. No task history is needed.
- A `run_in_background` shell is a `local_bash` entry.
- A background subagent (Agent tool) is a `local_agent` entry, with
  `subagent_type`. The subagent's own background shells appear as `local_bash`
  with `parent_task_id`; their `task_started` frames carry
  `owned_by_subagent: true`. A subagent can re-activate with a new `run_id` when
  its own task notifies.
- The snapshot empties only once the whole tree is done. Claude may emit several
  `result`s while it is non-empty (one probe: 4 results, 3 of them with tasks
  pending). Raw `claude -p` exits only after `tasks: []`, and when the task
  notifies it takes a follow-up turn on its own, with no injected prompt.
- Raw `claude -p` waits **forever** on a task that never ends, such as a dev
  server. Claude itself gives no bound.
- Subagent `assistant` frames share the parent's stdout and carry
  `parent_tool_use_id`. They are not parent activity: Claude's semantics ignore
  them, so a busy subagent can't reset the parent's nudge timer or clear
  `awaiting_done`.

**The hold.** `harness/claude_drain.py` (`ClaudeCompletionEvidence`) keeps only
the latest snapshot, updates in constant time per frame, and never reads the
filesystem. The connection exposes it as `HarnessConnection.completion_evidence()`
(default `None`). `build_drain_plan()` picks the resident plan when a connection
has a `resident_backend` **or** supplies completion evidence. The assessment sets
`completion_hold` and `native_hold`:
1. A `result` while the snapshot is non-empty is held.
2. An empty snapshot alone does not finalize. The next `result` does.
3. A Claude child with no background work finalizes at its first `result`, with no
   added latency.
4. A malformed snapshot warns and holds, bounded by the deadline.

**Policy under `native_hold`.** The resident deadline runs from the first held
`result` through the follow-up; task activity never renews it.
- Rearm only extends the deadline; natural completion still finalizes `succeeded`.
- An explicit `done` authorizes success with tasks left over, and managed-scope
  teardown kills them. A `done` sent mid-turn waits for the turn's closing
  `result` up to the deadline, then publishes `succeeded` with the latest report.
- Without `done`, the deadline gives `timed_out`.
- Native nudges name the pending tasks (`description (task_type)`, plus
  `(subagent)` for entries with `parent_task_id`), say `done` stops them, and
  offer rearm only in the timeout-soon form.
- The resident profile polls the exact `done`/`rearm` signal files itself; the
  evidence adapter has no timers.

**Delivery.** Nudges are stream-json user messages on Claude's stdin
(`{"type":"user","message":{...}}`, `claude_ws.send_user_message`), sent through
the manager's serialized `inject()` with source `completion_nudge`. That is the
same path `meridian spawn inject` takes: any process reaches a live runner through
`spawns/pN/control.sock`. Cancel is SIGINT.

**What changed for Codex/OpenCode:** main's behavior and test assertions are kept,
except:
1. stale validation is cleared for each new candidate;
2. nudge urgency compares against the resolved interval, not a fixed 270 s;
3. the shared nudge interval is `min(270, deadline / 2)`, inert at the 1800 s
   default.

Their `done` still beats the deadline, during validation too, and their explicit
rearm still holds until `done`.

### Rearm budget

Each `rearm` signal grants one deadline extension. Grants are unlimited by
default (`None`). `--resident-rearm-budget` / `MERIDIAN_RESIDENT_REARM_BUDGET` /
profile `resident-rearm-budget` / `timeouts.resident_rearm_budget` opts into a
maximum grant count. The running count is spawn-scoped: initialized from the
persisted `resident_rearm_count` in `state.json`, monotonic across resident rearm
signals, and preserved across process recovery.

Exhaustion (only when a budget is configured) produces `timed_out` with
`resident_rearm_budget_exhausted`. Pre-expiry `timeout_soon` nudge behavior is
unchanged from the deadline model. Rearm count telemetry is always emitted
regardless of whether a budget is configured.

The Mars launch-bundle schema does not currently expose a per-model resident
rearm budget; `resident-rearm-budget` is a top-level profile default.

For resident drains, `raw_terminal_frames_authoritative=False`: a terminal harness
frame is a turn boundary candidate, not finalization authority by itself. The
coordinator owns terminal finalization while resident. If the resident deadline
expires, it finalizes the parent as `timed_out` with
`resident_deadline_expired`. After that outcome is published, async best-effort
teardown cancels active descendants through the normal cancel pipeline so
children converge to terminal `cancelled` instead of surviving as orphaned
backend launches.

## Timeout carrier

`execution_policy.timeout` (minutes) is the single timeout carrier for both
`--timeout` and `MERIDIAN_TIMEOUT`. Seconds conversion happens at the runner edge
only (`streaming_runner.py`). The former dual model -- `ExecutionBudget.timeout_secs`
alongside `execution_policy.timeout` -- was deleted because CLI armed one carrier
but not the other, and env armed the reverse, producing incorrect behavior in both
paths. The outer attempt timer is non-renewing and defaults to `None`; neither
resident rearms nor Pi child waves can reset it.

## Related pages

- [completion-drain-coordination.md](completion-drain-coordination.md) — shared completion mechanism, profile policy, and evidence authority.
- [D-claude-native-hold](../decisions/completion-drain.md#d-claude-native-hold) — why Claude children hold through the resident profile.
- [spawn-finalization.md](spawn-finalization.md) — terminal write authority and store-level finalization.
- [concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — resident backend capability seam.
