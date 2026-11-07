# Session state

A chat's native key identifies its harness, store and session. Bindings are fill-only except for explicit relocation, so a conflict cannot silently redirect a conversation. See [native session identity](../../decisions/native-session-identity.md) and the source checkout's `src/meridian/lib/state/.context/CONTEXT.md`.

## `sessions.jsonl` is one global log

Session state is event-sourced into **one file per runtime**. Every chat appends to
`sessions.jsonl`, the file only grows (46.6 MB and 18.9 MB on the development box in
October 2026), and most reads replay it. **Sessions are not stored per chat.** The
only per-chat files are the lifetime lock and lease below, plus each chat's `work` and `task` links, which are projections of
the row and never authority. A `chats/` directory
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

#588 removed the worst holder (the append guard) and stopped history-index readers
from waiting on the lock. #634 and #651 then moved the per-operation readers
(bindings, start, model selection and observation dedup, observed chats, the
launch work-id and execution-context reads, spawn witnesses) onto the keyed seam
above. They no longer decode or keep other chats' rows, so their memory and decode
time stopped growing with the log. Their reads still pass over every byte, and some
still hold `sessions.jsonl.flock` while they do; a primary close makes 10 filtered
passes and 0 full folds. Measured costs are on
[per-operation cost](../per-operation-cost.md#session-state-is-one-global-replay-log).
The automatic archiver still folds once per candidate.

A scoped `SessionBindings` (`scope` is a frozenset of chats) holds only those chats'
history. Binding any other chat raises `ChatOutsideBindingScope`, and cross-chat
uniqueness checks need the full snapshot (`scope=None`). A corrupt row poisons only
its own chat: other chats still read and bind, while reading that chat, or the full
list, raises.

**The accepted plan is not built:** per-chat journals,
`session-state/chats/<cN>.jsonl`, each the only authority for its chat, plus small
per-native-ID files for the facts that cross chats. A one-time cutover splits the
old log, and rows that older builds keep appending are absorbed at the next
command. The seam shipped in #651; replacing its implementation awaits open design questions
([D-session-per-chat-journals](../../decisions/state.md#d-session-per-chat-journals)).
Until it lands, describe and test sessions as the global log above.

Two disposable projections sit beside the journal. Neither holds a fact the journal
lacks:
- **Metadata index** (`history-index/history-v<N>-sessions-v<S>.sqlite3`, named by
  index schema and session event format; schema 12 since the session-browse redesign, PR #714, which skipped 11; 10 came
  with #631; 8 and 9 were written only by unreleased #616 builds; see the flag on
  [D-session-per-chat-journals](../../decisions/state.md#d-session-per-chat-journals)):
  cross-record browse, discovery, aliases and spawn records for reclaimed refs.
  Since v12 it also projects lineage edges and the latest model observation per native
  session, and drops the preview cache. Its
  `sessions` table folds native keys with the authority's own step,
  `project_session_generation`, and persists the working state as `session_chats`.
  Recency selects a chat's highest-ordinal generation bucket.
  That can differ from the accepted generation when a generation key is reused;
  making the index follow the accepted generation requires a schema bump.
- **Search projection** (`history-index/native-search-v1.sqlite3`): normalized display
  text keyed by native key, with no chat IDs. Chats are joined at query time from the
  fold with `by_native_key`
  ([native transcript reads](../native-transcript-reads.md#search-projection)).

`state/session_identity.py` joins sessions and spawn rows: the owner chat of a spawn,
the spawn that owns a chat (`chat_owner_spawn`), and `session_records_for_spawns`. That
last one returns the exact session generation a run started, not the chat's current
binding. `chat_owner_spawn` is the only chat→spawn answer. It walks the chat's
generations newest first and works only because each start event already names its
spawn ([D-link-at-start](../../decisions/chat-spawn-ownership.md#d-link-at-start)).
Only the start event writes that link
([D-owner-on-start](../../decisions/chat-spawn-ownership.md#d-owner-on-start), since
meridian-cli v0.17.1). A raw harness ID reaches its chat
through `session_store.native_generations`, the one native-ID lookup
([D-ref-by-shape](../../decisions/ref-classification.md#d-ref-by-shape)).

Browse uses the metadata index's recent-session view, which can include ZIP and
inert historical records; on the session-browse branch, Status: Archived selects
archived rows. The detail view and re-entry resolve authoritative session and spawn
facts before acting.
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
- `work`, `task` — symlinks to the execution's work directory and source checkout,
  published by `launch/session_links.py`; see
  [per-execution work and checkout links](../../decisions/session-work-links.md)

## A later start closes abandoned generations

A later accepted start closes earlier unstopped generations in the replay fold. A new start requires the chat lock, so the previous launcher must have released it; an observed-chat start uses a fresh ID. This lets capture recover from an abandoned generation without treating an unstopped row alone as a live writer. See [history storage](../../decisions/history-storage.md#d-abandoned-generation-closure).

Conversation lineage and original creation time belong to the chat, not each launch generation; continuation preserves them. See [state decisions](../../decisions/state.md#d-lineage-conversation-identity).
