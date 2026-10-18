# Agent Profiles

An **agent profile** is a Markdown file in `.mars/agents/` that declares a
named agent's behavioral defaults: model, harness, skills, approval mode, and
other runtime settings. The file body becomes part of the system prompt. The
YAML frontmatter provides the overrides that feed into Pass 2 of the resolution
merge.

## Loading

`scan_agent_profiles()` walks `.mars/agents/*.md` in each content root: the
primary Mars root (the nearest `mars.toml` at or above the project root, else
the user root `~/.meridian/mars/`), then the user root when the installed Mars
supports `--fallback-root` ([Mars user scope](../../architecture/mars-user-scope.md)):

1. Reads all `.md` files in each root's `.mars/agents/`; the primary root's
   profile shadows a user profile of the same name
2. Parses YAML frontmatter + body for each file
3. Duplicate names: the first profile seen for a name wins, without a warning

Source: `scan_agent_profiles()` in `src/meridian/lib/catalog/agent.py`

`load_agent_profile_with_fallback()` selects a profile via:

1. Explicit requested agent name (from `-a <name>`)
2. Configured default agent (`config.default_agent`, usually `__meridian-subagent`)
3. No profile (degraded config — no system prompt body, no skills)

A missing default profile emits a warning but doesn't fail the spawn.
Source: `src/meridian/lib/launch/resolve.py:19-60`

## Profile Schema

```yaml
---
name: coder
description: "Implements code changes"
model: sonnet            # alias or concrete model ID
harness: claude          # optional; overrides model-derived harness
skills:                  # or a flat list, meaning all load
  load: [meridian-spawn, kb-conventions]
  available: [diagnose]
approval: auto           # default | confirm | auto | yolo
effort: high             # low | medium | high
mode: subagent           # primary | subagent
model-policies:
  - match:
      alias: gpt55
    override: {}          # participates in fallback, no overrides
  - match:
      alias: codex
    override:
      effort: high
---

# Coder Agent

You implement code changes...
```

The profile's `model` field is an alias or bare model ID, not a harness.
For catalog lookups, `resolve_model()` turns it into a concrete `AliasEntry`.
For PRIMARY/SPAWN_PREPARE launches, the profile name is forwarded to Mars via
the bundle request and Mars resolves the model internally.

## Frontmatter Fields

### `mode: primary | subagent`

Declares whether the agent runs as a primary interactive session agent or as a
subagent spawned by an orchestrator. Affects grouping in `meridian list` output.
Default: `subagent`.

- `primary` — e.g., `__meridian-orchestrator`
- `subagent` — e.g., `coder`, `reviewer`, `kb-writer`

### `model-invocable: true | false`

Controls whether this agent appears in the model-facing agent inventory prompt
(the `# Meridian Agents` system prompt block). Default: `true`.

- `true` (default) — agent is listed in the inventory prompt; the model can
  suggest or invoke it.
- `false` — agent is excluded from the inventory prompt. The model never sees
  it in the menu. Explicit invocation via `meridian spawn -a <name>` still works;
  this field does not restrict CLI users.

The filter runs when Mars renders the launch-bundle
`prompt_surface.inventory_prompt`, not in `scan_agent_profiles()`. The catalog
scanner returns all profiles regardless of this field; only the model-facing
inventory renderer applies the visibility rule. Meridian embeds the Mars-rendered
inventory string verbatim and has no Python fallback renderer.

Use `model-invocable: false` for internal orchestration agents, deprecated agents,
or any agent that should not appear in the model's awareness.

The rendered inventory is persisted on the launch policy snapshot as
`bundle_inventory_prompt` so resume/replay keeps the same model-facing menu without
re-calling Mars.

See [decisions/launch.md](../../decisions/launch.md#d-model-invocable-filter-at-inventory-prompt-boundary-not-at-catalog-scan)
for the rationale behind the filter-seam choice.

### `model-policies: [...]`

Typed selector rules that apply overrides when a specific model or alias is
selected, and declare fallback candidates for harness-availability fallback.
Rules participate in fallback by default (for `model` and `alias` match types).
Set `no-fallback: true` on a rule to exclude it from fallback while still
applying its overrides. List order determines fallback priority.

See [model-policies.md](model-policies.md) for the full match/override semantics.

## Profile vs Direct Model

When both `-a profile` and `-m model` are passed, the model override wins per
config precedence (CLI flags beat profile frontmatter). This lets you use a
profile's skills and system prompt body while routing to a different model:

```bash
meridian spawn -a coder -m gpt55   # coder's skills + prompt, gpt55 model
```

## Harness Availability Fallback

Profile `model-policies` rules with `model` or `alias` match types can
participate in harness-availability fallback by default. `no-fallback: true`
opts a rule out, and `model-glob` rules never participate.

For PRIMARY/SPAWN_PREPARE launches, harness-availability fallback is handled
by Mars within the bundle. The `model-policies` rule schema (match type, list
order, `no-fallback` flag) controls which rules Mars treats as fallback candidates.

See [model-policies.md](model-policies.md#fallback-participation) for the
candidate semantics and the
[target-constrained fallback decision](../../decisions/model-resolution.md#target-constrained-model-policy-fallback)
for the decision rationale.

## Primary Agent

`config.primary_agent` is the same concept as `default_agent` but for primary
interactive launches (not subagent spawns). It defaults to
`__meridian-orchestrator`.

## Skill Attachment

A profile's `skills:` is a flat list (every name loaded) or a mapping with
`load` and `available` lists. Mars reads it from the profile file, adds any
`--skills` names to `load`, and returns the launch bundle's `skills.loaded`
(body with the harness variant applied), `skills.available` (name, type, description) and
`skills.missing`. Meridian composes the loaded bodies into the system prompt on
every launch, so they survive session compaction. Missing skills produce a
warning, not a launch failure.

A loaded skill arrives in full even when it is `model-invocable: false`. See
[skill-loading.md](../skill-loading.md) for what load, available and
`model-invocable` each control, and
[composition-pipeline.md](../composition-pipeline.md) for where skill documents
sit in the prompt.

Mars is the only reader of a profile's `skills`. Meridian's `AgentProfile` has
no skills field, and Meridian sends Mars only the caller's `--skills` names. An
exact continue replays the skill bodies stored in its launch snapshot rather than
looking skills up again; a snapshot without them replays with no loaded skills
([D-mars-sole-skill-resolver](../../decisions/package-management.md#d-mars-sole-skill-resolver-meridian-never-resolves-skills)).

## Related

- [overview.md](overview.md) — full bundle-based resolution pipeline
- [aliases-and-routing.md](aliases-and-routing.md) — how the profile's model
  name is resolved to a concrete model ID and harness
- [model-policies.md](model-policies.md) — `model-policies` rule semantics
- [concepts/skill-loading.md](../skill-loading.md) — load, available and `model-invocable`
- [concepts/skill-schema.md](../skill-schema.md) — skill frontmatter and variant selection
- [architecture/launch-system.md](../../architecture/launch-system.md) — where
  profile loading sits in the launch factory
