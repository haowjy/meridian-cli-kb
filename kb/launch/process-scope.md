# Process-scope ownership

Harnesses leave descendants behind after their root exits, some reparented to PID 1. Meridian records a durable process scope for each launch, so normal teardown and crash recovery use the same containment boundary, and session-owned scopes can outlive a disposable launcher. Contract: the source checkout's `src/meridian/lib/platform/process_scope/.context/CONTEXT.md`; untangling work is in its `FUTURE`.

## Why durable scopes

A process-tree snapshot alone misses descendants reparented after an intermediate exits. Harness-specific cleanup would duplicate policy and miss generic tool children. A PID read from metadata needs a birth check before it is signalled, because PIDs are reused; unknown identity is never proof that signalling is safe.

## Start gate and guardian

A harness can't execute until its scope and verified birth are durable. A small POSIX guardian watches for runner death through a pipe and stops the owned process group; a parent-death signal alone reaches only a direct child and misses binaries behind wrappers. A TUI that shares the shell's process group gets root-only termination so the shell isn't signalled, and on that path the guardian sends one TERM, never a delayed KILL that could hit a reused PID. The start gate also covers the window before exec, which is why it replaced a launcher watchdog.

## The continue sweep is the backstop

Before a continue starts the next launch, the holder of the prior launch lock sweeps that launch's remaining scopes, refusing a live owner and rechecking process identity before each signal ([spawn launches](../state/spawn-launches.md#the-launch-lock-fences-each-launch)). Managed primaries share the same model ([primaries](primaries.md)).

## Implementation hazards

- A process group ID can be reused after its last member exits. A guardian inside the group pins it during staged termination; an unpinned PID can't safely get a delayed second signal without a birth check.
- A guardian must drop harness stdio before waiting, or it can hold off EOF and deadlock the launcher. A hand-written fork/exec path must close inherited descriptors as the subprocess path does, and a proof of concept for descriptor handling needs the production topology, PTYs and EOF readers included.
- Moving a blocking wait to a thread moves its cleanup out of the main thread's exception unwinding, so the main thread must stop the child and join the waiter on interruption. A dead terminal raises `termios.error`, which is not an `OSError`.
- Select processes to stop by a handle recorded when starting them, never by command-line text: agent prompts appear in other processes' arguments.

## Known gaps

- A descendant that creates its own session or process group escapes group cleanup after its parent dies, and a process-table scan after reparenting can't reconstruct the original tree.
- Fallback paths that signal a group whose root birth time is unreadable, and the three competing termination implementations: the source checkout's `src/meridian/lib/platform/process_scope/.context/FUTURE`.
- Background-worker or metadata-only paths may own long-lived processes without recording a scope; PID fallback gives weaker containment there.
