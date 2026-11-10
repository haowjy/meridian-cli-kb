# Launch Process Ownership

Managed startup, stop and cleanup share a lifecycle gate: otherwise stop can return before an in-flight launch publishes a child, or a new start can overlap cleanup of the old one. Slow cleanup retains ownership even when foreground waiting is bounded. See [managed-primary lifecycle](../architecture/managed-primary-lifecycle.md).

### Durable process-scope ownership over POSIX-only or psutil-only cleanup

Durable scope ownership lets normal teardown and recovery use the same containment boundary. A process-tree snapshot alone can miss descendants reparented after an intermediate exits; harness-specific cleanup duplicates policy and misses generic tool children. Metadata PID signals require birth checks to avoid PID reuse. See [process scope](../architecture/process-scope.md).
