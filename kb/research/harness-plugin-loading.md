# How harnesses load plugins

There is no shared plugin format. Claude Code's format is the richest, and
Codex can install a Claude-format plugin from a Claude catalog, so one
Claude-format tree serves both. OpenCode needs a JavaScript module and cannot
add agents. Cursor documents its own format but was never probed. Mars's
`mars build plugin` therefore emits three physical trees: shared Claude for
Claude/Codex/Grok, native Cursor, and executable OpenCode. Its scope and rationale are in
[D100](../decisions/package-management.md#d100-a-package-publishes-one-plugin-holding-its-full-resolved-graph-2026-10-24).

Versions probed on 2026-10-24: Claude Code 2.1.310, Codex CLI 0.164.0,
OpenCode 2.1.3. Codex and OpenCode findings were also read from source at
those tags. Each probe ran in an isolated home with only auth files copied.
Recheck a fact before relying on it against a later harness version.

## At a glance

| | Claude Code | Codex CLI | OpenCode | Cursor |
|---|---|---|---|---|
| Plugin manifest | `.claude-plugin/plugin.json` (only `name` required) | own `.codex-plugin/plugin.json`, or reads Claude's and Cursor's | JS module: `package.json` + default export with `setup` | `.cursor-plugin/plugin.json` (documented, unprobed) |
| Catalog | `.claude-plugin/marketplace.json` | `.agents/plugins/marketplace.json`, and also reads `.claude-plugin/marketplace.json` | path or npm entry in `opencode.json` | documented, unprobed |
| Skills | loaded, named `<plugin>:<skill>` | loaded, named `<plugin>:<skill>` | not from a plugin dir; registered via JS, or read from `.claude/skills` and `.agents/skills` | — |
| Agents | loaded, with a field allowlist (below) | **not loaded** | **cannot register a new agent** | — |
| Hooks | `hooks/hooks.json` | reads `hooks/hooks.json` | in-process JS callbacks only | — |
| MCP | `.mcp.json` | listed by `codex mcp`, **not callable** by the agent | separate config | names approved as `plugin-<p>-<s>` |
| Commands | loaded | migrated into generated skills | — | — |
| Plugin-root variable | `${CLAUDE_PLUGIN_ROOT}` | `PLUGIN_ROOT`, and honours `CLAUDE_PLUGIN_ROOT` | none | `CURSOR_PLUGIN_ROOT`, and `CLAUDE_PLUGIN_ROOT` in MCP |
| Plugin `dependencies` | resolved (see below) | ignored | — | — |

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
does not reject unknown fields. Installation copies the tree into a cache
under `CODEX_HOME` and writes enablement into user config, so a fresh clone
is never self-contained.

What a Claude-format tree gets on Codex:

- **Skills and hooks work.** Hook discovery injects `PLUGIN_ROOT`,
  `CLAUDE_PLUGIN_ROOT`, `PLUGIN_DATA` and `CLAUDE_PLUGIN_DATA`, so Claude
  hook commands run unchanged.
- **Agents are not loaded.** Codex has no plugin agent component; subagents
  come from host TOML configuration. The agent files survive in the cache and
  do nothing. A plugin also cannot install loose `.codex/agents/*.toml` into a
  user's project. Mars prints one warning that Codex plugin users get skills
  and hooks but not agents; users who want the agents install with mars.
- **MCP servers are registered but not callable** in 0.164.0.
- **`commands/*.md` are migrated** into generated skills under
  `.codex-plugin/migrated-command-skills/`.
- **Plugin `dependencies` are ignored**, whether declared in `plugin.json`
  or in the catalog entry. Installing the top plugin installs only it.

A Codex-only hook (a `codex.json` fragment with no Claude fragment) cannot
ride the shared tree; it would need a Codex-own tree with
`.codex-plugin/plugin.json` and `${PLUGIN_ROOT}`. When both catalogs list the
same plugin name, which one Codex prefers is unprobed.

## OpenCode

OpenCode's plugin is executable JS: a `package.json` plus a default export
whose `setup` can register skills, commands and MCP from bundled files under
`pkg:name` ids, locating its files through `import.meta.url`. A plugin
directory of `skills/`, `agents/` and `commands/` is not read, and the agent
editor has no `add`, so a plugin cannot create agents. No plugin-root
variable exists. OpenCode already reads `.claude/skills` and `.agents/skills`
from projects, so mars-installed users have skills without a plugin. A
project `opencode.json` may point at a relative plugin path, which loads in
place from a fresh home. Mars's generated `index.mjs` imports each targeted
hook from an in-memory TypeScript data URL and isolates every load/setup call.
The build's default-export recognizer is advisory: an unrecognized fragment is
warned about but still ships for runtime validation.

## Cursor

No `cursor-agent` binary was available, so everything here is from
documentation and a CLI changelog claim that Cursor imports Claude plugins.
Cursor's own format is `.cursor-plugin/plugin.json` with skills, rules,
agents, commands, hooks and MCP. Hooks receive `CURSOR_PLUGIN_ROOT`; MCP
expands `CURSOR_PLUGIN_ROOT` and `CLAUDE_PLUGIN_ROOT` but not
`PLUGIN_ROOT`. Mars emits that documented native tree but labels Cursor
plugin support docs-only until a live probe confirms the loader behavior.

## Sources

Synthesized from the probe and source-study reports of
`work:harness-plugin-export` (archived with the work item):
`spawn:p9291` (install probe across harnesses), `spawn:p9290` (Codex and
OpenCode loader source at rust-v0.164.0 and v2.1.3), `spawn:p9288`
(published formats), `spawn:p9297` (OpenCode JS registration),
`spawn:p9305` (project-local enablement), `spawn:p9328` (Claude
cross-marketplace dependencies), `spawn:p9329` (Claude agent `skills:`
preload), `spawn:p9340` (same-marketplace dependencies). Claude plugin
references: [plugin components](https://code.claude.com/docs/en/plugins/components),
[subagent frontmatter](https://code.claude.com/docs/en/sub-agents#frontmatter-reference).
