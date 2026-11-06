# Spawn Identity and Launch Containment

A spawn has one public ID across continuations. Fork and `--from` create a new spawn. Individual launches remain internal, with separate artifacts and launch-pinned waits and cancellation. Cost and duration aggregate across launches. See [spawn launches](../architecture/state-system/spawn-launches.md) and [derived status](../architecture/state-system/derived-status.md).

## Continue reopens the spawn {#d-continue-reopens}

The launch lock serializes continuation and fences lifecycle writes; the previous launch's outcome and artifacts remain separate. Reopening without a fence would let a late writer from an older launch change the new launch. Cancel is pinned to the launch it observed, so a later continuation cannot be signalled by that request.

A dead launch with a durable report derives success even when cancellation was requested; otherwise cancellation wins over crash. Live runner outcome classification also considers the observed exit and explicit verdict. Selection for repair uses stored active state, while liveness uses the derived state: choosing only derived active rows can skip dead rows that still need finalization.

## One public identity {#d-spawn-identity-v2}

A separate public launch identity was rejected because it spread a second name through references, output and ownership while the lock can supply the needed fence. A launch count remains internal to disambiguate artifacts, delivery and cancellation. The runner holds the lock for its lifetime and only a holder publishes the terminal outcome.

### Launch lock and sweep {#pr-2-as-built-the-launch-lock-and-the-sweep}

Before another launch starts, its lock holder sweeps the prior launch's recorded scopes. The sweep is the containment backstop if the guardian dies or cannot terminate a process. Exact process birth checks matter because PIDs are reused; unknown identity is not proof that signalling is safe.

### Derived status and dead launch finalization {#pr-3-as-built-read_derived-and-one-dead-launch-finalize}

Readers derive liveness from the launch lock without persisting state. A repair or cancellation must claim that lock before writing a dead launch's outcome. This makes repair converge with the same outcome readers observed and prevents a late writer from affecting a new launch.

## Start gate and guardian {#d-start-gate-guardian}

A harness cannot execute until its process scope and verified birth are durable. A small POSIX guardian observes runner death and terminates the owned process group; a later birth-checked sweep is the backstop. A parent-death signal alone reaches only a direct child and misses binaries behind wrappers. On the root-only terminal path the guardian sends one TERM, not a delayed KILL that could hit a reused PID. Processes that leave the owned group can survive the sweep. See [process containment](../lessons/process-containment.md).

