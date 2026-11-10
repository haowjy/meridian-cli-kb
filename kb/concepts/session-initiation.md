# Session initiation modes

`--continue` resumes the same transcript; `--fork` branches it with identity-shaping inputs locked, so the branch keeps the source's prompt-cache prefix; `--fork-fresh` allows identity changes; `--from` starts a fresh session with prior output as reference material in the user turn. The flag contract is in meridian-cli `docs/commands.md`; why per-run material sits in the user turn is in [launch system](../architecture/launch-system.md#the-system-prompt-depends-only-on-what-defines-the-conversation).

## A spawn's goal is spawn state

A goal belongs to the spawn, not to the work item and not only to the prompt text: spawns in one work item can have different goals, and consumers shouldn't have to parse a prompt to find one. A goal doesn't select model, harness or permissions. Its text is trimmed without template substitution, so persisted and delivered text agree.

## Headless deny is checked before the row exists

A spawn's headless deny policy is checked during preparation, before a background spawn creates a row and worker: checking only in the worker delays an actionable refusal and leaves a transient row for callers to wait on. Bind checks persisted worker requests too, since those may not have passed through preparation.

Re-entry from the session picker is in [session browse](../architecture/session-browse.md).
