# Troubleshooting

Recovery for failures that span subsystems. User-facing recipes are in meridian-cli `docs/troubleshooting.md` and `docs/debugging.md`; upgrade-specific failures (the Codex shell-rule hook refusing a launch, the idle limit and `spawn rearm`) are in `docs/upgrading.md`. Health checks: [doctor and pruning](health-checks.md).

## A spawn shows `crashed`

The runner died without writing an outcome: a hard kill, OOM, host reboot, or a crash while extracting the report. The kernel freed its `launch.lock`, so every reader derives `crashed`, or `succeeded` when a durable report exists, or `cancelled` when a cancel was requested ([derived status](../architecture/state-system/derived-status.md)). Until something persists it, `state.json` still says `running`; plain `meridian doctor` lists it under `dead_spawns_unfinalized`. The next `--continue`, `spawn cancel`, `meridian doctor --kill-orphans`, or the background repair of a later primary launch writes the outcome and sweeps leftover process scopes.

What survives depends on when the runner died: killed mid-turn leaves only the harness's native transcript; killed while finalizing may leave a complete or partial `report.md`. The in-memory attempt facts are lost either way. Read the transcript with `meridian session log pN --tail 30`; an unbound chat has no transcript to recover ([native transcript reads](../architecture/native-transcript-reads.md)). Salvage `report.md`, then relaunch or `--continue`.

## A primary, its harness and its spawns vanish together

Usually the host ran out of memory inside the tmux pane's systemd scope. With systemd's default `OOMPolicy=stop`, the kernel's OOM kill of one process (often a large browser started by an agent) makes systemd stop the whole scope: the launcher, the TUI and every spawn from that pane get SIGTERM, and leftovers SIGKILL about 90 s later. Primary launchers handle SIGTERM and SIGHUP and record `cancelled` (143 or 129) with a session stop, if finalization finishes before the SIGKILL.

Check with `journalctl --user -b | grep -iE 'oom|tmux-spawn'` and `journalctl -k -b | grep -i 'out of memory'`. `meridian --continue cN` resumes the chat; the new start closes the old generation, and capture of the crashed run is retried when the resumed run stops ([session state](../architecture/state-system/session-state.md#a-later-start-closes-abandoned-generations)).

To contain it, give pane scopes `OOMPolicy=continue` (machine configuration, not something Meridian sets), close browser sessions when done, and cap the browser's own memory. If `systemd-oomd` is active it kills whole cgroups regardless of `OOMPolicy`.

## A managed primary's launcher died

A Codex or OpenCode `kind: primary` spawn shows `crashed`, and the TUI may show app-server connection errors. The launcher was SIGKILLed, OOM-killed or crashed (SIGTERM and SIGHUP make it finalize `cancelled` itself). Guardians normally stop the backend group and send the TUI one SIGTERM, so nothing is left running; launcher death before the TUI exits is the defect to diagnose. Read `spawns/pN/primary_meta.json` and the latest `launches/<n>/stderr.log`. `meridian doctor --kill-orphans` persists the outcome and terminates recorded scopes, or the PIDs in `primary_meta.json` when none were recorded, checking each PID's birth time ([managed primary lifecycle](../architecture/managed-primary-lifecycle.md#repair-and-cleanup)).

## A child was cancelled though nobody cancelled it

Its parent closed: `spawn done` or a reply to the idle-limit ask cancels running descendants and stops local background tasks, and a dead parent backend cancels them too ([completion hold](../architecture/completion-drain.md#a-close-takes-what-it-started)). `meridian spawn show <parent>` tells these apart: a dead parent shows `failed` with `backend_dead_while_awaiting_done`. A job that must outlive its spawn runs detached; a parent that still needs a child runs `spawn rearm --reason "..."` when asked.

## `pre_init_failed`

The spawn failed before the harness started, often before a spawn directory existed. Read the line above the error first: it usually names the cause. A common one is `[spawn] deny_headless_harnesses` in `meridian.toml` blocking a headless Claude spawn. Check work-item state (`meridian work show`) and config (`meridian config show`) before suspecting the harness or provider.

## `work_task_dir_missing`

A work item's `task_dir` no longer exists (typically a worktree removed after merge) and the project-root fallback is missing too; normally a stale `task_dir` only warns and falls back. Restore the directory, reassign it with `meridian work start <slug> --task-dir <path>`, or pass `--task-dir` on the spawn. A running agent's `task` link falls back to the project root silently, since a link can't carry a warning; if an agent edited the project root instead of its worktree, `meridian work task-dir` still prints the recorded path ([work items](../architecture/state-system/work-items.md#checkout-pins)).

## "Spawn state quarantined"

A row from the early native-history build carries fields the strict schema rejects, and index-backed commands (`session archive`, `session search`, browse) may refuse until it is migrated. Run `meridian doctor` once: it migrates the rows and re-arms the history index. A row it can't migrate is reported as `dogfood_spawn_rows_failed`. A quarantined row of any other shape still fails the index closed ([#530](https://github.com/haowjy/meridian-cli/issues/530)).
