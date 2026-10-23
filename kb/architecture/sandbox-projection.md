# Architecture: Sandbox Permission Projection

When Meridian spawns a child agent, it projects filesystem paths into the child's sandbox. This page covers the **projection policy** — what can be projected, what cannot, and how code that reaches for unprojected paths should behave.

Related: [startup/sandbox decisions](../decisions/startup-health-sandbox.md#sandbox-projection-policy) — rationale for the fixed projection constraint. [architecture/startup-pipeline.md](startup-pipeline.md) — how the startup pipeline handles per-project repairs in sandboxed contexts. [operations/health-checks.md](../operations/health-checks.md) — doctor tier split driven by sandbox constraints.

## Fixed Projection Constraint

Child sandboxes may receive:

- **Project runtime roots** — `~/.meridian/projects/<uuid>/` for the current project
- **Project lock directory** — `~/.meridian/projects/.locks/`, so a sandboxed command can take the project's lifetime lock
- **Workspace/context roots** — work, kb, archive, strategy directories, configured git context clones
- **System temp** — platform temp directory

Child sandboxes **shall not** receive the whole `~/.meridian/` tree, nor `~/.meridian/projects/`
itself. The global user home is not projected. This is a fixed constraint, not a tunable.

The lock directory is shared across projects, so it is a wider grant than the runtime root:
a sandboxed agent can reach another project's lock file. [Decisions](../decisions/sandbox-platform-access.md#d-sandbox-state-roots)
records that risk as open for human review.

The design question is therefore not "how do we make user home writable?" but "which global paths are actually valid in nested execution, and how should each class of path behave?"

## Path Classification Framework

```mermaid
flowchart TD
    A[Code wants ~/.meridian/... path] --> B{What kind of state is it?}
    B -->|Per-project runtime state| C[Keep — runtime_root already projected]
    B -->|Automatic global cache| D[Delete — no projection, no fallback]
    B -->|Cross-project coordination| E[Wrap — skip gracefully on failure]
    B -->|Global service state| F[Bound — allow scoped cases, cap recursion]
    B -->|Optional user config read| G[Wrap — best-effort read + log warning]
    B -->|Explicit global maintenance| H[Gate — root-only library guard in entrypoint]
```

| Class | Examples | Nested policy |
|---|---|---|
| Per-project runtime state | `~/.meridian/projects/<uuid>/...` | Already projected — no change |
| Project lifetime lock | `~/.meridian/projects/.locks/<id>.lock` | Directory projected, created at bind (see Fixed Projection Constraint) |
| Repo/context state | work, kb, archive roots, context git clones | Already projected — no change |
| Automatic global cache | `doctor-cache.json`, background global scan | **Delete** — expensive automatic global work is wrong |
| Cheap per-project repairs | Stale session lock cleanup, orphan run reconciliation | **Keep** — daemon thread, current project root only |
| Cross-project coordination | Lock files serializing shared resources | **Wrap** — skip gracefully on `PermissionError` |
| Optional user config | `~/.meridian/config.toml` | **Wrap** — best-effort implicit read + structured log warning |
| Explicit global maintenance | `doctor --global`, prune/reaper-style scans | **Gate** — library-level guard before cross-project traversal |

## Governing Principle

**Do not do expensive things automatically.**

This principle drives the classification:

- Default startup must not launch background global maintenance to print a warning
- Cheap per-project repairs are acceptable on `PRIMARY_LAUNCH` paths if they touch only the projected runtime root
- `meridian doctor` without flags must stay cheap and per-project
- Cross-project cleanup is valid, but only when the user asks explicitly with `--global`

## Context-Root Existence Filtering

Meridian projects context roots (work, work_archive, kb, and extra context dirs) into the harness sandbox as environment variables (`MERIDIAN_CONTEXT_WORK_DIR`, etc.) and as sandbox-visible filesystem paths. Before projection, `_collect_context_projection_roots()` filters out roots that don't exist on disk.

**Why this is load-bearing:** Codex (bubblewrap) bind-mounts every projected root at exec time. If a source path is missing, `bwrap` aborts the **entire exec namespace** — every command in the worker fails, not just operations targeting the missing path. A not-yet-created context dir (commonly `work_archive` before the first archive) would silently break all command execution while the spawn still reported `succeeded` (see [terminal status semantics](../concepts/harness-abstraction.md#event-semantics-and-terminal-outcomes)).

Critically, a sandbox-broken worker cannot run **any** Meridian command either — including `meridian spawn done` — so it cannot self-report the failure. Only the harness-written report survives (Codex `-o` flag), with the blocker described in prose.

**Implementation:** Missing context roots are skipped at debug log level, not warning. A missing context dir is the normal pre-archive state — it must not pollute stderr on every launch.

**The lock directory is created, not filtered:** bind creates `~/.meridian/projects/.locks/` before projecting it, so the filter never drops it. A missing directory would abort Codex's namespace the same way a missing context root does.

**Git clone roots are unaffected:** `_collect_git_context_clone_roots` returns configured clone roots for git-backed context entries. These are intentionally still projected even when the clone hasn't happened yet — they exist on disk by launch time after `mars sync`.

**Other harnesses:** opencode uses `external_directory` allow-lists (missing entry is harmless — just not in the allow-list). Claude `--add-dir` to a missing path is non-fatal (Claude warns but continues). Codex is the only hard-affected harness.

## Gap-by-Gap Behavior

### Gap 1: Doctor Cache (Deleted)

The old `doctor-cache.json` existed only to support a startup nag from a background global scan. The subsystem was deleted:

- `doctor_cache.py` module removed
- `maybe_start_background_doctor_scan()` removed — no background global scans at startup
- `consume_doctor_cache_warning()` removed — no cache-based warning display
- Cheap per-project repairs (stale session locks, orphan runs) are retained as a background daemon thread on `PRIMARY_LAUNCH`; they touch only the current project's runtime root

Users discover cross-project stale state by running `meridian doctor --global` explicitly.

### Gap 2: Autosync Locks (Wrap — Skip on Failure)

Autosync lock files live in `~/.meridian/locks`, not projected in nested sandboxes. Lock acquisition failure now returns a skipped hook result:

- `PermissionError`/`OSError` during lock parent directory creation or lock file opening → `HookResult(outcome="skipped", skip_reason="lock_permission_error", success=True)`
- Existing `TimeoutError` handling is unchanged
- No new projection paths added

**Why not project `~/.meridian/locks`?** (The projects `.locks` directory above is projected despite the same kind of risk; see its decision.) Projecting `~/.meridian/locks` as shared writable creates cross-project interference — any nested child could create arbitrary lock files or hold them to force autosync skips. Also, lock identity is derived from clone paths that depend on user config which may not be visible in nested contexts — projecting the lock dir would not guarantee correct serialization anyway.

### Gap 3: User Config Read (Wrap — Best-Effort + Warning)

`~/.meridian/config.toml` is read-only and optional. Nested processes may not have access:

- If the implicit default path probe raises `OSError` → return `None` (treat as absent)
- If probe fails and `is_nested_meridian_process()` is true → emit a structured log warning including the inaccessible path (one-shot per process, to config module logger, not stderr)
- If user explicitly supplied a config path (via argument or `MERIDIAN_CONFIG`) → preserve fail-fast behavior

### Gap 4: Global Doctor Maintenance (Explicit Tier + Gate)

Cross-project doctor work intentionally traverses `get_user_home() / "projects"`:

- `doctor_sync(global_=False)` → no cross-project enumeration under `get_user_home() / "projects"`; per-project preparation (config surface, current project runtime root) is allowed because those paths access the *current* project's own state
- `doctor_sync(global_=True)` in nested execution → `RuntimeError` before any cross-project traversal; guard lives in `doctor_sync()` itself, not just CLI layer
- `doctor_sync(global_=True)` in root process → cross-project scan/prune allowed

## Sandbox vs. Root: Detection

`is_root_side_effect_process()` is the authoritative check for whether the current process may perform cross-project or global side effects. It returns false when `_MERIDIAN_DEPTH > 0`. Code that gates on this function:

- `reconcile_active_spawn()` — the reaper never persists outcomes or signals from nested processes
- `doctor_sync(global_=True)` — global maintenance rejected before user-home traversal

## What Was Not Changed

- **Autosync lock location** — stays in `~/.meridian/locks`. Temp directories are subject to platform cleanup (Apple 3-day policy, Windows Storage Sense, systemd-tmpfiles) and are unsuitable for coordination locks.
- **Config forwarding** — nested invocations remain fresh config readers; no parent-resolved config is forwarded via env var. `LaunchRuntime.config_snapshot` is an in-process prepare/execute boundary, not a cross-process transport.

## Cross-References

- [../operations/health-checks.md](../operations/health-checks.md) — doctor tier split and background per-project repairs
- [../decisions/startup-health-sandbox.md](../decisions/startup-health-sandbox.md#sandbox-projection-policy) — sandbox projection policy decisions
- [../architecture/startup-pipeline.md](startup-pipeline.md) — startup pipeline design
- [../concepts/hooks-and-plugins.md](../concepts/hooks-and-plugins.md) — autosync hook behavior
