# Harness Override Passthrough

**Canonical term:** harness override passthrough. **Field path:** `harness-overrides.<harness>`.

`harness-overrides.<harness>` is a target-native passthrough block in agent profiles. It is not portable Meridian semantics. Mars selects the block for the resolved harness and carries it as `execution_policy.native_config` in the launch bundle.

**Related pages:**
- [../architecture/mars-launch-bundle.md](../architecture/mars-launch-bundle.md) — launch-bundle system and projection architecture
- [skill-schema.md](skill-schema.md) — universal skill frontmatter and harness lowering
- [../decisions/package-management.md](../decisions/package-management.md) — decision rationale

---

## What It Is

Some harnesses expose config knobs that have no cross-harness equivalent. Rather than requiring Mars/Meridian to claim portable semantics for every harness option, `harness-overrides.<harness>` provides a pass-through channel:

```yaml
effort: low
skills: [planning]
tools: [ask_user]

harness-overrides:
  codex:
    effort: high                  # target-native key; does not replace top-level effort
    skills: [codex-only-skill]     # target-native key; does not replace top-level skills
    sandbox_workspace_write.network_access: true
```

For a Codex launch bundle, Mars preserves the `codex` block exactly under `execution_policy.native_config`. Top-level `effort`, `skills`, and `tools` remain the Mars semantic controls.

---

## Structural Separation from Portable Policy

Harness passthrough and portable policy are independent concerns:

| Concern | Schema location | Mars role | Meridian role | Portable? |
|---------|----------------|-----------|---------------|-----------|
| Tool policy | top-level `tools`, `disallowed-tools`, `mcp-tools` | Parse, validate, resolve into bundle `tools` field | Project per harness to CLI flags / config | Yes |
| Execution policy | top-level `effort`, `approval`, `sandbox`, `autocompact*` | Parse, validate, resolve into bundle `execution_policy` fields | Project per harness at launch time | Yes |
| Harness passthrough | `harness-overrides.<harness>` | Validate outer mapping and serializability only | Interpret/project per resolved harness | No |

A key inside `harness-overrides.<harness>` does not replace the top-level field with the same name. If a profile has both top-level `tools` and `harness-overrides.codex.tools`, the top-level value controls `bundle.tools`; the override value is preserved in `execution_policy.native_config.tools` for the Codex adapter/runtime to interpret.

---

## Schema

### Location in profile YAML

```yaml
harness-overrides:
  <harness-id>:
    <target-native-key>: <serializable-value>
    <another-target-native-key>:
      nested: values
```

### Value constraints

| Property | Constraint |
|----------|-----------|
| Harness keys | Known keys: `claude`, `codex`, `opencode`, `cursor`, `pi`. Unknown keys warn and are preserved for forward compatibility. |
| Target keys | Strings only. No interpretation by Mars. |
| Values | YAML/TOML/JSON-serializable: booleans, integers, floats, strings, arrays, maps. No null values. |
| Nesting | Arbitrary depth for map values. |
| Empty | Empty harness block is valid and equivalent to omitting for launch-bundle native config. |

### Validation (Mars)

Mars validates shape and serializability only. Mars does not interpret key names, enforce key semantics, or warn when a target-native key happens to match a portable Mars field name.

### Harness scoping

Mars selects the block matching the resolved launch harness and ignores other blocks for `execution_policy.native_config`. The source profile still preserves the full `harness-overrides` table.

---

## Bundle Field

`execution_policy.native_config` in the launch bundle contains the matching block exactly:

```json
{
  "execution_policy": {
    "effort": "low",
    "native_config": {
      "effort": "high",
      "skills": ["codex-only-skill"],
      "sandbox_workspace_write.network_access": true
    }
  }
}
```

Omitted from JSON when null (`skip_serializing_if = "Option::is_none"`).

---

## Per-Harness Projection Strategies

Meridian's harness adapter layer owns projection. Each harness declares its own strategy; do not assume all harnesses have Codex-style dotted CLI overrides.

### Codex

**Strategy:** Repeated `-c <key>=<toml-value>` CLI flags where supported by the Codex adapter.

Dotted notation can represent nested Codex config. Nested map values may require harness-adapter-specific handling.

```yaml
harness-overrides:
  codex:
    sandbox_workspace_write.network_access: true
```

Projected conceptually: `codex ... -c sandbox_workspace_write.network_access=true`

### Claude

**Strategy:** Temporary settings JSON file or CLI flags owned by the Claude adapter.

```yaml
harness-overrides:
  claude:
    permissions:
      allow:
        - "Bash(*)"
        - "Read(*)"
    env:
      CLAUDE_CODE_MAX_TURNS: "50"
```

### OpenCode

**Strategy:** Deep-merged into `OPENCODE_CONFIG_CONTENT` env var JSON where supported.

```yaml
harness-overrides:
  opencode:
    permission:
      external_directory:
        "/data/**": "allow"
        "/models/**": "allow"
    context:
      max_tokens: 200000
```

