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

Terminal publication is one-way. The drain loop publishes the profile-owned
terminal outcome and resolves the completion future before it starts one async,
best-effort cleanup. A stuck descendant cleanup or connection stop therefore
cannot hang publication. Cleanup reports are telemetry and cannot replace the
outcome; the startup reaper reconciles incomplete cleanup after a crash.

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
   correctness does not depend on the nudge succeeding.

Selection is capability-driven through `connection.resident_backend`, not harness-id
branching. See [concepts/harness-abstraction.md](../concepts/harness-abstraction.md)
for the connection seam.

For resident drains, `raw_terminal_frames_authoritative=False`: a terminal harness
frame is a turn boundary candidate, not finalization authority by itself. The
coordinator owns terminal finalization while resident. If the resident deadline
expires, it finalizes the parent as `timed_out` with
`resident_deadline_expired`. After that outcome is published, async best-effort
teardown cancels active descendants through the normal cancel pipeline so
children converge to terminal `cancelled` instead of surviving as orphaned
backend launches.

## Related pages

- [completion-drain-coordination.md](completion-drain-coordination.md) — shared completion mechanism, profile policy, and evidence authority.
- [spawn-finalization.md](spawn-finalization.md) — terminal write authority and store-level finalization.
- [concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — resident backend capability seam.
