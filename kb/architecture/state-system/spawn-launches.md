# Spawn identity and launches

A spawn has one public ID across continuations; fork and `--from` create a new spawn. Each continue reopens the same spawn as a new internal launch with its own files. Cost and duration aggregate across launches. Contract: the source checkout's `src/meridian/lib/state/AGENTS.md`.

## One public identity, launches stay internal

A separate public launch identity would spread a second name through references, output and ownership, and the launch lock already supplies the fence it would provide. The launch number stays internal, to keep artifacts, delivery and cancellation apart.

## The launch lock fences each launch

The runner holds `launch.lock` for its whole life, and only a holder publishes the terminal outcome. Without that fence a late writer from an older launch could change the new one.

- **Reopen.** The lock holder settles the previous launch and sweeps its recorded process scopes before committing the next; the sweep is the containment backstop if the guardian died or couldn't kill a process ([process scope](../process-scope.md)). A continue can reparent the spawn and its subtree, and cycle detection walks the full stored ancestor chain, which can be longer than the spawn depth limit.
- **Pinning.** Operations that don't hold the lock (cancel, wait, late artifact writers) pin the launch they observed, so they can't act on a launch reopened since. A descendant that escaped cleanup but still holds the spawn ID can issue spawn-level commands, so the fence is not full containment.
- **Prepare before reserving.** Preparation runs before the spawn row is reserved, so setup errors create no row.

How a dead launch's outcome is derived and repaired is in [derived status](derived-status.md).

## Usage across launches

Claude and Codex report session-cumulative usage, so a launch subtracts the prior recorded cumulative value instead of counting it twice. A fork needs its source session's recorded seed for the same reason. A row from before launches existed lacks the seed, so its first launch after a reopen can be counted twice.

## Results are owed per launch

Delivery and consumption records are keyed by `(spawn_id, launch)`: keyed by spawn alone, consuming launch 1 would hide launch 2 forever. The keys live inside the files; the CLI never shows them. The protocol is in `src/meridian/pi_runtime/.context/delivery-contract.md`; the reader-facing behavior is in [child result delivery](../../concepts/child-result-delivery.md).

- **A wait pins its launch.** `spawn wait pN` re-runs as new processes, so the pin is stored in the caller's ledger and expires one yield interval ahead of each poll. A wait that lost its pin pins the current launch; the earlier result is still owed, so nothing is lost.
- **Reopen keeps an unconsumed launch owed.** Owed discovery selects children by current parent and launch mode, so after a reopen an unconsumed background launch would vanish; reopen records it as owed.
- **New ledger files, not a version bump.** An older Python build reads an unknown version as absent and rewrites the whole document; an older Pi extension throws on it and stops delivering. A bare legacy entry reads as launch 1, which is exact because older builds can't read a reopened row.
- **`spawn collect` and `spawn ack` keep bare IDs.** A `:` in the ID breaks the Claude mod's pattern, and a bare `ack` must not consume a running launch. `ack` consumes exactly what `collect` handed out. A delivered launch that is no longer current carries its report inline, because `spawn report show` now shows the next launch.
- **Lifecycle events.** `spawn.created` fires once; every launch emits `spawn.started` and `spawn.finished`. Event IDs hash the launch so launch 1 keeps the ID older builds gave it (hook dedupe still holds) and launch 2 doesn't dedupe against it.

## Older builds on the same runtime

Older readers can reject newer rows or repeat a delivery after an upgrade. An older Pi extension still loaded after a continue can mark launch 1 consumed without showing it, so restart Pi sessions after upgrading. Procedure: the source checkout's `docs/upgrading.md`.
