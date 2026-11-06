# Workspace architecture

Workspace config controls filesystem access projected to a harness; context config supplies working memory. Keeping them separate avoids silently making every context directory a filesystem grant. User-global workspace entries do not apply across projects because that would widen access without project-specific authorization.

The source checkout's `docs/configuration/workspace.md` documents configuration and harness projection. See [workspace decisions](../../decisions/workspace.md) and [context resolution](../../concepts/context-resolution.md).

