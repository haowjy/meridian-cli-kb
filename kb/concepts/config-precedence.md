# Config Precedence

Meridian resolves spawn configuration through two distinct systems: a file-backed settings loader (`MeridianConfig`) and a per-spawn runtime override stack (`RuntimeOverrides`). Understanding which system handles which setting is the key to predicting what any given spawn will actually use.

## Two Systems, Two Problems

**`MeridianConfig`** holds persistent, project-wide operational settings: retry policies, timeouts, verbosity, retention. These are loaded from TOML files at startup and apply broadly across all spawns unless overridden.

**`RuntimeOverrides`** holds per-spawn behavioral choices: which model, which harness, which approval mode. These are assembled fresh for each spawn from CLI flags, environment variables, agent profile frontmatter, and config file defaults — then merged with first-non-`None` semantics.

Conflating the two systems would make it impossible to cleanly answer "what should this specific spawn do?" versus "what are the project-wide defaults?" — see [launch decisions](../decisions/launch.md) for the reasoning behind this separation.

## RuntimeOverrides: Launch Policy Precedence

Each field in `RuntimeOverrides` resolves independently via `resolve(*layers)` in `src/meridian/lib/core/overrides.py`. The merge is first-non-`None` across layers in order:

```
CLI flags / request
  → env vars (MERIDIAN_MODEL, MERIDIAN_APPROVAL, …)
    → [agents.<name>] overlay (meridian.local.toml > meridian.toml > ~/.meridian/config.toml)
      → agent profile frontmatter
        → project/user config defaults
          → model alias defaults
            → hardcoded harness fallback
```

