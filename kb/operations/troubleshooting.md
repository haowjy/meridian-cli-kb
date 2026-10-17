# operations/troubleshooting — Common Failure Patterns and Recovery

This page covers what goes wrong, why, and how to recover. Each pattern includes the symptoms, the cause, and the commands to diagnose and fix.

## Spawn Failure Patterns

### Orphan Run (`orphan_run`)

**Symptom:** `meridian spawn show <id>` shows `status: failed`, `error: orphan_run`. The harness produced no `report.md`.

**What happened:** The runner process (PID recorded in the spawn row) died during normal execution — hard kill, OOM, host reboot. The heartbeat file went stale (no update within 120s window), the reaper confirmed the PID is dead, and finalized the spawn as failed.

**Likelihood of work product:** Low. The harness was killed mid-execution. Whatever the harness wrote before the kill is in its native transcript. The runner's in-memory attempt facts are lost, so the spawn is finalized from `report.md` and lifecycle facts only.

**Diagnosis:**
```bash
# Check what the spawn produced
meridian spawn show p42
meridian session log p42 --tail 30   # the chat's native transcript, labeled as the entry-based view

# Inspect raw runner stderr
cat ~/.meridian/projects/<uuid>/spawns/p42/stderr.log
```

`session log p42` never reads runner `history.jsonl`. If the chat was never bound to a native session, it reports `unbound`, and there is no transcript to recover ([native-only history](../decisions/native-only-history.md)).

**Recovery:** Relaunch the spawn, or continue its chat with `--continue cN` when the chat is bound. Pass the relevant part of the native transcript as context if needed.

---

### Primary Crashed With No Exit or Stop Recorded

**Symptom:** an interactive primary, its harness and its spawns all vanish at
once. The primary later shows `failed` / `orphan_run`. `runner-lifecycle.jsonl`
has no exit for it and `sessions.jsonl` has no stop for its generation. Before
#621 and #631, resuming it with `--continue` could draw
`ValueError: Cannot capture an active native owner: pN` over the new TUI, and
capture of that native session kept failing afterwards. Both are merged: launcher
output during a TUI goes to `launcher.log`, and capture defers to a live owner
instead of failing.

**What happened (2026-10-15 incident, chat `c11462`):** the host ran out of
memory. The kernel OOM-killed one process, a 1.5 GB agent-browser Chrome, inside
the tmux pane's systemd scope (`tmux-spawn-<uuid>.scope`). systemd's default
`OOMPolicy=stop` then stopped the **whole scope**: the Meridian launcher, the
Claude TUI and every spawn started from that pane got SIGTERM, and leftovers were
SIGKILLed about 90 s later. The scope had peaked at 34.3 GB of memory plus 9.2 GB
of swap. The OOM kill of one process takes down the pane.

The launcher recorded nothing because a Claude/Pi primary's launcher had no
SIGTERM/SIGHUP handler (#623). The default action killed it before its
`finally` blocks wrote the runner exit and the session stop
([gap](../architecture/primary-terminal-ownership.md#known-gaps)). PR #658 (merged
2026-10-17, `8055a1f77`) adds the receiver, so on builds with it a scope stop records
`cancelled`/130 and a session stop, provided finalize completes before
systemd's SIGKILL. Codex/OpenCode managed primaries already register a receiver
and finalize under the same condition.

**Diagnosis:**
```bash
journalctl --user -b | grep -iE 'oom|tmux-spawn'   # scope stop, OOM kill, leftover SIGKILLs
journalctl -k -b | grep -i 'out of memory'          # which process the kernel killed
meridian spawn show pN --verbose                    # orphan_run, no runner exit
```

**Recovery:** `meridian --continue cN` resumes the chat. Since #631
(`8ecd44989`), the new start closes the old generation on replay, and capture
of the crashed run is retried when the resumed run stops
([mechanism](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations)).
On builds before #631, a resume made before stale-lock cleanup ran left the old
generation open and native capture was refused on every later pass; running
`meridian doctor` before `--continue` avoided the race.

**Prevention:** keep heavy children out of the pane's memory budget. Close
agent-browser sessions when they're done
([leftover resources](../corrections.md)). On the owner's machine (systemd 255,
tmux 3.4, `systemd-oomd` inactive), pane scopes now get `OOMPolicy=continue`
from a user drop-in, `~/.config/systemd/user/tmux-spawn-.scope.d/oom.conf`
(`[Scope]` `OOMPolicy=continue`, `MemoryAccounting=yes`), installed 2026-10-16
(`work:resume-after-crash` R18, `evidence/oompolicy-research.md`). An OOM kill
then takes only the victim, and the rest of the pane keeps running. After
`systemctl --user daemon-reload`, every live pane scope reported
`OOMPolicy=continue` with the drop-in in `DropInPaths`, panes that existed before
the reload included (24 of 24, checked 2026-10-16). Check a pane with
`systemctl --user show <scope> -p OOMPolicy -p DropInPaths`.
The drop-in targets tmux scopes only, not a user-wide `DefaultOOMPolicy`. It is
machine configuration, not something Meridian sets. If `systemd-oomd` is active,
it kills whole cgroups regardless of `OOMPolicy`. Capping the agent-browser
Chrome's own memory is the stronger fix, filed as
haowjy/meridian-dev-workflow#8.

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

### Pre-Init Failed (`pre_init_failed`)

**Symptom:** `meridian spawn show <id>` shows `status: failed`,
`error: pre_init_failed`. Often no spawn directory is created, so there is no
log to read.

**What happened:** The spawn failed before the harness process started.

**Known causes and remedies:**

| Cause | Diagnostic clue | Fix |
|---|---|---|
| Claude harness blocked by `deny_headless_harnesses` | `[spawn] deny_headless_harnesses` in `meridian.toml` blocks headless Claude spawns. The spawn needs the native Agent tool instead. | Use a different harness, or spawn through the Agent tool from within a Claude session. |

**Diagnosis approach:** When `pre_init_failed` appears, read the line above
the error in the spawn output -- it often names the cause directly. Do not
assume a harness or provider problem before checking work-item state
(`meridian work show`) and config (`meridian config show`).

---

### Work Task Dir Missing (`work_task_dir_missing`)

**Symptom:** Spawn fails with `error: work_task_dir_missing`. The CLI
`meridian task-dir` emits a warning and falls back to the project root.

**What happened:** A work item's configured `task_dir` no longer exists on
disk (the common case after a worktree is removed post-merge), AND the
fallback project root also does not exist. Normally, stale task_dir degrades
gracefully with a warning and fallback to the next valid tier; this error
appears only when even the fallback fails.

**Fix:** Restore the directory, or reassign the work item's task_dir:
`meridian work start <slug> --task-dir <valid-path>`. An explicit `--task-dir`
on the spawn command will also override the stale value.

---

### Spawn State Quarantined After Upgrading Past PR 3

**Symptom:** `meridian spawn show pN` says "Spawn state quarantined: …/state.json; run
`meridian doctor` to migrate it". Index-backed commands (`session archive`,
`session search`, browse) may fail with `History index initialization failed: Spawn
state quarantined: … Automatic initialization will not retry for this index
generation`. `session archive --prune-runner-history` lists the IDs under `Skipped
quarantined`.

