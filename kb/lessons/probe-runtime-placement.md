# Probe Runtime Placement

Keep scratch homes, copied runtime state, environments and databases under `/tmp`, outside autosynced work directories. Autosync may stage and push these large or private files.

A scratch home alone does not isolate inherited project variables or a shared harness binary. Start state-mutating probes with a clean environment and scratch home, and disable self-updates for any shared harness binary. Git worktrees share repository configuration; use a separate clone for disposable configuration, or pass temporary identity per Git command. See [dogfooding builds](dogfooding-pr-builds.md).
