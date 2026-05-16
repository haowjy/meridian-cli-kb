# Aliases and Harness Routing

Model names in a spawn request resolve to concrete model IDs via **Mars aliases**,
then route to a harness via a priority cascade. This page covers the alias
mechanism, the identity/routing split, and how a final harness is selected.

## Alias Authority: Mars

Meridian has **zero built-in alias definitions**. All aliases come from Mars
packages declared in `mars.toml` `[models]` sections. When you run
`meridian mars sync`, Mars writes a merged alias table to
`.mars/models-merged.json`. At resolution time:

1. `mars models list --json` → fully resolved alias list (preferred)
2. Fall back to `.mars/models-merged.json` if the Mars binary is unavailable
3. Fall back to empty alias table if neither works — bare model IDs still work

Source: `src/meridian/lib/catalog/model_aliases.py:390-411`

Keeping aliases in `mars.toml` co-locates them with the agent packages that
use them. An alias change propagates automatically on the next `meridian mars sync`.

## AliasEntry

`resolve_model()` returns an `AliasEntry` (frozen dataclass):

```python
class AliasEntry(BaseModel):
    alias: str
    model_id: ModelId
    resolved_harness: HarnessId | None   # None if mars doesn't specify
    description: str | None
    default_effort: str | None
    default_autocompact: int | None
    harness_candidates: list[RunnablePath]      # per-harness (harness_id, model_id) pairs
    runnable_paths: dict[HarnessId, ModelId]    # fast lookup: harness → harness-specific model string

    @property
    def harness(self) -> HarnessId:
        if self.resolved_harness is not None:
            return self.resolved_harness
        return pattern_fallback_harness(str(self.model_id))

    @property
    def mars_provided_harness(self) -> HarnessId | None:
        return self.resolved_harness

    def harness_model_id_for(self, harness_id: HarnessId) -> ModelId:
        """Return the harness-specific model string, or canonical model_id if not in runnable_paths."""
        return self.runnable_paths.get(harness_id, self.model_id)
```

Source: `src/meridian/lib/catalog/model_aliases.py:28-48`

`mars_provided_harness` returns `None` when Mars doesn't specify a harness
for the alias. Harness assignment then falls to `resolve_harness_routing()`.

## Harness-Specific Model IDs

Some models carry different concrete model strings per harness. For example,
`gpt-5.5` may be `gpt-5.5` on Codex but `openai/gpt-5.5` on OpenCode (which
routes to an HTTP API requiring provider-prefixed IDs).

Mars encodes these per-harness strings via **`RunnablePath`** entries:

```python
@dataclass(frozen=True)
class RunnablePath:
    harness_id: HarnessId    # which harness this path applies to
    model_id: ModelId        # the harness-specific model string for that harness
```

`AliasEntry.harness_candidates` is the ordered list of `RunnablePath` entries
for the alias. `AliasEntry.runnable_paths` is a dict derived from it for O(1)
lookup: `{harness_id: model_id}`.

At bind time, `context.py:bind_launch_context()` calls
`entry.harness_model_id_for(resolved_harness)` to retrieve the harness-specific
model string. This string is stored as `ModelSelectionContext.harness_model_id`
and passed to the harness adapter when building the subprocess command.

If `runnable_paths` has no entry for the resolved harness, `harness_model_id_for()`
falls back to the canonical `model_id` — preserving behavior for harnesses that
don't need a provider-prefixed form.

Source: `src/meridian/lib/catalog/model_aliases.py` — `RunnablePath`,
`parse_harness_candidates()`, `parse_runnable_paths()`.

## Identity vs. Routing Split

`resolve_model()` establishes **identity** only — what model are we talking about?
It returns an `AliasEntry` with `model_id` and optional `mars_provided_harness`.
It does not raise if the harness is unknown.

**Harness routing** is handled separately by `resolve_harness_routing()`, where
all override sources (profile, config, CLI) are visible. This split lets identity
resolution succeed independently of harness assignment.

## resolve_model() Algorithm

```
1. Strip whitespace from input
2. Call mars models resolve <name> --json
   - If mars returns same model_id as input → return it as AliasEntry
   - If mars returns a different model_id → run exact-ID guard via
     mars models list --all --json to protect against prefix collisions
     (e.g. gpt-5.4 resolving to gpt-5.4-mini)
3. If mars cannot resolve → treat input as raw model ID, apply pattern fallback
```

Source: `src/meridian/lib/catalog/models.py:44-136`

Aliases are resolved **exactly once** in `resolve_policies()`. The resulting
`AliasEntry` threads through the rest of the pipeline without re-resolution,
eliminating a previous double-resolution bug. See
[decisions/model-resolution.md](../../decisions/model-resolution.md).

## Pattern Fallback for Raw Model IDs

When a model name doesn't match any Mars alias, `pattern_fallback_harness()`
maps it to a harness by prefix:

| Patterns | Harness |
|---|---|
| `claude-*`, `opus*`, `sonnet*`, `haiku*` | Claude |
| `gpt-*`, `o1*`, `o3*`, `o4*`, `codex*` | Codex |
| `gemini*` | OpenCode |

