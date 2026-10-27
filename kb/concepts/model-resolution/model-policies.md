# Model Policies

`model-policies` are typed selector rules in agent profile frontmatter or
`[agents.<name>]` overlays. They apply behavioral overrides when a specific
model or alias is selected for a spawn and can contribute fallback candidates.
They do not control which rows appear in the model inventory.

## model-policies Rules

Agent profile frontmatter can declare a list of `model-policies` entries. A
project/user config overlay (`[agents.<name>]`) can inherit, suppress, or prepend
those entries for **settings matching** without editing generated `.mars/agents/`
files. Launch backups still come only from the profile list. When the resolved
model matches a rule's selector, the rule's overrides are applied.

```yaml
model-policies:
  - match:
      alias: gpt55          # exact alias match
    override:
      effort: medium
  - match:
      model: claude-opus-4  # exact model ID match
    override:
      effort: high
  - match:
      model-glob: "gpt-4*"  # glob pattern match
    override:
      sandbox: read-only
```

### Match Types

Three selector forms, evaluated in priority order:

| Match type | Example | How it matches |
|---|---|---|
| `model` | `claude-opus-4` | Exact match against canonical model ID |
| `alias` | `gpt55` | Exact match against the alias string |
| `model-glob` | `gpt-4*` | Glob match against canonical model ID |

Priority: exact `model` > exact `alias` > `model-glob`. Within the same rank,
matching is **first-match wins** — the first rule in list order that matches is
applied. Duplicate same-specificity rules (e.g., two `alias: gpt55` rules) do not
raise a parse error; later rules are simply unreachable (silently shadowed).

### Override Fields

Under Mars schema v1 (mars-agents 0.20 and later, Meridian v0.15.0 and later) an
`override` sets only `effort`, `approval`, `sandbox` and `network`, in a profile and in an
overlay alike. Mars rejects `autocompact`, `autocompact_pct` and `harness` in an overlay
override with a load error, and `mars migrate` deletes them from profiles: compaction now
comes from the `[models.<alias>]` entry, and the harness from the profile or the alias.
meridian-cli `docs/upgrading.md` ("Mars 0.20.0 (schema v1)") lists each removed key and
its replacement.

See [D72](../../decisions/model-resolution.md#d72-agentsname-overlay-config-as-the-project-scoped-override-surface)
and [future work](../../open-questions/future-work.md#agent-overlay-follow-ups-2026-05-05-after-agent-model-overrides-shipped).

### Fallback Participation

Rules with `model` or `alias` match type participate as **harness-availability
fallback candidates** by default. When the primary harness is unavailable, the
runtime walks model-policies rules in list order and tries each eligible rule's
`match_value` as a fallback model token.

  - `no-fallback: true` on a rule opts it out of fallback. The rule still applies
  its overrides when matched during primary launch resolution; it just won't be
  tried as a fallback candidate.
  - `model-glob` rules are **always** `no-fallback: true`. Glob rules apply
  overrides during primary launch resolution only and never become fallback
  candidates.
- An empty `override: {}` is valid for a candidate-only rule (backup without
  extra settings).

### Overlay Prepend Semantics

`[agents.<name>]` overlays use a three-state model for `model-policies`:

| Overlay state | Effective policy source |
|---|---|
| Key absent | Inherit profile `model-policies` unchanged |
| `model-policies = []` | Suppress all conditional model overrides |
| One or more entries | Overlay rules **prepend** to profile rules for settings matching |

When an overlay provides one or more entries, those rules are tried first for
**settings**. Profile rules apply for tokens no overlay rule matched. Overlay
and global rules are not extra launch backups; Mars still scans the profile list
in declaration order for fallback candidates.

Use `model-policies = []` to fully suppress all profile rules when needed.

### Where model-policies Fit in the Cascade

For **policy fields** (`effort`, `approval`, `sandbox`, `network`), a matched rule sits
below explicit user overrides and `[agents.<name>]` overlay scalars, and above the
profile's own top-level values:

```
CLI flags > env vars > agent overlay scalar > matched model-policy rule > profile top-level value > config default > alias default
```

For PRIMARY/SPAWN_PREPARE launches, model-policies are evaluated by Mars within
the bundle. Execution-policy overrides from matched rules are returned in the bundle
payload and applied post-bundle in Meridian's `_resolve_bundle_execution_policy()`.
See [config-precedence.md](../config-precedence.md#routing-vs-policy-fields).

For prelaunch fallback, Mars scans concrete profile rules in declaration order,
independent of the primary's matched rule, index or settings layer. A per-entry
`no-fallback` flag skips that entry, never the remaining search. Alias entries
retain alias resolution; model entries retain literal identity. There is no
policy-transform demotion step or separate fanout backup list.

Enabled targets constrain eligible harnesses before auth and model support checks.
Explicit model pins disable model backups; explicit harness pins only that dimension.
See [the decision](../../decisions/model-resolution.md#target-constrained-model-policy-fallback)
for the reason primary settings and backup enumeration are separate.

## Model Inventory Is a Separate Concern

Mars separates Possible discovery, display-only Curated rules, and Selection
routing. `model-policies` participate in Selection; Curated rules affect only
the human-facing `models list` view and never filter aliases, resolve results,
or launch-bundle routes. See the [model-catalog decision](../../decisions/model-resolution.md#model-catalog-layers-2026-09)
for command responsibilities and cross-repo boundaries.

## Migration: fanout → model-policies

The `fanout:` key has been removed from the profile schema. Profiles containing
`fanout:` fail at parse time with a descriptive error. Express fallback candidates
through `model-policies` list position instead. Rules participate in fallback by
default; `no-fallback: true` opts out. `model-glob` rules are automatically excluded.

Similarly, `fallback-order:` within a `model-policies` rule is removed. Fallback
order is now implicit from list position.

Old:
```yaml
fanout:
  - alias: gpt55
  - alias: codex
```

New:
```yaml
model-policies:
  - match: {alias: gpt55}
    override: {}        # participates in fallback, no overrides
  - match: {alias: codex}
    override: {}
```

## Related

- [overview.md](overview.md) — full resolution pipeline
- [aliases-and-routing.md](aliases-and-routing.md) — where model-policies sit
  in the Mars bundle routing path
- [agent-profiles.md](agent-profiles.md) — how model-policies are declared in
  profile frontmatter
- [../config-precedence.md](../config-precedence.md) — where the agent overlay
  tier sits in the runtime override ladder
- [decisions/model-resolution.md](../../decisions/model-resolution.md) — why
  model-policies replaced the legacy `models:` mapping and overlay prepend semantics
