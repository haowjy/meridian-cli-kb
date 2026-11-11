# Workspace

A workspace entry grants a harness filesystem access to a directory; context names working memory ([work items](../state/work-items.md)). The two can name the same directory but stay separate, so a context directory never silently becomes a filesystem grant, and workspace paths never become prompt instructions or context environment variables. Schema, resolution and harness projection: meridian-cli `docs/configuration/workspace.md`.

## Named entries, committed and local

Committed configuration describes a team's conventional layout; local configuration overrides machine-specific paths by entry name. Names give stable merge keys when paths differ between machines; path-keyed or append-only arrays can't express a local override reliably. A dedicated loader validates the named entries, because Pydantic's `extra="allow"` would leave arbitrary named subtables as untyped dictionaries.

## No global workspace

There are no user-global entries: a hidden default would widen filesystem access in every project, so grants are declared per project.

## Missing roots are skipped and reported

A missing root is excluded from projection; there is no `enabled` field, so a committed root that exists can't be disabled. A missing committed path is often a normal partial checkout, while a missing local path is more likely a stale or mistyped override, so the two get distinct diagnostics. Excluding them also matters for Codex, whose bind mount of a missing path can fail the whole execution namespace ([sandbox projection](sandbox-projection.md)).

## Projection

Projected roots are Meridian-owned policy, kept apart from user passthrough arguments so permissions never have to be recovered by reparsing a flat argument list. On OpenCode, grants merge with inherited config: suppressing projection when a parent already set config would silently lose the child's access.

Granting a root can expose instructions in it: Claude discovers skills in projected roots.

Sandbox scope and approval policy are independent. On Codex, a file-change approval prompt alone doesn't prove a root is missing; the remote TUI and the managed backend have separate permission surfaces (`docs/codex-tui-passthrough.md`).
