# Sandbox permission projection

## Fixed Projection Constraint

A child gets the current project's runtime and configured roots, not the whole shared Meridian home or project-lock directory. Exposing shared writable locks would allow cross-project interference. The exact project lifetime lock is materialized for a real launch and opened read-only by nested POSIX processes. See [sandbox platform access](../decisions/sandbox-platform-access.md#project-lock).

Missing context roots are excluded from projection: Codex's bind mount can otherwise fail the entire execution namespace, including commands that might diagnose it. Cross-project maintenance is explicit and gated at the library boundary; optional global configuration and shared coordination may be inaccessible in a child. See [startup and sandbox decisions](../decisions/startup-health-sandbox.md#sandbox-projection-policy).

