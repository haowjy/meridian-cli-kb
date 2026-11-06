# Task Directory and Reference Anchors

Relative file references resolve from the selected task directory, without per-file fallback: fallback makes the chosen file depend on what happens to exist on a machine. `kb:` references resolve within the project authority's KB even when the task directory is a worktree. See [reference resolution](../concepts/reference-resolution.md).

### D-stale-task-dir-graceful-fallback: Stale work-item task_dir degrades with warning, not hard error

A missing work-item task directory falls back with a warning; reading does not silently rewrite the work item. An explicit task directory takes precedence. See [checkout pins](session-work-links.md#d-checkout-pin-sources).

An explicit work selection bounds work-item lookup: using an ambient attachment as a fallback would make the source of the selected checkout unpredictable.
