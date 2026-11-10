# Profile tool permissions

Profiles express tool, sandbox and approval policy; harnesses enforce what their native mechanisms can express. Current behavior and gaps are in meridian-cli `docs/agent-profiles.md` and `src/meridian/lib/safety/.context/CONTEXT.md`.

## Mars owns native names

Permission spellings and callable tool names can differ: Claude scopes a write with `Edit(path)` while the callable tool is `Write`. Deriving one from the other can silently leave a gap.

## Native enforcement

A universal interception hook would create a second policy layer that can disagree with the harness. A sandbox is not equivalent to tool denies. Where a harness lacks a scoped rule but offers a hook, Meridian compiles rules into that seam.

## Codex shell hook

Scoped Bash rules use a Meridian `PreToolUse` hook. It guards against accidents, not adversarial bypass: wrappers, substitutions, scripts and alternate execution tools can evade a rule unless explicitly matched. A shell parser was rejected because it adds complexity without closing those paths. Codex can silently skip untrusted hooks and fail open on hook errors, so Meridian verifies hook trust and enabled state before the first turn and makes hook crashes refuse the call. See `src/meridian/lib/harness/projections/.context/CONTEXT.md` and `docs/upgrading.md`.

## Approval `auto`

`approval: auto` uses each harness's own automatic approval mechanism. Codex's `--approve-for-me` was rejected because it conflicts with sandbox selection and is unavailable on the app server. Claude's auto mode can widen a confined profile through classifier decisions, including network and writes outside working directories; the classifier and Codex reviewer are not security boundaries. See `docs/upgrading.md`.

## Report actual projection

An enforcement report is produced with the projection so it describes what reached the harness. The lost-restriction invariant in meridian-cli's root `AGENTS.md` governs reporting.

## Primaries

Profile rules bind primaries as well as spawns. Native user settings or passthrough flags can widen enforcement; these are reported rather than stripped.

## Headless ask

A spawn has no human to answer a supported `ask`, so it is refused rather than routed to the parent. A primary keeps `ask`.

> [!FLAG] **Needs human review.** The OpenCode divergence below remains open for review.

OpenCode lowers `ask` to native `deny` in spawns. The model receives OpenCode's refusal, and a tool removed from the catalog produces no attempt to journal. A custom reason would require a mediation hook or an upstream seam; see meridian-cli `src/meridian/lib/harness/.context/TODO`.

## List form

A tools list grants named tools; it does not mean only those tools. Imported Claude agent lists retain their restrictive meaning. A map with `'*': deny` expresses a whole-profile allowlist.

## Permission set merging

Tool grants and denies and network host sets add across surfaces. A package profile cannot replace a consumer's deny set: a dependency must not erase the consumer's restriction.

## Platform rules

Meridian adds control-plane grants needed for its operation; see [sandbox platform access](sandbox-platform-access.md). A child needs a completion command to signal completion.

## Unset sandbox

An unset sandbox means the harness default. Inferring it from the tool list conflates separate controls.

## User settings

Claude's user, project and local settings stay in force; their allows may widen an allowlist. Removing them would also discard their hooks. The report exposes this limitation.
