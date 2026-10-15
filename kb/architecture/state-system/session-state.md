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

Restored sessions are immutable. Landing in PR #588, restore mints their generation
with the reserved `historical-` prefix, and the append guard checks the event's own
generation in O(1). On `main` the guard still folds the whole log under the writer
lock on every update, stop and model-selection append. The prefix rules are enforced
at append time only, never in replay, so an older build's unprefixed restore cannot
fail every read
([decision](../../decisions/history-storage.md#d-history-index-readers-never-wait-on-session-writers)).

## `sessions.jsonl` is the remaining global O(n) log

Every chat in a runtime appends to one file, and it only grows: 46.6 MB and 18.9 MB
on the development box in October 2026. Spawn state left this shape in May 2026.
Replaying the global `spawns.jsonl` made every status read O(total history), and its
one flock serialized every writer. Each spawn now owns `spawns/<id>/state.json` with
its own lock
([spawn state v2](../../decisions/state.md#spawn-state-v2-per-spawn-statejson-over-global-jsonl-2026-05)). Sessions never got that split. Any
path that folds the whole log costs time proportional to every chat that ever ran,
and if it holds `sessions.jsonl.flock` it serializes every other session writer.

The rule to keep is that no per-spawn path replays `sessions.jsonl` while holding
`sessions.jsonl.flock`. #588 removes the worst violator (the append guard, about four
folds per spawn) and stops history-index readers from waiting on the lock at all.
Several paths still violate the rule, tracked in
[#591](https://github.com/haowjy/meridian-cli/issues/591):

| Path | When | Cost at 46.6 MB |
|---|---|---|
| `SessionBindings.__init__` | once per spawn or primary start (native-ID bind), repair, legacy import | 1,157 ms median hold |
| `start_session()` lookups | every start | full fold under the lock |
| `record_model_selection()` / `record_model_observation()` dedup | model events | full fold under the lock |
| `get_or_create_observed_chat()` | observed-chat binding | full fold under the lock |
| Automatic archiver, one fold per candidate (no lock on the log) | each archiver run | 612 folds, 4.7 s warm, >20 s cold |

Since #588, these paths add latency to spawn start but no longer stall index
readers. The fix is a design decision (`decision-needed` on #591). Binding conflict
checks may need every chat's records, so a per-chat read alone may not be enough.
The likely shape is per-chat session files that follow the per-spawn `state.json`
precedent, or a compacted log. Either way, the projection must stay rebuildable from
files (files as authority).

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
