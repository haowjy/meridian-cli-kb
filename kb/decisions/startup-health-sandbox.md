# Decisions: Startup, Health, and Sandbox Policy

Startup decisions cover descriptor-driven CLI startup, read/write bootstrap separation, process-seam telemetry install, doctor behavior, and sandbox projection policy. See [state-and-launch.md](state-and-launch.md) for the split-domain map.

## Doctor and Health Checks

### Doctor post-reconcile snapshot is authoritative for active spawn computation

**Decision:** `doctor_sync()` refreshes the spawn-store snapshot after orphan-run reconciliation. The post-reconcile snapshot (not the pre-reconcile snapshot) drives active-spawn-ID computation, stale-artifact scanning, and warning generation.

**Why:** The pre-reconcile snapshot included rows that the reaper was about to repair to terminal. Using it caused two bugs: (1) artifact dirs for repaired spawns were wrongly excluded from same-run pruning; (2) repaired spawn IDs could appear in the live-session warning, implying doctor had failed. Post-reconcile snapshot removes both bugs cleanly.

See [operations/health-checks.md](../operations/health-checks.md) — `doctor_sync()` control flow.

---

### Doctor delegates liveness to reaper, not independent classification

**Decision:** Doctor's orphan-run repair calls `reconcile_spawns()` (the reaper's authoritative logic) rather than implementing its own liveness heuristic.

**Why:** Duplicating liveness classification in doctor would drift from the reaper over time and create two places to maintain Windows-safe process-liveness semantics. Single source of truth in `state/reaper.py`.

---

### `live_active_spawns_remain` warning replaces `active_spawns_present`

**Decision:** The warning for active spawns that doctor chose not to prune is renamed `live_active_spawns_remain`, with explicit post-reconcile semantics. The old `active_spawns_present` code is removed.

**Why:** `active_spawns_present` fired on the pre-reconcile snapshot and could include stale orphaned rows already cleaned up. Users couldn't tell whether it meant "doctor failed" or "live sessions intentionally left alone." The new warning makes semantics unambiguous: if it fires, doctor checked, reconciled, and these spawns are genuinely live.

---

### Prune advice omitted for live-only doctor warnings

**Decision:** `summarize_doctor_output()` appends "Run 'meridian doctor --prune --global' to clean up." only when `stale_orphan_dirs > 0` or `stale_spawn_artifacts > 0`. `live_active_spawns_remain` warnings alone do not trigger the prune advice.

**Why:** Prune advice is only actionable when stale artifacts exist. Showing it for live-session warnings would be misleading — pruning would not remove the sessions and users would be confused when re-running doctor.

---

## Startup Pipeline

### Descriptor-driven startup: CommandDescriptor catalog owns startup policy

**Decision:** The CLI startup pipeline is driven by a lightweight command catalog of `CommandDescriptor` objects importable without triggering ops/harness/server module imports. The catalog is the single source of truth for routing validation, default output mode, redirects, and help metadata.

**Why:** Before this redesign, the startup path queried the extension registry for command routing, output mode defaults, and validation. The extension registry eagerly constructs full `ExtensionCommandSpec` objects including handlers, Pydantic models, and schemas — expensive imports that added ~174ms to startup even for read-only commands. Separating catalog metadata from execution specs removes this coupling.

**Alternatives rejected:**
- Keep extension registry as CLI source of truth, lazy-import only handlers — CLI startup still needs routing/validation/output-mode before handlers import; leaving the catalog coupled to extension spec construction doesn't help.
- Put all startup logic inside Cyclopts hooks — Meridian needs policy decisions before Cyclopts resolves lazy commands (trivial fast paths, read/write state separation).

See [architecture/startup-pipeline.md](../architecture/startup-pipeline.md).

---

### Thin entrypoint: root help/version avoids importing main

**Decision:** Root `--help` and `--version` are handled in `entrypoint.py` before `meridian.cli.main` is imported. Measured improvement: ~16ms vs ~190ms (11.6× faster for the trivial case).

**Why:** The import chain for `main.py` pulls in bootstrap, extension registry, and telemetry infrastructure. None of that is needed to print `--help` or `--version`. Lazy import strings in Cyclopts allow parent `--help` rendering without resolving child command handlers.

---

### Bootstrap split: `resolve_*` is pure, `ensure_*` may mutate