**Agent overlay** (`[agents.<name>]` in any config file) sits between env vars and profile frontmatter. It lets project operators override per-agent runtime policy without editing generated `.mars/agents/` profiles. See [decisions/model-resolution.md](../decisions/model-resolution.md#d72-agentsname-overlay-config-as-the-project-scoped-override-surface) for full rationale.

The fields in `RuntimeOverrides` are: `model`, `harness`, `agent`, `effort`, `sandbox`, `approval`, `autocompact`, `timeout`.

**Each field resolves independently.** A CLI override of `-m gpt-5.4` affects model selection but does not affect approval mode. Approval mode runs its own precedence chain separately. There is no "whoever wins model also wins everything else."

### Routing vs Policy Fields

Fields are split into two categories, codified in `_ROUTING_OVERRIDE_FIELDS`:

- **Routing fields** (`model`, `harness`, `agent`) — determine *where* to run
- **Policy fields** (`effort`, `approval`, `sandbox`, `autocompact`, `timeout`) — determine *how* to run

`model_policy_scope()` on `RuntimeOverrides` returns a copy containing only the policy fields (i.e. everything not in `_ROUTING_OVERRIDE_FIELDS`). It's used by the bundle execution policy resolver in `policies.py` to isolate the policy field resolution path. Because the exclusion is set-based, any new policy field added to `RuntimeOverrides` is automatically included — no hand-curation needed.

**Policy fields have a more granular cascade than the simple layer order above.** The full 7-tier ladder for policy fields (`effort`, `approval`, `sandbox`, `autocompact`, `timeout`):

```
1. CLI flags (--effort, --approval, …)
2. Env vars (MERIDIAN_EFFORT, MERIDIAN_APPROVAL, …)
3. Matched model-policy rule — from overlay if overlay.model_policies is set, else from profile
4. Agent overlay generic defaults ([agents.<name>].effort etc.)
5. Profile generic defaults (top-level frontmatter)
6. Config-level defaults (primary.effort etc.)
7. Alias defaults
```

The **effective model-policy source** for tier 3 is three-state:
- Overlay has `model_policies = None` (key absent) → inherit profile model-policies.
- Overlay has `model_policies = ()` (key present, empty array) → suppress all conditional overrides.
- Overlay has `model_policies = (rules...)` → replace profile model-policies entirely.

This means a `model-policies` rule can override a profile's generic `approval: auto` for a specific model, while CLI `--approval` overrides both. Agent overlay generic defaults (tier 4) sit between the matched rule and profile defaults — they apply when no rule matches but an overlay value was set.

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

**`MERIDIAN_HARNESS` is not read by `from_env()`** — it is intentionally excluded because harness is spawn-local, not a user policy override. It does influence the config layer via `primary.harness`, but it does not enter the `RuntimeOverrides` env layer. See `src/meridian/lib/core/overrides.py:126-146`.

### What goes where

| Setting | Where to put it |
|---|---|
| Default model for all spawns | `meridian.toml` → `defaults.model` |
| Default model for primary (interactive) | `meridian.toml` or `meridian.local.toml` → `[primary].model` |
| Model alias definitions | `mars.toml` → `[models]` (travel with the package) |
| Per-agent model (in profile) | Agent profile frontmatter → `model:` |
| Per-agent override (project experiment, committed) | `meridian.toml` → `[agents.<name>]` |
| Per-agent override (local experiment, uncommitted) | `meridian.local.toml` → `[agents.<name>]` |
| One-off model override | CLI → `-m model_name` |
| Environment-scoped override | `MERIDIAN_MODEL=...` env var |

**Agent overlay example** (override `tech-lead` for a local experiment):
```toml
# meridian.local.toml
[agents.tech-lead]
model = "gpt55"
effort = "medium"

[[agents.tech-lead.model-policies]]
match = { model-glob = "gpt*" }
override = { effort = "medium", autocompact = 40 }
```

## MeridianConfig: File Precedence

`MeridianConfig` is a `pydantic_settings.BaseSettings` subclass. Its source order (highest to lowest) is defined in `settings_customise_sources()` at `src/meridian/lib/config/settings.py:1136-1201`:

```
init args
  → env alias overrides
    → meridian.local.toml
      → meridian.toml
        → ~/.meridian/config.toml (or MERIDIAN_CONFIG path)
          → built-in defaults
```

In practice: `defaults < user config < project meridian.toml < local meridian.local.toml < env vars`.

### Key config sections

```toml
# meridian.toml (or meridian.local.toml or ~/.meridian/config.toml)

[defaults]
model = ""             # empty = harness picks; default harness is "codex"
harness = "codex"
max_depth = 3          # max spawn nesting
max_retries = 3

[timeouts]
wait_minutes = 30.0
kill_grace_minutes = 5.0

[primary]              # overrides for interactive primary launch
model = "claude-sonnet-4-5"
harness = "claude"
approval = "auto"
autocompact = 80

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

## Project Root Discovery

`MeridianConfig` loads relative to the resolved project root. `resolve_project_root()` in `src/meridian/lib/config/project_root.py` does not require git:

1. Explicit argument / `-C` flag (see below)
2. `MERIDIAN_PROJECT_DIR` env var
3. Ancestor directory containing `.mars/`
4. Ancestor directory containing legacy `.agents/skills/`
5. Ancestor directory containing `.git` (boundary heuristic, not a requirement)
6. Current working directory fallback

### The `-C` / `--directory` Flag

`-C <path>` (alias `--directory`) is the CLI boundary mechanism for explicit project root override. It is a **global flag** — it precedes any subcommand:

```bash
meridian -C ~/gitrepos/mars-agents spawn list
meridian -C ~/gitrepos/meridian-cli session log c8
```

**How it works:** `main.py` sets `MERIDIAN_PROJECT_DIR` to the given path in `os.environ` before any subcommand runs. The resolution layer reads from `os.environ` — no parameter threading through intermediate callsites. The `-C` path IS `MERIDIAN_PROJECT_DIR` for the duration of the command.

**Env cleanup:** When `-C` is active, `main.py` also unsets `MERIDIAN_RUNTIME_DIR` (if set). Because `-C` changes the project identity, a stale runtime-dir override from a different project would be silently wrong. Unsetting it forces the runtime root to be re-derived from the new project identity. See [state-model.md](state-model.md#user-root-resolution) for the full derivation chain and the decision rationale at [decisions/state.md](../decisions/state.md#directory-env-scope-eliminates-meridian_directory_explicit).

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
