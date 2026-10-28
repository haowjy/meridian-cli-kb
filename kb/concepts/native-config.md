# Harness Override Passthrough

**Canonical term:** harness override passthrough. **Field path:** `harness-overrides.<harness>`.

`harness-overrides.<harness>` is a target-native passthrough block in agent profiles. It is not portable Meridian semantics. Mars selects the block for the resolved harness and carries it as `execution_policy.native_config` in the launch bundle.

Two entries are not harnesses: `meridian-cli` and `meridian-flow` are consumer namespaces that Mars reserves and never routes to. meridian-cli reads its own block (`resident-rearm-budget`, `capabilities`) directly from the profile ([D105](../decisions/package-management.md#d105-meridian-hosts-own-full-name-namespaces-under-harness-overrides-2026-10-27)).

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
tools:
  ask_user: allow

harness-overrides:
  codex:
    model_reasoning_effort: high   # reserved: Meridian sets this from top-level effort, so it is skipped
    sandbox_workspace_write:
      network_access: true          # deprecated exception for one warning release; use top-level network
```

For a Codex launch bundle, Mars preserves the `codex` block exactly under `execution_policy.native_config`. Top-level `effort`, `skills`, `tools`, and `network` remain the Mars semantic controls. A passthrough key normally never overrides a setting Meridian computes from them; the deprecated Codex network key is the one warning-release exception described below.

---

## Structural Separation from Portable Policy

Harness passthrough and portable policy are independent concerns:

| Concern | Schema location | Mars role | Meridian role | Portable? |
|---------|----------------|-----------|---------------|-----------|
| Tool policy | top-level `tools`, `disallowed-tools` (MCP as `mcp(...)` rules inside them) | Parse, validate, resolve into typed `tools.rules` with each harness's native names | Compile per harness into flags, settings, config agent or extension, and report gaps ([tool permissions](../decisions/tool-permissions.md)) | Yes |
| Execution policy | top-level `effort`, `approval`, `sandbox`, `network`, `autocompact*` | Parse, validate, resolve into bundle `execution_policy` fields | Project per harness at launch time | Yes |
| Harness passthrough | `harness-overrides.<harness>` | Validate outer mapping and serializability only | Apply as written on Codex; report not applied elsewhere ([bundle fields](../decisions/bundle-field-consumption.md#d-native-config-codex-only)) | No |

A key inside `harness-overrides.<harness>` does not replace the top-level field with the same name. If a profile has both top-level `tools` and `harness-overrides.codex.tools`, the top-level value controls `bundle.tools`; the override value is preserved in `execution_policy.native_config.tools`, and Meridian passes it to Codex as the literal Codex key `tools`. Meridian does not read it as tool policy.

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

## Meridian Projection

Meridian applies the block on Codex only. Every other harness reports each key
`[unsupported] native_config: ... -> not applied for <Harness>` in `--dry-run`.

**Codex:** nested mappings flatten to dotted keys and values become TOML literals, emitted
as repeated `-c key=value` on both the subprocess and app-server paths:

```yaml
harness-overrides:
  codex:
    sandbox_workspace_write:
      network_access: true
```

Projected during its warning release: `codex ... -c
sandbox_workspace_write.network_access=true`. This key is deprecated in favor of top-level
`network`. A boolean wins for compatibility only when the ordinary native-config emitter can
address its segmented key and encode the value as a well-formed `-c` pair. A matching emitted
value reports native and a conflict reports partial. A literal dotted key, null, non-boolean,
empty map, or other unemitted candidate is reported ignored and leaves the portable network
projection in place. The next release reserves the key and removes this exception.

Other keys Meridian computes itself (model, reasoning effort, compaction, sandbox, approval,
writable roots, notify, web search, `features.*`, `agents.*`) are skipped with a report
line, so a passthrough cannot undo the profile's sandbox or tool switches. Keys `-c` cannot
address and values TOML cannot hold are reported unsupported; nothing in the block fails the
launch. The block can change any other Codex setting, including providers and MCP server
commands, so a package that carries one is trusted code. Rationale and the full skip rules:
[D-native-config-codex-only](../decisions/bundle-field-consumption.md#d-native-config-codex-only).
