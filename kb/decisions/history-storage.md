# History Storage

Native capture is a complete post-stop observation associated with the exact record, not a guarantee of the transcript at the instant of launch completion: a continuation can overtake observation. Codex forks can depend on ancestor rollouts, and an OpenCode transcript snapshot is not a backup of its whole native database. See [portable history](../architecture/state-system/portable-history.md).

### D-history-file-authority: readable transcript files are authoritative; SQLite is a derived index (2026-09-11) {#d-history-file-authority}

Files and verified archives retain transcript content and identity; SQLite indexes are disposable projections. This keeps history transferable without a database or running service and makes repair a rebuild. Native transcripts are the authority for bound chats; see [native-only history](native-only-history.md).

### D-history-reclaim-gate: hash under shared capture, revalidate and retire briefly under exclusive ownership (2026-09-14)

Capture hashes under shared ownership, then revalidates and retires briefly under exclusive ownership. Checksums are necessary, but repeating a full hash and recursive deletion under the global gate would serialize unrelated writers. See [portable history](../architecture/state-system/portable-history.md).

### D-history-index-initialization: initialization has its own bounded gate (2026-09-14) {#d-history-index-initialization}

A cold index build has a separate bounded ownership phase from ordinary queries. Failures of an owned build latch; contention, cancellation and a peer building do not. A single quarantined non-dogfood row can still fail the index closed; the policy is open in [#530](https://github.com/haowjy/meridian-cli/issues/530).

### D-history-index-readers-never-wait-on-session-writers: shorten the holder, don't widen the wait (2026-10-15) {#d-history-index-readers-never-wait-on-session-writers}

A busy index must not answer from a stale projection that could omit a new spawn. Shortening lock holders is preferable to increasing the reader wait. On a 46.6 MB journal, append lock time fell from 1,463 ms to about 16 ms; reader P95 under two writers fell from 2,001 ms to 101 ms. Lock-free projection depends on newline-committed appends and atomic replacement of the only rewriter. See [session state](../architecture/state-system/session-state.md) and [per-operation cost](../architecture/per-operation-cost.md).

### D-abandoned-generation-closure: a later start of the same chat closes earlier generations; capture defers to real owners (2026-10-15) {#d-abandoned-generation-closure}

A later accepted start closes earlier unstopped generations of the same chat in the fold. Without that repair, an abandoned generation can block capture forever after a crash. Liveness comes from actual owners; capture defers while one can write and retries after it stops. This depends on start writers holding the chat lock. See [abandoned generations](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations).

### D-history-index-schema-namespace: each index schema owns its own files; never migrate in place (2026-09-26) {#d-history-index-schema-namespace}

Each index schema and independently versioned input has its own files and is rebuilt from authority rather than migrated in place. A still-running older process cannot understand a newly migrated index, and sharing its queues or failure latch would let builds interfere. Old-build archive changes may still require an index rebuild. See [compatibility](compatibility-versioning.md).

### D-history-portable-digest-stored-json: verify archive digests over the stored record JSON (2026-09-26) {#d-history-portable-digest-stored-json}

Verify the digest over the stored record facts, excluding local-only fields, rather than reserializing through current models. New model defaults otherwise change the digest of an unchanged old archive. Member hashes alone do not protect record facts. The repeat-restore compatibility issue is tracked in [#537](https://github.com/haowjy/meridian-cli/issues/537).

## Identity and restore boundaries

Portable history identity is distinct from local aliases and process ownership. Restore publishes historical, inert records; a missing selected snapshot must not silently fall back to an older one. Only a verified copy of the same digest can substitute.
