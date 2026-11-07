# Package management

Mars resolves packages and produces the artifacts Meridian consumes. The source checkout's [src/AGENTS.md](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/src/AGENTS.md), [src/compiler/AGENTS.md](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/src/compiler/AGENTS.md) and [src/build/AGENTS.md](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/src/build/AGENTS.md) describe its contracts. See also [Mars targeting](../architecture/mars-targeting.md), [launch bundles](../architecture/mars-launch-bundle.md), and [user scope](../architecture/mars-user-scope.md).

Meridian asks Mars or reads Mars-produced artifacts for effective settings instead of independently merging Mars configuration. A second resolver can drift when Mars adds a configuration layer; the same boundary applies to skills. When Mars does not expose a needed fact, add an interface there.

## User configuration {#the-user-mars-layer-is-the-existing-configtoml-below-the-project-2026-10-28}

Mars's user configuration is below project configuration. A separate user-local layer above the project would hide project choices; user package content is a separate concern. See [Mars user scope](../architecture/mars-user-scope.md).

## Plugin builds {#d100-a-package-publishes-one-plugin-holding-its-full-resolved-graph-2026-10-24}

A plugin contains the resolved package graph, rather than requiring consumers to install each graph node separately. See [harness plugin loading](../research/harness-plugin-loading.md) for harness constraints.

## OpenCode model selection

An OpenCode plugin leaves the model unset when it cannot determine a provider-qualified model. Choosing a provider on the user's behalf can make the agent unusable when that provider is unavailable. See the Mars checkout's [src/build/AGENTS.md](https://github.com/haowjy/mars-agents/blob/release/v0.28.0/src/build/AGENTS.md).

## Warnings

The project invariant requires lost restrictions to be reported by default and other lost fields to remain available under verbose output. See the [meridian-cli invariants](https://github.com/haowjy/meridian-cli/blob/main/AGENTS.md) and Mars's compiler guidance.
