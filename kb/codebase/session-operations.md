# Codebase: Session Operations

Session operations (`meridian session log`, `session search`, `session export`, `session browse`) read agent conversation transcripts and present them in human-readable form. They're the primary tool for understanding what a spawn or primary session did.

## Session Browse

`meridian session browse` is an interactive full-screen picker over recent
primary sessions. It lists recent c-ids (primary chats only, current project
scope), lets the user filter by metadata or deep-search transcripts via `/`,
preview the highlighted session's current segment, and on Enter exec's
`meridian --continue <c-id>` for stopped sessions or `meridian --fork <c-id>`
for live ones. A live session is never double-attached — fork allocates a new
c-id by construction.

Bare `meridian --continue` (no ref) opens the same picker. Both entry forms
are identical after argv canonicalization; there is no origin flag and no
provenance-dependent behavior.

Non-TTY invocations or `--plain` degrade to a plain table listing (no
interaction), so the command stays scriptable and testable. Invocations from
inside a managed Meridian session (detected by `MERIDIAN_SPAWN_ID`) refuse
interactive mode with an explicit message — exec-ing `--continue` from inside
a running harness would nest the selected session under the original one.
`--plain` still works inside managed sessions.

The picker is built on prompt_toolkit's full-screen `Application`
([decision rationale](../decisions/tui-framework.md)). The TUI is pure
presentation over four ops-layer seams: `session_list_sync` (listing),
`session_log_sync` (preview), `iter_session_subset_search` (deep search), and
`resolve_session_reentry` (Enter-time re-entry decision). The TUI never touches
`lib/state`, `lib/harness`, or transcript files directly.

See [../concepts/session-initiation.md](../concepts/session-initiation.md) for
the re-entry model (Resume/Fork/Blocked) that governs what Enter does.

## Transcript Source Resolution

