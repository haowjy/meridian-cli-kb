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

**Publication policy (installed 0.9.3, idle-only):** The watcher checks Pi's native idle state immediately
before queueing and rechecks explicit waits after asynchronous formatting. The
formatter uses a non-consuming spawn wait. This yields no follow-up when the
terminal Bash wait already returned, while leaving an unattended result eligible
for its normal idle delivery. Wait reservations suppress only while the caller's
PID, process-birth identity, and expiry lease remain valid.

**Pending: tool-boundary delivery (PR #560, not merged or released).**
Idle-only publication leaves a parent that keeps calling tools unaware of finished
children until it stops. The settled direction adds native steer at an awaited
successful tool-bearing `turn_end`, keeps the idle follow-up for genuinely late
work, and leaves receipts, public-event fence and on-disk state unchanged. A
consuming wait finishes before the boundary, so it still suppresses the notice.
Rejected: steering as soon as a poll notices completion (races an explicit wait)
and deferring all late results to the next turn (changes policy and can strand
spawned RPC work).

The first candidate bolted a boundary drain with formatter cancellation, preemption
and a 30-second budget onto always-on 500 ms scanning and the general `spawn wait`
CLI. Measurement on a frozen 2,360-directory corpus showed that machinery was the
cost (about 65 % of a core while busy with no owned children), so the user approved
replacing it in the same PR. The calibration: keep native boundary steer and exact
receipt semantics; delete the cancellation layer rather than tune it. Do not
describe the budget or preemption as current behavior.

Settled shape: one finite serialized collector; no timer scans while Pi is active;
idle generation invalidated on agent start, rebind and stop, with final
idle→snapshot→idle; direct bounded report reads; candidate IDs are hints narrowed
to canonical own rows. **Hints never prove absence:** a bare Bash completion first
triggers fresh canonical discovery, because a missed root event could hide a child
that owns the launcher's obligation. Failed or superseded full discoveries stay
dirty and only the newest scan commits; the negative decision is bound to the
specific discovery's still-current revision, while positive revalidation of an
already selected child rereads its own canonical ID so unrelated root churn cannot
delay it. Every `session_shutdown` stops the watcher before Pi invalidates its
context; reload keeps the owner and queued claims and restarts at fresh
`session_start`. The accepted tradeoff is about 30.5 s
(plus collector and filesystem time, not a hard deadline) of idle latency for a
child row whose publication was missed; that is not a bug.

**Bounded aggregate delivery (implemented in the pending PR source).** The first
replacement capped each report at 64 KiB but opened every owed report at once and
joined them into one message; a 512-child probe at a 256 file-descriptor limit then
failed with `EMFILE` and delivered nothing, so a per-item cap had become an
unbounded aggregate. The review blocker was fixed in the PR, not deferred: one
automatic message takes at most 32 members and 64 KiB of UTF-8 content, reports are
read sequentially, and the rest of the backlog stays owed. Queueing, then exact
admission, closes the current idle opportunity: there is no new timer, controller or
drain-until-empty loop, so later tool boundaries or genuine idle transitions carry
the remainder. This is not a one-inference-for-unbounded-backlog promise. Every
included member keeps a visible ID, outcome and a concrete full-result lookup; only
optional detail is truncated, and a member whose mandatory frame does not fit the
remaining packet is deferred rather than shown clipped. Terminal `error` is a typed
canonical field. Canonical limits live in the source delivery contract.

Open limits (pre-existing, not reproduced defects): a queued message Pi never
admits keeps its IDs reserved until admission or restart, and full discovery at each
boundary still scans all history, a known performance opportunity. See
[future work](../open-questions/future-work.md#pi-result-delivery-limits). Broader
speed, storage and resource follow-up from the PR's efficiency audit is tracked in
[issue #561](https://github.com/haowjy/meridian-cli/issues/561), not here. Fold this
section into the decision above once the PR merges and releases.

Completion keeps that causal delivery obligation separate from running execution:
an explicit done may still override known running work, but it cannot bypass a
terminal result awaiting publication, admission, public-event observation, or the
parent's reply. Pi suppresses done nudges for delivery blockers and requires the
parent to be idle before resolving done.
