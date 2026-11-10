# History Storage

Native capture is a complete post-stop observation associated with the exact record, not a guarantee of the transcript at the instant of launch completion: a continuation can overtake observation. Codex forks can depend on ancestor rollouts, and an OpenCode transcript snapshot is not a backup of its whole native database. See [portable history](../architecture/state-system/portable-history.md).

### Readable transcript files are authoritative; SQLite is a derived index

Files and verified archives retain transcript content and identity; SQLite indexes are disposable projections. This keeps history transferable without a database or running service and makes repair a rebuild. Native transcripts are the authority for bound chats; see [native-only history](native-only-history.md).

### Hash under shared capture, revalidate and retire briefly under exclusive ownership

Capture hashes under shared ownership, then revalidates and retires briefly under exclusive ownership. Checksums are necessary, but repeating a full hash and recursive deletion under the global gate would serialize unrelated writers. See [portable history](../architecture/state-system/portable-history.md).

### Initialization has its own bounded gate

A cold index build has a separate bounded ownership phase from ordinary queries. Failures of an owned build latch; contention, cancellation and a peer building do not. A single quarantined non-dogfood row can still fail the index closed; the policy is open in [#530](https://github.com/haowjy/meridian-cli/issues/530).

### Shorten the holder, don't widen the wait

A busy index must not answer from a stale projection that could omit a new spawn. Shortening lock holders is preferable to increasing the reader wait. On a 46.6 MB journal, append lock time fell from 1,463 ms to about 16 ms; reader P95 under two writers fell from 2,001 ms to 101 ms. Lock-free projection depends on newline-committed appends and atomic replacement of the only rewriter. See [session state](../architecture/state-system/session-state.md) and [per-operation cost](../architecture/per-operation-cost.md).

### A later start of the same chat closes earlier generations; capture defers to real owners

A later accepted start closes earlier unstopped generations of the same chat in the fold. Without that repair, an abandoned generation can block capture forever after a crash. Liveness comes from actual owners; capture defers while one can write and retries after it stops. This depends on start writers holding the chat lock. See [abandoned generations](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations).

### Each index schema owns its own files; never migrate in place

Each index schema and independently versioned input has its own files and is rebuilt from authority rather than migrated in place. A still-running older process cannot understand a newly migrated index, and sharing its queues or failure latch would let builds interfere. Old-build archive changes may still require an index rebuild. See [compatibility](compatibility-versioning.md).

### Verify archive digests over the stored record JSON

Verify the digest over the stored record facts, excluding local-only fields, rather than reserializing through current models. New model defaults otherwise change the digest of an unchanged old archive. Member hashes alone do not protect record facts. The repeat-restore compatibility issue is tracked in [#537](https://github.com/haowjy/meridian-cli/issues/537).

## Identity and restore boundaries

Portable history identity is distinct from local aliases and process ownership. Restore publishes historical, inert records; a missing selected snapshot must not silently fall back to an older one. Only a verified copy of the same digest can substitute.
