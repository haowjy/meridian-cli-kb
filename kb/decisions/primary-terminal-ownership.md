# Primary Terminal Ownership

See [terminal ownership](../architecture/primary-terminal-ownership.md) for the current mechanism.

## Exit text

Append Meridian's hint after the harness exits; erasing or rewriting its exit text can hide the harness's own resume instructions.

## Quiet launcher

Launcher output must not draw over a live TUI. Redirecting only Python logging misses C writes and inherited children; file-descriptor redirection covers them. Direct `/dev/tty` writers can still draw, and writes racing the swap may split across it.

## Native terminal

Codex and OpenCode use the real terminal because their managed connections provide observation without a PTY relay, whose nested PTY mangled resizes. Claude and Pi retain the relay. A TUI killed with SIGKILL cannot restore its terminal, so a reset may be needed.

## Shell job control

The native TUI and launcher share the shell's foreground job. A separate process group would require Meridian to implement terminal job control itself.

## Process containment

A parent-death signal reaches only a direct child and misses descendants behind wrappers. The [start gate and guardian](../architecture/process-scope.md) replaced the proposed launcher watchdog and protect the pre-exec window as well.

## Cancellation order

A primary is cancelled when the launcher observes termination before TUI exit; otherwise the observed exit stands. Harnesses differ in their SIGHUP exit codes, so exit code alone cannot identify a closed terminal. Signal-handler state must be lock-free or reentrant to avoid self-deadlock.

## Black-box signal latch

The signal handler latches cancellation; normal execution performs termination and finalization outside the handler. Calling code that takes a non-reentrant lock from the handler can deadlock against the interrupted thread.
