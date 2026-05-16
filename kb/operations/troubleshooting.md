# operations/troubleshooting — Common Failure Patterns and Recovery

This page covers what goes wrong, why, and how to recover. Each pattern includes the symptoms, the cause, and the commands to diagnose and fix.

## Spawn Failure Patterns

### Orphan Run (`orphan_run`)

**Symptom:** `meridian spawn show <id>` shows `status: failed`, `error: orphan_run`. The harness produced no `report.md`.

**What happened:** The runner process (PID recorded in the spawn row) died during normal execution — hard kill, OOM, host reboot. The heartbeat file went stale (no update within 120s window), the reaper confirmed the PID is dead, and finalized the spawn as failed.

**Likelihood of work product:** Low. The harness was killed mid-execution. Any partial output is in `history.jsonl` but may be truncated.

**Diagnosis:**
```bash
# Check what the spawn produced
meridian spawn show p42
meridian session log p42 --last 30

# Inspect raw output
cat ~/.meridian/projects/<uuid>/spawns/p42/stderr.log
cat ~/.meridian/projects/<uuid>/spawns/p42/history.jsonl | tail -20
```

**Recovery:** Relaunch the spawn. The previous spawn's `history.jsonl` preserves what was emitted before the kill; pass it as context if needed.

---

### Orphan Finalization (`orphan_finalization`)

**Symptom:** `status: failed`, `error: orphan_finalization`. Spawn was in `finalizing` state when the reaper acted.

**What happened:** The runner explicitly entered controlled drain (`finalizing` status) — meaning harness process exited, runner was in the report-extraction/finalization phase — then crashed or was killed before completing finalization. The heartbeat went stale while in `finalizing`.

**Likelihood of work product:** Higher than `orphan_run`. The harness finished executing; `report.md` may be partially written. Check before relaunching.

**Diagnosis:**
```bash
meridian spawn show p42
cat ~/.meridian/projects/<uuid>/spawns/p42/report.md    # may be partial but useful
meridian session log p42 --last 50
```

**Recovery:** Read the partial `report.md` for salvageable output. Relaunch with the prior session context if the work is incomplete.

---

### Missing Runner PID (`missing_runner_pid`)

**Symptom:** `status: failed`, `error: missing_runner_pid`. No active spawns visible.

**What happened:** The spawn started but the PID was never committed to the event stream. Spawn started but either the harness failed to launch or the PID-recording step failed. The spawn was in `running` state with no PID, past the 15-second startup grace window, with no heartbeat activity.

**Diagnosis:**
```bash
meridian spawn show p42
cat ~/.meridian/projects/<uuid>/spawns/p42/stderr.log  # check for launch errors
```

**Recovery:** Check `stderr.log` for the launch error (bad model name, harness binary not found, permission issue). Fix the root cause and relaunch.

---

### Managed Primary Orphan (`orphan_primary`)

**Symptom:** `meridian spawn show <id>` shows `status: failed`,
`error: orphan_primary` for a `kind: primary` Codex or OpenCode spawn. The TUI
may also show remote app-server connection errors if the backend later exits or
is explicitly cleaned up.

**What happened:** The managed-primary launcher wrapper died or became
unreliable. For Codex/OpenCode managed primaries, the launcher, backend, and TUI
are separate roles. Launcher death is abnormal and should be diagnosed as a
wrapper failure, but it does not prove the backend/TUI were already dead.

**Reaper behavior:** Passive read paths may record `failed/orphan_primary`. They
do not terminate backend/TUI processes on Skip decisions — that protects an
interactive TUI that may still be usable. Once the reaper finalizes the row as
failed and metadata safely identifies runtime children, it may terminate those
tracked children as a cleanup safety net. Missing or corrupt `primary_meta.json`
for a Codex/OpenCode primary is treated conservatively as a managed-primary
candidate, not as a generic worker to kill.

**Diagnosis:**
```bash
meridian spawn show p42
cat ~/.meridian/projects/<uuid>/spawns/p42/primary_meta.json
cat ~/.meridian/projects/<uuid>/spawns/p42/stderr.log
meridian session log p42 --last 50
```

Check whether the launcher died from an exception, signal, OOM, or host crash.
Launcher death before TUI exit is the primary defect; passive reconciliation is
only recording that state.

**Recovery / cleanup:** If the TUI/backend are stranded and should be stopped,
run explicit cancel:

