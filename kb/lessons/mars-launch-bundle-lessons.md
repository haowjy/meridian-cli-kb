# Mars Launch-Bundle Lessons

The `mars-launch-bundle-design` work item (2026-05) designed the cross-repo
Mars/Meridian launch-bundle system, including native-config passthrough, portable
tool-policy preservation, and Cursor as an experimental target. This page captures
structural and process lessons. For the steady-state architecture, see
[architecture/mars-launch-bundle.md](../architecture/mars-launch-bundle.md).

---

## Deep Module Over Split Modules for Schema Files

`compiler/agents/mod.rs` reached 1203 lines of production code (plus ~495 test
lines inline). A reflexive response is to split into `types.rs / parse.rs /
diagnostics.rs`.

**Why that split is wrong here:** Every schema addition (native-config proved this)
touches types, parse helpers, diagnostics, AND tests in a single PR. Splitting into
three files would create shallow modules with high coupling between them — the
interface cost of the split exceeds the reading cost of the current file.

**Correct diagnosis:** `compiler/agents/mod.rs` has one concern — agent profile
schema definition and parsing. Types, parse helpers, diagnostics, and public parser
all change together for the same reason. That's one module.

**Lesson:** When everything changes together for the same reason, it's one module.
Split when concerns actually change independently, not just when lines grow.

**What to split:** Only the tests. The inline `#[cfg(test)] mod tests` (488 lines,
36 functions) was moved to `src/compiler/agents/tests.rs` via standard Rust
convention (`#[cfg(test)] mod tests;` as an external file). Production code stays
focused; test file gets its own reader context. Mechanical move — private helpers
remain accessible via `use super::*`.

---

## Launch Bundle Tests: Split Along Contract Boundaries

`tests/launch_bundle.rs` grew to 1659 lines across 34 tests covering 8+ distinct
contract areas. Splitting was warranted.

**The right split axis is contract boundaries, not line count:**

```
tests/
  launch_bundle/
    mod.rs              # shared helpers: setup_bundle_project*
    schema.rs           # output shape, slots, prompt-file rejection, model required
    prompt_surface.rs   # skills, system instruction, inventory, ordering
    routing.rs          # alias resolution, provider derivation, precedence chain
    execution_policy.rs # CLI overrides, profile, harness-override, alias
    tool_policy.rs      # mixed allow/deny, harness override replacement
    cursor.rs           # Cursor-specific behavior
    native_config.rs    # emission, precedence, invalid shape
    errors.rs           # unknown harness, invalid field, missing agent
```

Shared setup helpers (`setup_bundle_project`, `setup_bundle_project_with_agents`)
stay in `mod.rs` — importable from all submodules without duplication.

**Why Rust directory modules work identically:** Rust integration tests in `tests/`
work as separate crates. A `tests/launch_bundle/` directory with `mod.rs` is one
test crate, same as `tests/launch_bundle.rs`. The `#[test]` attribute controls test
discovery, not file location. Submodule files are internal organization.

**Lesson:** Test splits should be along contract boundaries. Split when adding
coverage for one contract area requires understanding an unrelated area. A 200-line
Cursor addition that forced reading 1400 lines of unrelated tests was the signal.

---

## OpenCode Has No `-c` Flag — Verify Before Assuming

The OpenCode CLI does not expose a Codex-style `-c key=value` runtime config
override. This was verified against local help output before committing to the
projection strategy.

**Consequence:** Native config for OpenCode must go through `OPENCODE_CONFIG_CONTENT`
env var or static config files. The design did not assume a flag shape that does not
exist.

**Lesson:** Always probe the real CLI surface before designing around assumed flag
shapes. Codex's `-c` flag is distinctive; other harnesses may not have an equivalent.
"It probably has a flag like Codex" is not sufficient justification — check first.

---

## Env Var Coordination Between Independent Projectors

For OpenCode, both the native-config projector and the tool-policy projector need
to write `OPENCODE_CONFIG_CONTENT`. If both projectors write independently, the
second one overwrites the first.

**Resolution:** Establish a defined composition ordering before implementation.
Native-config projector runs first. Tool-policy projector receives the native-config
projector's output as `parent_env`. Deep-merge at the JSON level for overlapping env
vars. The `merge_projections()` function enforces this ordering.

```python
def merge_projections(native, tools):
    return (
        native.args + tools.args,
        {**native.env_overrides, **tools.env_overrides},
        native.temp_files + tools.temp_files,
        native.warnings + tools.warnings,
    )
```

For env vars that both projectors set, the tool-policy projector's value wins in the
simple dict merge — but because tool-policy receives native-config output as its
`parent_env` and deep-merges, the final `OPENCODE_CONFIG_CONTENT` contains both.

**Lesson:** When two independent projectors can write the same env var, establish a
composition ordering and merge protocol before implementation, not after. The
interface question is: who receives whose output as input?

---

## Cursor Experimental Stability Is a Bundle-Level Signal

Cursor's config surfaces are actively evolving. Rather than treating it as a silent
equal of first-class harnesses, the bundle carries an explicit stability signal:

```json
{
  "provenance": {
    "harness_stability": "experimental"
  },
  "warnings": [
    "Cursor is an experimental launch-bundle target. The contract may change without notice."
  ]
}
```

This lets Meridian consumers (and users) know the contract is not stable, without
blocking use. Best-effort projection with clear warnings is better than either
blocking usage or silently claiming stability that doesn't exist.

**Lesson:** Experimental support with explicit signals is better than delayed support.
Surface the stability information in the artifact itself so consuming systems can
display it appropriately.
