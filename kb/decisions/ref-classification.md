# Classifying and Locating References

## Classification by shape

A reference's shape determines whether it names a spawn, chat, selector, or native session. A missing chat or spawn is an error, not an untracked native session: otherwise a mistyped chat ID could resume a different conversation. Untracked native IDs are accepted only where operating on a session Meridian never saw is meaningful. The grammar and command behavior are in meridian-cli `src/meridian/lib/ops/.context/CONTEXT.md` and `docs/commands.md`.

## Command projections

A spawn reference names that exact spawn, even if another run later occupies its chat. A chat reference names the live occupant for spawn actions, falling back to its owner when the occupant index cannot answer. Following native `/resume` is best effort. Reading may use historical fallbacks that resuming must refuse: reading an old transcript is less risky than resuming the wrong session.

`--from` a chat uses that chat's transcript; a spawn reference uses its current chat. After `/clear` those can differ. See [chat ownership](chat-spawn-ownership.md) and [native session identity](native-session-identity.md).
