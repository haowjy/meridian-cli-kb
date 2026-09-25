# Decisions: Launch Harness Compatibility History

This page preserves harness- and platform-specific launch decisions. Native
Windows work is historical and superseded by the current POSIX-first stance.
Claude trampoline detection stays, but its record *repair* is superseded: the
successor never rewrites the entry chat and never names an exit chat; it is a
diagnostic.

## Decision Map

| Concern | Status | Current authority |
|---|---|---|
| Native Windows launch work | Superseded / legacy best effort | Root `AGENTS.md` POSIX-first rule |
| 2026-05 structural simplification | Historical rationale | [Launch architecture](../architecture/launch-system.md) |
| Claude TUI trampoline ID repair | Repair superseded; successor is a diagnostic only | [Native session identity](native-session-identity.md) |

## Windows Compatibility (Superseded)

Historical. Five decisions (D-win-signal, D-win-cancel, D-win-output-log,
D-win-ci, D-win-stdout) addressed portable signal handlers, cancel semantics,
output-log passthrough, CI gating, and stdout encoding for Windows. All are
superseded by the POSIX-first platform stance adopted in 2026-07 (PR #375).
Existing `os.name` branches may remain in source as untested best-effort code.

## Structural Simplification (Historical)

Completed in PR #184 (2026-05). Three waves removed proxy/wrapper indirections,
consolidated session carriers from `**kwargs` to typed dataclasses
(`SessionRequest`, `PrimarySessionMetadata`), and flattened harness launch specs
into a single `ResolvedLaunchSpec`. The rationale was uniform: each removed
layer was pure ceremony with no semantic value. The resulting architecture is
documented in [launch-system.md](../architecture/launch-system.md).

## Claude TUI Trampoline Session-ID Reconciliation (2026-06)

**Superseded in part (2026-09-24, 2026-09-25).** Detection is kept, but the successor
no longer rewrites records. Under the [native session identity](native-session-identity.md)
rule a chat's key is immutable, and a same-project prompt match is inference rather
than an owned signal. The successor is now persisted only as the diagnostic
`trampoline_successor_id`. Briefly it also served as the run's exit key; that was
removed because the correlation cannot distinguish an unrelated concurrent chat (see
[Claude native sessions](../architecture/claude-native-sessions.md#tui-trampoline)).
The original decision follows because it explains why detection is file-based.

**Decision:** Repair Claude TUI trampoline session IDs at finalization time via
file-based `history.jsonl` evidence rather than an interactive `claude
--resume` probe.

Claude's new TUI creates a transient trampoline session when entering `/tui
fullscreen`, then writes the durable transcript under a different session ID.
Meridian may record the trampoline ID during launch. The fix checks
`~/.claude/history.jsonl` at finalization: find the `/tui fullscreen` marker,
identify the successor same-project prompt, verify the successor transcript's
first user message matches, and update records to the durable ID.

**Why file-based over probe:** A `claude --resume <real-id> --print
--output-format json 'Reply: RESUME_PROBE_OK'` was tested and rejected. It
performs model work that hits the configured budget cap, making it
non-deterministic and inappropriate for finalization-time reconciliation.

**Why adapter override over generic finalization:** The trampoline pattern is
specific to Claude's new TUI and depends on Claude's `history.jsonl` format and
`projects/<slug>/` transcript layout. Embedding this in `ClaudeAdapter` keeps
trampoline-specific helpers alongside the existing transcript-path utilities
they depend on. Codex, OpenCode, and Pi do not have this pattern and need no
changes.

**Conservative fallback:** When evidence is insufficient (no trampoline marker,
no unique successor, transcript mismatch), the recorded ID is preserved
unchanged. This means existing behavior is preserved — the system fails or
succeeds on the recorded ID rather than guessing.

**Implementation:** `reconcile_tui_trampoline_session_id()` in
`src/meridian/lib/harness/claude.py`, wired into
`ClaudeAdapter.observe_session_id()` at the time; today it runs from
`ClaudeAdapter.observe_after_exit()` as a diagnostic only. Current mechanism: [../architecture/claude-native-sessions.md#tui-trampoline](../architecture/claude-native-sessions.md#tui-trampoline).

## Related

- [Launch composition decisions](launch.md)
- [../architecture/native-session-binding.md](../architecture/native-session-binding.md)
- [Harness abstraction](../concepts/harness-abstraction.md)
