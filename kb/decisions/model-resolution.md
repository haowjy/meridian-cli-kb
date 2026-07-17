# Decisions: Model Resolution

Decisions about model alias authority, routing architecture, profile schema, policy matching, and harness derivation.

For mechanism, see:
- [concepts/model-resolution/overview.md](../concepts/model-resolution/overview.md) — how model names become concrete models
- [concepts/model-resolution/aliases-and-routing.md](../concepts/model-resolution/aliases-and-routing.md) — alias entries, pattern fallback, harness routing
- [concepts/model-resolution/agent-profiles.md](../concepts/model-resolution/agent-profiles.md) — profile loading, skill attachment, fallback chain
- [concepts/model-resolution/model-policies.md](../concepts/model-resolution/model-policies.md) — visibility, superseded models, fanout

---

## Alias Authority

### Mars owns model alias resolution; Meridian calls `mars models list --json`

**Decision:** Model alias resolution (human-readable names like `sonnet`, `codex`, `gpt55` → concrete model strings) is delegated to Mars. Meridian calls `mars models list --json` at resolve time rather than maintaining its own alias table.

**Why:** Model aliases change frequently as providers update naming, new models are released, and organizational routing preferences evolve. Maintaining aliases in Meridian source would require a Meridian release for every alias change. Mars as the package manager already has the catalog refresh mechanism (24h TTL cache) and the organizational package override system. Delegating avoids duplication of the catalog layer.

**Why aliases travel with packages:** The same `mars.toml` that installs the `coder` agent declares which model name the `coder` profile was designed for. One config change propagates to all spawns that use that package.

**Fallback chain when Mars is unavailable:**
1. Run `mars models list --json` → structured alias definitions
2. If Mars binary unavailable → fall back to `.mars/models-merged.json`
3. If neither → empty alias table (bare model IDs still work via pattern fallback)

**Alternatives rejected:**
- Hardcoded alias table in Meridian — breaks on every new model, requires a Meridian release
- Alias TOML in the repo — requires a repo commit for every model change, defeats the cache benefit

---

## Routing Architecture (MR1–MR3)

### D52: Identity separated from routing in `resolve_model()` (MR1)

**Decision:** `resolve_model()` returns an `AliasEntry` with `model_id` and `mars_provided_harness` (may be `None`). Harness routing is handled downstream by the Mars launch-bundle path. The function no longer raises an error when harness can't be determined.

**Why the split:** Previously, `resolve_model()` raised `ValueError` if no harness could be determined — even in contexts where the harness would be provided by profile frontmatter, config defaults, or explicit CLI override. This forced callers to pass dummy harness context just to satisfy the resolver. The split lets identity resolution succeed independently; harness assignment happens downstream where all override sources are available.

**Implementation note:** `validate_harness_compatibility()` was moved from `policies.py` to `resolve.py` to avoid a circular import introduced when `policies.py` imported from `resolve.py` while `resolve.py` was also importing from `policies.py`.

---

### D53: `ModelSelectionContext` introduced as routing context type (MR2+MR3)

**Decision:** `ModelSelectionContext` is a frozen dataclass carrying `requested_token`, `selected_model_token`, `canonical_model_id`, `mars_provided_harness`, `resolved_entry`, and `harness_provenance` through the pipeline. It threads from `resolve_launch_policy()` through `LaunchContext` to dry-run output.

**Why:** Routing provenance ("why was this harness selected?") was previously lost after `resolve_launch_policy()` returned. Debugging wrong harness selection required reading source code, not just inspecting the dry-run output. `ModelSelectionContext` makes provenance a first-class value.

**`requested_token` field:** Specifically preserves the user's original input — so dry-run output can say "you asked for `sonnet`, resolved to `claude-sonnet-4-5` via claude harness (mars-provided)".

---

### D55: Resolve-once pattern: model aliases resolved exactly once

**Decision:** Model aliases are resolved exactly once in `resolve_launch_policy()`. The resolved `AliasEntry` is threaded through harness derivation and final model selection. Config normalization no longer calls `resolve_model()` — it only normalizes field shapes.

