# Decision: Idle Compaction Actuator

This page records the decision that idle compaction takes precedence over
unsent composer text, including the safety boundary for Codex's tmux actuator.
The current adapter sequence and runtime contract live in
[Idle Adapter Architecture](../architecture/idle-adapters.md).

## D-idle-compact-over-drafts — Compaction wins over unsent text

**Status:** decided by the user (2026-10-15); merged in PR #583 (`8bd2f4775`),
released in v0.10.5. `work:idle-compaction-floor` ("User: compact even with a
draft"), evidence `evidence/drop-draft-guard.md`.

**User rule:** "I would rather we compact than keep any drafts." The user said
this after asking what happens when a tmux pane is mangled or resized.

**Problem:** compaction used to skip when the composer held unsent text, and
also when the adapter could not tell. Only Claude and Pi could see a draft
through their host APIs. Codex and OpenCode guessed from the tmux pane. The
OpenCode 2.x pane parser looked for a footer separator that 2.0.28 never
renders, so every 2.x compaction skipped as `draft-unknown`. Its replacement, a
parser anchored on the composer border, still depended on a pane that resizes
and redraws. A rare lost draft costs less than a guard that silently disables
compaction.

**Decision:** delete the draft fact and its guard everywhere: the core guard,
`IdleFacts`/`GuardFacts`, the CLI `--draft` flag, the OpenCode pane parser, and
the Claude and Pi draft vetoes. Claude, Pi and OpenCode compact through APIs
that do not touch the composer. Codex still compacts by typing into its TUI,
so it has to clear the composer first. The busy check, the `/compact`
verify-before-Enter step and the TUI-alive check stay.

**Codex clears the composer with a screen-gated Ctrl+C.** Codex 0.162.1 has
no single key that clears a multi-line composer: Ctrl+U deletes only to the
start of the line. Ctrl+C clears the whole composer, including attachments,
but only when the composer is non-empty. On an empty composer the key falls
through to the outer handler, which interrupts running work or quits Codex.
The double-press quit confirmation is disabled in this release. So the
sequence is:

1. Capture the pane and require the focused composer layout: two blank rows
   above a column-zero `›` prompt, indented continuation rows, the blank
   bottom margin, an indented status row, and the footer (`… for shortcuts`
   when empty, blank with a draft). Reject `Working (`.
2. Type the primer `ix`, which makes the composer non-empty in both normal and
   Vim mode (`i` enters insert mode). Confirm that the visible text is exactly
   the previous text plus `ix`.
3. Re-check busy and liveness, then send Ctrl+C.
4. Confirm the composer shows empty, type `/compact`, verify it, press Enter,
   and wait for a new `compacted` rollout record.

A slash-command popup takes the blank rows above the prompt. Modals, the
agents overview and transcript views replace the footer. None of them match
the layout, so Ctrl+C is never sent to them. The first version without the
layout gate was sent back in review. In that version a key could reach a modal
or the agents overview, and an unprimed Ctrl+C could quit Codex or interrupt
it. The deleted empty-`›` draft check had been blocking those screens by
accident.

**An unreadable screen retries, then notifies.** If the layout doesn't match,
the actuator re-captures for about 10 s (three reads). If it still can't
recognize the screen, it vetoes with `screen-not-recognized`, the same reason
it uses when the primer or the cleared composer fails to verify. No key is
sent. Core recognizes that one reason and sends a push-only notice:
"Codex not compacted: Meridian couldn't read its screen", plus the minutes of
cache left. The decision about who to notify stays in the idle policy layer,
not the adapter (user decision). Other vetoes and failures keep the generic
compaction notice.

**Verified live** (codex-cli 0.162.1, `gpt-5.6-luna`, scratch homes):

- A three-line draft was cleared and compacted, and Codex kept running.
- An open `/` popup gave `composer-not-focused` before any key was sent.
- Five resizes just before compaction still compacted cleanly.
- A popup left open gave the `screen-not-recognized` veto after 10.02 s, with
  exactly one notice.

**Accepted residual risk:** the pane can change between the capture and the
keys. No tmux key sequence is atomic with the TUI's handling. The primer makes
the quit path unreachable in a normal composer, and the exact `/compact` check
blocks submission if stray text remains.

**Rejected:**

- **Keeping the guard with a better OpenCode pane parser.** A parser was built
  (`spawn:p7902`) and then deleted unmerged. It still depended on the pane, and
  the user preferred compaction.
- **Silent vetoes for an unreadable Codex screen.** Compaction would stop
  without the user knowing. One notice per stretch costs little.

**Related, not decided here:** Codex and OpenCode TUIs resize correctly when
run natively, but their layout gets mangled under Meridian primaries. That is
tracked in `work:primary-resize-mangling`.


## Related

- [Idle Notifications and Idle Compaction](idle-notifications.md): shared guard order, stretch lifecycle, and hosting rationale.
- [Idle Adapter Architecture](../architecture/idle-adapters.md): current Codex, Claude, Pi, and OpenCode actuator mechanisms.
- [Idle Compaction Growth Floor](idle-compaction-growth-floor.md): break-even guard applied before the actuator.
