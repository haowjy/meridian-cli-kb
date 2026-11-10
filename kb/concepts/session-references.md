# Spawn and chat references

A reference's shape decides whether it names a spawn (`pN`), a chat (`cN`), a selector or a native session ID. The grammar and per-command behavior are in meridian-cli `src/meridian/lib/ops/.context/CONTEXT.md` and `docs/commands.md`.

## A missing chat or spawn is an error

A missing tracked ID never falls through to an untracked native session: otherwise a mistyped chat ID could resume a different conversation. Untracked native IDs are accepted only where operating on a session Meridian never saw makes sense. Tracked references need an exact recorded binding ([native session binding](../architecture/native-session-binding.md)).

## A spawn names itself; a chat names its occupant

A spawn reference names that exact spawn, even when another run later occupies its chat. For spawn actions, a chat reference names the run currently occupying the chat, otherwise its own latest linked session, never a child. Resolving to the newest spawn in the chat's family could make `cancel` act on an unrelated child. Following a native `/resume` is best effort.

`--from` a chat uses that chat's transcript; `--from` a spawn uses the spawn's current chat. After `/clear` the two can differ.

Reads may use historical fallbacks that resuming refuses: showing an old transcript is less risky than resuming the wrong session.

## Ownership starts with the session

Each session generation links to its spawn when it starts, and a run becomes the chat's owner only then: publishing ownership earlier would make every failed launch path undo it. The chat lifetime lock prevents simultaneous continues, and until the new session starts a chat reference still names the previous owner. An exact continue reserves its row against its target chat, so an early failure doesn't name the caller's conversation.

Linking at start replaced reader-side scans for missing links, which cost about 0.9 s per miss on a project with 2,750 spawns. Older chats that were never linked stay addressable by spawn ID.
