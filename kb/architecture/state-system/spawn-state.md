# Spawn state

Per-spawn state gives individual reads cost independent of total history. On a dataset of about 189 MB and 35,000 events, primary launch fell from 12–13 seconds to 0.67 seconds after the change; these historical figures are not current guarantees. Remaining scan costs are tracked in [per-operation cost](../per-operation-cost.md).

### Published Spawn Artifact Lifetime

A published row owns the lifetime of its artifacts. Late writers and deletion coordinate on the same lock so a writer cannot recreate an aggregate after deletion. See [state decisions](../../decisions/state.md) and the source checkout's `src/meridian/lib/state/AGENTS.md`.