```bash
meridian spawn cancel p42
```

`spawn cancel` performs best-effort managed-primary cleanup even when the spawn
is already terminal with `error: orphan_primary`.

If multiple orphaned managed primaries or browser/backend child processes have
accumulated, use:

```bash
meridian doctor --kill-orphans
```

---

### Stale Session Locks

**Symptom:** `meridian doctor` reports `stale_session_locks` in `repaired`. Or a primary session fails to start because the lock is held.

**What happened:** A session lock file (`~/.meridian/projects/<uuid>/sessions/<chat_id>.lock`) was not cleaned up when the session ended — typically from a hard kill or crash.

**Resolution:** `meridian doctor` repairs these automatically. Run `meridian doctor` if sessions are blocked.

```bash
meridian doctor  # detects and repairs stale locks
```

---

## Configuration and Launch Failures

### Mars Binary Not Found

**Symptom:** `RuntimeError: mars binary not found` or similar on `meridian mars sync` or package operations.

**Cause:** The `mars` binary (from `mars-agents`) is not on `PATH` or not installed.

**Fix:**
```bash
# Check if mars is available
which mars

# Install/reinstall via the project's documented method
# (see repository documentation for installation instructions)
```

---

### Model Resolution Failure

**Symptom:** Launch fails with an error about an unknown model alias or no matching harness.

**Cause:** The model name in `-m`, agent profile, or config doesn't match any entry in the model catalog.

**Diagnosis:**
```bash
# List available models and their harness mappings
meridian mars models list

# Check current config resolution
meridian config show
```

**Fix:** Use a model name from `meridian mars models list`. Aliases like `"sonnet"` or `"gpt-5"` are resolved through the catalog; bare version strings or typos will fail.

---

### Workspace Config Invalid

**Symptom:** All launches fail with a workspace validation error.

**Cause:** A `[workspace.<name>]` entry in `meridian.toml` or `meridian.local.toml` has a TOML syntax error or schema violation, such as an invalid entry name or missing/empty `path`. This blocks all launches — the workspace gate in `build_launch_context()` raises immediately. Legacy `workspace.local.toml` can still be read during migration when no `[workspace]` entries exist.

**Diagnosis:**
```bash
meridian config show   # shows workspace status and findings
meridian doctor        # surfaces workspace_invalid / workspace_* diagnostics
```

Inspect the relevant config file named in the diagnostic: `meridian.toml`, `meridian.local.toml`, or legacy `workspace.local.toml`.

**Fix:** Correct the TOML syntax or schema. A `"present"` workspace with missing roots is non-blocking: committed missing roots are silently skipped, and local missing roots emit `workspace_local_missing_root`. Only `"invalid"` blocks launch.

Common causes:
- Unquoted paths with special characters
- `[workspace]` scalar values instead of `[workspace.<name>]` tables
- Missing or empty `path` in a workspace entry
- Invalid entry names; use lowercase letters, numbers, hyphens, and underscores, starting with a letter

---

### Rebase Conflicts in Git-Autosync

**Symptom:** Git-autosync hook emits a conflict error and aborts; no data loss.

**Cause:** The autosync hook attempted to rebase local KB/work commits onto the remote and encountered a conflict.

**Behavior:** Autosync is designed to detect and abort on conflict — it does not attempt to resolve. Local commits are preserved. No data is lost.

**Fix:**
```bash
# Check git status to understand the conflict
git status
git log --oneline -10

# Resolve manually: rebase, merge, or force-push as appropriate
# Then re-run sync
```

---

## Diagnostic Command Reference

| Situation | Command |
|---|---|
| Check spawn status and error | `meridian spawn show <id>` |
| Read spawn output | `meridian session log <id> --last N` |
| Full spawn transcript | `meridian session log <id>` |
| Health check | `meridian doctor` |
| Health check + clean | `meridian doctor --prune --global` |
| List active spawns | `meridian spawn list` |
| Check config resolution | `meridian config show` |
| List available models | `meridian mars models list` |
| Check workspace config | `meridian config show` (workspace section) |

## Cross-References

- [health-checks.md](health-checks.md) — doctor flow, pruning, cache
- [../concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — spawn statuses and reaper decisions
- [../architecture/managed-primary-lifecycle.md](../architecture/managed-primary-lifecycle.md) — managed-primary roles and passive/explicit cleanup boundary
- [configuration-guide.md](configuration-guide.md) — workspace and config setup
