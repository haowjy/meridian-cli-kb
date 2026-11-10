# Cursor harness

Cursor encodes effort in its model slugs, with ordering and spelling that differ across model families. Mars keeps the raw probe slugs rather than decomposing them into a family-specific schema, and resolves the full launch slug before Meridian invokes Cursor; Meridian uses the returned `harness_model` and never applies effort again. Fast variants are filtered out of discovery; whether speed belongs in effort or a separate axis is undecided. Contract: mars-agents `src/models/probes/.context/CONTEXT.md` and [model resolution](../concepts/model-resolution.md).

Cursor is unmaintained in Meridian: it keeps building, without parity work (`src/meridian/lib/harness/projections/.context/CONTEXT.md`).
