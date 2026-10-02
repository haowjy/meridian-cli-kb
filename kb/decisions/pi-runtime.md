# Decisions: Pi Runtime Ownership and Delivery

This page records why Pi task ownership and completion delivery rely on separate,
typed evidence. Current data flow and file contracts live in
[Pi Runtime Coordination](../architecture/pi-runtime/coordination.md).

## D-Pi-live-execution-owner — Keep process authority with the live owner

**Decision (2026-10-02):** A launch-scoped managed-Bash owner retains its process
handles through extension reload and rebinds current hooks. A cold process may
retain the last durable record, but it must treat formerly running work as
`ownership_lost` and must never signal a PID read from that record. Detach only
releases quiescence tracking; normal shutdown terminates process groups the
current runtime still owns.

**Why:** A PID and a historical record cannot prove that the same process still
owns the shell after a process restart. Reattaching could signal a reused PID or
claim completion without observing the command. Keeping ownership in one live
actor makes reload recovery useful while making cold recovery honest. Where the
actor is gone, manual inspection or explicit detach is safer than fabricated
ownership.

**Advisory failure boundary:** A ping callback is not task execution. If it
fails, release the ping claim and keep the shell running. Task, log, or durable
record publication failure can still fail the task because those are required
for trustworthy execution state.

## D-Pi-result-delivery-evidence — Prove admission and public observation separately

**Decision (2026-10-02):** Keep terminal Bash and canonical direct-child results
owed until an explicit terminal wait consumes the Bash result or an exact
unattended message is admitted and its matching public event is observed. The
native message receipt records exact `delivery_id` and `work_ids`; Python records
the corresponding public event separately. Only canonical `parent_id` rows
establish child membership. `originating_bash_id` transfers a launcher
obligation to its child but cannot invent membership.

**Why:** `sendMessage()` reports queueing, not native admission. A timestamp or a
remembered scan cannot identify which work was admitted, and a native receipt
alone cannot prove Python observed the public event used for completion. Exact
membership and an exact event fence let explicit waits, user-visible follow-ups,
history pruning, and quiescence agree about the same work.

**Failure contract:** Cross-store admission and observation cannot commit
atomically through a Pi process crash. A receipt without the matching observed
event is unknown and fails closed at a fixed deadline. Do not convert this
window into a false success or an atomic exactly-once claim. The retired
`last-notification.json` timestamp is ignored because it proves neither
membership nor admission.

**Publication policy:** The watcher checks Pi's native idle state immediately
before queueing and rechecks explicit waits after asynchronous formatting. The
formatter uses a non-consuming spawn wait. This yields no follow-up when the
terminal Bash wait already returned, while leaving an unattended result eligible
for its normal idle delivery. Wait reservations suppress only while the caller's
PID, process-birth identity, and expiry lease remain valid.

Completion keeps that causal delivery obligation separate from running execution:
an explicit done may still override known running work, but it cannot bypass a
terminal result awaiting publication, admission, public-event observation, or the
parent's reply. Pi suppresses done nudges for delivery blockers and requires the
parent to be idle before resolving done.
