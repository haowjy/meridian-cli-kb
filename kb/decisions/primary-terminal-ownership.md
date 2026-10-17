# Decisions: Primary Terminal Ownership

How much of Meridian sits between an interactive primary's TUI and the user's
terminal, and what follows from that. Current mechanism:
[architecture/primary-terminal-ownership.md](../architecture/primary-terminal-ownership.md).
Parent-death linkage on `main` and its decided replacement:
[architecture/process-scope.md](../architecture/process-scope.md#parent-death-linkage).

Provenance for the whole page: `work:native-terminal-primaries` (`decisions.md`,
evidence), `work:relay-exit-text` (archived), `work:resume-after-crash`
(`decisions.md` R1, R5, R7, R9), `chat:c7524`.

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
Commit `95d429f10` (in PR #590, since closed unmerged) answered with `ESC[J`,
which erased the leftover characters but also erased the harness's real exit
text.

**Decision:** Meridian appends its hint after a harness exits and never moves
the cursor up or erases the harness's exit text. While a TUI owns the terminal,
Meridian writes nothing to it; [D-tui-quiet-launcher](#d-tui-quiet-launcher--launcher-output-never-reaches-a-live-tui)
extends that to every launcher thread and child. The relay-level fix is
documented in the [current mechanism](../architecture/primary-terminal-ownership.md).

**Rejected:** erasing the leftovers with `ESC[J`; scrubbing or rewriting
harness exit text, including Codex's `resume <uuid>` reconnect line, in any
mode.

**Revisit if:** a harness leaves the terminal so broken that its own exit text
is unreadable. Even then, append a reset; don't erase.

---

## D-tui-quiet-launcher — Launcher output never reaches a live TUI

**Status:** decided by the user (2026-10-15). Shipped: PR #621
(`fix/launcher-quiet-under-tui`) merged 2026-10-16 as `23eb2d6d2`, released in
v0.10.11. Work-item IDs R1, R5, R7, R9, R17.

**User rule:** "it still garbles the screen when claude hasn't redrawn --- we
shouldn't garble the text at all". The text vanishing on the TUI's next redraw
is not a fix. This applies to every harness.

**Problem:** D-exit-text-append said Meridian writes nothing while a TUI owns
the terminal, but only Meridian's own printing code obeyed it. A resumed Claude
primary's startup-repair thread logged a warning and traceback to `sys.stderr`,
which was the pane, and they were drawn over the live TUI. Any launcher thread,
any C-level write and any child that inherits fd 2 can do the same, for every
harness.

**Decision:** while a primary TUI owns the terminal, the launcher's fd 0 points
at `/dev/null` and its fds 1 and 2 at a dedicated `spawns/<id>/launcher.log`.
The TUI, relays and launchers use saved duplicates of the terminal fds, and a
relay owns its duplicates until its `wait()` returns. After exit, one pointer
line is printed below the harness's exit text, only if the log grew during this
window. The window prints it itself when an error leaves the window. Mechanism:
[launcher output during the TUI](../architecture/primary-terminal-ownership.md#launcher-output-during-the-tui-goes-to-launcherlog).

**Rejected:**
- **Swapping Python logging handlers or `sys.stderr`.** It misses inheriting
  children, C-level writes and any writer that doesn't go through Python's
  streams. Only an fd-level redirect covers every writer.
- **Reusing `stderr.log`.** The Codex and OpenCode backends and Claude's
  primary stderr log write to it during normal sessions, so the pointer would
  show after ordinary Codex runs.
- **A SIGTERM/SIGHUP barrier and signal masking inside the window.** If the
  launcher dies by a default signal, where its fds point doesn't matter. A
  barrier would also be a hidden change to signal policy, which belongs to the
  runner (on #614, to `SignalCoordinator`). A `ctypes` `fflush(None)` that
  flushed C stdio buffers nothing used was deleted with it.
- **Deciding the pointer by "the log is non-empty".** Once a spawn directory
  outlives one launch (#616's run model, closed unmerged, and identity-v2's
  continue-keeps-ID both reuse it), a stale log from an earlier launch would
  print a pointer on a clean run. Three error paths in `runner.py` used that test. They
  were deleted, and the window owns the one rule.

**Accepted limits:** programs that open `/dev/tty` directly can still draw on the
screen. A write that races the fd swap can split across it just after the TUI
exits.

**Revisit if:** a native-terminal driver needs to hand the TUI something other
than the window's saved fds, or launcher-spawned helpers start prompting on
`/dev/tty`.

---

## D-native-terminal — Codex/OpenCode inherit the real terminal; Claude/Pi keep the relay

**Status:** decided by the user (2026-10-15: "option B is better"; Claude/Pi:
"i didn't really have any problems resizing claude and pi so its fine").
Implemented on branch `feat/native-terminal-primaries`, PR #614, open and not
merged; `main` still relays all four. Work-item IDs D1, D5, D8.

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
native-terminal branch, PR #614, open and not merged. **Mechanism superseded** on
2026-10-16 by [D-start-gate-guardian](spawn-identity.md#d-start-gate-guardian)
(identity-v2 PR 1, #633). That replaces this watchdog, PDEATHSIG and
`parent_watchdog.py` with one `sh` guardian per harness, on every launch path.
The diagnosis below still holds: PDEATHSIG misses wrapper descendants, and an
unknown birth time is never safe to signal. The Python watchdog, its
registration protocol and its 17 MB do not. #614 has to adopt the gate when it
rebases.

**Problem:** killing a native launcher with `kill -9` left the TUI alive and
"Reconnecting…". The backend had a parent-death link and the TUI had none. The
first fix gave the TUI `PR_SET_PDEATHSIG`. Review found that PDEATHSIG reaches
only the **direct child**. Codex's command is a Node wrapper that starts the
real binary, so the inner `codex` TUI and inner `codex app-server` both
survived. `main` already leaks the inner app-server this way.

**Decision:** on POSIX, each launcher starts one PID/birth-guarded watchdog
before any child. It owns the backend's process group and the TUI's process
tree; PDEATHSIG is removed. Background Codex/OpenCode spawns use the same
ownership model for their backend. Children register over a control pipe after
they are spawned; pipe EOF without `shutdown` means the launcher died.

**Cost accepted:** the watchdog's initial package imports cost about 34 MB RSS;
the isolated implementation reduced that to about 17 MB.

**Rejected:**
- **PDEATHSIG on the TUI**, and per-child watchdogs or PDEATHSIG elsewhere.
  Neither contains wrapper descendants, and per-child watchdogs multiply the
  RSS cost.
- **Unknown birth time treated as "probably fine".** The watchdog kills by PID,
  and PID reuse makes that unsafe.
- **A pre-exec parent-death hook** to close the spawn-to-registration window.
  It would run Python after fork in a threaded launcher and would still cover
  only one PID. The guardian closes that window another way: the harness's
  first exec is an `sh` stub that blocks until its scope is durable.

**Follow-ups:** #609 (fold `platform/terminate.py` and
`fallback.terminate_tree` onto `scope_termination`, leaving one termination
path), #610 (macOS instant-exit TUI reported as a link failure).

---

## D-cancel-by-order — Cancellation is decided by which came first

**Status:** decided 2026-10-15 (work-item D9). Shipped for managed attach with
the native terminal (PR #646); extended to black-box primaries by PR #658
(open), see [D-blackbox-signal-latch](#d-blackbox-signal-latch).

**Decision:** a native primary run is cancelled (exit 130) if the launcher
received SIGHUP/SIGTERM before it observed the TUI's exit; if the TUI's exit
was observed first, the real result stands. The recorder and teardown ordering
are in the [current mechanism](../architecture/primary-terminal-ownership.md).

**Why not exit codes:** a closed pane sends SIGHUP to the TUI as well as the
launcher. OpenCode exits 0 on SIGHUP, so "the TUI died by signal" recorded a
closed pane as `succeeded 0`, a regression from `main`'s cancelled 130.
Harnesses don't agree on SIGHUP exit codes. The order in which the launcher saw
the two events is the only fact that means the same thing everywhere.

**Constraint:** whatever records the order is touched from a signal handler,
which runs on the main thread. A plain `Lock` could deadlock the launcher
against itself. Any state a `SignalCoordinator` receiver touches must be
lock-free or reentrant. #614's recorder used an `RLock`; shipped code reads a
plain bool latch instead.

---

## D-blackbox-signal-latch — Black-box primaries cancel on SIGTERM/SIGHUP through a latch {#d-blackbox-signal-latch}

**Status:** decided 2026-10-16 (`work:relay-signal-exit` S1, S6, S7, with the
identity-v2 lead). Implemented in PR #658 (`fix/relay-signal-exit`, #623),
merged 2026-10-17 (`8055a1f77`).

**Problem:** a Claude/Pi PTY primary's launcher had no SIGTERM/SIGHUP handler.
A pane kill, terminal close or systemd scope stop killed it at the default
disposition, so it never recorded an exit, concluded the native run or stopped
its session, and the harness was orphaned. Under identity-v2's derived status
that run would read as crashed forever.

**Decision:**
1. **The handler only latches.** `run_primary_process_with_capture()` moves
   `running.wait()` to a daemon thread, and the main thread polls every 100 ms.
   It calls `terminate()` and `cancel_wait()` once each, outside handler
   context. Rejected: calling the gate or `cancel_wait()` from the handler,
   because `LaunchGate` and `threading.Event.set()` take non-reentrant locks the
   interrupted main thread may hold
   ([mechanism](../architecture/primary-terminal-ownership.md#signal-handlers-run-inside-the-main-thread)).
2. **No new kill path and no state written in the handler.** The child stops
   through the start gate, and exit, native-run conclusion, lifecycle finalize
   and session stop each run once on the normal return path. That keeps one
   call for identity PR 4 to reroute. Rejected: `os.kill`/`killpg` of the
   child, or an exit record written from the handler.
3. **Same outcome as managed attach.** `PrimaryProcessResult` gains
   `cancelled`, and `PrimaryExecutionResult.managed_cancelled` becomes
   `cancelled` for both paths; exit is 130
   ([D-cancel-by-order](#d-cancel-by-order--cancellation-is-decided-by-which-came-first)).
4. **Repeat escalation on.** The coder first shipped without it, reasoning that
   a `SIG_DFL` re-raise skips finalize. Review (F3) showed the cost: if the wait
   wedges (the relay blocked in `waitpid`, a child ignoring TERM), only SIGKILL
   could end the launcher. The 1 s grace already routes a terminal-close burst
   (HUP, HUP, TERM) to the latch, so a later repeat can safely mean "force".
   The rapid HUP+TERM probe still recorded one cancel and one session stop.

**Accepted:** a signal before `start()` still launches the harness and cancels
it within one poll; a signal in the microseconds after the child exits turns a
clean exit into cancelled.

**Handed on:** native black-box has no KILL after its one TERM (#657). The
receiver is unregistered before finalize, and 130 replaces the raw exit code at
the process layer. Both are identity-v2 PR 4 acceptance criteria
([gaps](../architecture/primary-terminal-ownership.md#known-gaps)).

**Revisit if** identity PR 4 moves the outcome write: the receiver should then
span that write, not just the wait.
