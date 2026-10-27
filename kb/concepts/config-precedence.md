# Config Precedence

Meridian resolves spawn configuration through two distinct systems: a file-backed settings loader (`MeridianConfig`) and a per-spawn runtime override stack (`RuntimeOverrides`). Understanding which system handles which setting is the key to predicting what any given spawn will actually use.

## Two Systems, Two Problems

**`MeridianConfig`** holds persistent, project-wide operational settings: timeouts,
verbosity, retention, and spawn depth. These are loaded from TOML files at startup
and apply broadly across all spawns unless overridden.

**`RuntimeOverrides`** holds per-spawn behavioral choices: which model, which harness, which approval mode. These are assembled fresh for each spawn from CLI flags, environment variables, agent profile frontmatter, and config file defaults — then merged with first-non-`None` semantics.

Conflating the two systems would make it impossible to cleanly answer "what should this specific spawn do?" versus "what are the project-wide defaults?" — see [launch decisions](../decisions/launch.md) for the reasoning behind this separation.

## RuntimeOverrides: Launch Policy Precedence

Each field in `RuntimeOverrides` resolves independently via `resolve(*layers)` in `src/meridian/lib/core/overrides.py`. The merge is first-non-`None` across layers in order:

```
CLI flags / request
  → env vars (MERIDIAN_MODEL, MERIDIAN_APPROVAL, …)
    → [agents.<name>] overlay (mars.local.toml > mars.toml, delivered in the Mars bundle)
      → agent profile frontmatter
        → project/user config defaults
          → model alias defaults
            → hardcoded harness fallback
```

**Agent overlay** (`[agents.<name>]` in `mars.toml` or `mars.local.toml`) sits between env vars and profile frontmatter. Meridian's own loader rejects an `[agents]` section in `meridian.toml` with an error pointing at `mars.toml`. It lets project operators override per-agent runtime policy without editing generated `.mars/agents/` profiles. See [decisions/model-resolution.md](../decisions/model-resolution.md#d72-agentsname-overlay-config-as-the-project-scoped-override-surface) for full rationale.

The fields in `RuntimeOverrides` are: `model`, `harness`, `agent`, `effort`, `sandbox`, `approval`, `autocompact`, `timeout`.

**Each field resolves independently.** A CLI override of `-m gpt-5.4` affects model selection but does not affect approval mode. Approval mode runs its own precedence chain separately. There is no "whoever wins model also wins everything else."

### Routing vs Policy Fields

Fields are split into two categories, codified in `_ROUTING_OVERRIDE_FIELDS`:

- **Routing fields** (`model`, `harness`, `agent`) — determine *where* to run
- **Policy fields** (`effort`, `approval`, `sandbox`, `autocompact`, `timeout`) — determine *how* to run

`model_policy_scope()` on `RuntimeOverrides` returns a copy containing only the policy fields (i.e. everything not in `_ROUTING_OVERRIDE_FIELDS`). It's used by the bundle execution policy resolver in `policies.py` to isolate the policy field resolution path. Because the exclusion is set-based, any new policy field added to `RuntimeOverrides` is automatically included — no hand-curation needed.

**Policy fields have a more granular cascade than the simple layer order above.** The full 7-tier ladder for the policy fields Mars resolves (`effort`, `approval`, `sandbox`, `network`):

```
1. CLI flags (--effort, --approval, …)
2. Env vars (MERIDIAN_EFFORT, MERIDIAN_APPROVAL, …)
3. Agent overlay scalars ([agents.<name>] effort, approval, sandbox, network)
4. Matched model-policy rule — first match in the overlay's rules followed by the profile's
5. Profile generic defaults (top-level frontmatter)
6. Config-level defaults (primary.effort etc.)
7. Alias defaults
```

The **effective rule list** for tier 4 is three-state:
- Overlay has no `model-policies` key → the profile's rules alone.
- Overlay has `model-policies = []` → no rules apply.
- Overlay has rules → they come first, followed by the profile's rules.