**Why:** Previously, the same alias could be resolved twice: once in `_normalize_toml_payload()` (during config loading) and again in `resolve_launch_policy()`. If the alias catalog changed between the two calls (rare but possible), the results could be inconsistent. The double-resolution also wasted compute. The `requested_token` field on `ModelSelectionContext` preserves what the user originally asked for, so downstream consumers can distinguish "user said gpt55" from "resolved canonical ID claude-sonnet-4-5".

---

## Profile Schema

### D54: New profile schema: `model-policies`, `fanout`, `mode`

**Decision:** Three new agent profile frontmatter fields were introduced as part of the routing architecture overhaul:
- `model-policies` — typed selector rules (match on `model`, `alias`, or `model-glob`; apply override fields). Replaces legacy `models:` mapping.
- `fanout` — structured list of fallback model candidates. Replaces flat `models:` list as fan-out surface.
- `mode: primary | subagent` — declares agent role for inventory grouping. Default `subagent`.

Legacy `models:` still parsed but emits a deprecation warning when present without `model-policies` or `fanout`.

**Why `model-policies` over `models:`:** The legacy `models:` was a flat mapping from alias-or-model-id to `{effort, autocompact}`. It couldn't express harness routing decisions, had ambiguous priority semantics when the same model matched multiple ways, and conflated fan-out display with policy overrides. `model-policies` is explicit: typed match selectors with a defined priority order (exact model > exact alias > glob) and an `override` block that supports all scalar fields.

**Parse error vs. warning:** `model-policies` uses hard parse errors (raises `ValueError`) while legacy fields use soft warnings. The strict parser is isolated from the lenient parser to prevent a messy dual path.

---

## Delivery Mechanism

### D51: Phase 9 narrowed — keep `--agent`/`--agents` for Claude; keep Codex `base_instructions` split

**Decision:** The original Phase 9 plan (eliminate `--agent`/`--agents`, switch to `--append-system-prompt` for all harnesses) was rejected. Each harness's existing delivery mechanism is preserved. The V0 scope was read-path migration only.

**Two blocking architectural issues identified:**
1. **Claude `--agent` semantics** — `--agent` makes the profile body **replace** Claude's default system prompt. `--append-system-prompt` appends to it. These are different instruction models. Agents that rely on being the governing prompt (most of them) would have weaker authority when appended to Claude's default coding prompt.
2. **Codex `base_instructions` split** — Codex intentionally separates `adhoc_agent_payload` → `base_instructions` (agent role — higher priority) from `appended_system_prompt` → `developer_instructions` (Meridian runtime context). Collapsing the two layers would silently change instruction priority.

**What changed in Phase 9:** Nothing. Phase 9 became a probe that confirmed the existing composition flows correctly from `.mars/` — no code changes required.

**Delivery mechanism unification** was explicitly deferred for a future work item with proper behavioral evaluation.

---

## Filtering and Environment

### D56: Phase 13 — prompt-hygiene filtering deferred as unnecessary

