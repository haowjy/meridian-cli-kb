# Session State


Sessions track one immutable native key per chat (`harness`, `native_store`,
`harness_session_id`), work-item attachment, primary-spawn relationships,
and lifecycle (created → active → closed). The key only gains fields. Every write and
replay goes through one pure rule, `bind(prior, attempted) → Bound | Same | Conflict`,
and a conflict is never applied. The pure replay fold is public as
`state/session_fold.py`, with `by_native_key` for the inverse lookup. Legacy multi-ID
arrays are ignored on read (see
[native session binding](../native-session-binding.md#binding)). Chats from before the key
existed get their key only from the one-time
[legacy import](../legacy-native-import.md#legacy-import). It binds through the same
lock-scoped `state/session_binding.py` path with `source: legacy_import`, and it
records its outcome in `legacy-native-import-v1.json` at the runtime root.

`sessions.jsonl` is the sole authority for these facts. There is no session SQLite
index: `session_store` replays the journal on each read. `get_session_record` and
`get_session_records` filter events down to the requested chats before folding, so a
one-chat lookup does not fold the whole journal, but it still reads and parses every
line. `list_all_session_records` folds everything. A `sessions-index.sqlite3` or
`sessions-append-state.json` found in an old runtime root was left by an older build;
no current code reads either one.

Restored sessions are immutable. Since #588 (v0.10.4), restore mints their generation
with the reserved `historical-` prefix, and the append guard checks the event's own
generation in O(1) instead of folding the log. The prefix rules are enforced at
append time only, never in replay, so an older build's unprefixed restore cannot
fail every read
([decision](../../decisions/history-storage.md#d-history-index-readers-never-wait-on-session-writers)).

## `sessions.jsonl` is one global log

Session state is event-sourced into **one file per runtime**. Every chat appends to
`sessions.jsonl`, the file only grows (46.6 MB and 18.9 MB on the development box in
October 2026), and most reads replay it. **Sessions are not stored per chat.** The
only per-chat files are the lifetime lock and lease below. A `chats/` directory
appears empty in every runtime: it was created for a `chats/<chat_id>/lifecycle.jsonl`
primitive (2026-04) that nothing ever wrote. #418 (2026-07) prescribed per-session
authority files, and they were never built.

Spawn state left this shape in May 2026. Replaying the global `spawns.jsonl` made
every status read O(total history), and its one flock serialized every writer.
Each spawn now owns `spawns/<id>/state.json` with its own lock
([spawn state v2](../../decisions/state.md#spawn-state-v2-per-spawn-statejson-over-global-jsonl-2026-05)).
Sessions never got that split. Every path that folds the whole log costs time
proportional to every chat that ever ran. If it also holds `sessions.jsonl.flock`,
it serializes every other session writer.

#588 removed the worst holder (the append guard, about four folds per spawn) and
stopped history-index readers from waiting on the lock. What remains, measured on a
real launch with a fake harness at 46.6 MB (v0.10.4,
[per-operation cost](../per-operation-cost.md#1-session-state-is-one-global-replay-log-418)):

| Reader | When | Cost at 46.6 MB |
|---|---|---|
| `session_bindings()` / `SessionBindings.__init__`, under the flock | native-ID bind at spawn or primary start (twice per launch), repair, legacy import | 1.12 s for both reads |
| `get_session_active_work_id` | every launch | 0.60 s |
| `conclude_native_run` → observed chat, under the spawn lock | every native run's exit | 0.58 s |
| `_session_execution_context` re-read (plus `session_scope`'s own re-read) | every launch | 0.46 s for the first |
| `start_session` lookups, `record_model_selection` / `record_model_observation` dedup, `get_or_create_observed_chat`, under the flock | start, model events, observed chats | one full fold each |
| `get_session_record_for_spawn`, `session_records_for_spawns` | wait, show, archive witness | one full fold each |
| Automatic archiver, one fold per candidate (no lock on the log) | interval-gated, on primary stop | 612 folds, 4.7 s warm, >20 s cold |

That adds up to 9 full-log reads per foreground launch, and a no-op launch that takes
9.12 s at 46.6 MB against 3.59 s at 1 MB. These readers lengthen launch but no
longer stall index readers.

**The accepted plan, not built:** per-chat journals,
`session-state/chats/<cN>.jsonl`, each the only authority for its chat, plus small
per-native-ID files for the facts that cross chats. A one-time cutover splits the
old log, and rows that older builds keep appending are absorbed at the next
command. The design is accepted and pending the owner's approval; it ships as
#418 B1–B3
([D-session-per-chat-journals](../../decisions/state.md#d-session-per-chat-journals)).
Until it lands, describe and test sessions as the global log above.

Two disposable projections sit beside the journal. Neither holds a fact the journal
lacks:
- **Metadata index** (`history-index/history-v6.sqlite3`; the file is named by schema): cross-record browse,
  discovery, aliases, spawn records for reclaimed refs and the preview cache. Its
  `sessions` table folds native keys with the authority's own step,
  `project_session_generation`, and persists the working state as `session_chats`.
- **Search projection** (`history-index/native-search-v1.sqlite3`): normalized display
  text keyed by native key, with no chat IDs. Chats are joined at query time from the
  fold with `by_native_key`
  ([native transcript reads](../native-transcript-reads.md#search-projection)).

`state/session_identity.py` joins sessions and spawn rows: the owner chat of a spawn,
a chat's recorded primary spawn, and `session_records_for_spawns`. That last one returns
the exact session generation a run started, not the chat's current binding.

Normal browse uses the metadata index's recent-session view, which can include ZIP
and inert historical records. `--include-archives` adds archived rows to the picker.
Preview and re-entry resolve authoritative session and spawn facts before acting.
Transcript reads go through one resolver to the chat's bound native key. There is no
fallback scan, and a chat without a key is `unbound`
([native transcript reads](../native-transcript-reads.md#one-resolver-one-reader)).

Native capture preparation is narrower than presentation. It accepts only an
identified terminal local spawn and reads the key of that run's exact session
generation. A missing key refuses capture as `unbound`, and a conflict raises. Native
identity never changes re-entry authorization. See
[Portable history](portable-history.md#archive-capture-is-native-only).

`sessions.jsonl` also records `model_selection` events (legacy `initial_seed` or
`invocation_started`). That is Meridian-selected intent for later continue, not
an observation of the last model that executed. Writes happen only at
accepted-running. Native ID may bind later without moving the log row. The
original launch-policy snapshot is not rewritten.

Session-ID counter (`session-id-counter`) is monotonically incremented under `platform.locking.lock_file()` so concurrent spawns never collide.

Per-session files under `sessions/<chat_id>/`:
- `<chat_id>.lock` — held for active session duration
- `<chat_id>.lease.json` — PID + generation token for staleness detection


## Related Pages

- [../native-session-binding.md](../native-session-binding.md)
- [../../decisions/native-session-identity.md](../../decisions/native-session-identity.md)
- [State system overview](overview.md) — state roots and subsystem map
- [Spawn state](spawn-state.md) — per-spawn authority and artifact lifetime
- [Portable history](portable-history.md) — cross-record discovery, exact identities, ZIP retention, and inert restore
- [Reconciliation](reconciliation.md) — read-time projection and durable repair
