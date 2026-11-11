# Cursor

Cursor encodes effort in its model slugs, with ordering and spelling that differ across model families. Mars keeps the raw probe slugs rather than decomposing them into a family-specific schema, and resolves the full launch slug before Meridian invokes Cursor; Meridian uses the returned `harness_model` and never applies effort again. Fast variants are filtered out of discovery; whether speed belongs in effort or a separate axis is undecided. Contract: mars-agents `src/models/probes/.context/CONTEXT.md` and [model resolution](../mars/model-resolution.md).

Cursor is unmaintained in Meridian: it keeps building, without parity work (`src/meridian/lib/harness/projections/.context/CONTEXT.md`).

## Sandbox and approval

A Cursor probe found that sandboxed workspace writes could succeed while deletion was approval-gated. Cursor's binary sandbox can't represent every Meridian sandbox policy, and its default approval behavior has no explicit confirm flag, so Meridian reports approximations ([tool permissions](../permissions/tool-permissions.md)). Mars doesn't write Cursor's undocumented `.cursor/sandbox.json`: it would conflict with user configuration, couple Mars to an internal format and need cleanup ownership ([mars-agents #91](https://github.com/haowjy/mars-agents/issues/91)).
