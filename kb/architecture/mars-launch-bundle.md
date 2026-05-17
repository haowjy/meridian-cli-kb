# Mars Launch-Bundle System

Cross-repo system spanning `mars-agents` (builds the scaffold) and `meridian-cli`
(injects per-spawn content and launches the harness). The public verb is `build`
(`mars build launch-bundle`).

**Related pages:**
- [mars-compiler.md](mars-compiler.md) — compiler internals, config-entry pipeline
- [mars-targeting.md](mars-targeting.md) — static `.mars/` targeting vs. launch-bundle
- [../concepts/native-config.md](../concepts/native-config.md) — native-config passthrough concept
- [../decisions/package-management.md](../decisions/package-management.md) — D80–D85 for decisions made during this work
- [../lessons/mars-launch-bundle-lessons.md](../lessons/mars-launch-bundle-lessons.md) — implementation lessons from the launch-bundle work item

---

## Ownership Boundary

**Mars builds the static scaffold.** At `mars build launch-bundle` time, Mars has
the compiled agent/skill graph. Mars cannot see per-spawn content — prompt files,
context files, goal text, prior session history — because those are runtime concerns
owned by Meridian.

**Meridian injects per-spawn dynamic content and launches.** Meridian reads the
bundle, injects prompt + context files + goal + session history into scaffold slots,
then starts the harness process.

```mermaid
graph TD
    PROFILE["Agent profile\nmars.toml + source"]
    BUILD["mars build launch-bundle\n--agent name --harness target"]
    BUNDLE["JSON launch scaffold\nstatic, harness-targeted"]
    INJECT["Meridian injects\nprompt file, context files,\ngoal, session history,\nspawn metadata"]
    LAUNCH["Harness process\nclaude / codex / opencode / ..."]

    PROFILE --> BUILD
    BUILD --> BUNDLE
    BUNDLE --> INJECT
    INJECT --> LAUNCH
```

**Mars never receives the prompt file.** The prompt file is per-spawn confidential
content. It enters the flow only at Meridian injection time.

---

## Two-Phase Model

| Phase | Owner | Input | Output |
|-------|-------|-------|--------|
| `mars build launch-bundle` | Mars | agent identity, target harness, launch options | JSON scaffold (bundle) |
| Meridian injection + launch | Meridian | bundle + prompt file + context files + goal + session history | running harness process |

Mars produces a bundle with scaffold slots — placeholder markers where Meridian will
inject per-spawn content before assembling the final system prompt.

---

## Bundle Structure

Top-level fields in the JSON schema (version 1):

| Field | Type | Description |
|-------|------|-------------|
| `version` | integer | Schema version (currently `1`) |
| `agent` | string | Resolved agent name |
| `routing` | object | `model`, `model_token`, `harness` |
| `execution_policy` | object | Portable execution settings + `native_config` |
| `prompt_surface` | object | System instruction + supplemental documents |
| `scaffold_slots` | object | Placeholder positions for Meridian injection |
| `tools` | object | Resolved portable tool policy (`allowed`, `disallowed`, `mcp`) |
| `skills_metadata` | object | Skill loading metadata |
| `provenance` | object | Source attribution for each resolved field |
| `warnings` | string[] | User-visible warnings emitted during build |

### `execution_policy` key fields

```json
{
  "execution_policy": {
    "effort": "high",
    "approval": "auto",
    "sandbox": "workspace-write",
    "autocompact": null,
    "autocompact_pct": 80,
    "timeout": null,
    "native_config": {
      "sandbox_workspace_write.network_access": true
    }
  }
}
```

`native_config` is omitted (`skip_serializing_if = None`) when no native config
is declared. See [../concepts/native-config.md](../concepts/native-config.md) for
full semantics.

### `tools` field

```json
{
  "tools": {
    "allowed": ["Bash", "Bash(meridian spawn *)", "Bash(meridian session *)"],
    "disallowed": ["Agent", "Edit", "Write"],
    "mcp": ["plugin:context7:context7"]
  }
}
```

Both `allowed` and `disallowed` may be non-empty simultaneously. An empty list
means no entries of that kind — not "wildcard everything." Mars preserves both
sides; Meridian projects both sides per harness.

---

## Scaffold Slots

Mars emits placeholder markers into the assembled system prompt so Meridian can
inject per-spawn content at the correct positions:

| Slot | Placeholder | Content injected by Meridian |
|------|------------|------------------------------|
| Prompt | `__MERIDIAN_PROMPT__` | Prompt file content (user/delegator work instruction) |
| Context files | `__MERIDIAN_CONTEXT_FILES__` | `-f` files passed at spawn time |
| Goal | `__MERIDIAN_GOAL__` | `--goal` completion contract text |
| Prior session | `__MERIDIAN_PRIOR_SESSION__` | Prior session context (`--from`) |

