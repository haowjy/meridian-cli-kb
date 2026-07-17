# lessons/state-design-lessons — Why the State System Is Built This Way

These are the choices that shaped Meridian's state layer, with the reasoning that drove each one. The "we tried X" context is what makes these valuable — without it, the decisions look arbitrary.

## Why Dual-Root (Repo + User)

**The problem we were solving:** Projects get moved, renamed, and copied. A state root tied to the project path breaks when the directory changes. But committing high-churn runtime state (spawn records, artifacts, session transcripts) to git creates commit noise, merge conflicts, and bloated repositories.

**The solution:** Split state into two roots by volatility:

- **Repo `.meridian/`** — committed scaffolding: KB, work items, agent profiles. Moves with the repo. Goes into version control.
- **User `~/.meridian/projects/<uuid>/`** — runtime state: per-spawn `state.json`, session events, artifact directories. Keyed by UUID, not path. Never committed.

**The UUID binding:** `~/.meridian/projects/<uuid>/` uses a project UUID (stored in `.meridian/id`). When you rename or move the repo, the UUID stays the same, and the user-level state is still there. This was the key insight: project identity is the UUID, not the path.

**The migration moment:** This split was introduced in `v001 uuid_state_split` (0.0.34). Before that, all state lived under repo `.meridian/`. The legacy path still worked for reading; new writes went to the user root.

**What we'd do differently:** The split is right, but the migration tooling (currently a stub) should have been implemented before the split was released. Running the old structure and the new structure in parallel required read-fallback code that complicated path resolution.

---

## Why JSONL Over SQLite

**The temptation:** SQLite would give us transactions, queries, indexes, and a schema. Running `SELECT * FROM spawns WHERE status='running'` is cleaner than replaying a JSONL stream.

**Why we didn't:**

1. **Crash tolerance.** JSONL appends are atomic at the line level — a crash mid-write leaves an incomplete line that gets skipped on read. SQLite transactions require a write-ahead log that can get corrupted under hard kills. Crash-only design favors the dumbest possible write that can be safely recovered.

2. **Inspectability.** JSONL and per-spawn JSON files work with standard tools (`cat`, `jq`) without any Meridian tooling. This matters for debugging in CI, on bare machines, in restricted environments. SQLite requires `sqlite3` and schema knowledge.

3. **No service dependency.** SQLite files are exclusive-locked when open on some platforms. Multiple concurrent Meridian processes (nested spawns, parallel CLI invocations) need to read state concurrently. JSONL with `flock` sidecar scales to N readers + 1 writer without a connection manager.

4. **Simpler crash recovery.** A truncated JSONL file is recoverable by skipping the last partial line. A corrupt SQLite database requires `PRAGMA integrity_check` and may need manual reconstruction.

**The cost:** Querying requires reading and replaying the entire file. For spawn lists this is fast (hundreds of events, not millions). If spawns scale to tens of thousands, we'd need an index. That's a future problem.

---

## Why Mutable JSON for Work Items

**The asymmetry:** Everything else is append-only JSONL. Work items are mutable per-directory JSON (`<context.work>/<slug>/__status.json`). Why the inconsistency?

**The constraint:** Work items have slugs, and the slug IS the directory name. When a work item is renamed, the directory must also rename. Directory location is the primary authority for active-vs-archived state.

With JSONL, rename would mean appending a `rename` event and somehow rebuilding the path — but the directory and the slug must stay in sync, and you can't atomically rename both via event append.

**The solution:** Mutable JSON (`os.replace()` for atomic overwrites) with a rename intent file. Before any rename:
1. Write `work-items.rename.intent.json` (the intent)
2. Rename the directory
3. Rename the JSON file
4. Delete the intent

If the process dies between steps 2 and 3, the intent file survives and the rename is replayed on next startup/reconciliation. This is crash-safe, inspectable, and keeps slug + directory in sync.

---

## The Projection Authority Rule

**The problem:** The reaper runs on every read path. It may stamp an orphaned spawn as `failed` with `orphan_run` before the actual runner process has a chance to report success. If the runner then reports `succeeded`, which one wins?

**The naive answer is wrong:** "Last write wins" means the reaper's `orphan_run` can never be corrected. "First write wins" means a legitimate runner success can be overridden by a reaper that jumped the gun.

**The actual answer:** Origins have authority levels. `runner` and `launcher` are authoritative (they were executing the harness). `reconciler` (the reaper) is non-authoritative (it's inferring from liveness signals).

- Authoritative vs. authoritative: first writer wins (no revision)
- Authoritative overrides reconciler: always — runner ground truth beats reaper inference
- Reconciler vs. reconciler: first writer wins (concurrent reap calls don't revise each other)

**How it manifests:** `finalize_spawn()` accepts an `origin` parameter. Projection replays all finalize events and applies the authority rules to determine the canonical terminal state.

**The lesson:** When multiple processes can write terminal state for the same record, you need an explicit authority ordering — not just a timestamp or a mutex. Timestamps require clock synchronization. Mutexes block. Authority rules are append-safe and correct without coordination.

---

## Reaper Lessons

### Heartbeat-Based Liveness Over PID-Only

**Early design:** The reaper checked if the runner PID was alive. If dead, the spawn was orphaned.

**The problem:** PID reuse. A process finishes, its PID is recycled to a completely different process, and the reaper sees an "alive" PID and leaves the spawn in `running` forever.

**The fix:** Two guards:
1. Create-time comparison: if the process was created more than 30 seconds after the spawn's `started_at`, it's a PID reuse
2. Heartbeat recency: if any artifact in the spawn directory was touched within 120 seconds, the runner is considered active regardless of PID state

**The lesson:** PID liveness is a necessary but not sufficient condition. The heartbeat artifact provides a ground-truth liveness signal that doesn't depend on the OS PID table.

### Why Artifact-Agnostic Recency

**Original design:** The reaper checked only the `heartbeat` file for recency.

**The problem:** A runner that crashes while writing `stderr.log` or `history.jsonl` — just before writing the heartbeat — would be reaped immediately on the 120-second window, even though it was active a moment ago.

**The fix:** The recency check applies to any of: `{heartbeat, history.jsonl, output.jsonl, stderr.log, report.md}`. A write to any of these resets the recency window.

**The lesson:** Use the broadest safe set of liveness signals. The heartbeat is the primary signal (written every 30s unconditionally), but last-gasp activity on other artifacts should prevent premature reaping.

### Why PID Probes Are Skipped for `finalizing`

**The `finalizing` status** means the harness process has already exited — the runner is in controlled post-exit drain (report extraction, finalization). The runner PID is still alive (it's the runner, not the harness), but the worker (harness) is done.

**The problem:** PID-probing the runner during `finalizing` gives "alive" (runner hasn't exited yet), which would suppress reaping forever for a runner that crashed mid-drain.

**The fix:** For `finalizing` rows, the reaper uses only heartbeat recency, not PID liveness. A stale heartbeat during `finalizing` means the runner crashed mid-drain → `orphan_finalization`.

## Cross-References

- [architecture/state-system.md](../architecture/state-system.md) — full state system architecture
- [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — reaper decision logic
- [principles/design-principles.md](../principles/design-principles.md) — crash-only design, files as authority
- [principles/invariants.md](../principles/invariants.md) — projection authority rule, JSONL invariant
