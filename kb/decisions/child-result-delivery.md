# Decisions: Child Result Delivery

Why a Claude primary gets its forgotten background children's results from a
mod that calls Meridian, why owed discovery shares the durable consumption
evidence each delivery route already owns, and why no separate delivery-claim
subsystem ships.
Current mechanism: [Child result delivery](../concepts/child-result-delivery.md).

---

## D-claude-spawn-watch: core decides what is owed; a Claude mod only senses idle and submits (2026-10) {#d-claude-spawn-watch}

**Status:** lead decision after design review (`spawn:p8815`) and structure
review (`spawn:p8822`), `work:claude-spawn-watch`. Shipped in meridian-cli
v0.11.9 (PR #724). Follow-ups #717–#722; #719 is
[D-observed-spawns-location](#d-observed-spawns-location).

### Problem

A Claude primary that ran `meridian spawn --bg` and ended its turn without
`meridian spawn wait` stalled: nothing started its next turn when the child
finished. The spawn guidance already said to wait, and models still skipped it.
Pi primaries did not stall, because Pi's `meridian-spawn-watch` extension
delivers child results as a new turn. The goal was the same guarantee for
Claude from the runtime rather than the prompt: an idle lead that is owed
results receives them as a turn, as if it had waited.

### Decision

- **Split.** Python owns every decision: which children are owed, when a batch
  is ready, the text the lead reads, retries, and consumption. The mod senses
  idle, runs hidden `meridian spawn collect`, submits its text with
  `$.prompt.submit`, and runs hidden `meridian spawn ack` when the delivered
  turn starts. The mod reads no Meridian files.
- **Two hidden commands, not a `spawn wait` flag.** Owed discovery, the return
  rule, lease handling, consumption, rendering and exit triggers all differ
  from `spawn wait`. As `--owed` on `spawn wait` they would have added about
  eight mode branches to a 300-line function and given one command two output
  schemas. Hidden commands are a protocol with the bundled mod, outside the
  user-facing deprecation surface.
- **The ack is its own command, not `spawn wait <ids>`.** Waiting on an
  already-terminal id does record it, but exits 1 when that child failed,
  re-resolves references, and writes under the caller's spawn id like any wait.
- **One consumption record, read across the chat.** `collect` and `ack` use
  `observed-spawns.json`, the file `spawn wait` already writes, so explicit
  waits and automatic delivery never double-deliver. Writes stay under the
  caller's primary; `collect` reads every primary of the chat, because a
  resume creates a new primary and an observation from before it lives in the
  old one's file. No new authoritative state and no state-format change.
- **Background direct children only.** A foreground spawn's result already went
  to its command; grandchildren arrive through their parent's report.
- **Return rule.** If an owed child had already finished at the first look,
  deliver every finished one at once; otherwise wait for the whole owed set.
  Under a strict "all finished" rule, a lead the user talks to every 20 minutes
  would never get child A while child B ran for hours, because each turn
  restarts the collector.
- **No checkpoint turns.** `spawn wait` yields so a blocking Bash call hands the
  turn back; an idle lead has no turn to hand back, and wait's checkpoint text
  tells the lead to block in the foreground, which the watcher exists to make
  unnecessary. Cost: the prompt cache can go cold during a wait longer than an
  hour, as it does when the user steps away.
- **Consume only after the conversation has the text.** Claim in memory, submit
  with a nonce, ack on the main-loop `turn.start` carrying that nonce.
  `$.prompt.submit` resolving proves only that the prompt was queued
  ([host behaviour](../research/claude-code-mods-and-inbox.md)).
- **A failed ack keeps its ids excluded until the mod reloads.** Releasing them
  would redeliver the same results at every idle while the ack keeps failing.
  The cost is one duplicate after a restart.
- **`session.start` starts an idle run; `/clear` does not.** A resumed session
  is idle with nothing to end, so it would otherwise wait for the user's first
  turn (found by the runtime probe, `spawn:p8840`). After `/clear`, results stay
  owed until the next normal idle rather than landing in a context the user just
  cleared.
- **Its own plugin and config key.** `harness.claude.spawn_watch.enabled`,
  default true, mirroring `harness.pi.spawn_watch.enabled`. Idle notifications
  are off unless configured; spawn delivery defaults on, so they do not share a
  gate.

### Rejected

- **A launcher idle sidecar.** It does not run for Claude primaries (PTY path,
  no asyncio loop, no idle sensor), and could deliver only through Claude's
  inbox socket, whose mid-turn behaviour is unspecified for this use and which
  needs Claude Code 2.1.224 or later outside bare mode.
- **The mod reads `state.json` itself, as Pi's extension does.** The mod API
  has no file watch, so it would poll from TypeScript and re-implement the
  spawn schema, lock-derived crash status and wait leases in a second language.
- **Extending `meridian-idle`.** Different concern, different default.
- **Shared TypeScript between the two Claude mods.** Claude Code refuses an
  import from outside the plugin's folder and does not follow a symlink to a
  sibling, so the gate, CLI queue and store log are duplicated.

[D-claude-primary-inbox](primary-inject.md#d-claude-primary-inbox) rejected a
`$.prompt.submit` mod for `spawn inject`: it only queues until the turn ends,
fails silently when the mod doesn't load, and adds a process. For delivery the
first is the point (results are wanted only at idle). A mod that fails to load
leaves the lead where it was without one, needing an explicit wait.

### Revisit when

- Claude Code gives a mod a mid-turn insertion point that the model sees
  (#722).
- The per-idle CLI start shows up as a cost (#721); a hint that skips `collect`
  when nothing changed is the planned fix, never a correctness input.
- Pi adopts `collect` and the background-only filter (#718).

The one-record statement describes the original Claude/CLI design. The later
[bounded no-arg wait decision](#d-no-arg-wait-owed-bounded) also made owed
discovery honor Pi's native admission receipts and explicit cleared IDs.
Existing observation leases remain the only temporary coordination state.

---

## D-claude-delivery-header-only: automatic delivery carries lookup headers, not reports (2026-10) {#d-claude-delivery-header-only}

**Status:** current choice, set by the owner in `chat:c8820`
(`work:read-consumes`). Shipped in meridian-cli v0.12.8 (PR #743, `53038dbba`).

The owner's words: header-only came from "also I don't think the plugin should
show the full report... it should just return [the pasted delivery header] the
top part"; the lead chose the exact lines kept. Keeping reads pure came from
"tbh... shouldn't spawn wait just be enough...?", which reversed the owner's
earlier "i think show should be an ack tho?" and "yeah, i think session log
should count as well". The framing of the repeat as an accepted one-ping cost
is the lead's.

### Problem

Claude's collector originally rendered report bodies into the automatic turn,
sharing a 64 KiB budget across at most 32 children. Reading a report directly
did not consume it, so a later automatic delivery could send the same body
again. Making `spawn show`, `spawn report show`, and `session log` consume what
they displayed would avoid that repeat, but would also turn read operations
used by the CLI, server, UI, and MCP into observation writers. The rule would
have needed caller-context, terminal-state, and partial-transcript-window
exceptions, and each write would take the parent's published-spawn lock.

Once the automatic delivery contains only enough information to identify and
inspect a child, a repeat after a direct read is one short lookup notice rather
than a repeated report. That makes keeping reads pure the simpler and safer
tradeoff.

### Decision

- **Claude delivery is header-only.** Each child keeps its terminal
  status/outcome line, `Worker:`, `Failure:` when present, the distinct task
  directory line, and the exact `meridian spawn show <id>` lookup command.
  Report bodies, the no-durable-report fallback, and truncation notices are not
  delivered.
- **`spawn collect` does not read report files.** The report-byte budget and
  cross-member budget-sharing machinery are deleted. The existing 32-child
  cap remains.
- **Reads remain reads.** `spawn show`, `spawn report show`, and `session log`
  do not record consumption and gain no observation flag. Consumption remains
  with terminal results returned by `spawn wait` unless `--no-observe` is used,
  and with `spawn ack` after the Claude delivery turn starts.
- **Pi is unchanged.** Its TypeScript renderer continues to deliver bounded
  report excerpts under Pi's delivery contract.

> [!FLAG] **Needs human review**
> Open: should Pi deliveries also become header-only? Undecided; Pi keeps its
> excerpts until the owner decides. The lead asked twice in `chat:c8820` without
> an answer.

### Consequence

A lead that directly reads a finished child can receive one later header-only
Claude delivery for it. This duplicate is accepted: it is short, preserves
pure reads and the existing durable consumption protocol, and points back to
the explicit lookup command when the lead needs the report.

### Rejected and superseded

- **Record consumption from report and transcript reads.** An earlier
  work-item decision proposed opt-in op parameters and default-on CLI
  `--observe` flags, with terminal-child and transcript-final-entry checks. It
  was superseded before implementation because it made unrelated read surfaces
  share delivery policy and lock-taking write failure modes to avoid only a
  short duplicate header.
- **Keep bounded report excerpts in Claude delivery.** That preserves the
  costly duplicate that motivated read-consumption and keeps report I/O plus
  budget-sharing machinery on every automatic delivery.

---

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

**Status:** owner decision in `work:wait-since-started` (2026-10-24);
open as meridian-cli #753.

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
  that crashed with no recorded finish time (review p9267).
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
- **Tell the caller to run wait again.** The owner explicitly chose no hint
  line; the section reports state without prescribing another blocking action.
- **A second query shape or per-poll discovery.** Either would drift from the
  primary-chat/nested-subtree scope or add repeated index cost to the hot loop.

---

## D-observed-spawns-location: the consumption record lives in the parent's spawn directory (2026-10) {#d-observed-spawns-location}

**Status:** lead decision after correctness and version-skew review
(`spawn:p8936`) and containment review (`spawn:p8939`),
`work:observed-spawns-move`. Shipped in v0.11.10 (PR #729). Fixes #719.

### Problem

[D-claude-spawn-watch](#d-claude-spawn-watch) made `observed-spawns.json` the
consumption record for every harness, but the file stayed at
`<runtime>/pi-bash/<parent>/`, a directory owned by Pi's background-bash
extension. The path claimed a Pi ownership the file no longer had, and any
later implementation reading the state would inherit that misplacement. Moving it is a state-compatibility change: new
builds must read state older builds wrote, and an older `spawn wait` or Pi
session can keep running across an upgrade.

### Decision

- **Keyed by parent spawn, in that spawn's directory:**
  `spawns/<parent>/observed-spawns.json`.
- **Written under the parent's published-spawn lock, which never recreates the
  directory.** Pruning and archiving delete spawn directories; a plain atomic
  write would bring them back. When the parent's `state.json` is missing or
  quarantined (unreadable and set aside by the spawn repository), nothing is
  recorded: `spawn ack` reports an error, and
  `spawn wait` proceeds without a record. No reader selects children of such a
  parent, so nothing is redelivered.
- **One-release transition: read both, mirror for Pi.** Every reader (Python
  ops, the Pi disk watcher, the Pi TypeScript extension) unions the old and new
  files, because an older `spawn wait` may still write the old one. Writers
  also update the old file wherever `pi-bash/<parent>/` already exists, because
  a Pi session started before the upgrade reads only the old file. Claude
  parents never get that directory. Every transitional line is tagged
  `legacy-observed-spawns (#719)`, to be deleted one minor release after the
  move ships, keeping the `pi-bash` watch for Pi's other files.

### Rejected

- **One file per chat.** Spawned Pi sessions key consumption by their own spawn
  id; a per-chat file would change how they decide they are finished.
- **A separate lock file beside the new file.** It would guard nothing the
  published-spawn lock doesn't already.
- **Migrating and deleting the old file.** A still-running older waiter would
  recreate it.
- **Accepting a one-time redelivery for running Pi sessions** instead of the
  mirror write.

### Cost

Rolling back to an older build can deliver a Claude primary's results once
more: the older build reads only the old file, and Claude primaries are not
mirrored.

### Revisit when

- The compatibility window closes: delete the tagged transition code.