Mars never populates these slots — it only marks where they go. Meridian replaces
each placeholder with actual content or removes it if the slot is unused for a given
spawn.

---

## Harness Support Status

| Harness | Status | Notes |
|---------|--------|-------|
| Claude | First-class | Full native-config (`--settings`), tool policy (`--allowedTools`/`--disallowedTools`), full bundle support |
| Codex | First-class | Full bundle; native-config via `-c` flags; tool allow/deny not projected (warns) |
| OpenCode | First-class | Full bundle; native-config and tool policy via `OPENCODE_CONFIG_CONTENT` |
| Cursor | Experimental | Bundle supported; `provenance.harness_stability: "experimental"` + warning; projection best-effort; contract may change |
| Pi | Future | Pi contract still being developed; not in this slice; tool policy should be part of the future Pi contract |
| Gemini | Out of scope | Not a current Mars/Meridian target |

---

## Static Sync vs. Launch-Bundle: Intentional Divergence

Different build products for different consumers:

| | `mars sync` (static) | `mars build launch-bundle` (runtime) |
|---|---|---|
| **Consumer** | Harness-native agent discovery (e.g., Claude Code sidebar) | Meridian's runtime spawn flow |
| **Output** | `.mars/agents/*.md`, `.claude/agents/*.md`, etc. | JSON scaffold |
| **`native-config`** | Dropped from native artifacts (meridian-only in lossiness matrix) | Preserved in `execution_policy.native_config` for Meridian runtime projection |
| **Tool policy** | Preserved for harnesses that support it in native artifacts | Preserved in `tools` field for per-harness projection |

The static path serves harness-native discovery; the launch-bundle path serves Meridian.
The harness does not know how to apply `native-config` — that's Meridian's job.

---

## Projection Architecture

Two independent projection paths in Meridian:

```mermaid
graph TD
    BUNDLE["Launch bundle"]
    NC["execution_policy.native_config"]
    TP["tools allowed/disallowed/mcp"]

    NC_PROJ["Native-config projector\nper harness"]
    TP_PROJ["Tool-policy projector\nper harness"]

    MERGE["merge_projections()"]
    LAUNCH["Harness launch invocation\nargs + env + temp files"]

    BUNDLE --> NC --> NC_PROJ
    BUNDLE --> TP --> TP_PROJ
    NC_PROJ --> MERGE
    TP_PROJ --> MERGE
    MERGE --> LAUNCH
```

Results are merged: args concatenated, env vars deep-merged (native-config first,
tool-policy second for overlapping env vars), temp files tracked for cleanup,
warnings surfaced to user.

**Per-harness native-config projection:**

| Harness | Strategy |
|---------|----------|
| Codex | Repeated `-c <key>=<toml-value>` CLI flags |
| Claude | Temp settings JSON file, `--settings <path>` |
| OpenCode | Deep-merged into `OPENCODE_CONFIG_CONTENT` env var |
| Cursor | `mcp` key → `.cursor/mcp.json`; unknown keys warn |

**Per-harness tool-policy projection:**

| Harness | Strategy |
|---------|----------|
| Claude | `--allowedTools` + `--disallowedTools` (both emitted when non-empty) |
| Codex | Warning emitted; tool-level allow/deny not projected |
| OpenCode | Permission JSON via `OPENCODE_CONFIG_CONTENT` overlay |
| Cursor | Warning emitted; MCP tools via `.cursor/mcp.json` |

---

## Prompt-Surface Isolation Invariant

**`native-config` entries must never appear in `prompt_surface`, supplemental documents,
or any scaffold slot content.** Enforced structurally:

- **Mars side:** native-config extraction and prompt composition are separate code paths.
  Native config → `execution_policy.native_config`; prompt surface → `prompt_surface`/`scaffold_slots`.
- **Meridian side:** config projector reads `execution_policy.native_config`; content
  projector reads `prompt_surface` and fills scaffold slots. Separate projectors,
  no shared output.

Model context never contains raw harness config.

---

## Compatibility

`native_config` is optional and additive — existing bundles without it remain valid.
Cursor is an additive harness enum variant. Neither requires a version bump.

Meridian validates the version field at consumption time:

```python
if bundle.version > SUPPORTED_MAX_VERSION:
    raise IncompatibleBundleVersion(bundle.version, SUPPORTED_MAX_VERSION)
```
