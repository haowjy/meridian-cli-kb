# Portable history

Readable files and verified immutable ZIP archives are the authority for history; SQLite indexes are rebuildable projections. That keeps history transferable without a database or a running service, and makes repair a rebuild. Contract: the source checkout's `src/meridian/lib/state/.context/history.md`; user guide: `docs/history.md`.

## Identity is not ownership

A history UUID identifies a transcript across local alias changes; PIDs, leases and scopes describe runtime ownership. Restore keeps ownership facts as provenance and never turns them into authority to signal or resume a process. Restore publishes inert historical records.

A selected snapshot that is offline never silently falls back to older content under the same identity; only an independently verified copy with the same digest can substitute. Publication keeps loose sources until verification, and restore refuses conflicting content and leaves the ZIP intact.

## Archive capture

Capture selects the bound native source of the exact run generation, never runner output or a newer session ([native transcript reads](../native-transcript-reads.md)). It runs after stop and observes the bytes as they are then, so a continuation can get in first; it is not a reconstruction of the transcript at the instant the run ended. Codex forks can depend on ancestor rollouts, and an OpenCode transcript snapshot is not a backup of the whole OpenCode database. A transcript known to be incomplete can't authorize reclaiming the source.

- **Capture defers to a live owner.** A live owner, observer, exact-generation lease or unreleased serving scope can still write the transcript, so capture defers, rechecks before publishing, and retries when the owner stops. An unstopped generation alone is not proof of a live writer, since a crash can leave it unstopped ([session state](session-state.md#a-later-start-closes-abandoned-generations)).
- **Hash under shared ownership, retire briefly under exclusive.** Repeating a full hash and recursive delete under the global gate would serialize unrelated writers.
- **Digests cover the stored record JSON**, excluding local-only fields, so new model defaults can't change the digest of an unchanged old archive. Member hashes alone don't protect the record facts.

## Indexes

- **Each index schema owns its own files.** An index is rebuilt from authority, never migrated in place: an older build still running can't read a migrated index, and sharing its queues or failure latch would let the two builds' rebuilds interfere. The file name carries every independently versioned input format, so a change to any of them forces a rebuild ([compatibility](../../concepts/compatibility.md)).
- **A cold build has its own bounded gate.** Building is a separate ownership phase from queries. A failure of an owned build latches; contention, cancellation and a peer's build do not.
- **A caller with a deadline builds before starting its clock.** Otherwise a cold build can eat the query deadline and make a live child read as unknown ([test determinism](../../codebase/test-determinism.md#cold-index-builds-inside-a-test-deadline-a-flake-class)). Contention is not an empty result, and corrupt or incompatible state still needs explicit repair.

## Open

- A single quarantined row can still fail the index closed ([#530](https://github.com/haowjy/meridian-cli/issues/530)).
- Restore metadata hashes still depend on the reader's models, so restoring the same archive twice across a model change can disagree ([#537](https://github.com/haowjy/meridian-cli/issues/537)).
