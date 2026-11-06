# Completion Drain

A finished harness turn is a completion candidate, not proof that background work is finished. See [drain plans](../architecture/drain-plans.md) and [completion coordination](../architecture/completion-drain-coordination.md).

## Completion report {#d-completion-report-contract}

For resident holds, `done` accepts the retained candidate and its reply; `rearm` rejects it so a later candidate supplies a new reply. Pi retains its confirmation reply instead. The confirmation message must describe the applicable report rule: a short confirmation such as “Done” must not replace the actual report ([#725](https://github.com/haowjy/meridian-cli/issues/725)). If the rearm budget is exhausted, later work cannot replace the retained report; this is tracked in `src/meridian/lib/streaming/.context/FUTURE`.

## Completion channel {#d-completion-channel}

A child without a usable completion signal channel receives no confirmation turn. A read-only Codex child with spawn capability can use the scoped execpolicy escape; without spawn capability it has no reason to hold for child work. If sandbox policy is unset but Codex defaults to read-only, the signal may still fail because Meridian does not infer that policy. See [Codex child spawn](sandbox-platform-access.md#d-codex-child-spawn).

## Candidate confirmation

A confirmation is best effort; known blockers prevent silent success and unreadable evidence cannot authorize it. `done` leaves descendants running; cancellation cascade is tracked in [#671](https://github.com/haowjy/meridian-cli/issues/671). The calling harness's wait setting supplies the default completion deadline, separately from the caller's wait checkpoint.

## Claude native work {#d-claude-native-hold}

Claude's own background-task stream informs the shared completion hold. A harness mod cannot replace that signal: the finish decision and timeout must survive a wedged child, and the mod API did not expose background shell tasks. See [drain plans](../architecture/drain-plans.md#claude-native-holds).
