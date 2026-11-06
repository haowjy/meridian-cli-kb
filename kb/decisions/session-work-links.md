# Per-Execution Work and Checkout Links

Stable work and task links let a running execution follow work moves and checkout changes. The mechanism and lock order live in meridian-cli `src/meridian/lib/state/.context/CONTEXT.md` and `src/meridian/lib/launch/.context/CONTEXT.md`.

## Execution owns links {#d-execution-owns-links}

Each primary and spawn has its own chat links and attachment. Inheriting the parent's chat for a spawn can make a child work switch change the parent's attachment; concrete paths also go stale after a move.

## One projection per owner {#d-one-projection-per-owner}

Work-item links project work-store state; chat links project session state and point through the item links. Neither store reads a link as authority or holds its lock while taking the other's. A direct link joining both stores required bulk session rewrites and created deadlocks and partial updates. See [the state-design lesson](../lessons/state-design-lessons.md#a-link-that-spans-two-stores).

## Immediate moves {#d-done-moves-at-once}

Done and reopen move the directory immediately; links follow. When both source and destination exist, the move refuses without deleting either copy. Deferring the move until the last spawn ended would require another durable state and recovery sweep.

## Ended chats {#d-ended-chats-keep-their-item}

Rename updates live attachments, not ended chats. A reference to a missing item resolves to no item with a warning; only an explicit work selection can create it. This avoids bulk history rewrites and accidental resurrection of renamed or deleted work.

## Checkout pins {#d-checkout-pin-sources}

Pin checkouts only for facts a store cannot recompute: an explicit task directory, an outside-project launch, or an inherited execution pin. A managed parent's `MERIDIAN_TASK_DIR` is a projection, not evidence of a pin; saving it as one would stop an unpinned child following later work changes.

## Current work defaults {#d-work-commands-default-current}

Work commands default to the current attachment where possible, so an ID printed at launch cannot become stale after a switch. `current` is reserved as a selector.
