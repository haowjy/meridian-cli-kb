# Primaries

A primary stays open until the human ends it, so the [completion hold](../spawns/completion-hold.md) doesn't apply. A managed Codex or OpenCode primary has a Meridian launcher, a backend and the user-facing TUI: the TUI runs on the real terminal in the user's foreground process group, the backend has its own [process scope](process-scope.md), Meridian observes the backend through a separate connection, and guardians stop both if the launcher dies. A PTY relay's nested PTY mangled resizes, which is why those TUIs aren't relayed. Claude and Pi have no backend/TUI split and keep a PTY relay, which can restore terminal modes after the child dies; a TUI on the real terminal that is killed with SIGKILL can't, so a `reset` may be needed. Contracts: the source checkout's `src/meridian/lib/launch/process/AGENTS.md` and `.context/CONTEXT.md`.

The native TUI and the launcher share the shell's foreground job; a separate process group would make Meridian implement terminal job control itself.

## Start, stop and cleanup share one gate

Stop waits for an in-flight start or its cleanup: otherwise stop could report no child while the start then publishes one, or a new start could overlap cleanup of the old. Cleanup keeps the gate even after bounded foreground failure handling stops waiting for it. A second runtime lock, or a lock that guards only replacement, would leave the first-publication race open.

## Repair and cleanup

A dead launcher's free lock makes its row read as crashed; reads neither signal nor write. The reaper claims the free lock and records the outcome and a cleanup claim before terminating recorded scopes ([spawn outcome](../state/spawn-outcome.md#only-a-claim-authorizes-the-repair-write)). `spawn cancel` makes its own liveness and process-identity checks. A PID alone is never durable authority.

## Cancellation is decided by order, not exit code

A run is **cancelled** if the launcher received SIGHUP or SIGTERM before it observed the TUI exit, whatever code the TUI returned; it records `128+N`. If the launcher saw the TUI exit first, the TUI's result stands. A terminal Ctrl-C is a keystroke to the harness, so the harness's result is recorded. The exit code can't decide this: closing a pane sends SIGHUP to both the TUI and the launcher, harnesses react differently (OpenCode quits normally with 0), and the old exit-code rule recorded a closed OpenCode pane as succeeded. A signal in the microseconds between the child's exit and the wait being marked complete turns a clean exit into cancelled; that window was accepted.

## Terminal output

Launcher output must not draw over a live TUI. Redirecting only Python logging misses C-level writes and inherited children, so the launcher redirects file descriptors; direct `/dev/tty` writers can still draw, and a write racing the swap can split across it. Meridian appends its continuation hint below the harness's exit text rather than rewriting it, which could hide the harness's own resume instructions. A relay leaves the alternate screen only if the child hasn't already: repeating the escape can restore an old cursor position and overwrite the harness's output.

## Known gaps

- A native TUI that ignores TERM can keep the terminal after Meridian finalizes; the guardian has no KILL escalation on that path ([#657](https://github.com/haowjy/meridian-cli/issues/657)).
- One long suspended OpenCode run stopped again right after `fg` and hasn't reproduced since ([#611](https://github.com/haowjy/meridian-cli/issues/611)).
- On macOS, a native TUI that exits immediately can fail lifetime-link registration when it is already a zombie ([#610](https://github.com/haowjy/meridian-cli/issues/610)).
- A signal during extraction can precede the protected terminal write ([#831](https://github.com/haowjy/meridian-cli/issues/831)); a signal during `finalize_spawn` isn't in the stored result, and a second Ctrl-C a second or more later, the escape from a stuck hook or lock, can leave `crashed` ([#892](https://github.com/haowjy/meridian-cli/issues/892)).
