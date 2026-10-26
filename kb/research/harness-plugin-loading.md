# How harnesses load plugins

There is no shared plugin format, but one Claude-format tree goes a long way:
Claude Code, Codex and Grok all install the same `.claude-plugin/` tree from a
Claude catalog. Cursor has its own manifest, hook token and MCP file. OpenCode
loads only a JavaScript module and cannot add agents. Mars's
`mars build plugin` therefore writes three trees (Claude, Cursor, OpenCode) for
five harnesses; scope and rationale are in
[D100](../decisions/package-management.md#d100-a-package-publishes-one-plugin-holding-its-full-resolved-graph-2026-10-24),
and the generated support and verification tables are in mars-agents
`README.md` § Plugin Distribution (on the `release/v0.19.0` line, not yet
released).

Versions probed, 2026-10-24 and 2026-10-25: Claude Code 2.1.310, Codex CLI
0.164.0, Grok 1.0.40 and 1.0.70, OpenCode 2.1.3 (source) and 2.1.4 (live).
Cursor CLI 2026.10.22 installed but refused every plugin command without an
account, so every Cursor fact here comes from Cursor's docs and published JSON
schemas. Each probe ran in an isolated home with only auth files copied.
Recheck a fact before relying on it against a later harness version.

## At a glance

| | Claude Code | Codex CLI | Grok | OpenCode | Cursor (docs only) |
|---|---|---|---|---|---|
| Manifest | `.claude-plugin/plugin.json` (only `name` required) | own `.codex-plugin/plugin.json`, or reads Claude's and Cursor's | reads Claude's; no Grok files needed | JS module: `package.json` + default export | `.cursor-plugin/plugin.json` |
| Catalog | `.claude-plugin/marketplace.json` | `.agents/plugins/`, then `.claude-plugin/`, then `.cursor-plugin/` | Claude's; the plugin must sit inside the catalog's directory | git or path entry in `opencode.json` | `.cursor-plugin/marketplace.json` |
| Skills | loaded as `<plugin>:<skill>` | loaded as `<plugin>:<skill>` | loaded as `<plugin>:<skill>`, with `resources/` | registered by the module's `setup` | documented |
| Agents | loaded, with a field allowlist | **not loaded** | loaded; subagents preload `skills:` | **cannot add an agent** | documented |
| Hooks | `hooks/hooks.json` | reads `hooks/hooks.json` | reads `hooks/hooks.json` | in-process JS only | `hooks/hooks.json`, Cursor event names |
| MCP | `.mcp.json` | `.mcp.json`, callable; env only through `env_vars` | `.mcp.json` | registered by `setup` as `<plugin>:<server>` | `mcp.json` |
| Plugin-root variable | `${CLAUDE_PLUGIN_ROOT}` | `PLUGIN_ROOT`, and `CLAUDE_PLUGIN_ROOT` | `GROK_PLUGIN_ROOT`, and `CLAUDE_PLUGIN_ROOT` | none | `${CURSOR_PLUGIN_ROOT}` |
| Plugin `dependencies` | resolved (see below) | ignored | not probed | — | — |

## Claude Code

**Naming.** Claude prefixes every plugin component with the manifest name:
skill `code` in plugin `meridian-dev-workflow` is
`meridian-dev-workflow:code`. A bare-name skill lookup failed in one probe
(`Unknown skill: probe-skill`) and worked in a later one, so treat only the
qualified name as reliable. Mars writes `<plugin>:<name>` in every frontmatter
and body reference.

**Plugin agents ignore some fields.** Inside a plugin, Claude ignores an
agent's `permissionMode`, `hooks`, `mcpServers` and `initialPrompt`. Hooks
and MCP servers belong at plugin level. Mars drops those fields with a
warning rather than ship fields that silently do nothing.

**A session agent does not preload `skills:`.** Spawned as a subagent through
the Agent tool, a plugin agent preloads every skill in its `skills:` list,
bare or qualified. Started as the session agent
(`claude --agent <plugin>:<agent>`), the same file loads but none of its
`skills:` bodies are injected. One agent file can serve both roles, so the
skill list alone cannot guarantee loading. This is why Mars's skills lowering
also writes a "load these skills" line into Claude agent bodies
([D101](../decisions/package-management.md#d101-agent-skills-lower-by-harness-capability-2026-10-24)).

**Root variable.** `${CLAUDE_PLUGIN_ROOT}` expands to the installed plugin
directory in hook commands, stdio MCP servers and LSP servers. It does not
expand inside agent bodies, so a skill's relative `resources/` paths only
work when the skill itself is loaded, not when its text is copied into an
agent. A plugin installed from a local-directory marketplace runs in place:
the root is the source folder, and edits apply in the next process.

**Project-checked-in marketplaces are not portable.** A project
`.claude/settings.json` `extraKnownMarketplaces` entry with a relative
`path` stays relative and the plugin does not load; an absolute path works
but cannot survive a clone. This is one reason `mars sync` does not install
or enable plugins.

**Validation.** `claude plugin validate <dir>` is authoritative. Unknown
top-level manifest keys warn and are stripped; strict nested objects can fail
validation, so Mars keeps its ownership sentinel out of `marketplace.json`.
`file://` marketplace URLs are rejected; a local directory or a local git
checkout works.

**Plugin dependencies.** A plugin can declare `dependencies` on other
plugins. A dependency listed in the same marketplace installs and enables
automatically. A cross-marketplace dependency installs only when that
marketplace is already registered and the root catalog allowlists it
(`allowCrossMarketplaceDependenciesOn`); otherwise the install "succeeds"
and the plugin fails to load. A dependency names a registered marketplace,
never a URL. Mars does not use plugin dependencies (D100), partly because
Codex ignores them.

## Codex CLI

Codex looks for a root Agent Plugins `plugin.json`, then
`.codex-plugin/plugin.json`, `.claude-plugin/plugin.json` and
`.cursor-plugin/plugin.json`. Its legacy parser accepts Claude manifests and
does not reject unknown fields. Catalogs are searched in the order
`.agents/plugins/`, `.claude-plugin/`, `.cursor-plugin/`, so with a Cursor
catalog beside the Claude one Codex still installs the Claude tree.
Installation copies the tree into a cache under `CODEX_HOME` and writes
enablement into user config, so a fresh clone is never self-contained.

What a Claude-format tree gets on Codex:

- **Skills and hooks work.** Hook discovery injects `PLUGIN_ROOT`,
  `CLAUDE_PLUGIN_ROOT`, `PLUGIN_DATA` and `CLAUDE_PLUGIN_DATA`, so Claude
  hook commands run unchanged.
- **Agents are not loaded.** Codex has no plugin agent component; subagents
  come from host TOML configuration. The agent files survive in the cache and
  do nothing. A plugin also cannot install loose `.codex/agents/*.toml` into a
  user's project. Mars prints one warning that Codex plugin users get skills
  and hooks but not agents; users who want the agents install with mars.
- **MCP servers are callable**, but environment reaches them only one way.
  Codex forwards a variable listed by bare name in `env_vars`. It passes
  `"env": {"X": "${X}"}` and `$X` through literally, and a server with no
  `env` does not inherit the parent environment. Claude and Grok accept the
  same `env_vars` file, so the shared tree uses it for same-name
  pass-throughs. An earlier probe reported plugin MCP as listed but not
  callable; a later run on the same version called it.
- **`commands/*.md` are migrated** into generated skills under
  `.codex-plugin/migrated-command-skills/`.
- **Plugin `dependencies` are ignored**, whether declared in `plugin.json`
  or in the catalog entry. Installing the top plugin installs only it.

A Codex-only hook (a `codex.json` fragment with no Claude fragment) cannot
ride the shared tree; it would need a Codex-own tree with
`.codex-plugin/plugin.json` and `${PLUGIN_ROOT}`.

## Grok

Grok installs the Claude tree as built, with no Grok-specific files, and
picks it when a Cursor catalog and an OpenCode `package.json` sit beside the
Claude catalog. The plugin must be inside the catalog's directory: a catalog
entry pointing at `../plugin` registers but exposes nothing. Live sessions ran
a qualified skill and read its `resources/`, ran a plugin agent as a subagent
with its `skills:` preloaded, fired `SessionEnd` and `SubagentStop` hooks with
`CLAUDE_PLUGIN_ROOT` set, and called a plugin MCP server that received its
value through `env_vars`.

Grok rejects `mcpServers`, `hooks` and `permissionMode: bypassPermissions`
in plugin agents; the Claude tree already drops all three, because Claude
ignores them in plugin agents too. How Grok treats a Claude model name such as
`model: opus` is unobserved; the live turns chose the model at session level.

## OpenCode

OpenCode's plugin is executable JS: a `package.json` plus a module whose
default export is `{id, setup}`. `setup` can register skills, commands and
MCP from bundled files, locating them through `import.meta.url`. A plugin
directory of `skills/`, `agents/` and `commands/` is not read, and the agent
editor has no `add`, so a plugin cannot create agents. No plugin-root
variable exists.

- **Skills** registered from bundled files load with working `resources/`.
- **MCP** registered as `<plugin>:<server>` in OpenCode's
  `{type: "local", command: [cmd, ...args]}` shape is callable; the model
  sees the colon as `_`. Server processes start in the user's project
  directory, and there is no plugin-root variable, so an `args` path
  relative to the package resolves against the project and misses.
- **Hooks** are in-process callbacks; `hooks/hooks.json` is not read. A hook
  module must default-export `{id, setup}`. The named-export shape fails with
  `Plugin must export a default definition with an id and an effect or setup
  function`, and `opencode run` still exits 0. Mars's sync and plugin build
  both warn on a fragment with no default export
  ([mars-agents#213](https://github.com/haowjy/mars-agents/pull/213)).
- **Git install** (`opencode plugin add git+…`) honors the root
  `package.json`'s `main` and `files`, so a repo can expose one subdirectory
  as the package.

**Project MCP config.** A project's `.opencode/opencode.json` registers MCP
servers under `mcp` as `{type: "local", command: [cmd, ...args],
environment: {...}}`; `mcpServers` is ignored at runtime. Only `{env:VAR}`
expands in `environment` values (`${env:VAR}` and bare names stay literal).
`opencode mcp list` in 2.1.4 reports the background service's scope, not the
project's, so it shows "No MCP servers configured" even for a working project
config; an `opencode run` turn that calls the tool is the reliable check.
`mars sync` writes this shape from v0.19.0 (#213).

OpenCode already reads `.claude/skills` and `.agents/skills` from projects,
so mars-installed users have skills without a plugin. A project
`opencode.json` may point at a relative plugin path, which loads in place
from a fresh home. How Mars's generated `index.mjs` registers the tree and
isolates each hook is in mars-agents `src/build/.context/CONTEXT.md`.

## Cursor

Everything here is from Cursor's documentation and published schemas.
Cursor's own format is `.cursor-plugin/plugin.json` with skills, rules,
agents, commands, hooks and MCP. A root `plugin.json` (the "Agent Plugins"
form) carries only skills and MCP. Hooks receive `${CURSOR_PLUGIN_ROOT}`;
the docs don't list `CLAUDE_PLUGIN_ROOT` as a hook alias, so Claude hook
commands can't be assumed to resolve. MCP lives in `mcp.json` under
`mcpServers` with `${env:VAR}`. The CLI adds marketplaces from a git URL
but has no install subcommand; installs go through the Customize UI, and
`agent --plugin-dir <dir>` loads a tree for local development. Mars-built
manifests and catalogs pass Cursor's published `plugin.schema.json` and
`marketplace.schema.json`; nothing has run on the binary.

## One repository, several loaders

Claude and Grok run a local-marketplace plugin in place, inside the
checkout. A root `package.json` written for OpenCode therefore governs every
`.js` file in the Claude tree too: `"type": "module"` turns CommonJS hook
scripts and MCP servers into ESM and breaks them. Mars's root `package.json`
declares no `type`, and its OpenCode entry is `index.mjs`.

## Sources

Synthesized from the probe and source-study reports of
`work:harness-plugin-export` (archived with the work item):
`spawn:p9291` (install probe across harnesses), `spawn:p9290` (Codex and
OpenCode loader source at rust-v0.164.0 and v2.1.3), `spawn:p9288`
(published formats), `spawn:p9297` (OpenCode JS registration),
`spawn:p9305` (project-local enablement), `spawn:p9328` (Claude
cross-marketplace dependencies), `spawn:p9329` (Claude agent `skills:`
preload), `spawn:p9340` (same-marketplace dependencies), `spawn:p9473`,
`spawn:p9478` and `spawn:p9479` (Grok install and live turns),
`spawn:p9474` (Cursor docs and account wall), `spawn:p9475` and
`spawn:p9477` (OpenCode package, hooks, MCP, git install), `spawn:p9480`
(the `type: module` break), `spawn:p9533` (live verification of built
trees, Cursor schema validation), `spawn:p9535` (Grok MCP and Codex
environment forms), `spawn:p9556` and `spawn:p9560` (OpenCode project MCP
and `mcp list`). Claude plugin references:
[plugin components](https://code.claude.com/docs/en/plugins/components),
[subagent frontmatter](https://code.claude.com/docs/en/sub-agents#frontmatter-reference).
Codex `env_vars` parsing:
[plugin_config.rs](https://github.com/openai/codex/blob/main/codex-rs/codex-mcp/src/plugin_config.rs).
