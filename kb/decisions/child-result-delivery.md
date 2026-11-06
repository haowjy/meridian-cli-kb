# Decisions: Child Result Delivery

See [child result delivery](../concepts/child-result-delivery.md) for the current mechanism.

## D-claude-spawn-watch: core decides what is owed; a Claude mod only senses idle and submits (2026-10) {#d-claude-spawn-watch}

A Claude primary's mod senses idle and submits a result; core decides what is owed and when it is consumed. Submission proves only queuing, so acknowledgment waits for the delivered turn. Foreground results have already reached the caller; grandchildren reach it through their parent's report. A mod is useful here because delivery is wanted at idle, unlike mid-turn messaging. A failed acknowledgment excludes the IDs until reload to avoid repeated delivery; one duplicate after restart is possible.

## D-claude-delivery-header-only: automatic delivery carries lookup headers, not reports (2026-10) {#d-claude-delivery-header-only}

Claude automatic delivery carries lookup headers, not report bodies. This bounds repeated context and avoids report I/O. CLI output that actually shows the terminal report now consumes the result; diagnostic reads do not. Pi still delivers bounded report excerpts.

> [!FLAG] **Needs human review**
> Open: should Pi deliveries also become header-only? Undecided; Pi keeps its
> excerpts while the question remains open.

## D-no-arg-wait-owed-bounded: no-arg wait returns a bounded header-only owed batch (2026-10) {#d-no-arg-wait-owed-bounded}

**Status:** current choices, `work:read-consumes`, with review corrections
from `spawn:p9156` and `spawn:p9230`. Merged to meridian-cli `main` in PR #751
(`54ca93606`).

Sharing collect's owed definition with no-arg wait was the lead's proposal; the
owner agreed with "yeah, add it into that pr" (`chat:c8820`). Header-only owed
rows, the 32-row cap, and dropping the claim subsystem are the lead's calls; the
owner did not weigh in on them.

### Problem

No-arg `spawn wait` selected only active descendants. A child that finished just
before target discovery disappeared from wait even though it remained owed; the
Claude watcher could then deliver it after the lead believed wait had drained the
chat. Sharing collect's owed definition closes that gap, but the first
implementation exposed another one: real resumed chats had as many as 79
unconsumed finished background children. Returning every full report would dump
an unbounded stale backlog into one model turn.

### Decision

- **No-arg wait adds terminal children from collect's owed scope.** A primary
  owns background direct children across every primary generation of its chat;
  a nested caller owns only its own background direct children. Active
  transitive-descendant discovery is unchanged. Owed discovery subtracts CLI
  observations (current and legacy), Pi native-delivery receipts, and Pi
  cleared IDs. Their shared reader opens each source once per call; a damaged
  source is skipped and diagnosed without discarding the others.
- **Only the already-finished part is capped.** Target resolution takes at most
  the oldest 32 owed terminal rows, the same member bound as collect. Active
  targets are never capped; excess finished rows remain owed for the next wait
  or collection.
- **Resolve owed delivery once, outside the wait loop.** `owed.py` discovers
  the caller's scope, snapshots it, keeps finished rows, subtracts the
  blocked-on IDs, then applies the cap. Thus one child cannot appear in both
  lists, and the existing polling loop and pending table still operate on
  blocked-on IDs only.
- **The result model names both classes.** Target resolution places late
  delivery in `owed_spawns` and rows active then in `spawns`; a later status
  check cannot move them. Owed rows use the shared collect header (outcome,
  worker, failure, distinct task directory, and `spawn show` lookup) and omit
  `report_body`, including under `--full`. Blocked-on rows keep ordinary wait
  output and their requested report body.
- **Owed delivery is not waited work.** Run counts, single-result compatibility
  fields, fail-fast, aggregate failure, and the CLI exit code use only
  `spawns`. A failed owed row still says `failed` in its header but does not
  make an otherwise successful no-arg wait fail.
- **Consumption stays with wait's existing observation lease.** An observing
  wait leases both classes, then its existing exit cleanup records owed rows
  and terminal blocked-on rows together. `--no-observe` uses the same scope,
  cap, and rendering but neither leases nor consumes, so its oldest batch may
  repeat.
- **At-most-once arbitration is not added.** Collect rechecks existing
  consumption and leases immediately before returning; wait does not recheck
  after acquiring its lease. If those actions cross, one duplicate header can
  appear. That short, occasional duplicate is accepted.

### Consequences

- Collect's last consumption check and wait's lease are separate steps, so a
  wait that leases a child just after collect checked it returns the same child
  collect delivers: one duplicate header. A delivery the mod already holds or
  has queued when a wait starts arrives too.
