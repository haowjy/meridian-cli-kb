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
renders, so every 2.x compaction skipped as `draft-unknown`. Its replacement,
a parser anchored on the composer border, still depended on a pane that resizes
and redraws. A rare lost draft costs less than a guard that silently disables
compaction.

**Decision:** Delete the draft fact and its guard everywhere: the core guard,
`IdleFacts`/`GuardFacts`, the CLI `--draft` flag, the OpenCode pane parser, and
the Claude and Pi draft vetoes. Claude, Pi and OpenCode compact through APIs
that do not touch the composer. Codex still compacts by typing into its TUI,
so it clears the composer first. The busy check, `/compact`
verify-before-Enter step and TUI-alive check stay. The screen-gated clear,
primer verification, retry window, veto reasons, and compaction-result
observation are mechanism owned by
[Idle Adapter Architecture](../architecture/idle-adapters.md#adapter-contracts).

**Why the boundary is strict:** Codex 0.162.1 clears a non-empty composer with
Ctrl+C, but an empty composer sends the key to the outer handler and can
interrupt or quit Codex. The actuator therefore must recognize the focused
composer and make it non-empty before clearing it; an unreadable screen must
veto rather than send a key. The core, not the adapter, decides that the
`screen-not-recognized` veto gets one push notice.

**Verified live** (codex-cli 0.162.1, `gpt-5.6-luna`, scratch homes):

- A three-line draft was cleared and compacted, and Codex kept running.
- An open `/` popup gave `composer-not-focused` before any key was sent.
- Five resizes just before compaction still compacted cleanly.
- A popup left open gave the `screen-not-recognized` veto after 10.02 s, with
  exactly one notice.

**Accepted residual risk:** the pane can change between the capture and the
keys. No tmux key sequence is atomic with the TUI's handling. The primer makes
the quit path unreachable in a normal composer, and the exact `/compact` check
blocks submission if stray text remains. The recognition and actuator details
remain in the architecture contract linked above.

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