A matched rule beats the profile's own top-level value, so a profile can set `effort: max`
and lower it for one model. Before Mars schema v1 the profile's top-level value won, which
the owner ruled a bug ([D-model-policy-rule-beats-profile-default](../decisions/model-resolution.md#d-model-policy-rule-beats-profile-default)).
An overlay scalar beats every rule, and CLI and env beat both. Mars resolves tiers 3–5 in
the launch bundle; the user-facing ladder is meridian-cli `docs/configuration/mars.md`
("Precedence").

### Factory methods

`RuntimeOverrides` instances are constructed from specific sources via factory classmethods:

| Factory | Source |
|---|---|
| `from_launch_request(request)` | CLI flags passed to the invocation |
| `from_env()` | `MERIDIAN_MODEL`, `MERIDIAN_EFFORT`, `MERIDIAN_APPROVAL`, etc. |
| `from_agent_profile(profile)` | Profile frontmatter |
| `from_config(config)` | `config.primary.*` for primary launch |
| `from_spawn_config(config)` | `config.default_model` for spawned agents |
| `from_spawn_input(payload)` | `spawn create` REST API payload |

**The harness identity handle `_MERIDIAN_HARNESS` is not read by `from_env()`**, because it is spawn-local metadata, not a user policy override (see [Launch System](../architecture/launch-system.md#_meridian_harness-child-env-injection)). A plain `MERIDIAN_HARNESS` env var is not read anywhere; the harness comes from CLI flags, profile and config (`primary.harness`, `defaults.harness`).

### What goes where

| Setting | Where to put it |
|---|---|
| Default model for all spawns | `meridian.toml` → `defaults.model` |
| Default model for primary (interactive) | `meridian.toml` or `meridian.local.toml` → `[primary].model` |
| Model alias definitions | `mars.toml` → `[models]` (travel with the package) |
| Per-agent model (in profile) | Agent profile frontmatter → `model:` |
| Per-agent override (project experiment, committed) | `mars.toml` → `[agents.<name>]` |
| Per-agent override (local experiment, uncommitted) | `mars.local.toml` → `[agents.<name>]` |
| One-off model override | CLI → `-m model_name` |
| Environment-scoped override | `MERIDIAN_MODEL=...` env var |

**Agent overlay example** (override `tech-lead` for a local experiment):
```toml
# mars.local.toml
[agents.tech-lead]
model = "gpt55"
effort = "medium"

[[agents.tech-lead.model-policies]]
match = { model-glob = "gpt*" }
override = { effort = "medium", autocompact = 120000 }
```

## MeridianConfig: File Precedence

`MeridianConfig` is a plain Pydantic model (`src/meridian/lib/config/models.py`), not a
`pydantic-settings` class. There are no implicit settings sources: `load_config_with_sources()`
in `src/meridian/lib/config/loader.py` merges explicitly, low to high:

```
built-in defaults (model field defaults)
  < ~/.meridian/config.toml (or MERIDIAN_CONFIG path)
    < meridian.toml
      < meridian.local.toml
        < env vars (only names declared in the option catalog)
```

`layers.iter_config_layers()` discovers and reads the three files; the hook registry and
config inspection use the same seam. Each file is validated against `MeridianConfig` on its
own, so an error names the key and the file, then the validated layers deep-merge (tables
overlay, lists replace), env values overlay last, and the result validates once more as the
final `MeridianConfig`. `load_config_with_sources()` keeps one `ConfigValueSource` per set
field, which is what `meridian config show` renders.

**The models are the only validation path.** Constraints live on model fields and
validators. Unknown-key warnings come from a second validation of each section with
Pydantic's call-time `extra="allow"`, reading `model_extra` off the validated tree, so
Pydantic owns the traversal; the only warning-policy `ClassVar` is
`config_silences_extras`, and `extra="forbid"` models still hard-fail
([decision](../decisions/codebase-cleanup.md#d-unknown-keys-from-pydantic-pydantic-does-the-traversal-for-unknown-key-warnings)).
The alias-mapped `[defaults]`, `[spawn]` and `[timeouts]` keys are reported by the
loader's catalog mapping instead. `[env]`, `[hooks]`, `[work]`,
`[context]` and `[workspace]` validate through descriptor models in
`config/dynamic_sections.py`; errors format through `config/errors.py`. The hand-written
TOML normalizers and the re-entrant `BaseSettings` load were removed in cleanup phase 2
([cleanup program](../codebase/cleanup-program.md)); do not reintroduce a second
validator for a shape a model already declares.

### Key config sections

```toml
# meridian.toml (or meridian.local.toml or ~/.meridian/config.toml)

[defaults]
model = ""             # empty = harness picks; default harness is "codex"
harness = "codex"
max_depth = 3          # max spawn nesting

[timeouts]
wait_minutes = 30.0
kill_grace_minutes = 5.0

[primary]              # overrides for interactive primary launch
model = "claude-sonnet-4-5"
harness = "claude"
approval = "auto"
autocompact = 120000   # context tokens (min 1000); autocompact_pct is the 1-100 form

[state]
retention_days = 30    # -1 = never prune

[output]
verbosity = "normal"   # quiet | normal | verbose | debug
format = "text"        # text | json
```

Config loading preserves model tokens as strings — it does **not** resolve aliases during load. Alias resolution happens at launch time in the model resolution pipeline.

### Config CLI

```bash
meridian config show          # all resolved values annotated with their source
meridian config set/get/reset # operate on meridian.toml, not user config
```

`meridian config show` is the fastest way to understand what's actually active: it annotates each value with whether it came from builtin, user, project, local, or env.

`autocompact` is a token count and `autocompact_pct` is a percentage from 1 to
100. Claude takes the token count as `--autocompact` (100000–1000000) and the
percentage as `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`; Codex takes both as `-c` keys;
other harnesses report them not applied
([D-compaction-projected](../decisions/bundle-field-consumption.md#d-compaction-projected)).
Neither field has an "off" value.

### Declaring config keys

Each key is declared on its model in `lib/config/models.py` (or `harness_models.py`,
`history_models.py`) with
`config_field(canonical_key, value_kind=, file_aliases=, env_vars=, ...)`.
`build_option_catalog` walks the `MeridianConfig` model tree to build the option
catalog (`loader.OPTION_CATALOG`) behind `meridian config show/get/set` and the
env-var overlay. Before adding keys, know these rules:

- **Env names are explicit.** They are never derived from the TOML path. The
  loose convention is `MERIDIAN_<SECTION>_<KEY>` for plain sections and
  `MERIDIAN_HARNESS_<KEY>_<HARNESS>` for per-harness keys, so make the key and
  its env suffix match.
- **No metadata, no env var.** The loader reads env vars only from catalog
  entries, so an unannotated field validates from TOML but has no env var and
  no generic `config get/set` (several still do; `lib/config/.context/FUTURE`).
- **One annotated model can't be shared by harnesses.** If one annotated
  sub-model hangs off each harness profile, the catalog walks it once per
  harness and raises `Duplicate config canonical key`. Per-harness fields are
  therefore built by one profile factory, `harness_profile_model()` in
  `harness_models.py`, which gives
  each harness its own canonical key and env name (for example
  `MERIDIAN_HARNESS_WAIT_YIELD_SECONDS_PI`).
- **List values from env arrive as raw strings.** `parse_env_scalar` returns a
  `str_list` env value unsplit, and an empty env value is rejected, so an env
  var can't blank a file value.

`[notify]`, `[idle]`, and `[harness.<h>.idle]` use that generic path. The
per-harness idle models are still separate generated model classes, built by
`harness_idle_model()`, because each field needs a unique canonical key and env
name. Idle's effective-value resolver adds one feature-specific rule on top of
ordinary file loading: compare precedence level first, then use the
per-harness value over the global value within that level. Thus
`MERIDIAN_HARNESS_IDLE_ENABLED_CODEX` beats `MERIDIAN_IDLE_ENABLED`, either env
value beats every file, and a per-harness file key beats a global file key
after the file layers merge. See
[Idle Notifications](../decisions/idle-notifications.md#d-idle-config--separate-notify--idle-namespaces-standard-precedence).

## Project Root Discovery

`MeridianConfig` loads relative to the resolved project root. `resolve_project_root_resolution()` in `src/meridian/lib/config/project_root.py` resolves in three tiers (no ancestor walk):

1. Explicit argument / `-C` flag (see below)
2. `MERIDIAN_PROJECT_DIR` env var
3. Current working directory (literal CWD — no walk-up)

### The `-C` / `--directory` Flag

`-C <path>` (alias `--directory`) is the CLI boundary mechanism for explicit project root override. It is a **global flag** — it precedes any subcommand:

```bash
meridian -C ~/gitrepos/mars-agents spawn list
meridian -C ~/gitrepos/meridian-cli session log c8
```

**How it works:** `main.py` sets `MERIDIAN_PROJECT_DIR` to the given path in `os.environ` before any subcommand runs. The resolution layer reads from `os.environ` — no parameter threading through intermediate callsites. The `-C` path IS `MERIDIAN_PROJECT_DIR` for the duration of the command.

**Env cleanup:** When `-C` is active, `main.py` also unsets `MERIDIAN_RUNTIME_DIR` (if set). Because `-C` changes the project identity, a stale runtime-dir override from a different project would be silently wrong. Unsetting it forces the runtime root to be re-derived from the new project identity. See [state-model.md](state-model.md#root-resolution) for the full derivation chain and the decision rationale at [decisions/state.md](../decisions/state.md#directory-env-scope-eliminates-meridian_directory_explicit).

**Canonical use case:** Operating on a sibling repo from a different CWD — e.g., running spawn commands against `meridian-cli` while sitting in `mars-agents/`.

File locations derived from project root:
- `<project_root>/meridian.toml` — committed project config
- `<project_root>/meridian.local.toml` — local overrides (gitignored)
- `~/.meridian/config.toml` — user config (or `MERIDIAN_CONFIG` path)

## Approval Modes

Approval mode is one of the `RuntimeOverrides` fields and follows the same precedence chain. Valid values:

| Mode | Behavior |
|---|---|
| `default` | Harness decides (usually asks for confirmation) |
| `confirm` | User approves each tool call |
| `auto` | Auto-approve safe operations (harness-defined) |
| `yolo` | Approve everything |

Set via `--approval` flag, `MERIDIAN_APPROVAL` env var, profile frontmatter, or `[primary].approval` in config.

## Related

- [concepts/context-resolution.md](context-resolution.md) — how context paths (work, kb, extras) resolve separately from operational config
- [concepts/workspace-projection.md](workspace-projection.md) — workspace roots: a separate system from both config and context
- [concepts/model-resolution/overview.md](model-resolution/overview.md) — how model names become concrete models after precedence resolves
- [decisions/model-resolution.md#d72](../decisions/model-resolution.md#d72-agentsname-overlay-config-as-the-project-scoped-override-surface) — agent overlay config design rationale
- [decisions/model-resolution.md#d73](../decisions/model-resolution.md#d73-canonical-launch-parameter-compiler-in-compilerpy-with-pure-data-contract) — compiler-era rationale, now superseded by Mars bundle routing
- [operations/configuration-guide.md](../operations/configuration-guide.md) — practical config setup guide
- [decisions/idle-notifications.md](../decisions/idle-notifications.md) — built idle/notify namespaces and per-harness precedence
