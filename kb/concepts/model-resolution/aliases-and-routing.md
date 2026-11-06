# Aliases and Harness Routing

Mars owns alias identity and launch routing. Meridian consumes the selected route rather than guessing a harness from model-name patterns. A harness-native model token can differ from canonical identity; use the selected route at the native boundary. See [model resolution decisions](../../decisions/model-resolution.md) and `docs/configuration/mars.md`.

## Inventory and resolution surfaces

`models list` is the human-facing Possible-through-Curated harness/model view;
`models aliases` provides static alias definitions; `models catalog` provides
raw provider data; and `models resolve` resolves an alias. Meridian machine
consumers must not read the curated display list as alias inventory. An alias's
harness is an authored preference, not proof of an available route.

For runtime launch, the launch bundle remains the authority for harness
selection and the harness-specific model string. Catalog rows, Possible rows,
and aliases by themselves do not establish a runnable route. Exact cache,
fallback, and dry-run behavior is owned by Meridian's catalog package docs.


OpenCode continuation must preserve the committed native agent, model and variant; unsent TUI-local changes are not observable and observing state plus submitting a prompt is not atomic. See [continuation model selection](../continuation-model-selection.md).

Prompting guidance is caller-facing advice for briefing a model, separate from the model description; see [model resolution decisions](../../decisions/model-resolution.md).
