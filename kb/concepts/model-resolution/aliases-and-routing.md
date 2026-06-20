# Aliases and Harness Routing

Model names in a spawn request resolve to concrete model IDs via **Mars aliases**,
then route to a harness via the Mars launch-bundle. This page covers the alias
mechanism, the identity/routing split, and how harness-specific model strings work.

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

`mars_provided_harness` returns `None` when Mars doesn't specify a harness for
the alias. In the bundle path, Mars handles harness assignment in the
launch-bundle response.

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

**Harness routing** for PRIMARY/SPAWN_PREPARE is handled by Mars via the
launch-bundle. Meridian passes explicit CLI/env overrides to
`mars build launch-bundle` and Mars returns the resolved harness in the bundle
payload. This split lets identity resolution succeed independently of harness
assignment; the full routing resolution (including policy rules and fallback)
lives in Mars.

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

After bundle resolution, the routing context is captured in a frozen
`ModelSelectionContext`:

```python
@dataclass(frozen=True)
class ModelSelectionContext:
    requested_token: str          # original user input ("sonnet", "gpt-4o")
    selected_model_token: str     # alias or model_id used for policy matching
    canonical_model_id: str       # fully resolved model ID
    harness_model_id: str | None  # harness-specific model string (may differ from canonical_model_id)
    mars_provided_harness: HarnessId | None
    resolved_entry: AliasEntry | None
    harness_provenance: str       # why this harness was selected (from bundle provenance)
```

`requested_token` preserves the original user input — important for `--dry-run`
output that shows "you asked for X, resolved to Y via Z".

`harness_model_id` is the string actually passed to the harness subprocess. It equals
`canonical_model_id` for harnesses that use the canonical form, but diverges when
`AliasEntry.runnable_paths` has a harness-specific entry (e.g. `openai/gpt-5.5` for
OpenCode). Always use `harness_model_id` at the harness command boundary, never
`canonical_model_id` directly.

In the bundle path, `harness_model_id` comes from `bundle_result.harness_model`
returned by Mars; `harness_provenance` comes from the bundle provenance map
(`harness_source` key).

## Harness Routing via Mars Bundle

For PRIMARY and SPAWN_PREPARE launches, harness routing is owned by Mars. Meridian
forwards explicit routing overrides:

- CLI `--model` / `--harness` → `bundle_request.model_override` / `bundle_request.harness_override`
- CLI `--effort`, `--approval`, `--sandbox` → corresponding bundle request fields
- If model was set via CLI (not profile/config), model-derived harness takes precedence
  over any profile/config harness in the bundle

The bundle response includes:
- `routing.harness` — resolved harness ID
- `routing.harness_model` — harness-specific model string (if needed)
- `routing.model_token` — alias or token used for routing
- `provenance` map — per-field source attribution

Source: `src/meridian/lib/launch/bundle_adapter.py`, `src/meridian/lib/launch/policies.py:_resolve_policy_from_bundle()`

## Model Prompting Guidance

Each `[models.<alias>]` entry in `mars.toml` can carry an optional `prompting`
field — a string of caller-facing advice for how to prompt the resolved model
effectively. This is **not** a model description (what the model is, its
price/capability tier) — it's operational prompting guidance (how to brief it,
what style of instructions it responds to, what to emphasize or avoid).

```toml
[models.gpt55]
harness = "codex"
model = "gpt-5.5"
prompting = "Very literal executor. Before handing off, step back and write the complete technical plan..."
```

The `prompting` field is stored on `ModelAlias.prompting` in mars-agents and
serialized into the merged alias table. It is not used during spawn launch —
it's purely a caller-facing reference field retrieved on demand.

### Retrieval: `mars models prompting <ref>`

Prompting guidance is retrieved via the `mars models prompting` command, which
exists in mars-agents (not Meridian itself). The command accepts
`--refresh-models` / `--no-refresh-models` to control catalog refresh during
resolution.

The command uses **agent-first resolution**:

1. Try the ref as an agent name (with or without `@` prefix — equivalent).
   File-stem agent matches beat profile-name matches — consistent with
   launch-bundle agent resolution semantics.
2. If no agent matches, treat the ref as a model alias (catalog-backed).
3. If both an agent and model alias share the same name, the agent wins.
4. For agent matches: resolve through the canonical launch policy pipeline
   (`build::policy::resolve_policy` with `PolicyInput`). This applies
   model-policies, agent overlays, config defaults, harness routing, and alias
   resolution — the same path used for real agent execution. The returned
   `model_alias`, `model_name`, and `prompting` describe the final resolved
   runnable model after all routing decisions.
5. If launch resolution clears or omits a model (harness-only routing, model
   clearing via probe, or no matching candidate), the command returns no
   prompting guidance from a pre-routing token.

### Output from policy.routing

For agent refs, `model_alias`, `model_name`, and `prompting` in JSON output are
derived from the final `policy.routing` after canonical launch-policy resolution
—not from pre-routing overlay/profile/default tokens. Direct model-alias refs
bypass launch policy and read from the merged alias table plus catalog cache.

| Output field | Agent path (`resolve_policy`) | Direct model-alias path |
|---|---|---|
| `model_alias` | `routing.model_token` when non-empty and the token is a key in the merged alias table; otherwise omitted | Input alias name |
| `model_name` | `routing.harness_model` when non-empty, else `routing.model`; omitted when both are empty (model cleared) | Catalog-backed concrete model ID via `resolve_model_id_for_alias` and the models cache |
| `prompting` | `prompting` field from the alias entry keyed by `model_alias`; omitted when no alias key or alias has no `prompting` | `prompting` on the matched `ModelAlias` in the merged table |

JSON also includes `ref`, `ref_kind` (`agent` or `model`), `agent_name` (agent
path only), and `found`.

Known refs without `prompting` guidance exit 0 with a message showing how to
add a `prompting` field. Unknown refs exit non-zero.

Source: `src/cli/models.rs:run_prompting()` and `src/build/policy/mod.rs:resolve_policy()`
in mars-agents.

The spawn guidance block injected into agent system prompts points at this
command: `meridian mars models prompting <agent-or-model>`.
Source: `src/meridian/lib/launch/spawn_guidance.py:_SPAWN_PROMPTING`.

### Canonical command name

The canonical command spelling is `models prompting` (not `models prompt`).
The original design requirement called for `models prompt` because "the command
name should match how humans ask for prompt help: `prompt`, not `prompting`",
but the shipped version chose `prompting` to align with the field name and
avoid ambiguity with the `mars models prompt` command surface that could be
confused with a prompt-generation feature.

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
  profile frontmatter
- [architecture/launch-system.md](../../architecture/launch-system.md) — where
  these functions sit in the launch factory
- [architecture/mars-launch-bundle.md](../../architecture/mars-launch-bundle.md) —
  Mars launch-bundle schema, bundle request fields, and response structure
- [architecture/mars-routing.md](../../architecture/mars-routing.md) — mars-agents internal routing: slug primitive, SelectionKind/MatchEvidence split, acceptance layer, RouteDecisionReport DTO