**What happened:** the row was written by the PR 1 dogfood build, whose flat
run-boundary fields the strict schema rejects. PR 3 replaced the read-time translator
with a one-time migration.

**Fix:** run `meridian doctor` once. It migrates the rows (`repaired:
dogfood_spawn_rows`) and re-arms the history index, so the next command initializes
without a manual rebuild. A row it cannot migrate is reported as
`dogfood_spawn_rows_failed` with the failing field. A quarantined row that is *not*
dogfood-shaped (no doctor hint, only its path) is not repairable by doctor and still
fails the index closed (#530).
See [health checks](health-checks.md#dogfood-row-migration).

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
# Inspect the human-facing harness/model inventory and alias definitions
meridian mars models list
meridian mars models aliases

# Check current config resolution
meridian config show
```

**Fix:** Treat `models list` as a human-facing, curated harness/model view—not
as an alias API or proof that a route will be selected. Inspect aliases with
`meridian mars models aliases`, and use `meridian mars models resolve <alias>`
or the launch bundle for the project-specific runtime resolution. Curation only
changes display; it does not repair an unresolved alias or alter routing.

---

### Workspace Config Invalid

**Symptom:** All launches fail with a workspace validation error.

**Cause:** A `[workspace.<name>]` entry in `meridian.toml` or `meridian.local.toml` has a TOML syntax error or schema violation, such as an invalid entry name or missing/empty `path`. This blocks all launches — the workspace gate in `bind_launch_context()` raises immediately. Legacy `workspace.local.toml` can still be read during migration when no `[workspace]` entries exist.

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
| Show curated harness/model inventory | `meridian mars models list` |
| List model aliases | `meridian mars models aliases` |
| Show raw provider catalog | `meridian mars models catalog` |
| Check workspace config | `meridian config show` (workspace section) |

## Cross-References

- [health-checks.md](health-checks.md) — doctor flow, pruning, cache
- [../concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — spawn statuses and reaper decisions
- [../architecture/managed-primary-lifecycle.md](../architecture/managed-primary-lifecycle.md) — managed-primary roles and passive/explicit cleanup boundary
- [../architecture/primary-terminal-ownership.md#known-gaps](../architecture/primary-terminal-ownership.md#known-gaps) — launcher signal-handling gap and terminal-output ownership
- [../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations) — abandoned-generation repair
- [../architecture/state-system/portable-history.md#capture-defers-to-a-live-owner-and-retries-when-it-stops](../architecture/state-system/portable-history.md#capture-defers-to-a-live-owner-and-retries-when-it-stops) — capture deferral and retry
- [../decisions/history-storage.md#d-abandoned-generation-closure](../decisions/history-storage.md#d-abandoned-generation-closure) — decision and rejected alternatives
- [configuration-guide.md](configuration-guide.md) — workspace and config setup
