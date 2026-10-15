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

**Decision:** Meridian appends its hint after a harness exits and never moves
the cursor up or erases the harness's exit text. While a TUI owns the terminal,
Meridian writes nothing to it. The relay-level fix is documented in the
[current mechanism](../architecture/primary-terminal-ownership.md).

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

**Decision:** Codex and OpenCode run with native inherited stdio; Claude and Pi
stay PTY mediated. The adapter owns the default terminal mode, with no user
override or snapshot replay, and `TerminalSurfaceMode` is the single mode type.
The [current mechanism](../architecture/primary-terminal-ownership.md) records
the per-harness selection and process-launch details.

**Rejected:**
- **A PTY override for Codex/OpenCode.** The investigator proposed keeping one
  as an escape hatch. Rejected for simplicity; revisit only if native fails
  somewhere real.
- **Moving Claude/Pi native too.** Their relay reads nothing (no capture
  without `--print`, session id fixed at launch, idle from transcript or
  extension), but they resize fine. The owner kept the relay by choice, not
  because native mode is impossible; the mechanism page records what the relay
  still provides.
- **Keeping the relay with fixed resize forwarding (option A, #596)** for these
  two harnesses. It fixes the symptom and keeps the byte path.

**Cost accepted:** a `kill -9`ed native TUI can leave stale screen bytes; the
accepted consequence is described in the [current mechanism](../architecture/primary-terminal-ownership.md).

---

## D-shared-foreground-job — The shell keeps job control

**Status:** decided 2026-10-15 (work-item D2, confirmed by the long-stop
investigation). On the native-terminal branch.

**Decision:** the shell remains the job-control owner: the launcher and native
TUI share its foreground process group, and Meridian does not create a competing
job-control implementation. The [current mechanism](../architecture/primary-terminal-ownership.md)
records the signal windows and cancellation behavior.

**Why the SIGINT receiver goes through `SignalCoordinator`:** the first version
set `signal.signal(SIGINT, SIG_IGN)` directly. That bypassed the coordinator,
which owns handler install and restore, so the restore could be lost. The
receiver was first registered for all of `run()`, which swallowed Ctrl+C during
backend startup. It now covers only the TUI's lifetime; the resulting ordering
is in the [current mechanism](../architecture/primary-terminal-ownership.md).

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
tree; PDEATHSIG is removed. Background Codex/OpenCode spawns use the same
ownership model for their backend. Registration, release, and failure behavior
are the [current mechanism](../architecture/process-scope.md#parent-death-one-watchdog-per-launcher).

**Cost accepted:** the watchdog's initial package imports cost about 34 MB RSS;
the isolated implementation reduced that to about 17 MB. Its containment and
environment details belong to the [process-scope mechanism](../architecture/process-scope.md#parent-death-one-watchdog-per-launcher).

**Rejected:**
- **PDEATHSIG on the TUI**, and per-child watchdogs or PDEATHSIG elsewhere.
  Neither contains wrapper descendants, and per-child watchdogs multiply the
  RSS cost.
- **Unknown birth time treated as "probably fine".** The watchdog kills by PID,
  and PID reuse makes that unsafe.
- **A pre-exec parent-death hook** to close the spawn-to-registration window.
  It would run Python after fork in a threaded launcher and would still cover
  only one PID. See the [process-scope mechanism](../architecture/process-scope.md#parent-death-one-watchdog-per-launcher).

**Follow-ups:** #609 (fold `platform/terminate.py` and
`fallback.terminate_tree` onto `scope_termination`, leaving one termination
path), #610 (macOS instant-exit TUI reported as a link failure).

---

## D-cancel-by-order — Cancellation is decided by which came first

**Status:** decided 2026-10-15 (work-item D9). On the native-terminal branch.

**Decision:** a native primary run is cancelled (exit 130) if the launcher
received SIGHUP/SIGTERM before it observed the TUI's exit; if the TUI's exit
was observed first, the real result stands. The recorder and teardown ordering
are in the [current mechanism](../architecture/primary-terminal-ownership.md).

**Why not exit codes:** a closed pane sends SIGHUP to the TUI as well as the
launcher. OpenCode exits 0 on SIGHUP, so "the TUI died by signal" recorded a
closed pane as `succeeded 0`, a regression from `main`'s cancelled 130.
Harnesses don't agree on SIGHUP exit codes. The order in which the launcher saw
the two events is the only fact that means the same thing everywhere.

**Constraint:** the ordering recorder is touched from a signal handler, which
runs on the main thread. A plain `Lock` could deadlock the launcher against
itself, so it uses an `RLock`. Any state a `SignalCoordinator` receiver touches
must be lock-free or reentrant.
