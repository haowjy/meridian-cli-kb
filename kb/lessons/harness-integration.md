# Harness Integration Lessons

Native file-injection channels can give the same content different meanings across harnesses. Classify content before projecting it to harness channels; see [composition](../concepts/composition-pipeline.md).

Python signal handlers run on the main thread and may interrupt code that already holds a lock. A signal receiver must avoid blocking on a non-reentrant lock held by that thread; see `src/meridian/lib/launch/AGENTS.md`.

When a correctness-bearing internal parameter is added, an optional do-nothing default can hide missed callers. Make callers explicitly supply the value or explicitly choose absence. Function-local imports can hide dependency cycles from import-time checks; enforce boundaries through all import sites.

Codex fork snapshots must copy complete validated records from a bounded view of a live rollout. File publication and database registration cannot be one transaction: if registration's outcome is unknown, preserve the file rather than deleting a possibly committed fork. See [native session decisions](../architecture/native-session-binding.md).

## Bundled Mods Outlive the CLI They Call

A running harness retains the mod loaded at session start even after the CLI on PATH is upgraded. Changing a CLI flag or reply shape consumed by that mod can silently break existing sessions until they restart. See [idle TTL decisions](../architecture/idle-notifications.md#cache-ttl).

## A Dialect Switch Must Move Every Consumer of the Stream

When a harness changes event dialect, every consumer must follow the resolved transport, not just the main connection. Probe the version users actually run; a sensor that never fires needs positive evidence of activity, not merely absence of errors. See [OpenCode idle decisions](../architecture/idle-notifications.md#opencode-2x-sensor).

A scratch harness config hides user hooks. If a mod observes inputs another hook can rewrite, probe with representative hooks as well as a clean config. Before removing a workaround for an upstream bug, reproduce the original failure on the current binary: a closed issue does not establish it was fixed.

## In-Harness Code Calls Whichever `meridian` Is on PATH

**The mechanism:** plugin hooks, Pi extensions, Mars hooks and the model's shell all run
`meridian` by name. With an older build first on PATH, a Pi session launched by 0.19.5
ran `meridian spawn show --no-observe` on 0.14.0 and failed with "Unknown option" (#764).

**Reproducing it:** launch with the build's binary, not `uv run`. `uv run` puts the
worktree's `.venv/bin` first on PATH, so the skew vanishes and the bug looks fixed.

**A probe's source decides what it shows.** The same probe reported that interactive
Claude drops a prepended PATH, from sourcing an old `~/.claude/shell-snapshots` file.
A live check showed Claude 2.1.319 keeps it. Check a harness behaviour in a live
session, not from a file the harness once wrote.

The fix under review is in
[Escaped process inputs](../decisions/sandbox-platform-access.md#escaped-process-inputs).

## A Host Accepting a Message Is Not the Model Receiving It

**The mechanism:** a harness API that takes a message for the session can
return before the model has seen it. Claude Code's mod `$.prompt.submit` resolves when
the turn starts, but also when the prompt is merely queued behind a running
turn, and a queued prompt is lost if the user clears the queue (Esc) or Claude
dies. Pi's `sendMessage()` return likewise says nothing about whether Pi admitted
the message into the session.

**Where it bit:** both spawn watchers have to mark a child's result consumed,
and marking it on the send's return would lose results whenever the send
didn't become a turn. Pi's extension records the admission event Pi raises in
`delivery-receipts.json`, and Python confirms it from the session's public
event stream ([Pi runtime decision](../decisions/pi-runtime.md)). Claude's
mod puts a nonce line in the submitted text and acks only on the main-loop
`turn.start` whose text contains it
([child result delivery](../concepts/child-result-delivery.md)).

**The lesson:** record consumption from evidence that the model's turn began
with the text, never from the send call's return. Order it so that a crash
between the two can only cause a duplicate, never a loss: claim in memory,
send, observe the turn, then write the durable record.

**Where this lives:** `src/meridian/claude_runtime/meridian-spawn-watch/hooks/register.ts`,
`src/meridian/pi_runtime/extensions/meridian-spawn-watch/`. Host facts:
[Claude Code mods and the inbox](../research/claude-code-mods-and-inbox.md#mod-hooks-for-idle-delivery-21307).
