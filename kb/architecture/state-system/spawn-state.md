# Spawn state

Each spawn has its own state, so reading one spawn by ID costs the same however much history the project has. Listing and selection still materialize every indexed row; those scan costs
 are in [per-operation cost](../per-operation-cost.md). Contract: the source checkout's `src/meridian/lib/state/AGENTS.md`.

## A published row owns its artifacts

Late writers and deletion coordinate on the same lock, so a writer can't recreate an artifact after its spawn was deleted.

## Old rows upgrade at the read boundary

Known legacy rows upgrade when read; conflicting or unknown facts are quarantined rather than guessed. Treating an unreadable row as failed would invent an outcome, and discarding legacy rows would hide existing history. See the [legacy codec](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/state/spawn/legacy.py).