The contract from PR 2 (draft PR #526) is **ref → chat → the chat's
bound native key → that harness's exact reader**, through one function,
`ops/session_target.resolve_transcript_source`. `session log`, export, preview and
`search REF` all use it. The full ref table, view labels and search projection are in
[native transcript reads](../architecture/native-transcript-reads.md).

- **A tracked chat reads only its bound native key.** If the chat has no key, or
  its transcript is missing or still pending, resolution raises
  `NativeSessionUnavailable(reason="unbound"|"missing"|"ambiguous_native_file")`.
- **Nothing else can stand in:** not candidate IDs, primary metadata, adapter
  scans, ambient harness roots, index rows, or Meridian's runner `history.jsonl`
  ([native-only history](../decisions/native-only-history.md)).
- **`--file PATH`** reads a native file. A runner `history.jsonl` is rejected as
  "not a native transcript".
- **Untracked raw IDs** keep a labeled lookup (`untracked`); they are not on any
  tracked path.

### Claude: exact file in the recorded store

A tracked Claude chat reads exactly `<recorded native store>/<id>.jsonl`
(`resolve_native_session_file`), independent of the caller's cwd. There is no
trust-ordered root chain. A missing file is `missing`, not a reason to try
`~/.claude` or the ambient `CLAUDE_CONFIG_DIR`. The file's first-line `sessionId`
must equal the ID. A tracked record without a native store is `unbound`; its
persisted `claude_config_dir` hint is not used to rebuild a path. The legacy
`resolve_session_file` path serves only explicitly untracked references: a hint
expands only to `<hint>/projects/<slug>`, and without one it uses the current config
root. See
[Claude native sessions](../architecture/claude-native-sessions.md#reading-transcripts).

Persisting a resolved transcript *path* was rejected, because Claude transcripts are
re-seeded into other project stores on later launches. The key is store plus ID, and
the file is re-resolved inside it.

### Codex and OpenCode

Codex reads the exact rollout under its recorded home, header-checked. OpenCode reads
the session row in the exact recorded database, opened `mode=ro`, and models the
source as `TranscriptSource(kind="opencode_db", path=None)` rather than a fabricated
path. An empty OpenCode session gives an empty view; there is no fallback to runner
output. Reusable OpenCode SQLite fixtures live in `tests/support/opencode_db.py`.

### Archive capture

Archive capture resolves the exact native key recorded for the aggregate and
publishes its snapshot. A spawn with no native source stays loose with a reason.
Legacy runner-history members of old ZIPs restore as bytes and are never read as
transcripts ([portable history](../architecture/state-system/portable-history.md#archive-capture-is-native-only)).

### Pi: exact file in the recorded store

A Pi chat reads the single `*_<id>.jsonl` in its recorded store whose header ID
matches. There is no discovery and no cross-spawn glob. The journal is projected onto
Pi's reopen-default lineage before normalization, so abandoned sibling branches do not
render. See [../architecture/pi-native-sessions.md](../architecture/pi-native-sessions.md#journal-topology-and-readback).

## Runner-History Prune

`meridian session archive --prune-runner-history [--apply] [--after-days N]` deletes
runner-stream files that Meridian stopped writing in PR 3 and never reads, for spawns
whose exact native source makes them redundant. It lives in
`ops/runner_history_prune.py` and is wired from `session_archive_sync`. The output is
the `runner_history` field of `SessionArchiveOutput`. Why the rule has this shape:
[the prune rule](../decisions/native-only-history.md#the-prune-rule).

- **Dry run by default.** It prints each spawn it would prune with bytes, a total, a
  count and bytes per skip reason, the quarantined IDs with a `meridian doctor`
  pointer, and errors. `--apply` deletes. `--after-days` defaults to 14.
- **Refused combinations:** refs, `--eligible`, `--list` and `--destination`.
- **Never automatic.** `session_stop_maintenance` and `history.archive.automatic` do
  not call it, and a test covers this.

**Qualification.** Only spawns that still have runner-stream files are considered.
`_judge` returns a typed `_Verdict`, either a skip reason or the native source paths.
The first failing check is the skip reason:

| Order | Check | Skip reason |
|---|---|---|
| 1 | Not a historical (restored) record | `historical` |
| 2 | Status is terminal | `running` |
| 3 | `terminal.finished_at` parses | `no_terminal_time` |
| 4 | Finished more than N days ago | `recent` |
| 5 | `run_boundary` is `None` or `verified` | `exit_unresolved` |
| 6 | No unreleased, likely-serving process scope | `live_scope` |
| 7 | `session_target.resolve_run_sources(row, sessions)` resolves the log chat and the entry chat exactly | the `NativeSessionUnavailable` reason (`unbound`, `missing`, `ambiguous_native_file`), or `error` for any other exception |

Check 7 runs the same `_spawn_target` chain as `session log pN`: `continue_chat_id` →
`native_key()` → `adapter.resolve_native_session_file`. Adapters validate the header
or DB row. The sessions projection is built once per pass.

**What gets deleted.** `runner_stream_files` looks under `spawns/<id>` and legacy
`artifacts/<id>`, plus their `attempt-<n>/` subdirectories. In each it takes the
`RETIRED_RUNNER_STREAM_FILENAMES` (`history.jsonl`, `last-observed-event.json`) and
their `.<name>.*.tmp` atomic temps. It takes only regular files under `lstat`, and
skips symlinked directories and `attempt-N.tmp` staging directories. `state.json`,
`report.md`, logs, `pi-lifecycle.json`, native data, ZIPs and `sessions.jsonl` are
never touched.

**Apply.**
- The pass holds `history-archives/archive.lock`.
- Each spawn is unlinked inside `mutate_published_spawn_artifact`, which holds the
  shared history-mutation lock and the spawn lock. `can_mutate` requires the record
  to equal the planned one (prompt excluded) and re-runs `resolve_run_sources`.
  Otherwise the spawn is kept and reported.
- Each spawn's body runs in its own `try`; a failure becomes `"<id>: <error>"` and
  the pass continues.
- Files are unlinked one by one with `missing_ok`, so a rerun after a crash converges.
- `HistoryIndex.catch_up()` runs afterwards. The index does not project these files.

Capture and ZIP archive still work after pruning: capture needs a sealed native
snapshot, and runner files were never an archive source. A prune → capture → archive
test checks that the ZIP holds `native-transcript.jsonl` and no runner members.

## Segment Model

Harnesses compact a conversation when it grows beyond a threshold. The normalizer
splits the native transcript at each compaction boundary into **segments**. The
transcript is a sequence of segments, each a self-contained window of conversation
history. The latest segment is the default view.

Every segment has an **entry 0** — the segment setup slot:

- **Segment 0, entry 0**: the session's initial system prompt/prologue. If the harness makes it available, it's extracted and shown. If not, a placeholder is materialized.
- **Later segment, entry 0**: the compaction handoff or summary that seeded the new segment. If an explicit handoff text is extractable by the provider parser, it's shown. If not, a placeholder is materialized.

Entry 0 is always present — the segment always exists whether or not setup text was recoverable. Segment splitting is driven by explicit provider/harness boundary markers, not by whether summary text was found.

**Entry 0 extraction is provider-specific.** Generic scraping of arbitrary event keys must not decide what counts as a prologue or handoff. The parser is conservative: it reads known provider-specific fields rather than grepping for anything that looks like a summary. Missing content results in a placeholder, not a fabricated value.

The interaction entries (1, 2, 3, …) within a segment are the turn-based conversation: user messages, assistant responses, and tool round-trips grouped into logical entries.

## Default Navigation Model

Bare `meridian session log REF` defaults to **the last 5 interaction entries
from the current segment, shown oldest-to-newest (chronological)**. This narrows
the returned navigation window, not the underlying read work: at `4adeb355`, the
source is fully decoded, normalized, flattened, and grouped before the tail is
selected. One entry can contain arbitrarily many consecutive messages.

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

This truncation is a presentation limit on each message's cleaned content. It
does not bound source bytes, decoded messages, grouping, or the number of messages
rendered inside an entry. Browser pane clipping likewise happens after rendering.
Issue #495 tracks the still-unimplemented bounded-content preview contract; an
oversized-source unavailable state alone is not the accepted substitute for a
useful fast large-history preview.

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

- [../architecture/state-system/spawn-state.md](../architecture/state-system/spawn-state.md) — spawn directory layout, where history files live
- [../architecture/claude-native-sessions.md](../architecture/claude-native-sessions.md) — how Claude session IDs are captured
- [../decisions/native-session-identity.md](../decisions/native-session-identity.md)
- [harness-adapters.md](harness-adapters.md) — per-harness transcript format differences; provider-specific prologue/handoff extraction
- [../concepts/spawn-output-contract.md](../concepts/spawn-output-contract.md) — progressive disclosure: spawn report → session log → no-truncate
- [session-log-rendering.md](session-log-rendering.md) — internal rendering pipeline: ToolCall normalization, clean vs raw output, flag design, content pipeline order