**Decision:** Profile body filtering (stripping meridian-operational boilerplate from the agent's system prompt before launch) was evaluated and deemed unnecessary. The profile body passes through unchanged.

**Why:** The investigator probe found that profile bodies are typically 3–4% of the launched prompt. Meridian-operational boilerplate (spawn instructions, model routing info) is a small fraction of that. The cost of filtering (fragility, maintenance, prompting conventions conflict) outweighs the token savings.

**If profile bloat becomes a real problem:** The right fix is prompt authoring conventions in source packages, not parsing logic in Meridian.

**What remains open:** Agent catalog bloat (profiles listing too many agents in the inventory prompt) and skill frontmatter verbosity are structural concerns deferred to future work.

---

### D57: `MERIDIAN_HARNESS` is spawn-local, not a user-facing policy override

**Decision:** `MERIDIAN_HARNESS` written to child process environments by `build_launch_context()` is an informational spawn-local value. It is NOT consumed as a user policy override by `resolve_policies()`.

**Why:** `MERIDIAN_HARNESS` tells the child process "which harness you are running inside." If it were treated as a policy override, nested launches would inherit the parent's resolved harness, bypassing their own profile and config.

**Future work:** Rename the spawn-local env var to `MERIDIAN_SELECTED_HARNESS` (or similar) so a user-facing `MERIDIAN_HARNESS` env var can safely be consumed as a routing input.

See [launch decisions](launch.md#d57-meridian_harness-is-spawn-local-not-a-user-facing-policy-override) for the launch-side rationale.

---

### Model is optional — harness alone is sufficient for launch execution {#model-optional-empty-model}

**Decision:** An absent managed model override is a valid resolved launch state.
Harness adapters omit the managed model override from the launch command and let
the resolved harness use its own default. No layer between the caller and the
adapter should reject model absence as invalid.

**Context:** Prior to the 2026-05 spawn resolution refactor, the background worker's inline validation rejected empty model with a terminal event, orphaning `orphan_run` spawns for any profile that intentionally omits a `model:` field (model-optional profiles). These profiles rely entirely on the harness default.

The persisted representation for this state is legacy JSON `model=""`. Mars may
write that value in a launch-policy snapshot when it intentionally clears a model
during launch-policy resolution. Example: a profile or config supplies a model
token that the selected harness cannot run, but a higher-precedence harness
override wins. The harness choice is recorded, but Meridian must not pass a
managed model override.

**What empty model means:**
- The caller made no model selection, and the profile specifies none
- Mars cleared an incompatible profile/config model because a stronger harness override won
- Harness uses its internal default (e.g., Claude Code uses its configured default model)
- Persisted snapshots/spawn records may store `""` — this is not a placeholder but
  the durable JSON representation for harness-default model

Do not persist a fake model token such as `"none"` for this state. That would
make downstream launch binding treat the token as a Meridian-managed model
override. In new in-memory replay code, normalize persisted `""` to `None` so
absence is explicit. The durable JSON compatibility representation remains the
empty string.

**Validation rule (post-refactor):** Pre-launch and snapshot replay validation check `prompt` where applicable and `harness` (must be non-empty). Model is NOT validated — model absence is a legitimate harness-agnostic configuration, not an error. Snapshot model normalization belongs to policy snapshot replay; same-session continue consumes the normalized result through [launch decisions](launch.md#d-continue-replays-recorded-launch-contract-same-session-continue-is-not-live-policy-recomputation).

**Relationship to I-7:** I-7 requires "real resolved values" in the spawn row. The `"unknown"` sentinel that previously masked empty model was an I-7 violation — it was a placeholder, not a resolved value. Empty string is the correct representation.

See [launch decisions](launch.md) for the full background worker refactor decisions.

---

## Agent Overlay Config and Canonical Compiler (2026-05-05)

### D72: `[agents.<name>]` overlay config as the project-scoped override surface

**Decision:** Agent runtime policy can be overridden project-wide (or project-locally) via `[agents.<name>]` sections in `meridian.toml` / `meridian.local.toml` / `~/.meridian/config.toml`. These overlays feed the launch-parameter compiler without editing generated `.mars/agents/` profiles.

**Motivating use case:** Override `tech-lead` from its package default to `gpt55 + medium effort` for local experimentation — without touching `.mars/agents/tech-lead.md`, which is generated output overwritten by `meridian mars sync`.

**v1 supported scalar fields:** `model`, `harness`, `effort`, `approval`, `sandbox`, `autocompact`.

**`timeout` explicitly excluded from v1 overlay fields.** The overlay compiler does not yet thread resolved timeout into the launch pipeline's `execution_policy.timeout` field. Adding `timeout` to overlay would create "parses but does nothing" semantics. Timeout still resolves through CLI/env/profile/config paths as before; it just can't be set via `[agents.<name>]` in v1. (Note: `ExecutionBudget`, the former dual timeout carrier, was deleted in PR #375; timeout is now carried solely on `execution_policy.timeout` in minutes, with seconds conversion at the runner edge only.)

**List override fields (`skills`, `tools`, `disallowed-tools`, `mcp-tools`) rejected with warning in overlay.** In agent profile `model-policies`, these keys parse and are stored in `ModelPolicyRule.overrides` but are not yet applied at runtime (logged as "not-yet-supported"). In config overlays, they are actively rejected at parse time. Rationale: profile behavior is inherited from package authors and the silent-ignore is a known limitation; overlay behavior is user-authored and a silent-ignore would create false expectations about what the overlay controls.

**Three-state model-policies semantics** (key insight for correctness):
- Key absent in all config files → `model_policies = None` → inherit profile model-policies unchanged.
- Key present as empty array (`model-policies = []`) → `model_policies = ()` → suppress all conditional model overrides; neither profile model-policies nor legacy `models:` apply.
- Key present with entries → `model_policies = (rule1, rule2, ...)` → replacement; profile model-policies are suppressed entirely.

The `None`-vs-empty distinction is preserved through all config normalization layers. This is replacement semantics, not merge — merging rule sets from different sources creates ordering ambiguity and makes it impossible to suppress an inherited rule.

**Merge behavior across config files:** Deep merge per existing `_merge_nested_dicts()`. User config is base; project config overrides for any field it sets; project-local overrides both. `model-policies` is an array: stronger layer replaces (not concatenates).

**Alternatives rejected:**
- Editing `.mars/agents/` directly — overwritten by `meridian mars sync`.
- User-global config as the primary solution — doesn't satisfy project-scoped experiments without side effects on all projects.
- `config set/get` CLI support in v1 — deferred; users edit TOML files directly, consistent with `[workspace]` and `[context]` management.

---

### D73: Canonical launch-parameter compiler in `compiler.py` with pure-data contract

> **Superseded.** PRIMARY/SPAWN_PREPARE routing now resolves through Mars
> launch-bundle (`_resolve_policy_from_bundle()` → `bundle_adapter.request_and_resolve()`).
> `compiler.py` is explicitly deprecated in its module docstring. This record is
> preserved for historical context only.

**Decision:** A dedicated module `src/meridian/lib/launch/compiler.py` owns all launch-parameter resolution: routing field tier walks, policy field tier walks, model-policy matching, overlay application, harness derivation, and provenance tracking. The compiler accepts `CompilerRequest` (pure data) and returns `CompilerResult` (pure data). A separate `materialize.py` converts `CompilerResult` to runtime objects (`SubprocessHarness`, loaded skills, etc.).

**Why a dedicated compiler over continued growth of `resolve_policies()`:** `resolve_policies()` had become an implicit blob combining profile loading, model resolution, precedence, skill attachment, and harness derivation. Adding the new agent overlay tier would have made it worse. The compiler forces explicit typed inputs/outputs, testable in isolation, with clear provenance tracking.

**Pure-data contract requirement (EARS-072):** `CompilerRequest` and `CompilerResult` contain only scalars, tuples, dicts, and enums — no `Path`, no adapters, no loaded content. This enables: (a) unit testing without Meridian runtime setup, (b) a migration path to Mars ownership if warranted (see D74).

**`resolve_policies()` becomes a thin wrapper** that loads the profile, resolves the model via `CatalogSession`, builds a `CompilerRequest`, calls `compile_launch_params()`, calls `materialize_harness()`, and maps back to `ResolvedPolicies`. All existing callers continue working without changes.

**`FieldProvenance` for observability:** Each resolved field carries a `ProvenanceLevel` enum value (`CLI`, `ENV`, `AGENT_OVERLAY_POLICY`, `AGENT_OVERLAY_DEFAULT`, `PROFILE_MODEL_POLICY`, `PROFILE_DEFAULT`, `CONFIG_DEFAULT`, `ALIAS_DEFAULT`, `UNSET`). This feeds `--dry-run` output and `config show`, making precedence visible without reading source code.

**Effective model-policy source for harness availability fallback:** When the primary harness is unavailable and fallback is attempted, the compiler uses the *effective* model-policy source (overlay if overlay has a `model_policies` value, profile otherwise). This ensures overlay suppression/replacement of model-policies also controls which candidates are considered for fallback — otherwise a user suppressing profile model-policies would still have those policies applied during fallback selection.

---

### D74: Compiler lives in Meridian, not Mars (Option C over B)

> **Superseded.** The question of whether the compiler lives in Meridian vs. Mars
> was resolved differently: Mars now owns routing via launch-bundle. The compiler
> module (`compiler.py`) is deprecated. This record is preserved for historical context.

**Decision:** After an explicit adversarial review (three agent spawns: pro-Mars p4321, pro-Meridian p4322, boundary/risk analysis p4324), Option C was confirmed: the launch-parameter compiler lives in Meridian now, behind a stable JSON-serializable contract, movable to Mars later if warranted.

**Why Mars ownership (Option B) was rejected:**

1. **Different compilation concerns at different times.** Mars compiles at sync-time to produce disk artifacts (`.mars/agents/*.md`). Meridian compiles at launch-time to resolve "what does this agent launch with right now?" Different timing, inputs, purposes — they share input data but not logic.

2. **Mars lacks the runtime semantics the compiler needs.** Mars's `ModelPolicyEntry` is a marker struct with zero match logic. The model-policy matching engine (exact model > exact alias > model-glob, with precedence, with provenance) lives exclusively in Meridian. Mars has no concept of `[agents.<name>]` overlays or their three-state merge semantics.

3. **Agent overlays are Meridian-native config.** The motivating feature is pure Meridian config (`meridian.toml`, `meridian.local.toml`). Making Mars the compiler requires either teaching Mars Meridian's config semantics or having Meridian pre-resolve everything and pass it through — which eliminates the claimed benefit of centralization.

4. **Harness availability fallback cannot live in Mars.** Fallback depends on local `HarnessRegistry` state (which harness adapters are available on this machine), which is not a package-manager responsibility. Even with Mars as compiler, Meridian would apply post-compile corrections — the boundary stays muddy after paying the IPC/versioning cost.

5. **Cross-repo coupling cost is not worth theoretical cleanliness.** Every new overlay field, precedence tier, or provenance change would require coordinated Mars + Meridian releases. That is ongoing tax for a boundary that doesn't match current ownership.

**Conditions for revisiting Mars ownership:**
- A second consumer besides Meridian needs agent launch-parameter compilation.
- Mars gains its own runtime/launch semantics (Mars-native agent execution, not just artifact management).
- The `CompilerRequest`/`CompilerResult` contract proves stable for 2+ major versions and migration becomes mechanically trivial.

**Structural discipline required for Option C to stay correct:**
- `CompilerRequest`/`CompilerResult` must remain pure DTO (no `Path`, no adapters).
- Materialization stays fully separate from compile.
- After compile, callers must not re-run precedence logic from raw data.
- Contract shape tests prevent silent drift toward non-serializable types.

---

## Candidate Chain Semantics (2026-05-06)

### D75: Candidate-chain semantics: transform and demotion, not hidden-scan {#d75-candidate-chain-semantics-transform-and-demotion-not-hidden-scan}

**Decision:** When a harness is unavailable at spawn time, Meridian uses an
ordered candidate chain. `model-policies` rules contribute at most one
**policy-transformed candidate** at the head of that chain. The pre-transform
base candidate is demoted to the next position rather than discarded. Fanout
entries follow as raw availability-chain candidates compiled without policy
re-application.

**Rejected framing — hidden-scan:** Treat `model-policies` entries as an
additional pool of alternative tokens to scan when the primary harness is
unavailable. Under that framing, if the primary harness fails, Meridian would
scan all `model-policies` rules to find any rule whose resulting harness is
available, then walk `fanout`.

**Why hidden-scan was rejected:**

1. **Model-policies rules are context transforms, not token alternatives.** A
   rule fires when the resolved model matches a selector. The rule says "when
   you are using *this model*, apply these overrides." Scanning all rules for
   any available harness breaks that meaning — it would launch the agent using
   whatever harness a policy rule happens to specify, regardless of whether the
   resolved model actually matches that rule's selector.

2. **Demotion preserves the user's base intent.** The user asked for model X
   (via profile or config). A policy rule transformed X's launch parameters.
   If the policy-head's harness is unavailable, the right next step is "try X
   without the policy transform" — still model X, just without the harness
   override. Hidden-scan skips this and jumps to arbitrary policy entries,
   which may correspond to completely different model tokens.

3. **Fanout is the explicit availability-chain mechanism.** Profile authors
   declare `fanout:` precisely to say "if my primary can't run, try these in
   order." Conflating model-policies with fanout would make profiles harder to
   reason about: policy rules would have dual semantics (transform AND fallback
   candidate source).

4. **Recursive policy re-application on fanout entries would be wrong.** If a
   policy rule "reroute gpt55 to claude harness" applied when compiling a
   `gpt55` fanout entry, the fanout chain would re-trigger the same rule that
   failed. Fanout entries are compiled as base candidates to prevent this loop.

**The candidate chain:**

```
[policy-transformed head]     if a model-policy rule matched
[demoted base candidate]      pre-transform baseline, always retained
[fanout[0], fanout[1], …]     profile fanout entries, policy-free base candidates
```

**When fallback fires:** Only when `model_explicit = False` (routing came from
profile or config, not an explicit `-m` or `MERIDIAN_MODEL`). Explicit user
choices are honored or fail loudly — no silent rerouting.

**Overlay suppression scopes the chain:** Suppressing or replacing model-policies
via `[agents.<name>]` overlay controls the chain at its source. An empty overlay
`model-policies = []` removes the policy-transformed head from the chain (no
transform fires) and also neutralizes the legacy `models:` compatibility overrides.

**Implementation references:**
- `src/meridian/lib/launch/compiler.py` — deprecated compiler-era implementation that still carries the historical candidate-chain mechanics
- `src/meridian/lib/launch/policies.py` — live PRIMARY/SPAWN_PREPARE entry point that resolves and consumes the bundle-produced policy surface

---

## Candidate-Aware Harness Routing (2026-05-16)

### D76: Harness-specific model IDs via RunnablePath

**Decision:** Mars returns harness-specific model strings for aliases via a `RunnablePath` structure. `AliasEntry` carries `harness_candidates` (list of `RunnablePath` entries) and `runnable_paths` (dict of `{harness_id: model_id}` for fast lookup). At bind time, `select_harness_model_id()` looks up the resolved harness in `runnable_paths` and passes the harness-specific string to the harness adapter.

**Why:** A model token like `gpt-5.5` may need to be `openai/gpt-5.5` when sent to OpenCode's HTTP API, but bare `gpt-5.5` for Codex. The canonical alias ID is the right handle for policy matching and user communication, but the harness command boundary needs the provider-prefixed form. Using the alias token uniformly sends the wrong string to harnesses that expect provider-prefixed IDs.

**`ModelSelectionContext.harness_model_id`:** The resolved harness-specific model string threads from `resolve_policies()` through `LaunchContext` to `bind_launch_context()`. `context.py:bind_launch_context()` applies `harness_model_id` when building the harness subprocess command.

**Alternatives rejected:**
- Teach each harness adapter to do its own model-string transformation — puts transformation logic in the mechanism layer, not the policy layer. Fails when the same alias token maps to different provider paths across harnesses.
- Store only one model ID and add per-harness prefix rules in adapter config — equivalent complexity, lower locality, harder to audit.

---

### D77: Explicit harness = force; only adapter availability can reject

**Decision:** `validate_harness_compatibility()` no longer checks model compatibility for explicit harness selections. It only checks harness adapter availability (binary present/installed). An explicit harness selection via CLI `--harness`, `MERIDIAN_HARNESS` env, or a model-policy override is **never rejected for model incompatibility** — only for adapter unavailability.

**Why:** Explicit overrides represent deliberate user or profile-author choice. Rejecting them for model incompatibility assumes the catalog is authoritative, but models are added to harnesses faster than catalogs update. Users who force a harness may have out-of-band knowledge (new model added before catalog refresh, custom harness config). The right failure mode is "harness adapter not installed," not "harness doesn't list this model."

**Before (rejected behavior):** `validate_harness_compatibility()` checked whether the resolved model was in the harness's candidate list and raised `ValueError` if not — blocking any explicit override that referenced a model newer than the local catalog.

**Alternatives rejected:**
- Keep the compatibility check but add a `--force-harness` flag — adds surface area; `--harness` is already an explicit override with the same intent.
- Warn instead of reject — warning noise for an explicit user choice that may be entirely valid.

---

### D78: Invalid override key skip in model-policy rules

**Decision:** Model-policy rules where **all override keys are invalid** (unrecognized field names) are silently skipped during profile loading, not kept as phantom no-op rules.

**Why:** A phantom rule with invalid keys occupies a position in the policy list under first-match priority. A no-op rule at position N shadows all downstream rules at positions N+1, N+2, ... that might validly match the same selector. A rule that does nothing is strictly worse than no rule — it blocks rules that would have done something useful. Skipping is the correct behavior; it matches the intent of the profile author (add an override) rather than the broken artifact (malformed keys).

**Scope:** Applied at profile load time in `agent.py`. Logged at debug level for diagnostics. Does not affect rules with a mix of valid and invalid keys — partial-valid rules are applied with the valid keys only.

---

### D86: `mars models prompting` — canonical command name and agent-first resolution {#d86-mars-models-prompting}

> [!FLAG] **Needs human review** — D86 ID collision with [package-management.md](package-management.md) routing-parity D86 (`mars models resolve` / `mars build launch-bundle` share `routing::evaluate_candidates()`). Renumber or cross-link one decision before treating either as authoritative by ID alone. Flagged 2026-06-20.

**Decision:** `mars models prompting <ref>` is the canonical command surface for
retrieving model prompting guidance. The name `prompting` (not `prompt`) was chosen
to align with the `prompting` field name on `ModelAlias` and to avoid ambiguity
with a potential `mars models prompt` command (which could be read as
generate-a-prompt, not lookup-prompting-guidance).

**Resolution algorithm:** agent-first — interprets the ref as an agent name
before falling back to a model alias. Agent matches use the canonical launch
policy resolution path (`build::policy::resolve_policy` with `PolicyInput`),
not a shallow duplicate of model-token precedence.

1. Try the ref as an agent name (with or without `@` prefix — equivalent).
   File-stem matches beat profile-name matches, consistent with launch-bundle
   agent resolution semantics.
2. If no agent matches, try the ref as a model alias in the merged alias table.
3. If both an agent and model alias share the same name, the agent wins.
4. For agent matches: resolve through the canonical launch policy pipeline.
   This applies model-policies, agent overlays, config defaults, harness
   routing, and alias resolution — the same path used for real agent execution.
   The prompting guidance returned describes the final resolved runnable model
   after all routing decisions have been applied.
5. If launch resolution clears or omits a model (harness-only routing, model
   clearing via probe, or no matching candidate), the command returns no
   prompting guidance — it does not invent guidance from a pre-routing token.
6. Unknown refs exit non-zero with `found: false`.
7. Known refs without `prompting` guidance exit zero with a hint showing how to
   add a `prompting` field to the alias in `mars.toml`.

**CLI flags:** `--refresh-models` / `--no-refresh-models` control catalog
refresh during launch-policy resolution (for agent refs) and model name
resolution (for direct model alias refs).

JSON output identifies what was resolved: `ref`, `ref_kind` (`agent` or
`model`), `agent_name`, `model_alias`, `model_name`, `found`, and `prompting`.

**Why agent-first:** callers typically think in terms of "I'm spawning
explorer, what do I need to know about its model?" not "what's gpt55's
prompting style?" Agent-first lets the caller use the same name they'd use with
`meridian spawn -a explorer`.

**Why canonical launch policy over a shallow resolver (2026-06-20 correction):**
The initial agent resolution (commit `ab3fe73`) reimplemented overlay/profile/default
precedence without model-policies, harness routing, model clearing/rerouting, or
catalog-backed alias resolution. This produced prompting guidance for a model
token that could differ from the actual runnable model — misleading callers when
model-policy fallbacks, routing decisions, or model clearing changed which model
would execute. The fix (commit `1859a16`) reuses `resolve_policy()` with
`PolicyInput` so the prompting output matches the real launch outcome.

**Alternatives rejected:**
- `mars models prompt <ref>` — cleaner verb, but conflicts with the reading
  "generate a prompt for this model" vs the lookup semantics of "show me
  prompting guidance for this model."
- Model-alias-only lookup — forces callers to know the agent's effective model
  before getting guidance, defeating the purpose of a convenience command.
- Embedding prompting guidance in the spawn system prompt — adds token cost to
  every launch for guidance that's only useful when the caller is composing
  the prompt. Call-on-demand is lower token cost on average.
- Keeping the shallow duplicate resolver — produces wrong answers when
  model-policies, fallbacks, or routing change the runnable model.

Source: `src/cli/models.rs:run_prompting()` and `src/build/policy/mod.rs:resolve_policy()` in mars-agents.

---

## Related

- [launch.md](launch.md) — harness identity env var decisions (D32, D33, D57, D63); background worker resolution refactor
- [decisions/package-management.md](package-management.md) — Mars alias catalog, skill schema
- [concepts/model-resolution/overview.md](../concepts/model-resolution/overview.md) — resolution mechanism
- [concepts/config-precedence.md](../concepts/config-precedence.md) — how agent overlay tier fits in the precedence ladder
- [architecture/launch-system.md](../architecture/launch-system.md) — where `resolve_policies()` fits in the full launch factory
