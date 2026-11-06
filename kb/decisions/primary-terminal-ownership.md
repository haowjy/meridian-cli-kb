# Primary Terminal Ownership

See [terminal ownership](../architecture/primary-terminal-ownership.md) for the current mechanism.

## Exit text {#d-exit-text-append--meridian-never-covers-a-harnesss-exit-text}

Append Meridian's hint after the harness exits; erasing or rewriting its exit text can hide the harness's own resume instructions.

## Quiet launcher {#d-tui-quiet-launcher--launcher-output-never-reaches-a-live-tui}

Launcher output must not draw over a live TUI. Redirecting only Python logging misses C writes and inherited children; file-descriptor redirection covers them. Direct `/dev/tty` writers can still draw, and writes racing the swap may split across it.

## Native terminal {#d-native-terminal--codexopencode-inherit-the-real-terminal-claudepi-keep-the-relay}

Codex and OpenCode use the real terminal because their managed connections provide observation without a PTY relay, whose nested PTY mangled resizes. Claude and Pi retain the relay. A TUI killed with SIGKILL cannot restore its terminal, so a reset may be needed.

## Shell job control {#d-shared-foreground-job--the-shell-keeps-job-control}

The native TUI and launcher share the shell's foreground job. A separate process group would require Meridian to implement terminal job control itself.

## Process containment {#d-launcher-watchdog--one-watchdog-per-launcher-instead-of-pdeathsig}

A parent-death signal reaches only a direct child and misses descendants behind wrappers. The [start gate and guardian](spawn-identity.md#d-start-gate-guardian) replaced the proposed launcher watchdog and protect the pre-exec window as well.

## Cancellation order {#d-cancel-by-order--cancellation-is-decided-by-which-came-first}

A primary is cancelled when the launcher observes termination before TUI exit; otherwise the observed exit stands. Harnesses differ in their SIGHUP exit codes, so exit code alone cannot identify a closed terminal. Signal-handler state must be lock-free or reentrant to avoid self-deadlock.

## Black-box signal latch {#d-blackbox-signal-latch}

The signal handler latches cancellation; normal execution performs termination and finalization outside the handler. Calling code that takes a non-reentrant lock from the handler can deadlock against the interrupted thread.
