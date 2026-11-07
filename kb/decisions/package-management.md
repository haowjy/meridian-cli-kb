# Package management

These decisions cover how Mars resolves packages and produces artifacts Meridian consumes. The source checkout's [src/AGENTS.md](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/src/AGENTS.md), [src/compiler/AGENTS.md](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/src/compiler/AGENTS.md) and [src/build/AGENTS.md](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/src/build/AGENTS.md) describe its contracts. See also [Mars targeting](../architecture/mars-targeting.md), [launch bundles](../architecture/mars-launch-bundle.md), and [user scope](../architecture/mars-user-scope.md).

Meridian asks Mars or reads Mars-produced artifacts for effective settings instead of independently merging Mars configuration. A second resolver can drift when Mars adds a configuration layer; the same boundary applies to skills. When Mars does not expose a needed fact, add an interface there.

## User configuration {#the-user-mars-layer-is-the-existing-configtoml-below-the-project-2026-10-28}

Mars's user configuration is below project configuration. A separate user-local layer above the project would hide project choices; user package content is a separate concern. See [Mars user scope](../architecture/mars-user-scope.md).

## Plugin builds {#d100-a-package-publishes-one-plugin-holding-its-full-resolved-graph-2026-10-24}

A plugin contains the resolved package graph, rather than requiring consumers to install each graph node separately. Following references in prose would make a second, unreliable definition of the package. Plugin dependencies also cannot reliably provide the graph: Codex ignores them, and Claude cross-marketplace dependencies require prior marketplace registration and allowlisting. See [harness plugin loading](../research/harness-plugin-loading.md).

## Source and output boundaries

Hidden harness directories are not discovered as package source by default: scanning them would treat generated state as authored input and require harness-specific guesses. Foreign layouts can be imported explicitly. See [source selection](../concepts/package-management/self-source-selection.md).

Matching bytes do not establish ownership of an unowned output. See [sync model](../concepts/package-management/sync-model.md) for the interrupted-write recovery boundary.

Cross-source agent and skill collisions rename both items: first-wins would make one package disappear according to declaration order, while an error blocks legitimate combinations. A renamed-away config reference warns rather than silently rewriting user configuration. See [collision handling](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/docs/internals/conflicts.md).

## Engine compatibility

Package engine requirements filter incompatible versions and select the newest compatible candidate; sources with no alternate candidate fail. A hard filter avoids silently compiling a package with an engine too old to understand it. An advisory warning or soft preference would allow that incorrect result.

## Skill variants

Mars selects a skill body by harness, falling back to the base body. Model-specific selection was not built because no package or request needed it; adding it belongs in Mars, not a second Meridian resolver. See [skill loading](../concepts/skill-loading.md).

> [!NOTE] **Known gap**: Mars still indexes model-variant directories and warns about their layout, but does not select them. This can suggest support that does not exist.

## OpenCode model selection

An OpenCode plugin leaves the model unset when it cannot determine a provider-qualified model. Choosing a provider on the user's behalf can make the agent unusable when that provider is unavailable. See the Mars checkout's [src/build/AGENTS.md](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/src/build/AGENTS.md).

## Warnings

The project invariant requires lost restrictions to be reported by default and other lost fields to remain available under verbose output. See the [meridian-cli invariants](https://github.com/haowjy/meridian-cli/blob/main/AGENTS.md) and Mars's compiler guidance.
