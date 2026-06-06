# Agent Profiles

An **agent profile** is a Markdown file in `.mars/agents/` that declares a
named agent's behavioral defaults: model, harness, skills, approval mode, and
other runtime settings. The file body becomes part of the system prompt. The
YAML frontmatter provides the overrides that feed into Pass 2 of the resolution
merge.

## Loading

`scan_agent_profiles()` walks `.mars/agents/*.md` at project root:

1. Reads all `.md` files in `.mars/agents/`
2. Parses YAML frontmatter + body for each file
3. Duplicate names: identical-content duplicates are silently deduped; conflicting
   duplicates log a warning and first-seen wins

Source: `src/meridian/lib/catalog/agent.py:425-463`

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
skills:
  - meridian-spawn
  - kb-conventions
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
candidate-chain semantics and
[D75](../../decisions/model-resolution.md#d75-candidate-chain-semantics-transform-and-demotion-not-hidden-scan)
for the decision rationale.

## Primary Agent

`config.primary_agent` is the same concept as `default_agent` but for primary
interactive launches (not subagent spawns). It defaults to
`__meridian-orchestrator`.

## Skill Attachment

Skills listed in the profile's `skills:` field are loaded at spawn time from
`.mars/skills/*/SKILL.md`. They're re-injected fresh on each launch, so they
survive session compaction.

`resolve_skills_from_profile()`:
1. Builds a `SkillRegistry` from the project's `.mars/skills/` directory
2. Resolves variant selection for each skill based on resolved harness and model token
3. Returns `skill_names`, `loaded_skills`, and `missing_skills`

Missing skills produce a warning, not a launch failure.
Source: `src/meridian/lib/launch/resolve.py:71-109`

See [concepts/skill-schema.md](../skill-schema.md) for how variants are selected,
and [concepts/composition-pipeline.md](../composition-pipeline.md) for how skill
content gets routed into the prompt.

## Related

- [overview.md](overview.md) — full bundle-based resolution pipeline
- [aliases-and-routing.md](aliases-and-routing.md) — how the profile's model
  name is resolved to a concrete model ID and harness
- [model-policies.md](model-policies.md) — `model-policies` rule semantics
- [concepts/skill-schema.md](../skill-schema.md) — skill variant selection ladder
- [architecture/launch-system.md](../../architecture/launch-system.md) — where
  profile loading sits in the launch factory
