# Aliases and Harness Routing

Mars is the authority for model aliases and launch routing. Meridian parses the
resolved alias payload for display and launch context, but it does not infer a
harness from Python prefix tables.

## Alias Authority

Aliases originate in package and consumer `[models]` configuration. Mars
resolves them and supplies the launch bundle; `.mars/models-merged.json` is the
compiled alias catalog used for local catalog fallback. Meridian has no built-in
alias definitions.

`AliasEntry` is a frozen Pydantic model with canonical `model_id`, optional
`resolved_harness`, defaults, ordered harness candidates, and a tuple of
`RunnablePath` records:

```python
class RunnablePath(BaseModel):
    harness: str
    harness_model_id: str
    provider: str = ""

class AliasEntry(BaseModel):
    alias: str
    model_id: ModelId
    resolved_harness: HarnessId | None
    harness_candidates: tuple[str, ...]
    runnable_paths: tuple[RunnablePath, ...]
```

`AliasEntry.harness` returns the Mars-supplied harness and raises if it is
missing. `runnable_paths` is not a dict, and the entry has no
`harness_model_id_for()` helper. The harness-specific model string used for a
launch comes from `routing.harness_model` in the Mars launch bundle.

## Identity and Routing

Model lookup establishes identity; Mars launch-policy resolution establishes a
runnable route. Meridian forwards explicit model/harness/effort/approval/sandbox
overrides in the bundle request. The response supplies the resolved harness,
canonical and harness-native model strings, selected token, and per-field
provenance.

Raw model IDs follow this same Mars path. There is no Python
`pattern_fallback_harness()` or `model_policy.py` prefix router. If Mars cannot
supply a required harness, Meridian fails rather than guessing from the model
name.

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

See [decisions/model-resolution.md#d86-mars-models-prompting](../../decisions/model-resolution.md#d86-mars-models-prompting)
for the decision record and
[mars-model-refresh.md](../../architecture/mars-model-refresh.md) for catalog/probe
refresh behavior.

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
- [vocabulary.md](vocabulary.md) — glossary for model-resolution terms
- [../../decisions/model-resolution.md#d86-mars-models-prompting](../../decisions/model-resolution.md#d86-mars-models-prompting) —
  decision record for `mars models prompting`
- [architecture/launch-system.md](../../architecture/launch-system.md) — where
  these functions sit in the launch factory
- [architecture/mars-launch-bundle.md](../../architecture/mars-launch-bundle.md) —
  Mars launch-bundle schema, bundle request fields, and response structure
- [architecture/mars-routing.md](../../architecture/mars-routing.md) — mars-agents internal routing: slug primitive, SelectionKind/MatchEvidence split, acceptance layer, RouteDecisionReport DTO
- [architecture/mars-model-refresh.md](../../architecture/mars-model-refresh.md) —
  catalog and probe refresh control (`--refresh-models` / `--no-refresh-models`)
