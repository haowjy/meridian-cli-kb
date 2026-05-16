# Mars Compiler Cleanup Lessons

The `mars-compiler-cleanup` work item (2026-05) converted a rapid,
multi-phase implementation into a cleaner Mars compiler. This page preserves
the lessons that should guide future Mars work. For steady-state compiler
structure, see [architecture/mars-compiler.md](../architecture/mars-compiler.md).

---

## Windows Compatibility Needs to Be Designed Into Config Artifacts

### Hook Command Strategy

Hooks are shell scripts. Mars does not own the script content, only the
invocation. The cleanup kept `bash` as the interpreter and adjusted only
platform-specific quoting.

| Platform | Claude adapter | Codex adapter | OpenCode adapter |
|---|---|---|---|
| POSIX | `bash '...'` | `bash '...'` | `bash '...'` |
| Windows | `bash "..."` (adjusted quoting) | same | same |

Alternatives rejected:
- Dispatch by extension (`.sh` → bash, `.cmd` → cmd.exe, `.ps1` →
  powershell) — over-engineered for V0; all hooks were `.sh` files.
- Use `sh` instead of `bash` — hooks may use bash-isms.
- Require WSL — too restrictive.

### Other Windows Fixes

- **Agent profile name validation:** Names with Windows-invalid characters
  (`:`, `*`, `?`, `<`, `>`, `|`, `"`) emit a diagnostic error and skip
  the agent rather than producing an unwritable path.
- **`mars cache info --json`:** Paths now escape backslashes properly so
  output remains valid JSON on Windows.
- **Hook removal matching:** Separator-insensitive (`/` and `\` treated
  equivalently) so entries installed on one platform are found and removed
  on another.
- **Lock-file provenance paths:** Normalized before comparison to be
  separator-insensitive.
- **Filter exclusion patterns:** Matched separator-insensitively so globs
  authored on POSIX work on Windows.

---

## Index Lock Data Once Per Sync

Before the cleanup, lock item lookups in diff and target-sync loops were
O(n) per lookup. Three hot paths were affected: `diff.rs`,
`target_sync/mod.rs`, and `lock/mod.rs`.

**Fix:** Build a `DestPath → LockedItem` index once per sync run. All
subsequent lookups are O(1) amortized.

Two additional single-parse fixes:
- **Lock file:** Previously parsed twice (a version probe followed by a full
  parse). Now parsed once with a version-aware deserializer.
- **Model visibility:** `validate()` previously ran twice per sync. Now runs
  once.

Deferred because the complexity was higher than the cleanup scope:
- Agent files are still re-read from disk immediately after being written;
  future work should thread content through the pipeline instead.
- Tree walks still repeat across discovery, target build, and compiler phases.
- Frontmatter may still be parsed 3–7× per file across commands.

---

## Split Integration Tests by Behavior, Not by Historical Module

The integration test monolith (`tests/integration/mod.rs`, ~2,100 lines) was
split into focused top-level test files with a shared `tests/common/` module.

**Pattern:**

```
tests/
  common/
    mod.rs            # shared helpers: test roots, env bootstrap, etc.
  init_and_add.rs     # init + add command flows
  sync_behavior.rs    # sync scenarios, conflict resolution
  stale_cleanup.rs    # stale config entry removal
  windows_compat.rs   # Windows-specific paths and quoting
  ...
```

**Why separate files over submodules under `tests/integration/`:** Cargo
treats each file under `tests/` as an independent test crate, enabling
parallel execution and faster incremental builds. A `tests/integration/`
subdirectory would serialize all tests in it.

**Why not keep the monolith with section comments:** 2,100 lines in one file
is hard to navigate and creates large, opaque diffs. Splitting by concern
makes test changes reviewable in isolation.

All pre-existing test assertions were preserved through the restructure.

---

## Delete Scaffolding Once the Real Shape Emerges

The cleanup deleted substantial scaffolding left from rapid multi-phase
implementation:

| Item | Reason for deletion |
|---|---|
| `compiler/skills/` module | Placeholder for future feature, not needed now |
| `sync/translate.rs` | Forward-declared seam, never implemented |
| `lower_to_meridian`, `collect_lossiness_diags` | Unused paths in `agents/lower.rs` |
| `DroppedField`, `Exact` variants | Never matched |
| `raw` field (×2), `has_legacy_models`, `model_aliases`, `renames`, `package_depth` | Unused struct fields |
| `as_str` (×2), `is_empty` methods | Unreachable |

`sync_lock` retains `#[allow(dead_code)]` — it is an intentional RAII
keepalive for the file lock, not dead code in the semantic sense.

Post-cleanup: zero `dead_code` warnings, excluding the annotated exception.

---

## Route Warnings Through the Diagnostic Collector

V1→V2 lock promotion collision warnings were routed to `stderr` via
`eprintln!`, bypassing the `DiagnosticCollector` used everywhere else.

The fix routed those warnings through the collector, making them suppressible,
testable, and structurally consistent with the rest of Mars.
