# Primary terminal ownership

Codex and OpenCode TUIs use the native terminal; Meridian observes their backends through separate connections. Claude and Pi retain a PTY relay, which can restore terminal modes after a child dies. Native inheritance avoids a nested PTY resize race but cannot restore terminal state if the TUI is killed. See [terminal decisions](../decisions/primary-terminal-ownership.md) and the source checkout's `src/meridian/lib/launch/process/AGENTS.md`.

Signal callbacks run on Python's main thread and can interrupt code holding a lock. They latch state instead of acquiring locks or setting an Event, avoiding self-deadlock. Dead terminal cleanup catches `termios.error` as well as `OSError`.

## Cancellation is decided by order, not exit code

A run counts as **cancelled** if the launcher received SIGHUP or
SIGTERM **before it observed the TUI's exit**, whatever exit code the TUI
reported. It records the signal as `128+N` (129 or 143) rather than the TUI's code.
A SIGINT that reaches the launcher records 130 `cancelled`. A terminal Ctrl-C is a
keystroke to the harness, so Meridian records the harness's result. If the launcher saw the TUI exit first, the TUI's real result stands.
Managed attach reads its latch once, when
the main task resumes from the TUI wait. The black-box receiver latches
only while the wait thread has not yet marked the wait complete. A signal in the
microseconds between the child's exit and that mark turns a clean exit into
cancelled, which was accepted.

The exit code can't decide this, because a pane close sends SIGHUP to the TUI
as well as the launcher, and harnesses react differently. OpenCode treats SIGHUP
as a normal quit and exits 0. An earlier rule ("cancelled if the TUI died by
signal or the wait was cancelled") recorded a closed OpenCode pane as
`succeeded 0`. Order of observation gives the same answer for every harness.

---

## Terminal output

Meridian keeps its own output away from a live TUI and appends its continuation hint below the harness's exit text. A relay must only leave the alternate screen if the child has not already done so; repeating the escape can restore an old cursor position and overwrite the harness's output. The terminal window redirects launcher output to a separate log because backend stderr contains ordinary session noise.

## Known gaps

- A native black-box TUI that ignores TERM may retain the terminal after Meridian finalizes; the guardian has no KILL escalation on that path ([#657](https://github.com/haowjy/meridian-cli/issues/657)).
- One long suspended OpenCode run stopped again immediately after `fg`; it did not reproduce in subsequent probes ([#611](https://github.com/haowjy/meridian-cli/issues/611)).
- An immediately exiting native TUI on macOS can fail lifetime-link registration when its process is already a zombie ([#610](https://github.com/haowjy/meridian-cli/issues/610)).
- A signal during extraction can precede the protected terminal write ([#831](https://github.com/haowjy/meridian-cli/issues/831)).
- A signal during `finalize_spawn` itself is not in the stored result, and a double Ctrl-C at least a second apart before the write can leave `crashed`, because that repeat is the escape from a stuck hook or lock ([#892](https://github.com/haowjy/meridian-cli/issues/892)).
