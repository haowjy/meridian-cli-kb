# Codebase: Session Operations

Session operations (`meridian session log`, `session search`, `session export`) read agent conversation transcripts and present them in human-readable form. They're the primary tool for understanding what a spawn or primary session did.

## Transcript Source Resolution

`session_target.py` resolves a user ref into an ordered `SessionLogTarget.sources` plan.
`session_transcript.py` then parses sources in that order and uses the first one with
usable user/assistant interaction content. Fallback is source-list iteration, not
recursive target resolution.

### Claude: Trust-Ordered Root Chain

Claude transcript resolution uses a trust-ordered root chain when resolving
tracked sessions. The adapter seam `resolve_session_file()` accepts an
optional `config_root_hint` parameter, threaded from the persisted
`SessionRecord.claude_config_dir` / `SpawnRecord.claude_config_dir` through
every tracked ref form (bare harness session id, chat id, spawn id, corpus
search, and `session repair` paths).

The Claude adapter searches a deduped chain of roots, ordered by trust:

1. **Recorded config dir** (the dir where the session actually ran)
2. **Canonical `~/.claude`** (via `get_home_path()`) — covers the common
   case where a transcript was repaired to the canonical root while the
   ambient env points elsewhere
3. **Ambient `CLAUDE_CONFIG_DIR`** — last resort

First existing `<session_id>.jsonl` match wins. The chain is ordered by trust
because cross-root copies (not always symlinks) can diverge in content.

Untracked resolution passes `config_root_hint=None`, behavior unchanged.
Non-Claude harnesses (codex, opencode, pi) accept the hint parameter and
ignore it — their session roots are not env-relocatable this way.

Two rejected alternatives shaped this design:
- **Persisting a resolved transcript path** was rejected because Claude
  transcripts can be re-materialized into new roots on later launches,
  making a persisted path go stale.
- **Canonicalizing `CLAUDE_CONFIG_DIR` at session creation** was rejected
  because overlays exist for concurrent session isolation; canonicalizing
  defeats that purpose.

### OpenCode: SQLite Precedence

OpenCode completed-session precedence is:

1. OpenCode SQLite (`opencode.db`) when a matching `session.id` exists;
2. harness-native transcript file (legacy `storage/session_diff/...` JSON) when present;
3. Meridian spawn `history.jsonl` as fallback/debug/live output.

Non-file sources are modeled explicitly (`TranscriptSource(kind="opencode_db", path=None)`);
Meridian does not fabricate a session file path just to satisfy file-oriented code.
Spawn history remains a fallback, not the preferred transcript for completed OpenCode
sessions.

Test coverage is split along the same seams: target/source ordering and DB-backed
rendering live in session-log integration/unit tests, resident drain scope behavior
lives with streaming tests, and reusable OpenCode SQLite fixtures live in
`tests/support/opencode_db.py`.

## Segment Model

Claude compacts its conversation history when it grows beyond a threshold. Each compaction creates a new **segment**. The transcript is a sequence of segments, each being a self-contained window of conversation history:

- `history.jsonl` — current (latest) segment
- `history.compaction.1.jsonl`, `history.compaction.2.jsonl`, … — earlier segments, numbered oldest to newest

Every segment has an **entry 0** — the segment setup slot:

- **Segment 0, entry 0**: the session's initial system prompt/prologue. If the harness makes it available, it's extracted and shown. If not, a placeholder is materialized.
- **Later segment, entry 0**: the compaction handoff or summary that seeded the new segment. If an explicit handoff text is extractable by the provider parser, it's shown. If not, a placeholder is materialized.

Entry 0 is always present — the segment always exists whether or not setup text was recoverable. Segment splitting is driven by explicit provider/harness boundary markers, not by whether summary text was found.

**Entry 0 extraction is provider-specific.** Generic scraping of arbitrary event keys must not decide what counts as a prologue or handoff. The parser is conservative: it reads known provider-specific fields rather than grepping for anything that looks like a summary. Missing content results in a placeholder, not a fabricated value.

The interaction entries (1, 2, 3, …) within a segment are the turn-based conversation: user messages, assistant responses, and tool round-trips grouped into logical entries.

## Default Navigation Model

Bare `meridian session log REF` applies safe defaults: **last 5 interaction entries from the current segment, shown oldest-to-newest (chronological)**. This is the right starting point for agents reading recent context — narrow enough to stay cheap, ordered correctly for understanding flow.

