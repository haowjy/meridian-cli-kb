# Native Config Passthrough

**Canonical term:** native config passthrough. **Field name:** `native-config`.

`native-config` is a structured escape hatch in agent profiles for raw target-harness
configuration that is NOT portable Meridian semantics. It lives under
`harness-overrides.<target>.native-config`.

**Related pages:**
- [../architecture/mars-launch-bundle.md](../architecture/mars-launch-bundle.md) — launch-bundle system and projection architecture
- [skill-schema.md](skill-schema.md) — universal skill frontmatter and harness lowering
- [../decisions/package-management.md](../decisions/package-management.md) — D82 decision rationale

---

## What It Is

Some harnesses expose config knobs that have no cross-harness equivalent. Rather than
requiring Mars/Meridian to claim portable semantics for every harness option, `native-config`
provides a pass-through channel:

```yaml
harness-overrides:
  codex:
    sandbox: workspace-write        # portable Meridian semantics
    native-config:
      sandbox_workspace_write.network_access: true   # raw Codex config
```

The motivating case: Codex network access under `workspace-write` sandboxing. No
portable equivalent exists across harnesses.

---

## Structural Separation from Portable Policy

`native-config` and portable tool policy (`tools`, `disallowed-tools`, `mcp-tools`)
are independent concerns:

| Concern | Schema location | Mars role | Meridian role | Portable? |
|---------|----------------|-----------|---------------|-----------|
| Tool policy | `tools`, `disallowed-tools`, `mcp-tools` at root or `harness-overrides.<target>` | Parse, validate, resolve into bundle `tools` field | Project per harness to CLI flags / config | Yes |
| Native config | `native-config` under `harness-overrides.<target>` | Validate shape only | Project per harness at launch time | No |

They appear as independent fields in the launch bundle and go through independent
projection paths. A profile may have both; the bundle carries both.

---

## Schema

### Location in profile YAML

```yaml
harness-overrides:
  <harness-id>:
    # existing portable fields
    sandbox: workspace-write
    approval: auto
    tools:
      bash: allow

    # native-config — raw harness passthrough
    native-config:
      <string-key>: <serializable-value>
```

### Value constraints

| Property | Constraint |
|----------|-----------|
| Keys | Strings only. No interpretation by Mars. |
| Values | YAML/TOML-serializable: booleans, integers, floats, strings, arrays, maps. No functions, no YAML anchors, no YAML tags. |
| Nesting | Arbitrary depth for map values. |
| Empty | `native-config: {}` is valid and equivalent to omitting. |

### Validation (Mars)

Mars validates **shape only**: string keys, serializable values. Mars does NOT
interpret key names or enforce key semantics.

If a `native-config` key matches a known portable field name (`sandbox`, `approval`,
`effort`, `autocompact`, `autocompact_pct`, `skills`, `tools`, `disallowed-tools`,
`mcp-tools`), Mars emits a warning. The portable field takes precedence for Meridian
semantics; the `native-config` entry is preserved for native projection but flagged.

### Harness scoping

`native-config` under `harness-overrides.codex` applies only when the resolved target
harness is Codex. Mars selects the matching harness block and discards others. No
cross-harness merge.

### Precedence

CLI `--native-config` flag (future, not current slice) would take precedence over
profile. Within a profile, only the block matching the resolved harness contributes.
When multiple precedence layers provide `native-config` for the same harness, merge
is shallow replace at the top level (higher-precedence map replaces the lower
entirely). Deep merge of individual keys is deferred to avoid key-path conflict
semantics.

---

## Bundle Field

`execution_policy.native_config` in the launch bundle:

```json
{
  "execution_policy": {
    "sandbox": "workspace-write",
    "native_config": {
      "sandbox_workspace_write.network_access": true
    }
  }
}
```

Omitted from JSON when null (`skip_serializing_if = "Option::is_none"`). Only the
entries from the resolved target harness block are included.

---

## Per-Harness Projection Strategies

Meridian's harness adapter layer owns projection. Each harness declares its own
strategy — do not assume all harnesses have Codex-style dotted CLI overrides.

### Codex

**Strategy:** Repeated `-c <key>=<toml-value>` CLI flags.

