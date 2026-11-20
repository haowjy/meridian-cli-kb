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

Mars records, per harness, whether Meridian launches it. For one it doesn't (`agy` today), routing never picks it, because no launch would follow, and Mars writes its agent files even when Meridian manages routing, because nothing else puts them where that harness reads them. `.agy` is not a copy of `.mars/`: `agy` reads nothing from `.agy/` by itself, so Mars writes it as an `agy` plugin and owns one entry in the project's `.agents/plugins.json` that points at it, rather than writing into `.agents/skills/`, which users and other tools share (mars-agents `docs/config/mcp-and-hooks.md`, "Antigravity").

A host such as Meridian hands Mars its own skills, hooks and MCP servers with `mars provide`. A harness the host launches gets them from the host at launch, so sync installs provided content only into targets whose harness the host doesn't launch. The other way to reach those targets, a plain package of the host's that every project adds to `mars.toml`, would put an entry in every user's config and version it apart from the host. Records for provided content go to the machine-local `.mars/machine.lock`, never `mars.lock`: the staged bundle is an absolute path on one machine, and a provider never changes committed state. Mechanism: mars-agents `src/provided/.context/CONTEXT.md`.

Hooks and MCP entries use one target selection, so a server's configuration and its installed directory can't come from different winners. An entry that needs a packaged directory is withheld when an unmanaged or modified destination blocks installation. Dependency exclude filters don't yet match MCP server names ([mars-agents #237](https://github.com/haowjy/mars-agents/issues/237)).

## Sync and ownership

Each write is atomic and sync runs are serialized, but a sync is not a transaction with rollback. Exact lock evidence controls ownership and removal: matching bytes never prove Mars owns an unowned output, and unreadable owned input is not taken to mean the desired output is absent, so a later retry can still act instead of deleting something whose desired state is unknown. A durable write intent lets Mars reclaim a new canonical output after an interruption, once prior lock state and output identity both check out; native and config outputs aren't covered yet ([mars-agents #149](https://github.com/haowjy/mars-agents/issues/149)). Explicit force adoption is limited to safe shapes, and a failed apply after it isn't journaled, so an ordinary retry may still refuse.

A sync plans, writes and removes only the records of the targets the config names. A target it stops naming (a hand edit of `targets`, the retired `managed_root` key, the retired `.cursor` target) keeps its files and records, with one `unnamed-target` warning per sync, until `mars unlink` removes them or the config names it again. Removing them on the next sync would strip hooks and MCP servers from every project whose only target came from `managed_root`, and would delete records that a teammate's local-only target put in the shared lock. Keeping is undone by naming the target again; removal is undone only by a full re-sync. The cost: a hand edit removes nothing by itself, and dropping the deprecated `.agents` while `.agy` is linked leaves `.agents/skills/`, which `agy` also loads, so it sees each skill twice until `mars unlink .agents`.

The lock is split where it is loaded and joined where it is written, by the target each record serves, so no step of a sync holds a record outside its targets. A filter that set those records aside on load and restored them inside `finalize` failed twice: `settle` then compared old and new locks of different shapes, so a held warning repeated on every sync; and it chose records by `target_root`, so a hook's `.mars/hooks/<target>/` copy stayed in the pipeline and was deleted, after which `mars link` left a settings entry pointing at a missing hook directory. Mechanism: mars-agents `src/lock/.context/CONTEXT.md`.

One case stays open. A target that only one person's `mars.local.toml` names still gets its records in the committed `mars.lock`; a teammate's sync keeps them and warns to unlink, and if the teammate does, the owner's next sync asks for `mars sync --force`. Nothing is lost. Moving those records to `.mars/machine.lock` would close it, but changes what the committed lock holds, so it waits for the owner's approval; the design is `design/unnamed-targets.md` in the `antigravity-mars` work item.

When managed output changes shape, removal must recognize what earlier versions actually wrote: check migrations against real prior-version artifacts, since fixtures modeled on current output miss legacy formats. Legacy removal runs before replacements are written and only touches artifacts that need it, so it can't remove new or user-owned entries. An ownership check by content alone can follow a symlink to matching external bytes and claim the link; check filesystem structure too.

## Warnings say what the user loses

A problem is a `Warning` when installed behaviour does less than the package declares, or when the user can act on it; otherwise it is `Info`, printed only by `--verbose`, `mars check` and `mars validate`. A loss is reported once, by the sync that makes its lock record new or different, so an unchanged sync is silent. Letting the item's owner set the level hid real losses in dependencies, such as a public package's hook that installed nowhere: the person running agents gets the behaviour, whoever wrote the item. Rule and mechanism: mars-agents `AGENTS.md`, "Errors and Warnings".

## Plugins carry the whole resolved graph

A plugin build contains the package's full resolved graph rather than asking consumers to install each node. Following references in prose would be a second, unreliable definition of the package, and plugin dependencies can't carry the graph: Codex ignores them, and Claude's cross-marketplace dependencies need prior marketplace registration and allowlisting ([harness plugin loading](../research/harness-plugin-loading.md)). Sync and plugin builds share target selection and lowering, so a plugin holds what sync would install. When an OpenCode plugin can't determine a provider-qualified model it leaves the model unset: choosing a provider for the user could make the agent unusable where that provider isn't available.

## User scope

`meridian mars --user add` installs into the user root `~/.meridian/mars/`, which supplies agents and skills where no project package root exists, or as a content fallback beneath a project's own. Project configuration, aliases, targets and caches stay with the primary root only, so a user agent's model alias must resolve in the project using it. Mars's user configuration (`config.toml`) is a separate layer below project configuration; a user layer above the project would hide project choices. Content discovery walks upward while project identity doesn't, so a subdirectory can inherit its repository's packages yet keep its own ad-hoc state ([project identity](../state/project-identity.md)). Meridian prefers the Mars installed beside its own Python over one on `PATH`.

Related: [skills](skills.md), [portable hooks](package-hooks.md), [model resolution](model-resolution.md), [launch bundle](launch-bundle.md).
