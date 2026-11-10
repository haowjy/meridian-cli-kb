# Process Containment Lessons

A process group ID can be reused after its last member exits. A guardian inside the group pins it during staged termination; an unpinned PID cannot safely receive a delayed second signal without validating its birth. Unknown or inaccessible process identity is not proof of death. See [process scope](../architecture/process-scope.md).

A guardian must drop harness stdio before waiting: otherwise it can prevent EOF and deadlock the launcher. A hand-written fork/exec path must also close inherited descriptors like the subprocess path does. A proof of concept for descriptor handling must use production allocation and I/O topology, including PTYs and consumers that read to EOF.

Moving a blocking wait to a thread also moves its cleanup out of the main thread's exception unwinding. Arrange for the main thread to stop the child and join the waiter on interruption. A dead terminal can raise `termios.error`, which is not an `OSError`.

## Continue must lock the chat's current owner, not the named spawn

The unit that must not run twice is the native session. Resolving an old spawn to a chat requires checking the chat's current owner, and ownership must be published before the lock-holder allows another continue to proceed. See [spawn identity](../architecture/state-system/spawn-launches.md) for the current contract.
