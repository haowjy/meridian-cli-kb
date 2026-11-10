# Project identity

Committed identity (`[project].id` in `meridian.toml`) survives moves and is shared by clones; an ad-hoc identity is derived from the canonical path and Meridian writes nothing into the directory. Reads never mint either kind. Contract: the source checkout's `src/meridian/lib/state/.context/CONTEXT.md`.

- **No ancestor search.** Searching parent directories for an identity risks silently joining the wrong project; deriving every identity from its path would split clones and break moves.
- **Initializing a scratch directory starts a new identity.** Adopting the ad-hoc state would mean coordinating its live sessions, pending operations and crash recovery across the switch. The global listing keeps the shadowed ad-hoc state discoverable so it can be pruned.
- **Global pruning is explicit.** Its activity check uses several signals: the hourly origin-file touch alone leaves a gap near its boundary.

Control sockets use a short per-user path for the same reason projects can move: runtime paths can exceed the Unix socket limit (`src/meridian/lib/streaming/.context/CONTEXT.md`).
