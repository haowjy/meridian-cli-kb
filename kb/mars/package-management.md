# Package management

Mars resolves prompt packages, compiles them into the canonical `.mars/` catalog, and projects native files into each harness's directories. Meridian consumes what Mars produces instead of running a second package manager. Contracts live in the mars-agents checkout: `src/AGENTS.md`, `src/compiler/AGENTS.md`, `src/build/AGENTS.md`, `src/resolve/AGENTS.md`, `src/sync/AGENTS.md`; configuration in its `docs/config/`.

## Meridian asks Mars

Meridian reads Mars-produced artifacts, or asks Mars, for effective settings, never merging Mars configuration itself: a second resolver drifts the moment Mars adds a configuration layer. The same holds for skills. When Mars doesn't expose a fact Meridian needs, the interface is added to Mars.

## Resolution

Mars resolves the dependency graph before selecting items, so sync, package checks and plugin builds share one selection instead of filtering separately. Engine requirements are a hard filter: incompatible versions are excluded, staying excluded across resolution restarts, and the newest compatible version is chosen; a path or pinned source with no alternative fails. A warning or soft preference would let a package compile with an engine too old to understand it.

Hidden harness directories aren't discovered as package source by default: scanning them would treat generated state as authored input and need harness-specific guesses. Foreign layouts are imported explicitly.

## Current-package content

Content of the project itself is selected under a synthetic self owner. `.mars-src` overrides matching definitions of a declared package; dependencies are renamed before self items overlay installed destinations, which keeps dependency names and references intact when an authored name differs from the installed one. Discovery picks the shallowest occupied convention layer, so generated non-hidden target content can become source on a later sync ([mars-agents #161](https://github.com/haowjy/mars-agents/issues/161)); plugin output is excluded through its sentinel so an earlier build can't shadow authored content.

## Collisions rename both

When two sources provide an agent or skill with the same name, both are renamed. First-wins would make one package vanish depending on declaration order; an error would block legitimate combinations. A config reference to a renamed-away item warns rather than silently rewriting user configuration ([conflicts](https://github.com/haowjy/mars-agents/blob/main/docs/internals/conflicts.md)).

## Targets

`.mars/` is the canonical store; selected content is lowered into each harness's native format rather than assuming harnesses understand Mars profiles. Each harness gets its own target copy, which allows harness-specific lowering and avoids duplicate discovery through overlapping shared skill roots. When Meridian manages routing, native agent emission is suppressed to avoid a second agent catalog; standalone Mars emits native agents so harnesses can discover them. Known harness targets can constrain routing; generic targets only control materialization.

Hooks and MCP entries use one target selection, so a server's configuration and its installed directory can't come from different winners. An entry that needs a packaged directory is withheld when an unmanaged or modified destination blocks installation. Dependency exclude filters don't yet match MCP server names ([mars-agents #237](https://github.com/haowjy/mars-agents/issues/237)).

## Sync and ownership

Each write is atomic and sync runs are serialized, but a sync is not a transaction with rollback. Exact lock evidence controls ownership and removal: matching bytes never prove Mars owns an unowned output, and unreadable owned input is not taken to mean the desired output is absent, so a later retry can still act instead of deleting something whose desired state is unknown. A durable write intent lets Mars reclaim a new canonical output after an interruption, once prior lock state and output identity both check out; native and config outputs aren't covered yet ([mars-agents #149](https://github.com/haowjy/mars-agents/issues/149)). Explicit force adoption is limited to safe shapes, and a failed apply after it isn't journaled, so an ordinary retry may still refuse.

When managed output changes shape, removal must recognize what earlier versions actually wrote: check migrations against real prior-version artifacts, since fixtures modeled on current output miss legacy formats. Legacy removal runs before replacements are written and only touches artifacts that need it, so it can't remove new or user-owned entries. An ownership check by content alone can follow a symlink to matching external bytes and claim the link; check filesystem structure too.

## Plugins carry the whole resolved graph

A plugin build contains the package's full resolved graph rather than asking consumers to install each node. Following references in prose would be a second, unreliable definition of the package, and plugin dependencies can't carry the graph: Codex ignores them, and Claude's cross-marketplace dependencies need prior marketplace registration and allowlisting ([harness plugin loading](../research/harness-plugin-loading.md)). Sync and plugin builds share target selection and lowering, so a plugin holds what sync would install. When an OpenCode plugin can't determine a provider-qualified model it leaves the model unset: choosing a provider for the user could make the agent unusable where that provider isn't available.

## User scope

`meridian mars --user add` installs into the user root `~/.meridian/mars/`, which supplies agents and skills where no project package root exists, or as a content fallback beneath a project's own. Project configuration, aliases, targets and caches stay with the primary root only, so a user agent's model alias must resolve in the project using it. Mars's user configuration (`config.toml`) is a separate layer below project configuration; a user layer above the project would hide project choices. Content discovery walks upward while project identity doesn't, so a subdirectory can inherit its repository's packages yet keep its own ad-hoc state ([project identity](../state/project-identity.md)). Meridian prefers the Mars installed beside its own Python over one on `PATH`.

Related: [skills](skills.md), [portable hooks](package-hooks.md), [model resolution](model-resolution.md), [launch bundle](launch-bundle.md).
