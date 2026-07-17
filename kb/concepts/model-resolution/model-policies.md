# Model Policies

Model policies are two distinct mechanisms that both use the word "policy":

1. **`model-policies` rules** in agent profile frontmatter or
   `[agents.<name>]` overlays — typed selector rules that apply behavioral
   overrides when a specific model or alias is selected for a spawn.
2. **Model visibility policy** — filtering rules that determine which models
   appear in `meridian models list` output.

## model-policies Rules

Agent profile frontmatter can declare a list of `model-policies` entries. A
project/user config overlay (`[agents.<name>]`) can inherit, suppress, or prepend
to those entries without editing generated `.mars/agents/` files. When the resolved
model matches a rule's selector, the rule's overrides are applied.

```yaml
model-policies:
  - match:
      alias: gpt55          # exact alias match
    override:
      harness: codex
      effort: medium
  - match:
      model: claude-opus-4  # exact model ID match
    override:
      effort: high
  - match:
      model-glob: "gpt-4*"  # glob pattern match
    override:
      harness: codex
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

Supported override fields in **profile** `model-policies`:

| Field | Type | Effect |
|---|---|---|
| `harness` | string | Override the harness for this model |
| `effort` | string | Override the effort level |
| `approval` | string | Override the approval mode |
| `sandbox` | string | Override the sandbox setting |
| `autocompact` | int | Override the autocompact threshold |
| `timeout` | int | Override the timeout |

List fields (`skills`, `tools`, `disallowed-tools`, `mcp-tools`) are parsed and
warn on load but are **not yet applied** to the spawn.

Source: `src/meridian/lib/catalog/agent.py:24-29, 68-76`

In **agent overlay** `model-policies`, v1 supports the same scalar policy fields
except `timeout`. Overlay `timeout` is rejected because the overlay compiler does
not yet thread resolved timeout into `execution_policy.timeout`; accepting it
would create a "parses but does nothing" setting. Overlay list fields
(`skills`, `tools`, `disallowed-tools`, `mcp-tools`) are also rejected with a
warning rather than accepted-but-ignored.

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
- An empty `override: {}` is valid for a fallback-only rule (one that participates
  in the fallback chain without applying any overrides).

### Overlay Prepend Semantics

`[agents.<name>]` overlays use a three-state model for `model-policies`:

| Overlay state | Effective policy source |
|---|---|
| Key absent | Inherit profile `model-policies` unchanged |
| `model-policies = []` | Suppress all conditional model overrides |
| One or more entries | Overlay rules **prepend** to profile rules |

When an overlay provides one or more entries, those rules are evaluated first
(highest priority). Profile rules apply for tokens no overlay rule matched. This
lets overlays add or override specific model routing without replacing the entire
profile policy list.

Use `model-policies = []` to fully suppress all profile rules when needed.

### Where model-policies Fit in the Cascade

For **policy fields** (`effort`, `approval`, `sandbox`, `autocompact`, `timeout`),
`model-policies` rules are tier 3 — above agent overlay generic defaults and
profile generic defaults, but below explicit CLI/env user overrides:

```
CLI flags > env vars > effective model-policy rule > agent overlay default > profile generic default > config default > alias default
```

A matched `model-policies` harness override is forwarded to Mars via the
launch-bundle request and affects the harness Mars selects. Routing-field resolution
is distinct from the policy cascade above.

For PRIMARY/SPAWN_PREPARE launches, model-policies are evaluated by Mars within
the bundle. Execution-policy overrides from matched rules are returned in the bundle
payload and applied post-bundle in Meridian's `_resolve_bundle_execution_policy()`.
See [config-precedence.md](../config-precedence.md#routing-vs-policy-fields).

For harness-availability fallback, model-policies rules with `no-fallback != true`
and `match_type` of `model` or `alias` serve as ordered fallback candidates. The
pre-transform base candidate (if a policy rule matched) is demoted and tried first;
then model-policies rules are walked in list order.
See [D75](../../decisions/model-resolution.md#d75-candidate-chain-semantics-transform-and-demotion-not-hidden-scan)
for the decision rationale.

## Model Visibility Policy

`ModelVisibilityConfig` controls which models appear in `meridian models list`
default output (not `--all`):

```python
class ModelVisibilityConfig(BaseModel):
    include: tuple[str, ...] = ()
    exclude: tuple[str, ...] = (
        "*-latest",
        "*-deep-research",
        "gemini-live-*",
        "o1*", "o3*", "o4*",
    )
    max_input_cost: float | None = 10.0   # $/M tokens
    max_age_days: int | None = 120
    hide_date_variants: bool = True
    hide_superseded: bool = True
```

Source: `src/meridian/lib/catalog/model_policy.py:13-39`

`is_default_visible_model()` applies these filters in order:

1. **Include/exclude globs** — `include` patterns must match; `exclude` patterns
   must not match
2. **Date variant hiding** — models with a date suffix (e.g., `claude-3-5-sonnet-20241022`)
   are hidden if the base model (`claude-3-5-sonnet`) also exists in the catalog
3. **Superseded hiding** — older models in the same provider+lineage group are
   hidden
4. **Age cutoff** — models older than `max_age_days` from their `release_date`
   are hidden
5. **Cost cap** — models above `max_input_cost` $/M tokens are hidden

Pinned aliases are always visible regardless of these filters.

### Superseded Model Detection

`compute_superseded_ids()` groups models by `(provider, lineage)` where lineage
is the model ID with version numbers and date suffixes stripped. Within each
group, all models except the newest release are marked superseded.

Example: `claude-3-5-sonnet`, `claude-3-5-sonnet-20240620`, and
`claude-3-5-sonnet-20241022` are one lineage group. The two older entries become
superseded.

Source: `src/meridian/lib/catalog/model_policy.py:139-196`

## Invariants

- **I-1: Parse-time validation** — `model-policies` rules are validated when
  the profile is loaded, not at resolution time. Unknown keys and malformed rules
  fail at load time.
- **I-2: Pinned aliases always visible** — `is_default_visible_model()` returns
  `True` immediately for pinned aliases regardless of filters.
- **I-3: List fields deferred** — profile `model-policies` list override
  fields (`skills`, `tools`, etc.) are accepted and warned but not applied;
  overlay list fields are rejected with a warning.
- **I-4: Overlay prepend, not replacement** — when an overlay provides one or
  more `model-policies` entries, those rules prepend to profile rules (evaluated
  first). `model-policies = []` suppresses all profile rules. Key absent inherits
  unchanged.
- **I-5: First-match wins** — within a given match-type rank, the first matching
  rule in list order wins. Duplicate same-specificity rules are silently unreachable,
  not a parse error.

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
