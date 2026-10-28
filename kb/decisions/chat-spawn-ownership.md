# Decisions: Chat→Spawn Ownership

Which spawn a chat ID names. A chat (`cN`) is one conversation, run across one
or more session generations; a spawn (`pN`) is one harness process run. Several
commands take either ID, so every one of them needs the same answer to "which
spawn is this chat?". This page records that answer and the write-side rule that
keeps it cheap. How `spawn message` delivers once it has a target is in
[primary inject](primary-inject.md).

---

## D-chat-names-own-session: a chat ID names that chat's own session, in every command (2026-10) {#d-chat-names-own-session}

**Status:** the rule itself is the owner's ruling (an invariant). The reader's
shape, its single implementation, and the deleted fallback are the tech lead's
current choices, taken from the closing reviews. Built on meridian-cli branch
`feat/spawn-message-peers` (PR not yet opened); not on `main`.

### Problem

The shared ref resolver (`resolve_spawn_ref` in `lib/ops/reference.py`) turned a
chat ID into the newest spawn in that chat's family, children included. A
session that launches a coder is the newest-spawn holder for its own chat only
until the coder starts. After that, `meridian spawn show c9677` returned p9987,
the coder, not p9985, the session the user named. `cancel`, `wait`, `children`
and `--continue` used the same rule, so `spawn cancel cN` could kill a child.

The bug surfaced while making `spawn message` accept chat IDs. The owner asked
for the generic framework rather than a message-only resolver ("spawn message
should resolve it to the chat's running session... don't we have a generic
framework to do this..."). The lead first changed only the message path and
proposed filing the other commands as a follow-up issue. The owner ruled the
family rule wrong everywhere: "still resolve a chat ID to the newest spawn in
that chat's family, children included - wtf this is bad???"

### Decision

1. **A chat ID never names a child spawn.** It names the chat's own session in
   `spawn show`, `status`, `children`, `wait`, `cancel`, `done`, `rearm`,
   `report`, `message`, `--continue`, `--fork` and `--from`. This is the owner's
   ruling; changing it needs the owner.
2. **Two questions, two owners.**
   - *Whose chat is this?* `session_identity.chat_owner_spawn` answers it alone. It
     walks the chat's session generations from newest to oldest and returns the
     first generation's `spawn_id` whose spawn row exists. It reads no index and
     has no fallback. Every chat→spawn path calls it: `resolve_spawn_ref`,
     `_resolve_chat_reference`, and `--from`'s primary selection.
   - *Who is live in it now?* Only `spawn message` asks. It prefers a running
     run whose `current_chat_id` is the chat (a run that adopted it through
     native `/resume`), then the owner if the owner is still running and still in
     that chat. A message has to reach a live process; `show` and `cancel` name
     the chat's owner whether or not it runs.
3. **A legacy chat whose launch failed before its spawn was linked no longer
   resolves by chat ID.** On the dev project that is one chat (c7494). Its spawn
   ID still works.

Known gap: when a second run later moves into an existing chat, the chat's start
events still name the first run. `show` and `cancel` then name that first run, while
`spawn message` reaches the live one (meridian-cli #800).

### Rejected

| Alternative | Why rejected |
|---|---|
| A message-only resolver beside the family rule | Two answers to one question; the family rule was the bug everywhere it applied |
| One resolver answering both questions for every command | Read every spawn's state on each chat lookup: about 0.8 s per chat ref, one corrupt spawn file broke every chat ref in the project, and `--fork` mixed spawns |
| A fallback that scans spawn rows when no generation is linked | About 0.9 s on every miss, measured on the dev project (about 2,750 spawns, 7,152 unlinked chats), to rescue one legacy chat |
| A fallback through the history index | Puts a rebuildable projection on the resolution path, so a damaged index changes the answer |

**Revisit when:** a run can hold a chat it did not start (re-observation, #800),
or chats gain a first-class owner field.

**Provenance:** `chat:c9677` (owner rulings, lead decisions);
reviews `spawn:p9988`, `spawn:p9991`, `spawn:p9992`, `spawn:p9996`,
`spawn:p9997`, `spawn:p9999`, `spawn:p10000`, `spawn:p10002`; trigger sessions
`chat:c9618` (meridian-cli) and meridian-flow `chat:c14995`.

---

## D-link-at-start: every session generation names its spawn in its start event (2026-10) {#d-link-at-start}

**Status:** current choice (tech lead, on the closing whole-system review
`spawn:p9999`). Built on meridian-cli branch `feat/spawn-message-peers`; not on
`main`.

### Problem

`chat_owner_spawn` can be a plain record walk only if the records carry the link.
Before this change they often didn't. Children wrote their spawn row first and
linked at start, but the primary runner needed the chat before it could reserve
its spawn, so it linked the chat with a later update event. A chat created by
`/clear` was never linked at all. Each reader rebuilt the missing link its own way.

Five review rounds each found a resolver bug, and each reader-side fix traded one
cost for another: a full spawn-state scan was slow and failed project-wide on one
corrupt file; an index read was fast but wrong when the index was damaged; a
spawn-row fallback was robust but cost about a second on every miss. The closing
review traced all of them to the writers.

### Decision

1. **Writers link at start.**
   - The primary runner reserves its spawn ID (`spawn_store.reserve_spawn_id`)
     before `session_scope`, and passes that ID to both the session start event
     and the `SpawnReservation`.
   - Children write the spawn row first, as before.
   - A chat created by `/clear` names the observing run in its start event.
2. **Readers do not rebuild the link.** No scan, no index, no spawn-row search. A
   link to a row that was never written (a crash between reserving the ID and
   writing the row) is skipped, and the walk moves to the previous generation.
3. **`history_id` still arrives by update.** The primary runner's post-start
   `update_session_spawn_id` stays: it is the only writer of the session's
   `history_id`, which exists only once the spawn row does. The session listing
   uses it to hide archived primaries. Only the entry generation shares the
   spawn's `history_id`; a generation created by observation keeps its own, so a
   reader matching generations to a spawn checks both.

Chats written by older builds keep their update-event links and resolve. Their
`/clear` chats stay unlinked and resolve only by spawn ID; there is no repair
step. A start event carrying `spawn_id` is valid state for older builds, which
already read the field on update events.

### The lesson

When readers keep disagreeing about a fact, check who writes it. Each reader-side
fallback for a missing link hit the same speed-versus-robustness trade-off, and
no fallback escaped it. Writing the link when the generation starts removed the
trade-off instead of choosing a side.

### Rejected

| Alternative | Why rejected |
|---|---|
| More reader-side fallbacks | Each traded speed against robustness to damaged state (see Problem) |
| Deleting the post-start update along with the late link | It also carried `history_id`; removing it broke hiding archived primaries in the session listing |

**Provenance:** `chat:c9677`; closing reviews `spawn:p9999` (whole-system) and
`spawn:p10000` (observed-chat link); alignment review `spawn:p10002`.

## Related

- [Session reference resolution](session-reference-resolution.md): recovering a
  ref's harness session ID once the spawn is known.
- [Primary inject](primary-inject.md#d-spawn-message-address): how a chat ID is
  addressed across projects.
- [Session state](../architecture/state-system/session-state.md): the session
  journal these start events live in.
