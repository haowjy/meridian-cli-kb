# Workspace

Workspace grants filesystem scope; context supplies working memory. Workspace paths do not become prompt instructions or context environment variables. See meridian-cli `docs/configuration/workspace.md` for the schema and projection.

## Shared and local entries

Committed configuration describes a team's conventional layout; local configuration overrides machine-specific paths.

## Named entries

Names provide stable merge keys when paths differ between machines. Path-keyed or append-only arrays cannot express a local override reliably.

## Missing roots

There is no `enabled` field: a missing checkout is skipped. Disabling a committed root that exists is not supported by the current schema.

## Missing-path diagnostics

A missing committed path can be a normal partial checkout; a missing local path is likely a stale or mistyped override. Both are reported, with distinct diagnostics.

## Workspace loader

The dedicated loader validates named entries and produces filesystem findings; Pydantic's `extra="allow"` would leave arbitrary named subtables as untyped dictionaries.

## No global workspace

A hidden global default would widen filesystem access in every project, so scope grants must be declared per project.

## Policy and passthrough {#d47}

Projected roots are Meridian-owned policy, separate from user passthrough arguments. This avoids reparsing a flat argument list to recover permissions.

## OpenCode inheritance

Workspace grants merge with inherited OpenCode config; suppressing projection when a parent already set config would silently lose child access. Malformed inherited config is rejected.

## Codex approvals

Sandbox scope and approval policy are independent. A file-change approval prompt alone does not prove a workspace root is missing. See [workspace projection](../concepts/workspace-projection.md).
