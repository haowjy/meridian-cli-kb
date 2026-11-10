# Chat and Spawn Ownership

## A chat names its own session

A chat reference names the run currently occupying that chat, otherwise its own latest linked session, never a child. Resolving to the newest spawn in the chat family could make a command such as cancel act on an unrelated child. See [reference classification](ref-classification.md#command-projections).

## Link at start

Each session generation links to its spawn at start. Reader-side scans for missing links traded robustness against latency: a scan on a miss took about 0.9 seconds on a project with 2,750 spawns and 7,152 unlinked chats. Older unlinked chats remain addressable by spawn ID.

## Ownership starts with the session

A run becomes the chat's owner when its session starts. Publishing ownership earlier requires every failed launch path to undo it. The chat lifetime lock prevents simultaneous continues; before the new session starts, a chat reference still names the previous owner. An exact continue reserves its row against its target chat, so an early failure does not name the caller's conversation.
