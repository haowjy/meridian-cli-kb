# Git Autosync Merge Strategy

Autosync merges rather than rebases and normally aborts a conflict, preserving local work at HEAD and leaving the remote available for inspection. A rebase can stop partway through replay and leave later syncs wedged; a merge has one conflict point. Conflict state is recorded separately rather than written into user instruction files. See [hooks and plugins](../concepts/hooks-and-plugins.md) and the builtin hook documentation for current behavior.
