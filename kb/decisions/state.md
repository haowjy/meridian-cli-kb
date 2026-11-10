# State decisions

The source checkout's [src/meridian/lib/state/AGENTS.md](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/state/AGENTS.md) and [state context](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/state/.context/CONTEXT.md) own state contracts. See [spawn state](../architecture/state-system/spawn-state.md), [session state](../architecture/state-system/session-state.md), and [history storage](history-storage.md).

## Project identity

Committed identity survives moves and is shared by clones; ad-hoc identity derives from the canonical path without writing into a scratch directory. Searching ancestors for an identity risks silently joining the wrong project, while using paths for every project would split clones and break moves. Initializing a scratch directory starts a new identity; adopting its old state would require coordinating live sessions, pending operations and crash recovery. See [project identity](../architecture/state-system/project-identity.md).

## Session journals

The global session journal remains authoritative. A per-chat split would reduce global scans and writer contention, but must account for writes from older builds without losing or duplicating events. See [session state](../architecture/state-system/session-state.md) and [the tracked migration](https://github.com/haowjy/meridian-cli/issues/418). 

The proposed split retains append-only journals because consumers read events; per-chat state files would add a derived schema and read-modify-write on every update. A global offset index would keep the shared writer lock. Absorbing old writes on every read would add locking and deadlock risk; ignoring them loses facts. Pruning and the archive's full scan remain outside the proposal.

## JSONL recovery

A newline commits a row. A valid final object without one stays invisible until a later append repairs it; showing it sooner would expose an uncommitted write. Repair preserves a complete object and discards a torn one. It replaces the inode rather than truncating in place: an unlocked buffered reader could otherwise combine old and new bytes into a fabricated event. See the [state context](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/state/.context/CONTEXT.md).

## Session reads

A prefilter may include extra rows but must not discard rows the decoded predicate selects. Multi-chat reads scan once: scanning once per chat makes bulk operations scale poorly.

## Conversation lineage

Fork and context ancestry belong to the chat and survive continuation. A disposable index cannot be the sole copy of those edges; see [session state](../architecture/state-system/session-state.md).

## Locking

Unlinking a held lock can split exclusion across two inodes. The validated lock removal contract is in the source checkout's [src/meridian/lib/state/AGENTS.md](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/state/AGENTS.md). A project-lifetime gate outside the deletable root lets pruning exclude active sessions without enumerating their locks.

Fixed lock stripes were rejected: collisions undermine reentrant exclusion, and sharing a stripe would make a session lock's availability cease to identify that session's liveness. Store mutations share lock, re-read, mutate and atomic-write semantics, but a generic store framework would have to parameterize their different paths, codecs and failure handling.

## Reading old state

Known legacy spawn rows upgrade at the read boundary; conflicting or unknown facts are quarantined rather than guessed. Treating an unreadable row as failed would invent an outcome, while discarding all legacy rows would hide existing history. See [spawn state](../architecture/state-system/spawn-state.md) and the source checkout's [legacy codec](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/state/spawn/legacy.py).
