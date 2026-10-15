# Decisions: Primary Terminal Ownership

How much of Meridian sits between an interactive primary's TUI and the user's
terminal, and what follows from that. Current mechanism:
[architecture/primary-terminal-ownership.md](../architecture/primary-terminal-ownership.md).
Watchdog mechanism: [architecture/process-scope.md](../architecture/process-scope.md#parent-death-one-watchdog-per-launcher).

Provenance for the whole page: `work:native-terminal-primaries` (`decisions.md`,
evidence), `work:relay-exit-text` (archived), `chat:c7524`.

---

## D-exit-text-append — Meridian never covers a harness's exit text

**Status:** decided by the user (2026-10-15); merged in PR #607 (`aac732f74`),
released in v0.10.9. Applies to every harness.

**User rule:** "why tf are we clearing the leftover text... we should just
append after it", "i don't want to cover the harness exit", "(for all
harneses)".

**Problem:** Meridian's quit hint overwrote Claude's resume line. The cause was
the PTY relay sending `ESC[?1049l` again after the harness had already left the
alternate screen. tmux read the second one as "restore the saved cursor".
Commit `95d429f10` (in open PR #590) answered with `ESC[J`, which erased the
leftover characters but also erased the harness's real exit text.

**Decision:** Fix the cause. The relay tracks the child's alternate-screen
state and sends `ESC[?1049l` only when the child exited while still inside it.
Restore codes that don't move the cursor stay unconditional. After a harness
exits, Meridian appends plain text only, never moving the cursor up and never
erasing. While a TUI owns the terminal, Meridian writes nothing to it.

**Rejected:** erasing the leftovers with `ESC[J`; scrubbing or rewriting
harness exit text, including Codex's `resume <uuid>` reconnect line, in any
mode.

**Revisit if:** a harness leaves the terminal so broken that its own exit text
is unreadable. Even then, append a reset; don't erase.

---

## D-native-terminal — Codex/OpenCode inherit the real terminal; Claude/Pi keep the relay

**Status:** decided by the user (2026-10-15: "option B is better"; Claude/Pi:
"i didn't really have any problems resizing claude and pi so its fine").
Implemented on branch `feat/native-terminal-primaries`, no PR yet; `main` still
relays all four. Work-item IDs D1, D5, D8.

**Problem:** the PTY relay's nested PTY mangled resizes (#596). Option A
patched resize forwarding inside the relay. Option B removes the relay where
nothing needs it. For Codex and OpenCode, nothing does: Meridian observes them
through its own backend connection, and the managed attach never captured TUI
output.

**Decision:**
- Codex and OpenCode run native inherit: the TUI gets the launcher's stdio.
  Claude and Pi stay PTY mediated.
- The adapter declares the default mode. There is no user override: no real
  users, one path. TTY detection no longer picks PTY for managed launches.
- The mode is not persisted in launch-policy snapshots (D8). It is adapter
  policy, not a user choice, so `--continue` uses the current default.
- One mode type: `TerminalSurfaceMode` drives the process selector, and the
  near-duplicate `ProcessSurfaceMode` was folded into it (D5).

**Rejected:**
- **A PTY override for Codex/OpenCode.** The investigator proposed keeping one
  as an escape hatch. Rejected for simplicity; revisit only if native fails
  somewhere real.
- **Moving Claude/Pi native too.** The tech lead proposed it after checking that
  their relay reads nothing (no capture without `--print`; session id from
  launch; idle from transcript or extension). The owner kept them on the relay
  because they resize fine. It is a choice, not a constraint, and Pi already
  declares native as supported.
- **Keeping the relay with fixed resize forwarding (option A, #596)** for these
  two harnesses. It fixes the symptom and keeps the byte path.

**Cost accepted:** with no relay, a `kill -9`ed native TUI leaves stale screen
bytes. Nobody runs a restore.

---

## D-shared-foreground-job — The shell keeps job control

**Status:** decided 2026-10-15 (work-item D2, confirmed by the long-stop
investigation). On the native-terminal branch.

**Decision:** the launcher and the native TUI share the shell's foreground
process group. Meridian calls neither `setpgid` nor `tcsetpgrp` and installs no
SIGTSTP, SIGTTIN, SIGTTOU or SIGCONT handler. Ctrl+Z is the TUI's own
`kill(0, SIGTSTP)`, which stops launcher and TUI together, and `fg` resumes both.
The launcher ignores SIGINT only while the TUI runs, through a no-op receiver
on `SignalCoordinator`. SIGHUP/SIGTERM handling is unchanged.

**Why the SIGINT receiver goes through `SignalCoordinator`:** the first version
set `signal.signal(SIGINT, SIG_IGN)` directly. That bypassed the coordinator,
which owns handler install and restore, so the restore could be lost. The
receiver was first registered for all of `run()`, which swallowed Ctrl+C during
backend startup. It now covers only the TUI's lifetime.

**Rejected:**
- **A separate process group with `tcsetpgrp`.** That means writing
  launcher-side SIGTTOU/SIGTTIN handling, a job-control implementation of
  Meridian's own, to replace what the shell already does correctly.
- **Launcher SIGTSTP handlers or watchdog terminal operations** to fix #611.
  No trace shows Meridian involved, and they would make the native surface less
  shell-compatible. Act only on a captured trace.

---

## D-launcher-watchdog — One watchdog per launcher instead of PDEATHSIG

**Status:** decided 2026-10-15 (work-item D3, amended after review). On the
native-terminal branch.

**Problem:** killing a native launcher with `kill -9` left the TUI alive and
"Reconnecting…". The backend had a parent-death link and the TUI had none. The
first fix gave the TUI `PR_SET_PDEATHSIG`. Review found that PDEATHSIG reaches
only the **direct child**. Codex's command is a Node wrapper that starts the
real binary, so the inner `codex` TUI and inner `codex app-server` both
survived. `main` already leaks the inner app-server this way.

**Decision:** on POSIX, each launcher starts one PID/birth-guarded watchdog
before any child. It owns the backend's process group and the TUI's process
tree, which can't be a process group because it shares the shell's. PDEATHSIG
is removed everywhere. Background Codex/OpenCode spawns get the same watchdog
for their backend.

- **Registration fails closed.** A scope with unknown birth time is refused.
  If registration fails, the child is terminated and the launch aborts, so no
  child exists without an owner.
- **Release only after a confirmed stop.** An owner releases its scope by
  `release_id` (required) only after the child is stopped and reaped. Codex
  releases only after its backend stop is confirmed. A graceful launcher close
  sends `shutdown`; EOF on the pipe without one means the launcher died.
- **Register after spawn, not pre-exec.** There is a millisecond-scale window
  between `Popen` and registration. A pre-exec hook would close it but would run
  Python after fork in a threaded launcher, and would still cover only one PID.

**Cost accepted:** the first watchdog imported Meridian's package initializers
and used about 34 MB RSS per launcher. It was accepted for correctness, then cut
to about 17 MB: the watchdog runs by file path with `python -S -OO`, imports
only the stdlib, psutil and the dependency-free `scope_termination.py`, and gets
a sanitized environment. Every inherited `PYTHON*` variable is stripped and
`PYTHONPATH` is set explicitly. That last part was a blocker: with
`PYTHONSAFEPATH` set, the script's directory dropped off `sys.path` and every
Codex/OpenCode launch failed.

**Rejected:**
- **PDEATHSIG on the TUI**, and per-child watchdogs or PDEATHSIG elsewhere.
  Neither contains wrapper descendants, and per-child watchdogs multiply the
  RSS cost.
- **Unknown birth time treated as "probably fine".** The watchdog kills by PID,
  and PID reuse makes that unsafe.

**Follow-ups:** #609 (fold `platform/terminate.py` and
`fallback.terminate_tree` onto `scope_termination`, leaving one termination
path), #610 (macOS instant-exit TUI reported as a link failure).

---

## D-cancel-by-order — Cancellation is decided by which came first

**Status:** decided 2026-10-15 (work-item D9). On the native-terminal branch.

**Decision:** a native primary run is cancelled (exit 130) if the launcher
received SIGHUP/SIGTERM before it observed the TUI's exit, whatever the exit
code. If the TUI's exit was observed first, the real result stands.

**Why not exit codes:** a closed pane sends SIGHUP to the TUI as well as the
launcher. OpenCode exits 0 on SIGHUP, so "the TUI died by signal" recorded a
closed pane as `succeeded 0`, a regression from `main`'s cancelled 130.
Harnesses don't agree on SIGHUP exit codes. The order in which the launcher saw
the two events is the only fact that means the same thing everywhere.

**Constraint:** the ordering recorder is touched from a signal handler, which
runs on the main thread. A plain `Lock` could deadlock the launcher against
itself, so it uses an `RLock`. Any state a `SignalCoordinator` receiver touches
must be lock-free or reentrant.
