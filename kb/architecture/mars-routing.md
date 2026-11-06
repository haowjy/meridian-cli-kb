# Mars routing

## Routing parity (PR #72)

Model preview and launch-bundle routing share evidence and a candidate evaluator so they cannot silently disagree. The Possible inventory and Curated display are separate from runtime selection; curation cannot change a route. See [model resolution decisions](../decisions/model-resolution.md).

Selection and match evidence are distinct: a fixed harness is a caller choice, not proof that its model is available. A confirmed match can outrank a passthrough candidate with unknown evidence. Cursor's inconsistent model slug conventions make structured family-specific decomposition fragile; Mars uses raw probe slugs and resolves effort to the launch model. See [Cursor](cursor-harness.md) and the mars-agents checkout's `src/routing/AGENTS.md`.

