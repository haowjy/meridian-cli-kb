# Architecture Refactor Pitfalls

Git subprocesses inherit `GIT_DIR` and `GIT_WORK_TREE` from hooks; those variables can redirect Git to another repository even when cwd is correct. Remove inherited Git routing variables when a subprocess must operate on its own cwd.

## A Capability-Gated Second Channel Drifts From the First

Duplicating a content delivery path lets conditions and fixes diverge. Verify content at the artifact the harness reads, not just an intermediate, and check it arrives once. An observability artifact becomes part of delivery when the harness reads it. Visibility and explicit skill loading are distinct: hiding a skill from discovery must not suppress an explicit load. See [launch system](../architecture/launch-system.md) and [package management](../concepts/package-management.md).
