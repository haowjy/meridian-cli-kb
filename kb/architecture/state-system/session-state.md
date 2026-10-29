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
index. Since #651 (v0.10.16) every per-operation read and every append goes through
one keyed seam in `session_store.py`
([D-session-read-seam](../../decisions/state.md#d-session-read-seam)):
- **Reads:** `chats_rows(root, chats)` reads the selected chats in one pass and
  returns `ChatRows`, a per-chat mapping plus `.ordered`, the same rows in global
  log order. `chat_rows`, `chat_record` and `chat_generations` are its single-chat
  cases; `chats_rows(root, None)` is the explicit full snapshot. `native_chats` and
  `native_observations` select by native key.
- **Selection:** a byte prefilter nominates rows that might name a selected chat,
  and only those are decoded. The decoded check decides. Every read still scans
  every byte of the file.
- **Appends:** `append_chat_rows` and `append_native_rows`. `SessionBindings.commit`
  publishes through `append_chat_rows` as one durable batch.
- **Format version:** rows carry `v` (1; absent reads as 1). A read skips a row
  with an unsupported `v` and warns. Paths that read rows to decide what to append
  (starting a chat, binding a native session, deduplicating model records, adopting
  an observed native session, repair binds) pass `require_supported` and refuse with
  `UnsupportedSessionFormat` when an unsupported row is in their chat or native
  scope. Blind appends read nothing and check nothing
  ([D-check-before-deciding](../../decisions/compatibility-versioning.md#d-check-before-deciding--version-checks-guard-decisions-and-mutations-not-blind-appends)).
- **Multi-chat callers make one call**, never a loop over single-chat reads, and
  never sort rows by timestamp.

`list_all_session_records`, `list_session_generations` and legacy import's raw reads
still fold everything outside the seam. A `sessions-index.sqlite3` or
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
[per-operation cost](../per-operation-cost.md#1-session-state-is-one-global-replay-log-418).
The automatic archiver still folds once per candidate.

A scoped `SessionBindings` (`scope` is a frozenset of chats) holds only those chats'
history. Binding any other chat raises `ChatOutsideBindingScope`, and cross-chat
uniqueness checks need the full snapshot (`scope=None`). A corrupt row poisons only
its own chat: other chats still read and bind, while reading that chat, or the full
list, raises.

**The accepted plan, B2 not built:** per-chat journals,
`session-state/chats/<cN>.jsonl`, each the only authority for its chat, plus small
per-native-ID files for the facts that cross chats. A one-time cutover splits the
old log, and rows that older builds keep appending are absorbed at the next
command. B1 (the seam) shipped in #651. B2 swaps the seam's bodies and waits on the
owner's open design questions
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
  making the index follow the accepted generation is B2's change, with its own bump.
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

*Shipped in PR #631 (merge `8ecd44989`, 2026-10-16). Before it, an abandoned
generation stayed open forever.*

Each launch of a chat opens a **session generation**: a start event with its own
`session_instance_id`, closed by a stop event that names the same generation.
`session_fold.py` applies a stop only to the generation it names. A launcher
writes its stop in `session_scope`'s `finally`. If the launcher dies before
that block runs (SIGKILL, a default-action SIGTERM, an OOM-driven scope stop),
nothing stops its generation.

`cleanup_stale_sessions` normally repairs that: it takes the dead owner's chat
lock and appends the stop. But it skips a chat whose lock is held. When the user
runs `--continue` right after the crash, the new launcher takes the lock first.
Its own stop later closes only its own generation. Neither `doctor`,
`session archive --apply` nor another `--continue` would ever stop the old one.
Before #631, capture treated an unstopped generation as a live writer, so that
native session's history could never be captured
([capture side](portable-history.md#capture-defers-to-a-live-owner-and-retries-when-it-stops)).

**The rule:** in the generation fold, a later accepted start of the same chat
closes every earlier unstopped generation of that chat, at the new start's
`started_at`. The fold keeps a set of open generation keys per chat, so each
start does constant work and doesn't scan the chat's history. A late real stop
for a closed generation still applies and records the real stop time. A start
rejected by `bind` (`Conflict`) closes nothing.

**Why it is sound:** the rule depends on one fact. A later start of chat `cN` can
only exist once every earlier owner of `cN` has released the chat's lock. Both
production `SessionStartEvent` writers satisfy that:

- `session_store.start_session` covers primary starts, `--continue`, forks and
  spawns. It holds the chat's exclusive `flock` before it appends, and a start
  into an existing chat refuses while `live_observer_for_chat` finds a live
  external observer (`session_store.py`, verified on `main` by `spawn:p8255`;
  #616's `ChatBusy` reopen refusal never shipped). The lock is a
  per-handle `flock` and the acquirer checks the inode, so neither a second
  handle in the same process nor replacing the lock file gets around it.
- `_get_or_create_observed_chat_locked` writes a start only for a freshly
  reserved chat ID and appends its stop immediately. It never starts a later
  generation of an existing chat.

Restores and imports write `SessionHistoricalEvent`, not starts. The audit and
review that established this are `spawn:p8150` and `spawn:p8156`. The premise is
"lock released means the launcher is gone", not "the native writer is gone". A
harness child that outlives its launcher is fenced by the scope's process-group
liveness, as it already is for `cleanup_stale_sessions`.

**Projection:** the rule changes replay semantics, so the history index moves
from schema 7 to 10 on `main` and the first indexed read rebuilds `history-v10.sqlite3`
from `sessions.jsonl`. No authority is rewritten. Existing journals heal on
replay with no synthetic stop. Incremental catch-up rebuilds the touched chat's
open set from its stored rows. It matched a cold rebuild over 120 random event
streams.

**Keep the premise true.** A future start writer that doesn't hold the chat lock
would make this rule close a live generation. Rationale and
rejected alternatives:
[D-abandoned-generation-closure](../../decisions/history-storage.md#d-abandoned-generation-closure).

## What a chat keeps across generations

Most start-event fields describe one launch: settings, model selection, and the
tmux `session:window` captured at that launch (blank outside tmux). A few describe
the conversation and must survive `--continue`:

- the chat's original `created_at`;
- `forked_from_chat_id` + `forked_from_history_id`, set by `--fork`;
- `context_from_chat_id` + `context_from_history_id`, set by a primary's `--from`.

The fold keeps each pair from the first start that set it and gives every later
generation the same values; `_CHAT_LINEAGE_FIELDS` and `_CHAT_OWNED_FIELDS` in
`session_fold.py` are the one list. A new conversation-level fact belongs on that
list, not on `SessionSettings`, which continuation replaces. Older journals simply
lack the fields and read as "no parent".
Why: [D-lineage-conversation-identity](../../decisions/state.md#d-lineage-conversation-identity).

## Related Pages

- [../native-session-binding.md](../native-session-binding.md)
- [../../decisions/native-session-identity.md](../../decisions/native-session-identity.md)
- [State system overview](overview.md) — state roots and subsystem map
- [Spawn state](spawn-state.md) — per-spawn authority and artifact lifetime
- [Per-operation cost](../per-operation-cost.md) — measured history-proportional
  readers and the constant-time acceptance bar
- [Contention and cost diagnosis](../../lessons/contention-diagnosis.md) — how the
  session-writer contention was identified
- [Spawn wait barrier](../../concepts/spawn-wait-barrier.md) — the wait path that
  consumes history-index discovery
- [Portable history](portable-history.md#capture-defers-to-a-live-owner-and-retries-when-it-stops) — cross-record discovery, exact identities, ZIP retention, inert restore, and live-owner capture deferral
- [History-storage decision](../../decisions/history-storage.md#d-abandoned-generation-closure) — why the fold closes abandoned generations before capture trusts liveness signals
- [State-design lesson](../../lessons/state-design-lessons.md#repair-the-state-before-relaxing-the-check) — repair persisted state before relaxing a safety check
- [Crash troubleshooting](../../operations/troubleshooting.md#primary-crashed-with-no-exit-or-stop-recorded) — the crash path that leaves a generation unstopped
- [Derived status](derived-status.md) — lock-derived spawn status and the dead-launch writer
