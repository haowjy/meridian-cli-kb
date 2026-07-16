# Child rows publish as complete directories

A new spawn row becomes visible through one same-filesystem directory
replacement. The publisher builds the complete row beneath
`spawns/.staging/<unique>/`, syncs it, then replaces that directory into
`spawns/pN/`.

```mermaid
flowchart LR
    A[Allocate pN] --> S[Create .staging/unique]
    S --> P[Write and fsync prompt]
    P --> R[Write and fsync state]
    R --> D[Fsync stage directory]
    D --> X[os.replace to spawns/pN]
    X --> F[Fsync spawns directory]
```

Readers see either no final row or the complete prompt/state pair. A stage
directory directly beneath `spawns/` would be unsafe because row scanners could
admit it; the non-row `.staging/` container is part of the protocol, not a
naming preference.

Publication runs under the spawn allocation/publication lock. Stage names are
unique, stage and destination share a filesystem, and collision checks occur
before replacement. Runtime-write startup garbage-collects abandoned
`.staging/*` entries under the same lock, so a crash before publication leaves
recoverable private debris rather than a partial row.

## Consequence for descendant evidence

Only a valid, published row with a reconciled `parent_id` relationship can
become a persisted-descendant blocker. Pi no longer scans raw spawn directories,
infers parentage from newer numeric IDs, or carries an allocation-uncertainty
barrier. `ReconciledDescendantEvidence` is the sole persisted-descendant
authority.

Atomic publication removes the partial-row visibility window; it does not make
every storage failure look like an empty tree. Store-wide read failure still
produces typed `unknown` and blocks completion.

## Durability boundary

The protocol and crash/concurrency probes establish the complete-row visibility
contract on the supported implementation path. Its fsync steps are deliberate:
file contents, the stage directory, and the publication directory must reach
their respective durability boundaries in order.

Platform filesystem semantics still matter. A platform that cannot provide
same-volume atomic directory replacement needs an explicit publication design;
raw newer-directory inference is not a fallback authority.

## Provenance

- `work:drain-convergence`
