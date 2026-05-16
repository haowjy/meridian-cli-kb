# operations/configuration-guide — Practical Configuration

Meridian configuration layers from project TOML, local TOML, user TOML, environment variables, and per-spawn runtime overrides. This guide covers what to configure where and how to verify resolution.

## Config Files

| File | Scope | Committed? |
|---|---|---|
| `meridian.toml` | Project — applies to everyone in this repo | Yes |
| `meridian.local.toml` | Local personal overrides for this repo | No (gitignored) |
| `~/.meridian/config.toml` | User — applies to all your projects | No |

File precedence is user < project < local. Environment variables override all files.

### Common Project Config

```toml
# meridian.toml

default_model = "sonnet"          # default model for spawns
default_harness = "claude"        # default harness

[primary]
model = "claude-opus-4-6"         # model for interactive primary (meridian with no spawn)
approval = "auto"                 # auto-approve safe tool calls
autocompact = 80                  # compact context at 80% usage
timeout = 3600.0                  # 1-hour session timeout

[state]
retention_days = 30               # prune artifacts older than 30 days
                                  # -1 = never prune, 0 = prune immediately

[output]
verbosity = "normal"              # quiet | normal | verbose | debug
format = "text"                   # text | json
```

### Common Local/User Config

```toml
# ~/.meridian/config.toml

[primary]
model = "claude-sonnet-4-6"       # personal default for primary launches
approval = "confirm"              # require confirmation for all tool calls

[state]
retention_days = 60               # keep artifacts longer on this machine
```

## Workspace Config — Multi-Repo Context

Workspace config lives in the `[workspace]` section of the standard project config files:

| File | Scope | Committed? |
|---|---|---|
| `meridian.toml` | Team-wide path conventions | Yes |
| `meridian.local.toml` | Per-developer machine overrides | No (gitignored) |

```bash
# Initialize local workspace config
meridian workspace init
```

### Adding Sibling Repos

**Shared convention (committed):**

```toml
# meridian.toml
[workspace.frontend]
path = "../meridian-web"

[workspace.docs]
path = "../docs-repo"
```

**Per-developer override (local, gitignored):**

```toml
# meridian.local.toml
[workspace.frontend]
path = "/home/me/different-location/meridian-web"   # my checkout is elsewhere

[workspace.private-data]
path = "/data/local-only"   # only I have this
```

Entry names (`frontend`, `docs`) are stable merge keys — local entries with the same name
replace committed entries. Local-only entries are appended.

Relative paths resolve against the project root (directory containing `meridian.toml`).
Tilde expansion works. Existing roots are injected into harness launches as
additional file-access directories.

**Important:** An `"invalid"` workspace section (TOML decode error or missing `path`
field) blocks all launches. A missing root on disk is skipped (committed) or emits a
`workspace_local_missing_root` diagnostic (local). Check `meridian config show` to see
workspace status and which roots were projected vs skipped.

### Per-Harness Workspace Behavior

| Harness | How roots are injected |
|---|---|
| Claude | `--add-dir <path>` flag per root |
| OpenCode | `OPENCODE_CONFIG_CONTENT` env var with JSON permission config |
| Codex | `--add-dir <path>` flag per root (active; read-only sandbox modes may still limit effective access) |

### Migrating from workspace.local.toml

If you have the old `workspace.local.toml` format, convert it:

```bash
meridian workspace migrate
```

This reads the old file, generates named entries, and appends them to `meridian.local.toml`.
Review the generated names — they are provisional (basename-derived) and may not match the
canonical names your team will use in the committed `meridian.toml` convention.

## Key Config Fields

### Timeouts

```toml
kill_grace_minutes = 0.033        # ~2 seconds between SIGTERM and SIGKILL
guardrail_timeout_minutes = 0.5   # 30 seconds for guardrail checks
wait_timeout_minutes = 30.0       # meridian spawn wait checkpoint interval
```

### Retry

```toml
max_retries = 3                   # spawn retries on transient failure
retry_backoff_seconds = 0.25      # initial retry backoff (doubles each retry)
```

### Depth

```toml
max_depth = 3                     # max zero-based delegated spawn depth
                                  # prevents runaway agent recursion
```

### Model and Harness Per-Harness Defaults

```toml
[harness]
claude = "claude-sonnet-4-5"      # model when harness is claude but no model specified
codex = "gpt-4o"
opencode = "gemini-pro"
```

## Environment Variable Overrides

Most environment variables override their corresponding config field; `MERIDIAN_HARNESS` is intentionally excluded because it is spawn-local runtime metadata, not a user policy override:

| Variable | Config field |
|---|---|
| `MERIDIAN_MODEL` | Per-spawn model |
| `MERIDIAN_HARNESS` | Spawn-local selected harness metadata; not consumed as a config override |
| `MERIDIAN_EFFORT` | Effort level |
| `MERIDIAN_APPROVAL` | Approval mode |
| `MERIDIAN_TIMEOUT` | Spawn timeout |
| `MERIDIAN_AUTOCOMPACT` | Autocompact percentage |
| `MERIDIAN_STATE_RETENTION_DAYS` | `state.retention_days` |
| `MERIDIAN_HOME` | User state root (default: `~/.meridian/`) |
| `MERIDIAN_RUNTIME_DIR` | Runtime state root override (bypasses UUID lookup) |

## Config Precedence in Practice

Full precedence chain, highest to lowest, for any per-spawn field:

```
CLI flags (-m, --approval, ...)
  ↓
Environment variables (MERIDIAN_MODEL, ...)
  ↓
Agent profile frontmatter (model: ..., approval: ...)
  ↓
meridian.local.toml / meridian.toml / user config
(primary.* for primary launches, defaults.* for spawned subagents)
  ↓
Built-in defaults
```

Each field resolves independently. You can override the model without affecting the harness, or set approval in the profile without affecting timeout.

**Example:** If you run `meridian spawn -m claude-sonnet-4-6 -a my-agent` and `my-agent` profile specifies `model: gpt-4o`, the CLI flag wins and `claude-sonnet-4-6` is used. The harness is then derived from `claude-sonnet-4-6`, not from the profile's default harness.

## Inspecting Resolved Config

```bash
# Show all resolved config values with their sources
meridian config show

# Show a specific field
meridian config get default_model

# Set a project config value
meridian config set default_model sonnet

# Reset a field to default
meridian config reset default_model
```

`meridian config show` annotates each value with its source, such as `builtin`, `user`, `project`, `local`, or `env`. This is the fastest way to debug unexpected behavior — if a value isn't what you expect, `config show` shows why.

## Context Config — Work and KB Locations

```toml
# meridian.toml

[context]
work = ".meridian/work"              # default; relative to repo root
kb = ".meridian/kb"                  # default
work_archive = ".meridian/archive/work"  # default
```

Override to point work/kb at external paths (e.g., a shared KB repo):

```toml
[context]
kb = "../shared-kb"                  # relative to repo root
```

Use `meridian context` to verify resolved paths:

```bash
meridian context         # show all context paths
meridian context kb      # just the KB path
meridian context work    # just the work path
```

## Cross-References

- [../concepts/config-precedence.md](../concepts/config-precedence.md) — conceptual model of config precedence and runtime overrides
- [troubleshooting.md](troubleshooting.md) — workspace config invalid pattern
- [../principles/design-principles.md](../principles/design-principles.md) — progressive disclosure principle
