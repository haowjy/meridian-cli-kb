# Decisions: Child Result Delivery

See [child result delivery](../concepts/child-result-delivery.md) for the current mechanism.

## D-claude-spawn-watch: core decides what is owed; a Claude mod only senses idle and submits (2026-10) {#d-claude-spawn-watch}

A Claude primary's mod senses idle and submits a result; core decides what is owed and when it is consumed. Submission proves only queuing, so acknowledgment waits for the delivered turn. Foreground results have already reached the caller; grandchildren reach it through their parent's report. A mod is useful here because delivery is wanted at idle, unlike mid-turn messaging. A failed acknowledgment excludes the IDs until reload to avoid repeated delivery; one duplicate after restart is possible.

## D-claude-delivery-header-only: automatic delivery carries lookup headers, not reports (2026-10) {#d-claude-delivery-header-only}

Claude automatic delivery carries lookup headers, not report bodies. This bounds repeated context and avoids report I/O. CLI output that actually shows the terminal report now consumes the result; diagnostic reads do not. Pi still delivers bounded report excerpts.

> [!FLAG] **Needs human review**
> Open: should Pi deliveries also become header-only? Undecided; Pi keeps its
> excerpts while the question remains open.

## No-argument wait includes owed results {#d-no-arg-wait-owed-bounded}

No-argument `spawn wait` adds up to the oldest 32 terminal children from
collect's owed scope, so a child finishing just before discovery does not vanish
from the result. A primary owns its chat's background direct children across
primary generations; a nested caller owns its own background direct children.
The active descendant barrier is uncapped. Owed discovery subtracts CLI
observations, Pi delivery receipts and clears; damaged sources are diagnosed
without discarding the others.

Owed rows are resolved once outside the poll loop and rendered as headers under
`owed_spawns`, even with `--full`. They do not affect wait counts, fail-fast,
aggregate failure or exit code. Active rows retain normal report rendering.
Observing waits lease and consume both classes; `--no-observe` does neither.
Wait consumes before printing, so a kill between those actions can lose output.

Collect's last consumption check and wait's lease are separate, so a race can
produce a duplicate header. A dedicated renewable claim system was rejected as
disproportionate to that window; a queued mod delivery can also duplicate.
Capping the combined set would leave live descendants outside the barrier, and
full reports for 32 owed children could flood the context.

## Children discovered during a wait {#d-no-arg-wait-since-started}

A no-argument wait keeps its initial barrier fixed and reports later in-scope
children in a `Since this wait started` trailer, also exposed as
`since_started_spawns`. Discovery is best effort at return; failures omit the
trailer without failing the wait. The trailer is advisory: it does not lease,
consume, wait on, or count those children, and does not affect fail-fast or the
exit code. Explicit-ID waits have no trailer. Children already discovered but
held back by the owed cap are excluded before applying the trailer's cap.

Adding newly launched children to the barrier could let a producer extend it
indefinitely. A return-time discovery avoids a separate query and repeated
poll-loop scans.

---

## D-cancel-consumes: a cancel that reports `cancelled` consumes the child's result (2026-10-24) {#d-cancel-consumes}

A cancel reporting `cancelled` consumes that outcome for the caller, since the result was already shown. A cancel that finds success or failure does not consume the unseen report. An unrelated caller's observation cannot suppress the owner's delivery; lock contention can result in one duplicate rather than blocking cancel.

## D-shown-results-consume: a terminal result printed to the caller consumes (2026-10-24) {#d-shown-results-consume}

CLI surfaces that print a terminal report consume it for the caller. Internal reads, summaries, logs and context references do not: reading a result for another purpose must not erase a delivery the lead has not seen. Observation is best effort after output, so contention or a read-only runtime can cause a duplicate instead of losing the report.

## D-observed-spawns-location: the consumption record lives in the parent's spawn directory (2026-10) {#d-observed-spawns-location}

Consumption is keyed by parent spawn and stored with that spawn. Published-spawn locking prevents a concurrent prune from recreating a deleted directory. Readers also account for legacy observations while old processes can still write them; see [consumption sources](../concepts/child-result-delivery.md#consumption-sources).