- Wait records consumption before the CLI prints, for blocked-on and owed rows
  alike. A wait killed in between loses its output, and the results stay
  consumed.

### Rejected

- **Limit owed discovery to the current primary generation.** Measured current
  generations still had up to 12 finished children, and a second owed definition
  would drift from collect across resumes.
- **Keep full reports but cap the batch.** Up to 32 reports is still a large,
  often already-read context dump.
- **Cap the combined target set.** That could strand live descendants outside
  the synchronization barrier.
- **Reclassify from terminal status while rendering.** A child that was active
  when the wait selected it would lose the full report it waited to return.
- **A renewable chat-scoped claim subsystem.** An implemented draft added a
  new `spawn-delivery/<chat>.json` authority, lifecycle commands, renewal and
  release paths, and plugin protocol solely to close the final-read/lease race.
  It was deleted before shipping: the cost was disproportionate to one short
  duplicate header, and it could not remove the broader
  plugin-held-but-unacknowledged duplicate window. Existing observation leases
  and at-least-once delivery are the settled tradeoff.
- **Post-lease consumption recheck.** It duplicated target resolution and the
  no-pending return without eliminating the plugin window, so it was deleted
  with the claim design.

---

## D-no-arg-wait-since-started: report later children without expanding the barrier (2026-10) {#d-no-arg-wait-since-started}

**Status:** open as meridian-cli #753.

### Problem

A no-arg wait freezes its blocked-on set at initial discovery. A child launched
after that point was neither waited on nor mentioned, even if it finished before
the wait returned. The fixed barrier remained correct, but its output could look
like the whole work tree had settled while newer work was still running or had
failed.

### Decision

- **Keep the barrier fixed; append an advisory trailer.** Every completed,
  fail-fast, or checkpoint no-arg return after successful initial discovery lists
  in-scope children that appeared while it ran under `Since this wait started:`.
  An initial discovery-pending checkpoint has no baseline and omits the trailer.
  Explicit-ID waits do not.
- **Rediscover once at return through the existing target resolver.** There is
  no query shape specific to the trailer and no discovery in the poll loop.
- **Running means current active scope outside the original blocked-on set.**
  Finished means owed terminal children whose ids the start-of-wait discovery
  never saw: not blocked on, and not among its finished owed ids, counting the
  rows the 32-row cap held back. That exclusion runs before the cap, so an old
  backlog cannot crowd out a new completion. Membership, not finish time: a
  first draft compared `finished_at` with the wait's start and dropped children
  that crashed with no recorded finish time .
- **The trailer is not delivery.** Its rows are not waited on, leased, consumed,
  counted, or included in fail-fast, aggregate failure, or exit-code decisions.
  `--no-observe` has the same trailer. JSON exposes `since_started_spawns`.
- **Status text must not hide failure.** Active lifecycle states render
  `running`, successful completion renders `finished`, and other terminal rows
  keep `failed`, `timed_out`, or `cancelled`.
- **Return discovery is best effort.** Any state or index failure omits the
  trailer and never fails the wait: the wait's `finally` consumes its results
  regardless, so raising there would lose them.

### Rejected

- **Add later children to the blocked-on set.** A producer could extend the
  barrier indefinitely and make its exit timing depend on work launched after
  the caller chose what to wait for.
- **Tell the caller to run wait again.** The section reports state without prescribing another blocking action.
- **A second query shape or per-poll discovery.** Either would drift from the
  primary-chat/nested-subtree scope or add repeated index cost to the hot loop.

---

## D-cancel-consumes: a cancel that reports `cancelled` consumes the child's result (2026-10-24) {#d-cancel-consumes}

A cancel reporting `cancelled` consumes that outcome for the caller, since the result was already shown. A cancel that finds success or failure does not consume the unseen report. An unrelated caller's observation cannot suppress the owner's delivery; lock contention can result in one duplicate rather than blocking cancel.

## D-shown-results-consume: a terminal result printed to the caller consumes (2026-10-24) {#d-shown-results-consume}

CLI surfaces that print a terminal report consume it for the caller. Internal reads, summaries, logs and context references do not: reading a result for another purpose must not erase a delivery the lead has not seen. Observation is best effort after output, so contention or a read-only runtime can cause a duplicate instead of losing the report.

## D-observed-spawns-location: the consumption record lives in the parent's spawn directory (2026-10) {#d-observed-spawns-location}

Consumption is keyed by parent spawn and stored with that spawn. Published-spawn locking prevents a concurrent prune from recreating a deleted directory. Readers also account for legacy observations while old processes can still write them; see [consumption sources](../concepts/child-result-delivery.md#consumption-sources).
