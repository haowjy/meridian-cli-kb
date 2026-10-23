# Decisions: Child Result Delivery

Why a Claude primary gets its forgotten background children's results from a
mod that calls Meridian, why owed discovery shares each delivery route's
consumption evidence, and how concurrent delivery is arbitrated across a chat.
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
[bounded no-arg wait decision](#d-no-arg-wait-owed-bounded) made owed discovery
also honor Pi's native admission receipts and explicit cleared IDs, and made a
chat-scoped claim the concurrency authority.

---

## D-claude-delivery-header-only: automatic delivery carries lookup headers, not reports (2026-10) {#d-claude-delivery-header-only}

**Status:** lead decision in `work:read-consumes` (Decision 3). Shipped in
meridian-cli v0.12.8 (PR #743, `53038dbba`).

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

### Consequence

A lead that directly reads a finished child can receive one later header-only
Claude delivery for it. This duplicate is accepted: it is short, preserves the
single durable consumption protocol, and points back to the explicit lookup
command when the lead needs the report.

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

**Status:** lead decisions in `work:read-consumes` (Decisions 4 and 5), with
review corrections from `spawn:p9156`; implemented on the unmerged meridian-cli
branch `fix/owed-wait` at `e9fac2a78`, not yet on `main`.

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
  transitive-descendant discovery is unchanged. Both paths subtract CLI
  observations, Pi native-delivery receipts and Pi cleared IDs, so a result
  delivered or dismissed in Pi is not replayed by the CLI.
- **Only the already-finished part is capped.** Target resolution takes at most
  the oldest 32 owed terminal rows, the same member bound as collect. Active
  targets are never capped; excess finished rows remain owed for the next wait
  or collection.
- **Classification belongs to the result model.** Target resolution places
  already-finished rows in `owed_spawns` and rows active then in `spawns`; a
  later status check cannot change classes. Owed rows use the shared collect
  header (outcome, worker, failure, distinct task directory, and `spawn show`
  lookup). Blocked-on rows keep ordinary wait output and their requested report
  body after wait blocks for them.
- **Owed delivery is not waited work.** Run counts, single-result compatibility
  fields, fail-fast, aggregate failure and the CLI exit code use only
  `spawns`. A failed owed row still says `failed` in its header but does not make
  an otherwise successful no-arg wait fail.
- **JSON names both classes.** `spawns` carries blocked-on results and
  `owed_spawns` carries late delivery. Owed entries omit `report_body`, including
  under `--full`; consumers need not infer the class from a null field.
- **Consumption stays with wait.** An observing wait leases the combined target
  set and consumes terminal rows on return. `--no-observe` uses the same scope,
  cap, and rendering but neither leases nor consumes, so its oldest batch may
  repeat.
- **One renewable chat claim arbitrates owed headers.** Wait and collect reserve
  under `spawn-delivery/<chat>.json`, so collectors and resumed primary
  generations cannot both select the same result. Collect's IDs are exclusive;
  wait makes only owed-at-resolution IDs exclusive so active barriers may
  overlap. After the initial
  compatibility lease write they re-read durable consumption and other leases;
  durable observation is written before a claim is removed. Claude renews its
  claim while it owns unadmitted text, releases it when submission does not
  happen, and lets it expire after a crash. Per-primary wait reservations in
  `observed-spawns.json` remain a projection for Pi and older builds, not the
  arbitration authority.

### Rejected

- **Limit owed discovery to the current primary generation.** Measured current
  generations still had up to 12 finished children, and a second owed definition
  would drift from collect across resumes.
- **Keep full reports but cap the batch.** Up to 32 reports is still a large,
  often already-read context dump.
- **Cap the combined target set.** That could strand live descendants outside
  the synchronization barrier.
- **Reclassify from terminal status while rendering.** A child that was active
  when the wait claimed it would lose the full report it waited to return.
- **Use per-primary observation leases as the only claim.** A resumed chat has
  several primary directories; separate locks cannot prevent two generations
  from selecting the same owed header.

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
