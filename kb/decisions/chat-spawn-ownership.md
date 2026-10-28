# Decisions: Chat→Spawn Ownership

Which spawn a chat ID names. A chat (`cN`) is one conversation, run across one
or more session generations; a spawn (`pN`) is one harness process run. Several
commands take either ID, so every one of them needs the same answer to "which
spawn is this chat?". This page records that answer and the write-side rules that
keep it cheap and true. How a typed reference reaches this question at all is in
[ref classification](ref-classification.md); how `spawn message` delivers once it
has a target is in [primary inject](primary-inject.md).

---

## D-chat-names-own-session: a chat ID names that chat's own session, in every command (2026-10) {#d-chat-names-own-session}

**Status:** the rule itself is the owner's ruling (an invariant). The reader's
shape, its single implementation, and the deleted fallback are the tech lead's
current choices, taken from the closing reviews. Shipped in meridian-cli v0.16.4
(#801) as `release:patch`: asked whether the old meaning was a contract, the owner
answered "why would the meaning be correct...?", so changing it fixed a bug. Point 2
is revised on branch `refactor/ref-resolution` (#800, not merged as of
2026-10-28).

### Problem

The shared ref resolver turned a chat ID into the newest spawn in that chat's family, children included. A
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
     has no fallback. Every chat→spawn path reaches it through
     `ops/locate.py` `chat_spawn`.
   - *Who is live in it now?* A running run whose `current_chat_id` is the chat
     (one that moved in through native `/resume`), else the owner if it is still
     running and still in that chat. On `main` after #801 only `spawn message`
     asks this. On `refactor/ref-resolution` every spawn command asks it first
     and falls back to the owner, so `show`, `cancel` and `message` agree after a
     takeover; `spawn message` still refuses a chat with no live run
     ([D-locate-projections](ref-classification.md#d-locate-projections)).
3. **A legacy chat whose launch failed before its spawn was linked no longer
   resolves by chat ID.** On the dev project that is one chat (c7494). Its spawn
   ID still works.

### Rejected

| Alternative | Why rejected |
|---|---|
| A message-only resolver beside the family rule | Two answers to one question; the family rule was the bug everywhere it applied |
| One resolver answering both questions for every command | Read every spawn's state on each chat lookup: about 0.8 s per chat ref, one corrupt spawn file broke every chat ref in the project, and `--fork` mixed spawns |
| A fallback that scans spawn rows when no generation is linked | About 0.9 s on every miss, measured on the dev project (about 2,750 spawns, 7,152 unlinked chats), to rescue one legacy chat |
| A fallback through the history index | Puts a rebuildable projection on the resolution path, so a damaged index changes the answer. The occupant query later became the one index read, because it only nominates a live run and falls back to the owner when the index can't answer ([D-locate-projections](ref-classification.md#d-locate-projections)) |

**Revisit when:** chats gain a first-class owner field.

**Provenance:** `chat:c9677` (owner rulings, lead decisions); `chat:c9695` (point 2
revision);
reviews `spawn:p9988`, `spawn:p9991`, `spawn:p9992`, `spawn:p9996`,
`spawn:p9997`, `spawn:p9999`, `spawn:p10000`, `spawn:p10002`; trigger sessions
`chat:c9618` (meridian-cli) and meridian-flow `chat:c14995`.

---

## D-link-at-start: every session generation names its spawn in its start event (2026-10) {#d-link-at-start}

**Status:** current choice (tech lead, on the closing whole-system review
`spawn:p9999`). Shipped in meridian-cli v0.16.4 (#801).

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

---

## D-owner-on-start: a run owns a chat only once its session starts (2026-10) {#d-owner-on-start}

**Status:** current choice (tech lead, after three review rounds on the same
concern, `chat:c9695`). Built on meridian-cli branch `refactor/ref-resolution`
(#800); not merged as of 2026-10-28.

### Problem

Continue moved a chat's owner before the new run had started. A `--bg` continue
published its spawn as the chat's owner from the parent, before handing off to
the worker, so that a second continue in the hand-off window would see the chat
busy instead of launching beside it. Once #800 made a held chat refuse a continue
within seconds, refused and failed continues became common, and each one left the
chat pointing at a run that never ran. The first fix withdrew the link on each
failure exit; the next review found more exits. Separately, a spawn row was
reserved with its caller's chat as a placeholder until its session bound its own,
and `continue_chat_id` falls back to that field, so a continue refused before its
session started resumed or read the caller's conversation.

### Decision

1. **The session start event is the only writer of a chat's owner.** The
   pre-start publish and its withdrawal are deleted. Two racing continues no
   longer both run a harness because `start_session` takes the chat's lifetime
   lock: it waits at most two seconds for it, then refuses with "Chat cN is
   held". The loser fails in its worker, and only one harness runs, for racing
   and for back-to-back continues alike.
2. **An exact continue reserves its row with its own target chat**, so a continue
   refused before its session starts still names the chat it was continuing.
3. **The continue sweep keeps refusing a live owner, including one that ran
   `/clear`.** A run holds its entry chat's lock until it exits, even after
   `/clear` moved it to another chat, so letting that continue through would
   only turn a clear refusal into a wait on the lock. The refusal now says the
   owner has moved on.

The cost: during the hand-off window, `spawn show`, `wait`, `cancel` and
`message` by chat ID name the previous run, and a racer's refusal arrives from
its worker rather than the command that launched it. A fresh child's
placeholder chat is unchanged (meridian-cli #810): reserving it with no chat
needs every reader that groups children by `chat_id` to switch to
`owner_chat_id`.

### The lesson

The same lesson as [D-link-at-start](#d-link-at-start), from the other side.
Writing ownership before it was true forced every failure exit to undo it, and
each review found an exit that didn't. Writing it at the moment it becomes true
left nothing to undo; the lock that already guards the chat stops the race.

### Rejected

| Alternative | Why rejected |
|---|---|
| Keep the pre-start publish and withdraw it on failure | Every failure exit has to undo it; review found exits that didn't |
| Refuse a live owner only while it is still in the chat (narrow the sweep) | The owner keeps its entry chat lock after `/clear`; the continue would wait on the lock until the owner exits |

**Provenance:** `chat:c9695`; reviews `spawn:p10031`, `spawn:p10048`,
`spawn:p10052`; probes `spawn:p10033`, `spawn:p10049`, `spawn:p10053`; fixes
`spawn:p10036`, `spawn:p10050`, `spawn:p10055`.

## Related

- [Ref classification](ref-classification.md): how a typed reference becomes a
  spawn or a chat before this page's question is asked.
- [Session reference resolution](session-reference-resolution.md): recovering a
  ref's harness session ID once the spawn is known.
- [Primary inject](primary-inject.md#d-spawn-message-address): how a chat ID is
  addressed across projects.
- [Session state](../architecture/state-system/session-state.md): the session
  journal these start events live in.
