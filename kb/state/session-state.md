# Session state

A chat's native key names its harness, store and session; how it is bound and why it never silently changes is in [native session binding](native-session-binding.md). Contract: the source checkout's `src/meridian/lib/state/.context/CONTEXT.md`.

## `sessions.jsonl` is one global journal

All chats in a runtime append to one authoritative session journal; the SQLite indexes are disposable projections. Scoped reads avoid decoding unrelated chats but still read the whole file, so their I/O grows with project history ([per-operation cost](per-operation-cost.md)).

- **A prefilter may over-select, never under-select.** It can include extra rows but must not drop a row the decoded predicate selects. A multi-chat read is one pass: one scan per chat made bulk operations scale with chat count.
- **Shorten the holder, don't widen the wait.** A busy index must not answer from a stale projection that could omit a new spawn, so the fix for contention is shorter lock holds by writers, not a longer reader wait. Lock-free projection depends on newline-committed appends and atomic replacement by the only rewriter.

### Per-chat journals: chosen, not built

Splitting the journal per chat is the accepted fix for global scans and writer contention ([#418](https://github.com/haowjy/meridian-cli/issues/418)). The split must absorb writes from older builds without losing or duplicating events. It keeps append-only journals, because consumers read events; per-chat state files would add a derived schema and a read-modify-write on every update. A global offset index would keep the shared writer lock. Absorbing old writes on every read would add locking and deadlock risk, and ignoring them loses facts.

## Lineage belongs to the chat

Fork and context ancestry belong to the chat and survive continuation. A disposable index cannot be the only copy of those edges.

## A later start closes abandoned generations

A later accepted start of the same chat closes earlier unstopped generations in the replay fold. Without that, a generation abandoned by a crash blocks history capture forever. Capture takes liveness from actual owners instead: it defers while one can still write and retries when it stops. This depends on start writers holding the chat lock, so a new start proves the previous launcher released it. The fix repaired the fold rather than relaxing capture's check: a relaxed check hides wrong state while other readers still depend on it.
