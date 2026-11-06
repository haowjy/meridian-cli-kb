# Cursor model projection

Cursor encodes effort in model slugs, with inconsistent ordering and spelling across families. Mars retains raw probe slugs instead of decomposing them into a family-specific schema, and resolves the launch slug before Meridian invokes Cursor. Meridian uses the returned `harness_model` rather than applying effort again. See the mars-agents checkout's `src/models/probes/.context/CONTEXT.md` and [routing](mars-routing.md).

Fast variants are excluded from the probe result; [mars-agents #256](https://github.com/haowjy/mars-agents/issues/256) tracks them.