Dotted notation for nested Codex config. Nested YAML map values at the top level
are not auto-flattened — the projector emits a warning and skips:
`"Codex -c does not support nested values; use dotted key notation instead."`

```yaml
native-config:
  sandbox_workspace_write.network_access: true
```

Projected: `codex ... -c sandbox_workspace_write.network_access=true`

Value serialization:

| TOML type | Serialization |
|-----------|--------------|
| Boolean | `true` / `false` |
| Integer | decimal string |
| Float | decimal string |
| String | TOML-quoted if special chars, bare otherwise |
| Array | TOML inline array |
| Table (top-level) | Warning + skip — use dotted notation |

### Claude

**Strategy:** Temporary settings JSON file, passed via `--settings <path>`.

Keys may be nested objects (merged directly) or dot-notation (expanded to nested
path). Meridian writes the JSON file, passes the path as `--settings`, and cleans
up after the harness process exits.

Does NOT merge with the user's `~/.claude/settings.json`. Claude's `--settings` flag
loads a standalone settings overlay per-session — user base settings remain untouched.

```yaml
native-config:
  permissions:
    allow:
      - "Bash(*)"
      - "Read(*)"
  env:
    CLAUDE_CODE_MAX_TURNS: "50"
```

Temp file (`/tmp/meridian-claude-settings-<uuid>.json`):
```json
{
  "env": {"CLAUDE_CODE_MAX_TURNS": "50"},
  "permissions": {"allow": ["Bash(*)", "Read(*)"]}
}
```

Projected: `claude ... --settings /tmp/meridian-claude-settings-a1b2c3d4.json`

### OpenCode

**Strategy:** Deep-merged into `OPENCODE_CONFIG_CONTENT` env var JSON.

Meridian already uses `OPENCODE_CONFIG_CONTENT` to inject workspace-root
permissions. Native-config generalizes this mechanism. Ordering:

1. Parent `OPENCODE_CONFIG_CONTENT` (inherited env)
2. Workspace root permissions (existing behavior)
3. `native-config` entries (highest precedence, native-config takes over workspace on overlap)

**No `-c` flag.** The OpenCode CLI does not expose a Codex-style `-c key=value`
override. All native config goes through `OPENCODE_CONFIG_CONTENT` or static config
files. Do not design around a `-c` flag.

```yaml
native-config:
  permission:
    external_directory:
      "/data/**": "allow"
      "/models/**": "allow"
  context:
    max_tokens: 200000
```

Parent `OPENCODE_CONFIG_CONTENT` (from workspace projection):
```json
{"permission":{"external_directory":{"/workspace/**":"allow"}}}
```

Merged result:
```json
{"context":{"max_tokens":200000},"permission":{"external_directory":{"/data/**":"allow","/models/**":"allow","/workspace/**":"allow"}}}
```

### Cursor (Experimental)

**Strategy:** Only the `mcp` key has a known projection surface (`.cursor/mcp.json`).
Unknown keys warn rather than silently drop — fail-clear semantics.

```yaml
native-config:
  mcp:
    servers:
      context7:
        command: npx
        args: ["-y", "@context7/mcp"]
  experimental_feature: true   # unknown key
```

Result:
- `mcp.servers.context7` → `.cursor/mcp.json`
- Warning: `native-config key 'experimental_feature' has no known Cursor projection surface; skipped`

### Pi (Future)

Not yet defined. Part of the future Pi contract.

---

## Risks

**R1: Schema drift.** Harness renames a config key → profile breaks silently. No automated
detection. Profile authors must track harness config changes.

**R2: Secrets in native-config.** Profiles are source artifacts that may be committed
to git. Guidance: use env vars for secrets, not profile YAML. Temp files (Claude `--settings`)
use restrictive permissions (`0600`). Mars does not scan values for secrets.

**R7: Accidental leakage into model prompt.** Enforced structurally — separate code paths
for native-config extraction and prompt composition. Tested with prompt-surface
isolation invariant.

---

## Promotion Path

If a native knob proves universally valuable across harnesses, it may be promoted
to a first-class portable field in a future schema revision. The escape hatch does
not block future standardization — it is the staging ground for it.

Known portable fields that went through this path are already first-class:
`sandbox`, `approval`, `effort`, `autocompact`, `autocompact_pct`.
