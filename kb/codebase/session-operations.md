# Codebase: Session Operations

Session operations (`meridian session log`, `session search`, `session export`) read agent conversation transcripts and present them in human-readable form. They're the primary tool for understanding what a spawn or primary session did.

## Compaction Segments

Claude compacts its conversation history after it grows beyond a threshold. Each compaction creates a new segment. `history.jsonl` covers the latest segment; earlier segments live in separate files.

`SessionLogInput.compaction` (default 0) selects which segment to read:
- `0` = latest
- `1` = one compaction back
- `2` = two compactions back

`SessionLogOutput` reports `total_compactions`, `has_earlier_segments`, and `has_newer`/`has_older` for pagination. The `--last N` flag returns the last N messages; `--offset N` pages forward.

## meridian session log

Read transcript messages from a session or spawn.

- `ref` — spawn ID (e.g. `p101`), chat session ID, `$MERIDIAN_CHAT_ID` for the primary session, or empty for the active primary
- `-c N` — select compaction segment (default 0 = latest)
- `--last N` — return last N messages (start narrow, widen as needed)
- `--offset N` — page forward within a segment

**Usage pattern:** Start with `--last 5`, widen to `--last 20` or `-n 0` (all) as needed.

## meridian session search

Text search across **all compaction segments** for a session. Iterates segments oldest-to-newest, case-insensitive substring match on content.

Each match includes:
- `segment` — which compaction segment
- `message_index` — position within that segment
- `role` — `user`/`assistant`/`tool`
- `content_preview` — up to 200 chars
- `nav_command` — `meridian session log <ref> -c <segment> --offset <idx-5> --last 10` for quick navigation

Useful for finding where a specific decision or file path was mentioned across a long session.

## meridian session export

Exports a full session transcript as clean Markdown. Renders all messages as `## [assistant]` / `## [user]` sections. Used for archiving session context before work items are closed, or sharing a transcript with collaborators.

## Related Pages

- [../architecture/state-system.md](../architecture/state-system.md) — spawn directory layout, where history files live
- [harness-adapters.md](harness-adapters.md) — per-harness transcript format differences
- [../architecture/claude-session-isolation.md](../architecture/claude-session-isolation.md) — how Claude session IDs are captured
- [../concepts/extension-system.md](../concepts/extension-system.md) — ops layer dispatch model
