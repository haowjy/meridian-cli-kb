# Session state

A chat's native key identifies its harness, store and session. Bindings are fill-only except for explicit relocation, so a conflict cannot silently redirect a conversation. See [native session identity](../../decisions/native-session-identity.md) and the source checkout's `src/meridian/lib/state/.context/CONTEXT.md`.

## `sessions.jsonl` is one global log

All chats in a runtime share the authoritative session journal. Scoped reads avoid decoding unrelated chats but still scan the log, so their I/O grows with project history; some reads also hold the journal lock. The SQLite indexes are disposable projections, not authority. See [per-operation cost](../per-operation-cost.md#session-state-is-one-global-replay-log).

## A later start closes abandoned generations

A later accepted start closes earlier unstopped generations in the replay fold. A new start requires the chat lock, so the previous launcher must have released it. This repairs abandoned generations without treating an unstopped row as a live writer. See [history storage](../../decisions/history-storage.md#a-later-start-of-the-same-chat-closes-earlier-generations-capture-defers-to-real-owners).
