# Open Questions: Process-Scope Cleanup

Process-scope cleanup gaps that remain after the durable process-scope ownership design. These items share one mechanism boundary: containment and recovery for subprocess trees that may outlive their owning spawn.

Related pages:
- [architecture/process-scope.md](../architecture/process-scope.md) — ownership model, scope projection, platform adapters, and cleanup policy
- [decisions/launch.md](../decisions/launch.md#d-process-scope-ownership) — decision rationale for durable process-scope ownership
- [future-work.md](future-work.md#state--spawn) — broader state/spawn deferred-work index

---

## Open Items

### PROC-004: Dead Wrapper / Live Reparented Child Escapes Group Kill

**Where:** `src/meridian/lib/platform/process_scope/posix.py`, reaper path

**The issue:** POSIX cleanup now handles the common dead-root case by attempting the
recorded process group and, when the group appears dissolved, scanning the full process
table for remaining members of that PGID. The remaining escape case is narrower: when
an intermediate process exits before the scope snapshot is taken and its children both
reparent to PID 1 and start a new process group, a subsequent `os.killpg` against the
original PGID cannot reach them. The psutil snapshot taken during scope setup also
misses them for the same reason.

**Observed scenario:** A harness wrapper forks a helper that itself forks a long-lived
tool worker. If the helper exits promptly, the tool worker is in the init group and
invisible to both the PGID kill and the psutil tree walk rooted at the helper.

**What's needed:** Either (a) a deeper containment mechanism (cgroup on Linux, fully
assigned Job Object on Windows covering all descendants at creation time), or (b) a
post-kill audit pass that checks for living processes with `MERIDIAN_SPAWN_ID` in their
env or cmdline. Option (a) requires OS-specific capabilities not yet scaffolded;
option (b) is heuristic.

**Why deferred:** Group kill plus the PGID sweep handles the common cases. The
double-reparent/new-PGID scenario requires harnesses that fork through disposable
intermediaries, which is uncommon in practice. Defer until there is a concrete observed
leak of this type.

**Decision context:** [decisions/launch.md](../decisions/launch.md#d-process-scope-ownership)
and [architecture/process-scope.md](../architecture/process-scope.md)

---

### PROC-007: Audit Remaining Metadata-Only Lifecycle Edges

**Where:** `src/meridian/lib/state/process_scope_projection.py`, background worker launch paths

**The issue:** Current managed backend and primary-attach paths record
`ProcessScopeSnapshot` entries in `process_scopes.json`, and `backend_lifecycle.json`
was removed. The remaining uncertainty is narrower: whether any background-worker or
metadata-only lifecycle path can still launch a long-lived process without recording a
scope sidecar. If such a path exists, cleanup falls back to recorded PIDs rather than
containment.

**What's needed:** Audit launch paths that do not go through `launch_managed_backend()`
or `PrimaryAttachLauncher` scope recording. Any path that can own a process tree should
record a `ProcessScopeSnapshot` before exposing the process.

**Why deferred:** The known managed-backend leak was fixed by centralizing launch and
recording scope snapshots. This item is now a coverage audit, not the primary cleanup
mechanism gap.

**Decision context:** [architecture/process-scope.md](../architecture/process-scope.md)

---

### PROC-009: `root_pid` Conflates Launcher, Health, and Containment Identity (issue #449)

**Where:** `src/meridian/lib/harness/connections/managed_backend.py`,
`src/meridian/lib/state/process_scope_projection.py`

**The issue:** `ProcessScopeSnapshot.root_pid` represents three concepts that can
diverge for managed backends: the launcher shim PID, the effective serving process
(health target), and the containment boundary (PGID). With npm Codex 0.144.4, the
recorded `root_pid` is Node (the npm shim); it forks a native `codex app-server`
child in the same PGID. Killing the recorded root reparents the native child to PID 1,
but `/readyz` remains HTTP 200 and the turn completes. The shim PID is therefore not
an effective health identity, while `pgid` remains the correct POSIX containment handle.

**Observed scenario:** Kill the recorded `root_pid` of a managed Codex backend.
The native vendor child reparents to PID 1, continues serving turns, and readiness
checks pass. The scope projection records a release for the root PID but the backend
is still alive and functional. Process scopes correctly clean up at normal terminal
time via PGID, so the terminal release path is not broken -- the issue is the interval
between shim death and terminal, during which registered backend liveness is wrong.

**Partially resolved (PR #467, 2026-07):** The diagnostics and liveness-projection
pieces shipped:
- `is_pgid_reachable(pgid)` in `state/liveness.py` — deliberately named *reachable*,
  not *alive*: no birth-time reuse guard, counts zombies; documented as diagnostic-only.
- `scope_liveness()` shared projection (`root_alive`, `pgid_reachable`,
  `likely_serving`) wired through all three reaper consumers: finalize evidence,
  `worker_or_backend_alive`, orphan diagnostics. Reaper evidence for a shim-rooted
  backend now shows `root_alive=false, pgid_reachable=true, likely_serving=true`
  instead of flat misleading "dead".
- Post-connect death enrichment: exit code + stderr excerpt in Codex connection-close
  events, routed through a single idempotent terminal owner
  (`_emit_connection_closed_once`).

**What remains:** The identity redesign. Model at least three identity roles:
launcher PID (the direct child of `create_subprocess_exec`), effective
serving/health PID (the process that owns the readiness endpoint), and
containment PGID. For Codex, discover and register the effective server child
after readiness (POSIX-first is acceptable) while retaining PGID for cleanup.

**Why the remainder is deferred:** The identity redesign is large — it touches
scope projection, reaper paths, and managed-backend lifecycle. The shipped
liveness projection makes diagnostics truthful without the redesign.
Investigation findings posted to issue #449 (comment 5054101045).

**Decision context:** [decisions/launch.md](../decisions/launch.md#d-process-scope-ownership-durable-process-scope-ownership-over-posix-only-or-psutil-only-cleanup)
and [architecture/process-scope.md](../architecture/process-scope.md)

---

### PROC-008: Thread Windows Job Handles Through Scope Termination

**Where:** `src/meridian/lib/platform/process_scope/base.py`,
`src/meridian/lib/platform/process_scope/windows_job.py`,
`src/meridian/lib/harness/connections/managed_backend.py`

**The issue:** Windows managed-backend launch creates a Job Object and keeps its
handle alive through `ParentDeathLink`, so parent-death cleanup works while the worker
process is alive. Durable `ProcessScopeSnapshot` records `containment="windows_job"`
and the `job_name`, but `ScopedProcessHandle.terminate()` and `terminate_scope_sync()`
cannot use a persisted OS handle. They currently fall back to PID-tree cleanup and mark
the cleanup result degraded.

**What's needed:** Thread the live Job Object handle into the async
`ScopedProcessHandle` termination path, and decide whether sync repair paths can do
anything stronger than PID-tree fallback after the original handle-owning process is
gone. The durable snapshot cannot itself resurrect an OS handle.

**Why deferred:** The parent-death linkage fixed the orphaned-backend failure mode
while the worker is alive. The remaining gap is termination quality for explicit
cleanup/repair paths on Windows after handle ownership is lost.

**Decision context:** [architecture/process-scope.md](../architecture/process-scope.md)

---