**Decision:** Bootstrap helpers follow a strict naming contract: `resolve_*` returns state without mutations (returns `None` when state doesn't exist); `ensure_*` may create directories, UUIDs, and `.gitignore` entries, and is only called on write-class command paths.

**Why:** The old `resolve_runtime_root_and_config()` hid write-side bootstrap inside a "resolve" helper. Commands like `spawn list` in a clean CI checkout would silently create `~/.meridian/projects/<uuid>/` directories as a side effect of "reading" state. This was a correctness bug for read-only commands and polluted CI environments.

---

### Telemetry install at process seam only, not in ops modules

**Decision:** `TelemetryBootstrap.install(plan: TelemetryPlan) -> TelemetryHandle` is called once per process, at the process entry seam (CLI entrypoint, spawn worker entrypoint, MCP server entrypoint). Operation-layer functions (`spawn_create_sync`, `run_chat_server`, extension dispatch) do not configure telemetry.

**Why:** Installing telemetry inside ops modules creates multiple entry points with divergent lifecycle behavior. The `BufferingSink → upgrade` pattern (install early, upgrade to a real sink once runtime root is known) adds complexity and only makes sense when telemetry is installed before the runtime root is resolved — which is no longer necessary once bootstrap is sequenced correctly.

**Consequences:** `setup_telemetry()` calls removed from `spawn/api.py`, `spawn/execute.py`, `chat_cmd.py`, and `server/main.py`. Read-only commands install no telemetry at all.

> [!FLAG] **Needs human review** — Current source still has `setup_telemetry()` call sites in `src/meridian/lib/ops/spawn/api.py` and no `TelemetryBootstrap` class was found during the 2026-05-05 structural check. Decide whether this decision record is aspirational, partially shipped, or stale.

---

### Output protocol: `to_cli_output()` dispatch, not `isinstance` in main

**Decision:** Command handlers emit results through a `to_cli_output()` protocol method on result types, rather than `isinstance` branches in `main.py`.

**Why:** `isinstance` dispatch in `main.py` requires importing spawn output types at module scope. That violates the startup pipeline boundary — command execution detail leaks into startup code, increasing import cost for every command even when the specific output types are irrelevant.

---

### Selective command registration: import only the group needed for the current invocation (cli-lazy-import-startup)

**Decision (2026-05):** `_register_commands_for_invocation()` in `main.py` reads the first positional token and registers only the command module group that token requires. `spawn` imports only spawn-related modules; `session` imports only session-related modules; etc. The full group-registration path fires only on `--help` (all commands needed for the help tree).

**Why:** The prior `_register_group_commands()` imported all command modules unconditionally at startup. Each module transitively pulled in ops, harness, and Pydantic models. Combined cost was ~300ms of import work even for read-only commands like `spawn list`.

**Performance:** `main.py` module-import time 460ms → 156ms. `--help` latency 140ms → 54ms.

---

### `lazy_dispatch.py`: defer command handler imports until invocation

**Decision (2026-05):** `src/meridian/cli/startup/lazy_dispatch.py` provides `make_lazy_command(lazy_target: str)` — returns a callable that imports and delegates to `"module.path:function_name"` only when invoked by Cyclopts. Command registration at startup is import-free.

**Why:** Cyclopts resolves lazy commands (imports and calls the handler) only when the user runs that specific command. Parent `--help` can be rendered without resolving any child commands. Before this, registration imported the handler module, defeating selective registration.

**Format:** `"module.path:function_name"` or `"module.path:group.function_name"`. The module and function path are separated by `:`. The function path supports attribute chains.

---

### `meridian.lib.__getattr__` for lazy library exports

**Decision (2026-05):** `src/meridian/lib/__init__.py` exports `Spawn`, `HarnessId`, `ModelId`, `SpawnId` via a module-level `__getattr__`. `import meridian.lib` no longer transitively imports `core.domain` or `core.types` unless a specific name is accessed.

**Why:** Many modules imported `from meridian.lib import HarnessId` at the top level. Each such import triggered the full `core.types` import chain. `__getattr__` defers the underlying module import until the attribute is accessed, which may be after startup has completed or may never happen for paths that don't use that type.

---

### `app_tree.py` defines app instances; command modules import from it, not vice versa

**Decision (2026-05):** `src/meridian/cli/app_tree.py` defines the top-level Cyclopts app instances (`spawn_app`, `session_app`, `work_app`, `ext_app`, etc.) without importing any command implementations. Command modules (`spawn.py`, `session_cmd.py`, `ext_cmd.py`, …) import the app objects from `app_tree.py` when they register their commands.

**Why:** The old pattern had `main.py` importing app objects, which required importing the modules that defined those objects, which pulled in all command implementations. The inversion breaks the circular dependency: `main.py` imports only `app_tree.py` (cheap); command modules are imported lazily by selective registration.

**`ext_app` note:** `ext_app` is defined in `app_tree.py` and imported by `ext_cmd.py`. This is the canonical inversion pattern — the app object lives in the tree file, not in the implementation module.

---

### Rootless commands: tool-level linters and config inspection run without a project (2026-06)

**Decision (#338):** Commands that operate on a path, CWD, or user-level config are classified as `StartupClass.READ_ROOTLESS` with `StateRequirement.NONE`. They run anywhere — no established project required. Commands re-tiered: `kg check`, `kg graph`, `qi`, `qi check`, `qi list`, `mermaid check`, `config show`, `config get`, `ext list`, `ext commands`, `doctor`.

**Why:** These commands were formerly classified as `READ_PROJECT` or `READ_RUNTIME`. In a clean no-project shell (no walk-up to find an ancestor project), dispatch hard-failed with "No Meridian project found" (exit 1) — even though the commands operate on file paths or user-level config and don't need a project. Walk-up (removed in #335) used to mask this by always finding *some* ancestor project, often the wrong one.

**Catalog helper:** `_read_rootless()` in `catalog.py` creates descriptors with `StateRequirement.NONE` while preserving the appropriate startup class (`READ_ROOTLESS`) and telemetry mode.

**`config get` / `config show`:** These degrade to user/builtin config layers when no project is established — they don't error, they show what's available.

### Established project: canonical resolver replaces scattered resolution + footgun (2026-06)

**Decision (#338):** Three near-duplicate project-root resolvers (`require_established_project_root`, `optional_cli_project_root_posix`, and the implicit resolution in `maybe_bootstrap_runtime_state`) were unified into one canonical resolver: `resolve_cli_project_root() -> CliProjectRoot` (typed, never raises). The old names became thin adapters.

**Why:**

1. **Single source of truth.** Three functions implementing slightly different policies created divergence risk. One canonical resolver with typed output eliminates the divergence.

2. **Footgun kill.** `require_established_project_root` raised `SystemExit` (a `BaseException`) directly. `maybe_bootstrap_runtime_state()` in `cli/bootstrap.py` wrapped the bootstrap in `except Exception`, which silently swallowed the `SystemExit` — causing confusing downstream crashes instead of "No Meridian project found." The fix: `resolve_cli_project_root()` never raises; the no-project exit is an explicit, intentional decision *outside* the `except Exception` guard.

3. **`exit_no_established_project()`** is the single CLI-edge `SystemExit(1)` — centralized so the exit message never diverges across call sites.

**Established-project predicate:** `cwd_has_project_id(cwd)` checks whether the literal CWD contains `.meridian/id` — **no ancestor walk**. This restores correct behavior when run from inside a project directory without `-C` / `MERIDIAN_PROJECT_DIR` (a pre-existing #335 over-correction had dropped the literal-cwd check). The predicate is intentionally one function; #341 will broaden it to `meridian.toml` / `mars.toml`.

**Not an established project:** CWD resolution where `cwd_has_project_id()` returns `False` — project-required commands still exit 1; rootless commands exit 0.

---

## Sandbox Projection Policy

### Fixed constraint: no `~/.meridian/` root projection into child sandboxes

**Decision:** Child sandboxes receive purpose-scoped projection only: project runtime roots, workspace/context roots, selected git clones, system temp. The global `~/.meridian/` tree is never projected.

**Why:** Projecting the whole user home into every child creates a cross-project interference surface — any nested child gains write access to sibling projects' state, global service files, lock namespaces. The constraint is fixed; the engineering question is how code that reaches for unprojected paths should behave.

See [architecture/sandbox-projection.md](../architecture/sandbox-projection.md).

---

### Doctor cache deleted — global scan moved to explicit opt-in

**Decision:** The `doctor-cache.json` subsystem was deleted. `maybe_start_background_doctor_scan()` (which ran `doctor_sync(global_=True)` as a background daemon thread at startup) and `consume_doctor_cache_warning()` are gone. `doctor_cache.py` and its tests are removed.

**Why:** The background global scan added automatic cross-project work, global writes, sandbox failures, and startup complexity in exchange for a one-line warning users can get by running `meridian doctor` explicitly. The tradeoff was wrong.

**What survives:** Cheap per-project repairs (stale session lock cleanup, orphan run reconciliation) remain as a non-blocking background daemon thread on `PRIMARY_LAUNCH` paths. These touch only the current project's runtime root — already sandbox-safe — and require no user action to stay useful.

**Governing principle:** "Do not do expensive things automatically." A proactive global nag is expensive in code complexity and runtime cost for low informational value.

---

### Doctor tier split: cheap local default, explicit global maintenance

**Decision:** `meridian doctor` without flags is per-project and cheap. `meridian doctor --global` is the explicit cross-project maintenance path, root-only.

Local doctor:
- Repairs stale session locks and orphan runs for the current project
- Scans/prunes stale spawn artifacts in the current project runtime root
- Does NOT traverse `get_user_home() / "projects"` for sibling projects
- Does NOT read or write global cache files

Global doctor:
- Walks `get_user_home() / "projects"` for orphan project-state directories
- Detects and prunes stale cross-project directories when `--prune --global`
- Guarded by `is_root_side_effect_process()` in `doctor_sync()` itself, not just CLI layer

**Why:** The old doctor always performed cross-project work implicitly. For nested sandbox contexts this was wrong (paths unprojected); for human invocations it was unexpectedly expensive and surprising. Splitting into explicit tiers aligns cost with intent.

---

### Chat is root-only in nested execution [SUPERSEDED]

> Superseded 2026-05-07 by [chat backend D32](chat-backend.md#d32).
> Nested `meridian chat` launch and management are now allowed below the
> project `max_depth` cap so delegated smoke/browser testers can exercise live
> chat flows. Chat state remains global; scoped nested runtime roots are tracked
> as deferred work in [future-work](../open-questions/future-work.md#scoped-nested-chat-runtime-roots).

**Decision:** All `meridian chat` subcommands fail fast with exit code 1 when `MERIDIAN_DEPTH > 0`. The rejection happens before any filesystem writes to `get_user_home()`. `server.py` module-level runtime state uses an unconfigured sentinel rather than a `get_user_home()` fallback, so the global dependency is explicit at command startup rather than hidden at import.

**Why:** Chat is global service state backed by `~/.meridian/chats/<id>/` and `chat-server.json`. These paths are not projected into child sandboxes. Moving chat state under project runtime roots would break the current client discovery model and change chat from a global service to a per-project concept — the wrong design. A root-only gate is the correct minimal fix.

**Alternatives rejected:**
- Move chat state under project runtime roots — changes product semantics; breaks current client discovery
- Project chat state paths into child sandboxes — too much projection expansion for state that delegated agents don't use

---

### Autosync lock failure skips gracefully (no new projection)

**Decision:** When `file_lock()` raises `PermissionError` or `OSError` during autosync lock acquisition, `GitAutosync.execute()` returns `HookResult(outcome="skipped", skip_reason="lock_permission_error", success=True)`. No new workspace projection paths added for the lock directory.

**Why:** Projecting `~/.meridian/locks` would create a cross-project interference surface and would not guarantee correct serialization anyway — lock identity depends on clone paths derived from user config that may be absent or different in nested contexts (Gap 4 dependency). Autosync is best-effort by design; failure to acquire the lock is already a valid skip condition.

---

### Implicit user config degrades gracefully in nested contexts

**Decision:** When the implicit default user config path (`~/.meridian/config.toml`) is inaccessible in a nested process, treat it as absent and emit a structured log warning (one-shot per process, to config module logger). Explicitly supplied config paths (via argument or `MERIDIAN_CONFIG`) keep fail-fast behavior.

**Why:** The config read is optional and read-only. Silent degradation would hide root/nested behavioral differences. A log warning makes the difference diagnosable without adding complexity of config forwarding via env vars.

**Rejected alternative:** Forward parent-resolved config to children via env var — wrong layer. It duplicates policy resolution state, complicates config precedence, and risks serving stale policy.

---

## Related

- [../architecture/startup-pipeline.md](../architecture/startup-pipeline.md) — startup architecture
- [../architecture/sandbox-projection.md](../architecture/sandbox-projection.md) — sandbox projection mechanism
- [../operations/health-checks.md](../operations/health-checks.md) — doctor behavior
- [state-and-launch.md](state-and-launch.md) — compatibility map for the previous combined decision page
