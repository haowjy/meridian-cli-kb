# Work items and checkout links

A named work item outlives sessions; each execution (primary or spawn) attaches to one through its own stable links, so a running agent follows the item when it moves and follows its checkout when that changes. Mechanism and lock order: the source checkout's `src/meridian/lib/state/.context/CONTEXT.md` and `src/meridian/lib/launch/.context/CONTEXT.md`. Context configuration: `docs/configuration/context.md`.

Context names working-memory paths; [workspace](../permissions/workspace.md) grants filesystem access. They serve different purposes even when they name the same directory.

## Each execution owns its links

Each primary and spawn has its own chat links and attachment. If a spawn inherited its parent's chat, a work switch in the child would change the parent's attachment, and concrete paths go stale after a move.

## One projection per owner

Work-item links project work-store state; chat links project session state and point through the item links. Neither store reads a link as authority, and neither holds its lock while taking the other's. A direct link joining both stores made each store's writers responsible for the other's state: it required bulk session rewrites and produced deadlocks and partial updates. A derived link is never read back as authoritative input.

## Location is the status

Whether a work directory is active or archived is decided by where it is, so an interrupted move can be retried without trusting stale metadata. Done and reopen move the directory immediately and the links follow. When both source and destination exist, the move refuses and deletes neither. Deferring the move until the last spawn ended would need another durable state and a recovery sweep.

## Renames and missing items

Rename updates live attachments, not ended chats. A reference to a missing item resolves to no item with a warning; only an explicit work selection can create one. This avoids bulk history rewrites and resurrecting renamed or deleted work.

## Activation means someone chose the item

A user or agent choosing a work item, including a primary attaching it, activates it. Inheritance, rename maintenance and clearing do not: counting inheritance would run the default blocking autosync for every child spawn, and work events have no spawn-kind filter ([hooks](../hooks.md)).

## Checkout pins

A checkout is pinned only for facts a store can't recompute: an explicit task directory, a launch from outside the project, or a pin inherited from the parent execution. A managed parent's `MERIDIAN_TASK_DIR` is a projection, not evidence of a pin; saving it as one would stop an unpinned child following later work changes.

A work item's task directory that no longer exists falls back with a warning, and the read does not rewrite the item. An explicit task directory wins. An explicit work selection bounds the lookup: falling back to the ambient attachment would make the chosen checkout depend on unrelated state.

## Commands default to the current attachment

Work commands default to the current attachment, so an ID printed at launch can't go stale after a switch. `current` is reserved as a selector.

## Compatibility

Work metadata travels between builds. Readers preserve unknown fields and refuse to mutate a newer format, so information written by a newer build survives ([compatibility](compatibility.md)).