Source: `src/meridian/lib/catalog/model_policy.py:33-67`

Pattern fallback is a last resort. Teams using Mars aliases don't rely on it
in production — it's mainly for bare model IDs in development.

## ModelSelectionContext

After `resolve_model()`, the resolved state is captured in a frozen
`ModelSelectionContext`:

```python
@dataclass(frozen=True)
class ModelSelectionContext:
    requested_token: str          # original user input ("sonnet", "gpt-4o")
    selected_model_token: str     # alias or model_id used for policy matching
    canonical_model_id: str       # fully resolved model ID
    harness_model_id: str         # harness-specific model string (may differ from canonical_model_id)
    mars_provided_harness: HarnessId | None
    resolved_entry: AliasEntry | None
    harness_provenance: str       # why this harness was selected
```

`requested_token` preserves the original user input — important for `--dry-run`
output that shows "you asked for X, resolved to Y via Z".

`harness_model_id` is the string actually passed to the harness subprocess. It equals
`canonical_model_id` for harnesses that use the canonical form, but diverges when
`AliasEntry.runnable_paths` has a harness-specific entry (e.g. `openai/gpt-5.5` for
OpenCode). Always use `harness_model_id` at the harness command boundary, never
`canonical_model_id` directly.

## Harness Routing: 7-Source Cascade

`resolve_harness_routing()` determines the final harness from multiple sources
in priority order:

```
1. Explicit CLI override (--harness)
   → provenance: "explicit-override"

2. Matched model-policies rule with harness override
   → provenance: "profile-model-policy"

3. Profile frontmatter harness field
   → provenance: "explicit-override"

4. Model alias → harness derivation (AliasEntry.mars_provided_harness)
   → provenance: "mars-provided"

5. Pattern fallback from model ID
   → provenance: "pattern-fallback"

6. Configured default (config default_harness or primary.harness)
   → provenance: "configured-default"

7. Harness-availability fallback (demoted base if policy transformed, then
   model-policies list candidates in order)
   → provenance: "availability-fallback"
```

The `harness_provenance` string surfaces in `--dry-run` output and debug logs.

**Model-derived override:** When the user explicitly sets a model via CLI,
the model-derived harness wins over profile or config harness. `meridian spawn -m gpt55`
routes to Codex even if the profile says `harness: claude`.

Source: `src/meridian/lib/launch/policies.py:382-626`

## Harness-Availability Fallback

When the selected harness is unavailable (binary not found) **and** the user
didn't explicitly specify a model, Meridian walks an ordered **candidate chain**
rather than scanning for any available option:

```
[demoted base candidate]      ← if a model-policy rule matched and transformed
[model-policies[0], [1], …]   ← rules where no-fallback != true and match_type is model or alias
```

Steps:

1. **Primary attempt** — compile the base launch candidate (profile/config/CLI).
   Apply the effective `model-policies` list. If a rule matches, its overrides
   become the primary candidate.
2. **Demoted base** — if the primary candidate's harness is unavailable and a
   policy rule transformed it, try the demoted base candidate (same model, no
   policy overrides applied).
3. **model-policies candidates** — if the demoted base is also unavailable (or
   no rule matched and the base itself is unavailable), walk `model-policies` rules
   in list order. For each rule where `no-fallback` is not `true` and `match_type`
   is `model` or `alias`, try to resolve `match_value` as a model token with an
   available harness. The first successful entry wins. The matched rule's overrides
   are preserved — injected at CLI-equivalent precedence so the fallback keeps its
   intended execution policy and harness routing. Model-policies are **not**
   re-applied recursively against the fallback token.
4. **Fail loudly** — if nothing in the chain resolves to an available harness,
   raise an unavailability error naming the harness.

`model-glob` rules never participate in fallback. They apply overrides during
primary compilation only.

The fallback only activates when routing came from a profile or config default,
not from an explicit user choice. `meridian spawn -m sonnet` never falls back
silently to Codex.

Source: `src/meridian/lib/launch/policies.py` — `_try_harness_availability_fallback()`,
`_fallback_candidates_from_policies()`, `_compiler_request_for_fallback_candidate()`.

## Known Limitation

`MERIDIAN_HARNESS` env var is **not** a `RuntimeOverrides.from_env()` policy
override — it's spawn-local, set by Meridian in child process environments for
yield timing. It influences config defaults only, not the runtime override layer.
Source: `src/meridian/lib/core/overrides.py:126-146`. A rename to clarify this
is tracked in [decisions/model-resolution.md](../../decisions/model-resolution.md).

## Related

- [overview.md](overview.md) — full resolution pipeline
- [model-policies.md](model-policies.md) — model-policies rules that override
  harness per selected model and declare fallback candidates
- [agent-profiles.md](agent-profiles.md) — how model-policies are declared in
  profile frontmatter and drive harness-availability fallback
- [architecture/launch-system.md](../../architecture/launch-system.md) — where
  these functions sit in the launch factory
