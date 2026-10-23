# Decisions: Launch-Bundle Field Consumption

Every field of the Mars launch bundle is either applied at launch, reported as
not applied for the target harness, or deliberately unread with the reason
written down. No field is dropped silently, and no field crashes a launch. This
page records that rule and the decisions it forced for `harness-overrides`
passthrough, compaction thresholds and MCP names.

**Status:** owner rulings of 2026-10-22 (`work:session-permissions`), made
alongside [Profile tool permissions](tool-permissions.md).

**Where the mechanism lives.** The unread list and its reasons: the module
docstring of meridian-cli `src/meridian/lib/launch/bundle_adapter.py`. Codex
passthrough encoding and the reserved-key list:
`src/meridian/lib/harness/projections/native_config.py`. Compaction:
`projections/compaction.py`. User-facing tables: `docs/agent-profiles.md`,
sections "Compaction by harness" and "Harness-native passthrough". The bundle
schema: [Mars launch bundle](../architecture/mars-launch-bundle.md).

## D-bundle-field-accounted: applied, reported, or deliberately unread {#d-bundle-field-accounted}

**Decided (owner):** each bundle field falls in exactly one of three classes.

| Class | What happens | Where it shows |
|---|---|---|
| Applied | Projected into the harness's flags, env or config | `--dry-run` command and report line `native` |
| Reported | The harness has no channel; the launch proceeds without it | Report line `[unsupported] ... not applied for <Harness>` |
| Deliberately unread | Meridian has its own source for the same thing | Listed with its reason in `bundle_adapter.py` |

Reports use the same `EnforcementReport` the permission compile returns, so a
user checks one place, `--dry-run`, for everything a profile asked for and did
not get. As with permission gaps, nothing is a launch warning.

**Deliberately unread:** `agent_body`, `prompt_surface.system_instruction`,
`supplemental_documents` and `scaffold_slots`, because Meridian composes the
prompt itself from the local profile, `skills.loaded` and its own context
blocks; the flat `tools.allowed`/`disallowed`/`mcp` lists, because
`tools.rules` replaces them. Meridian does read
`prompt_surface.inventory_prompt`.

**Why:** to a profile author, a field Meridian drops looks exactly like one that
works. Two did: `harness-overrides` reached Meridian as
`execution_policy.native_config` and was never applied on any harness, and
`autocompact` went into the wrong Claude variable. The three classes make a
dropped field visible in `--dry-run` instead.

## D-native-config-codex-only: `harness-overrides` applies on Codex {#d-native-config-codex-only}

**Decided (owner):** `execution_policy.native_config` (the
`harness-overrides.<harness>` block Mars selects for the resolved launch
harness) is applied on Codex as repeated `-c
key=value` overrides on both the subprocess and app-server launch paths. Nested
mappings flatten to dotted keys and values are written as TOML literals; dotted
keys under one table combine. On every other harness the whole block is
reported not applied.

What Codex skips, each with its own report line:

- **Keys Meridian computes** (model, reasoning effort, compaction thresholds,
  sandbox mode, approval policy, writable roots, notify, web search, and the
  `features.*` and `agents.*` tables). The block may add siblings to these but
  never replaces them; otherwise a passthrough could undo the profile's
  sandbox or tool switches. The deprecated
  `sandbox_workspace_write.network_access` key is a one-release exception: its value is
  resolved once during Codex permission compilation, a boolean still wins with a warning,
  and a non-boolean warns without being treated as allow. The next release reserves it.
- **Keys `-c` cannot address:** any component of the dotted key path that
  contains a dot, `=`, quote, whitespace or control character.
- **Values outside what Codex `-c` takes:** Meridian encodes booleans, int64
  integers, floats, strings, arrays and mappings; anything else (such as null),
  integers outside int64, and empty mappings are reported.

**Trust model:** a block applies as written and can change any Codex setting
that is not reserved, including model providers and MCP server commands. A
package that carries a `harness-overrides` block is trusted code, the same as
any hook it installs.

**Evidence:** a live probe showed the deprecated `-c
sandbox_workspace_write.network_access=true` enables network under `workspace-write`; the
portable top-level `network` field now owns the replacement projection.

**Rejected (2026-10-22):** a first-class `network` profile field. Only Codex and Claude's
sandbox have a network knob and they are spelled differently; passthrough
covers the need.

> **Superseded (2026-10-23):** the owner decided on a `network` profile field, implemented
> on the unmerged combined branch. See
> [sandbox-platform-access.md](sandbox-platform-access.md#d-network-field). The passthrough
> key survives for one warning release only.

## D-compaction-projected: thresholds reach each harness's own knob {#d-compaction-projected}

**Decided (owner):** `autocompact` is a token count and `autocompact_pct` a
percentage, and each goes to the harness setting of the same kind.

| Harness | `autocompact` (tokens) | `autocompact_pct` (percent) |
|---|---|---|
| Claude | `--autocompact`, for 100000–1000000; outside that range, reported | `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` |
| Codex | `-c model_auto_compact_token_limit` | `-c model_post_turn_compact_threshold_percent` |
| OpenCode, Pi, Cursor | reported not applied | reported not applied |

This ends GitHub #548, where the token count went into Claude's percentage
variable and `autocompact_pct` was never applied. Claude profiles that set
`autocompact` change behavior: they now set the token window, explained in
meridian-cli `docs/agent-profiles.md`, "Compaction by harness".

OpenCode has no `effort` channel in its TUI and streaming launches, so `effort`
is reported not applied there. Whether a model variant still reaches the
request is unverified; tracked in meridian-cli
`src/meridian/lib/harness/.context/FUTURE`.

## D-mcp-names-not-servers: `tools.mcp` is permission names {#d-mcp-names-not-servers}

**Decided (owner):** the bundle's `tools.mcp` holds Mars's allowed MCP
permission names, never server definitions. Meridian had been passing those
names to Claude as `--mcp-config` and to Codex as `name=command` server
entries, which configured nothing real. The whole Meridian `mcp_tools`
mechanism (request field, spec field, per-harness projections) is deleted.
`LaunchPolicySnapshot.mcp_tools` remains so older snapshots load, and replay
ignores it. MCP permissions travel as `mcp(...)` rules in `tools.rules` like any
other capability.

**Not built:** per-launch MCP server injection (for example on OpenCode). It
needs a new Mars bundle field that carries real server definitions.

**Mars side:** the `codex_rules` bundle field is deleted. Mars always emitted it
as null and no consumer read it; Meridian ignores unknown bundle fields, so an
older bundle that still carries it parses.

## Provenance

`work:session-permissions`: `design/implementation-decisions.md`, section
"Dropped bundle fields, `native_config`, compaction" (owner, 2026-10-22);
`p8931-bundle-fields.md`. Reviews: `spawn:p8937` (request changes:
`mcp_tools` half removed, dotted keys, floats), `spawn:p8943` (approve with
notes: DEL, int64, control characters). meridian-cli commits `90bf24c71`,
`1943f1327`, `c316960c0`, `b60cf7b24`; mars-agents `90a9d28`, `d7cea78`.
