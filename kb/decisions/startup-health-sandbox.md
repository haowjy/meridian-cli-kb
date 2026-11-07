# Startup, Health, and Sandbox Policy

## Doctor and Health Checks

Doctor defaults to a local scan; cross-project maintenance is explicit because it can be expensive and touch paths unavailable to nested sandboxes. Prune advice is useful only when stale artifacts exist, not when the remaining sessions are live. See [health checks](../operations/health-checks.md).

## Startup Pipeline

A lightweight command catalog and lazy imports keep trivial commands from paying for operation and harness dependencies. Read-only resolution must not create project state; `ensure` operations own mutation. See [startup architecture](../architecture/startup-pipeline.md).

## Sandbox Projection Policy

Projecting all of `~/.meridian` into a child would give it access to sibling projects' state and lock namespaces. Grant purpose-scoped paths instead; see [sandbox projection](../architecture/sandbox-projection.md).

Autosync is best effort: a lock acquisition failure is a skip, rather than a reason to grant the global lock directory. An inaccessible implicit user config may be treated as absent with a warning; explicitly requested config must fail visibly.
