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

**Publication policy:** Through 0.9.3 publication was idle-only. The watcher checks Pi's native idle state immediately
before queueing and rechecks explicit waits after asynchronous formatting. The
formatter uses a non-consuming spawn wait. This yields no follow-up when the
terminal Bash wait already returned, while leaving an unattended result eligible
for its normal idle delivery. Wait reservations suppress only while the caller's
PID, process-birth identity, and expiry lease remain valid.

**Tool-boundary delivery (PR #560, shipped in v0.9.4).**
Idle-only publication left a parent that kept calling tools unaware of finished
children until it stopped. The decision adds native steer at an awaited
successful tool-bearing `turn_end`, keeps the idle follow-up for genuinely late
work, and leaves receipts, public-event fence and on-disk state unchanged. A
consuming wait finishes before the boundary, so it still suppresses the notice.
Rejected: steering as soon as a poll notices completion (races an explicit wait)
and deferring all late results to the next turn (changes policy and can strand
spawned RPC work).

The first candidate (never released) bolted a boundary drain with formatter cancellation, preemption
and a 30-second budget onto always-on 500 ms scanning and the general `spawn wait`
CLI. Measurement on a frozen 2,360-directory corpus showed that machinery was the
cost (about 65 % of a core while busy with no owned children), so the user approved
replacing it before release. The calibration: keep native boundary steer and exact
receipt semantics; delete the cancellation layer rather than tune it. Do not
describe the budget or preemption as shipped behavior.

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

**Bounded aggregate delivery (shipped with the same change).** The first
replacement capped each report at 64 KiB but opened every owed report at once and
joined them into one message; a 512-child probe at a 256 file-descriptor limit then
failed with `EMFILE` and delivered nothing, so a per-item cap had become an
unbounded aggregate. The review blocker was fixed in the PR, not deferred: one
automatic message takes at most 32 members and 64 KiB of UTF-8 content, reports are
read sequentially, and the rest of the backlog stays owed. A greedy first version
let one full report consume the packet, which turned a 512-report backlog into 512
single-member component packets (not 512 measured native inferences). Review ruled
that avoidable turn amplification a PR-local blocker, so the final source reserves
every selected member's worst-case mandatory frame first and shares the remaining
optional-text bytes fairly (unused shares carry forward); the same fixture is now 16
packets of 32 with one open report handle. Queueing, then exact
admission, closes the current idle opportunity: there is no new timer, controller or
drain-until-empty loop, so later tool boundaries or genuine idle transitions carry
the remainder. An unbounded backlog still needs several genuine opportunities; there is no constant-total-inference promise. Every
included member keeps a visible ID, outcome and a concrete full-result lookup; only
optional detail is truncated, and a member whose mandatory frame does not fit the
remaining packet is deferred rather than shown clipped. A receipt for a member whose
optional text was truncated records admission of the notice, not that the full
report was read; the visible lookup is what makes the result recoverable. Terminal `error` is a typed
canonical field. Canonical limits live in the source delivery contract.

Open limits (pre-existing, not reproduced defects): a queued message Pi never
admits keeps its IDs reserved and can also pause unrelated automatic publication
until a real native lifecycle callback, reload or restart (timer wakeups alone do
not reopen it; no timeout-based repair is designed), and full discovery at each
boundary still scans all history, a known performance opportunity. See
[future work](../open-questions/future-work.md#pi-result-delivery-limits). Broader
speed, storage and resource follow-up from the PR's efficiency audit is tracked in
[issue #561](https://github.com/haowjy/meridian-cli/issues/561), not here.

### Unreleased managed-Bash publication candidate (#561 / #562 / PR #563)

The source candidate is draft PR #563 (reconciled against source head `accb1ce5`).
It is **not** v0.9.4, installed, merged, or deployed, and this page does not record
a final qualification result for those bytes. It narrows managed-Bash records
writes without changing the shipped delivery authority:

- Output appends update the log authority and in-memory progress; `log_bytes` in
  `bash-records.json` is only the size captured by a lifecycle write. What was
  eliminated is the per-chunk records-file rewrite: each chunk still appends to the
  combined and stream logs and the combined log is still statted for its size, so
  no claim is made that other I/O or stat calls disappeared. There is no
  progress timer or periodic flush, and arbitrary log/list reads do not become
  implicit result consumption. Explicit terminal wait/output consumption rules
  remain.
- **Publication owns ordering.** One serialized publication stages, writes and
  adopts a single immutable value; live handles, progress and first-fault evidence
  are owned separately by the running task and overlaid for readers. The earlier
  shape (a mutable runtime record, a serialization blacklist, and a metadata patch
  plus an arbitrary commit closure that had to agree) is gone. Rejected: an
  immutable whole-map swap alone, because it could overwrite a task or fault that
  arrived during I/O.
- **Task-fatal failure is claimed inside the failing slot.** The rule used to
  live at each caller's direct `await`, and a ping-claim failure handled one
  promise hop late let a queued terminal write stage clean success first. The
  publication boundary now runs the task-fatal policy synchronously before any
  successor stages. Caller-recoverable operations (detach, consumption, clear)
  keep their own error path.
- **Claiming is void.** It once returned the repair promise, so an output-queue
  rejection inherited the metadata wait and a foreground abort could stay blocked
  behind an unrelated held records write after the shell was already stopped.
  Physical stop and output drain, durable publication, and native admission stay
  three separate milestones: foreground interruption waits for the first, wait
  and kill results for the second, and nothing here implies the third.
- **Launch separates ownership from creation.** `OwnedLaunch` returns the owned
  process and, separately, the supervised observation that its creation row
  landed. Both foreground APIs attach cancellation as soon as the process is owned,
  so abort and output drain never await creation metadata (the older candidate
  covered terminal and repair publication but not this initial barrier: a held
  creation write kept an aborted foreground call pending while output kept
  growing). Cancellation noticed after preparation I/O but before spawn removes
  the empty logs and launches nothing. Positive results still follow publication:
  a start awaits creation, and a foreground hotkey/timeout handoff (named
  `launch` versus `handoff` phases) succeeds only if its slot finds a published,
  still-running row; a creation failure or terminal row ahead of it declines the
  handoff rather than resurrecting or backgrounding it. Creation failure keeps its
  queue-owned fatal supervision, with a rejection observed even when the foreground
  call has already returned.
- **Terminal publication** still happens after physical group exit and queued
  output. Until the terminal write lands, observers see `running` and no result
  can be consumed. Group-signalling authority retires when the live task observes
  its group gone, including an ESRCH signal, independent of delayed or failed
  persistence and stale in-flight `ps` results.
- **Result outcome contract.** Retrieval returns a typed outcome plus explicit log
  availability (`available`, `missing`, `unreadable`) instead of empty text; one
  shared outcome projection feeds tool results and automatic notices, so a
  nonzero exit or execution fault is never inferred from log text, and an
  acknowledgement is persisted only for a result that was actually returned. Wait
  cancellation is checked at the acknowledgement stage. A uniform visible outcome
  header, including for empty successful output, is intentional: this work targets
  metadata amplification, not header tokens.
- **Notices keep error presence mandatory.** The terminal-outcome prefix of an
  automatic spawn or Bash notice states that the result had an error; only the
  diagnostic text is optional detail. If even the complete mandatory frame cannot
  fit the packet, the item stays owed rather than being admitted looking clean.
  The 32-member and 64 KiB bounds and receipt/admission semantics are unchanged.
- UI inspection (`/ps` list, previews, `/ps:logs`, `p-*` inspection) stays
  non-consuming; the list loads metadata only and one selected preview.

Cold recovery of a durable `running` row adds `ownership_lost` to any existing
`execution_error` once; it never replaces earlier fault evidence and never
signals a stored PID. Concurrent retries of a failed recovery share one in-flight
attempt. The candidate sits on the v0.9.5 base with no schema change, so v0.9.5
stop/backup/no-old-reader guidance still applies when upgrading from 0.9.4; the
installed CLI is unchanged.

The terminal transition linearizes when it stages. A failure that has claimed the
record first prevents a clean `exited` snapshot; a failure ordered after staging
is repaired by its own write. A crash or failed repair in the rename window can
still leave a recoverable success, so this is not a crash-atomic exactly-once
error boundary. These limits are intentional and are not a reason to add a timer,
periodic flush, or arbitrary-read consumption. Polling cadence, CLI startup cost,
and log-follow cadence are separate work and are unchanged by this candidate.
Deferred follow-ups (optional foreground attachment ownership, broader post-shell
polling qualification) are in [future work](../open-questions/future-work.md#pi-result-delivery-limits).

Completion keeps that causal delivery obligation separate from running execution:
an explicit done may still override known running work, but it cannot bypass a
terminal result awaiting publication, admission, public-event observation, or the
parent's reply. Pi suppresses done nudges for delivery blockers and requires the
parent to be idle before resolving done.