```bash
meridian session log p107           # last 5 entries, current segment, chronological
meridian session log $MERIDIAN_CHAT_ID  # same for the primary session
```

Navigation is **segment-local by default**. Ordinals in `--from`, `--before`, `--around` refer to entry positions within the selected segment. The current/last segment is the default selection.

## Segment Selection

```bash
meridian session log REF --segment current     # current/latest segment (default)
meridian session log REF --segment previous    # segment before current
meridian session log REF --segment 0           # absolute segment index
meridian session log REF --segment 2           # third segment
```

Entry 0 is the segment setup slot for every segment. It's included in `--full` and `--global` output, and can be read explicitly:

```bash
meridian session log REF --segment N --from 0 --limit 1   # just the segment setup entry
```

## Navigation Flags

All positional selectors are segment-local by default (operate within the selected segment's entries):

| Flag | Behavior |
|---|---|
| *(no flags)* | Last 5 entries from current segment, chronological |
| `--tail` | Last 5 entries (explicit; same as default when no other flags given) |
| `--tail N` | Last N entries |
| `--full` | All entries in the selected segment, including entry 0 |
| `--full --no-truncate` | All entries, full content (no preview truncation) |
| `--from N --limit M` | M entries starting at entry N (segment-local) |
| `--before N --limit M` | M entries ending before entry N (segment-local) |
| `--around N --context M` | 2M+1 entries centered on entry N (segment-local) |
| `--segment N` | Select segment N (default: current) |
| `--global` | Cross-segment stream with unique global ordinals starting at 0 |

`--global` includes every segment's entry 0 setup slots with unique global ordinals. Ordinals in `--from`/`--before`/`--around` switch to global scope when `--global` is used. `--global` and `--segment` cannot be combined.

## Content Truncation

By default, oversized entries are preview-truncated (safe for reading in terminals and agents). Use `--no-truncate` to get full content for selected entries. `--no-truncate` combines with any navigation mode.

## meridian session search

Text search across **all segments** for a session (or a multi-session corpus). Searches both interaction entries and real segment setup content (entry 0 prologue/handoff text). Placeholders are excluded from search — only real extracted content matches.

Search scope flags (optional):

| Flag | Scope |
|---|---|
| *(no flag)* | Current project only |
| `--workspace` | Current project + configured workspace roots that are Meridian projects |
| `--global` | All Meridian project roots under user home |
| `--work WORK_ID` | Sessions associated with a specific work item |

Each match includes a deterministic `Open:` command for navigating to the exact location. Open commands are argv-based and platform-aware:

- **Entry 0 hit** → `meridian session log REF --segment N --from 0 --limit 1`
- **Interaction entry hit** → `meridian session log REF --segment N --around K --context 5`

Open commands use segment-local references and absolute entry ordinals — they stay valid as the session grows.

## meridian session export

Exports a full session transcript as clean Markdown. Renders all messages as `## [assistant]` / `## [user]` sections. Used for archiving session context before work items are closed, or sharing a transcript with collaborators.

## Common Patterns

```bash
# Safe recent read — start here
meridian session log p107

# Explicit tail
meridian session log p107 --tail
meridian session log p107 --tail 20

# Read the segment setup (prologue / compaction handoff)
meridian session log p107 --from 0 --limit 1

# Widen to full current segment
meridian session log p107 --full

# Full content, no truncation
meridian session log p107 --full --no-truncate

# Deterministic window around a known entry
meridian session log p107 --around 12 --context 5

# Previous segment (most recent compaction context)
meridian session log p107 --segment previous

# Cross-segment global view (all entries including every segment's entry 0)
meridian session log p107 --global --from 0 --limit 1

# Search this session
meridian session search "auth middleware" p107

# Search all sessions across the project
meridian session search "design decision"

# Search across all known Meridian projects
meridian session search "pattern" --global
```

## Related Pages

- [../architecture/state-system.md](../architecture/state-system.md) — spawn directory layout, where history files live
- [../architecture/claude-session-isolation.md](../architecture/claude-session-isolation.md) — how Claude session IDs are captured
- [harness-adapters.md](harness-adapters.md) — per-harness transcript format differences; provider-specific prologue/handoff extraction
- [../concepts/spawn-output-contract.md](../concepts/spawn-output-contract.md) — progressive disclosure: spawn report → session log → no-truncate
- [session-log-rendering.md](session-log-rendering.md) — internal rendering pipeline: ToolCall normalization, clean vs raw output, flag design, content pipeline order
