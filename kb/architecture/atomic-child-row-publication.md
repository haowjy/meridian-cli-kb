# Child rows publish as complete directories

The target publication point for a new spawn row is one same-filesystem
directory replacement: build the complete row beneath
`spawns/.staging/<unique>/`, sync it, then replace that directory into
`spawns/pN/`. The protocol has been proven against process crashes and concurrent
publishers on the tested Linux/POSIX filesystem, but it is not implemented in
the current checkout and is not yet a cross-platform guarantee.

```mermaid
flowchart LR
    A[Allocate pN] --> S[Create .staging/unique]
    S --> P[Write and fsync prompt]
    P --> R[Write and fsync state]
    R --> D[Fsync stage directory]
    D --> X[os.replace to spawns/pN]
    X --> F[Fsync spawns directory]
```

Readers must see either no final row or the complete prompt/state pair. A stage
directory directly beneath `spawns/` is unsafe: current canonical and Pi readers
can admit a state-bearing sibling regardless of its staging-looking name. The
non-row `.staging/` container is therefore part of the protocol, not a naming
preference.

## Evidence and current divergence

Crash injection stopped and killed the publisher after stage creation, prompt
write, state write, and final replace. Before replace, neither canonical
`scan_spawn_ids()` nor a real `PiDiskWatcher` admitted the nested stage. After
replace, both observed a complete typed row.

Eight concurrent publishers racing both readers produced no partial final row,
no admitted staging name, and no unresolved Pi candidate across 2,832 samples.
The current `start_spawn()` order—create the final directory, write the prompt,
then write state—produced 1,738 observations of a final `pN/` without
`state.json`. Pi's bounded allocation uncertainty exists to prevent early
completion during that window; it can be removed only after publication changes
and a zero-divergence soak.

## Cutover requirements

The publisher must use collision-safe ID allocation and unique stage names,
sync each file and the stage/publication directories, keep stage and destination
on the same filesystem, and make abandoned `.staging/*` recovery explicit.
Readers must continue to treat allocation uncertainty as fail-closed until the
new protocol is active and compared in production.

The existing evidence does **not** establish:

- Windows same-volume directory replacement or open-handle behavior;
- survival under real power loss or every filesystem/storage stack;
- same-destination publication collision behavior.

These are release gates, not reasons to weaken the nested-stage target. If a
platform cannot support the replacement contract, it needs an explicit
parent-scoped allocation-intent record with bounded resolution; permanent
newer-directory inference is not the fallback design.

## Provenance

- `work:drain-convergence`
- `spawn:p5096`
