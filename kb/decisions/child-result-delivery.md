# Decisions: Child Result Delivery

Why a Claude primary gets its forgotten background children's results from a
mod that calls Meridian, and why consumption is shared with `spawn wait`.
Current mechanism: [Child result delivery](../concepts/child-result-delivery.md).

---

## D-claude-spawn-watch: core decides what is owed; a Claude mod only senses idle and submits (2026-10) {#d-claude-spawn-watch}

**Status:** lead decision after design review (`spawn:p8815`) and structure
review (`spawn:p8822`), `work:claude-spawn-watch`. Final review approved; on the
unmerged meridian-cli branch `feat/claude-spawn-watch` as of 2026-10-21.
Follow-ups #717–#722.

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
