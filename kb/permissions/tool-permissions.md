# Profile tool permissions

A profile states tool, sandbox and approval policy; each harness enforces what its native mechanisms can express, and Meridian reports the rest. Current per-harness behavior and gaps: meridian-cli `docs/agent-profiles.md`, `src/meridian/lib/safety/.context/CONTEXT.md` and `src/meridian/lib/harness/projections/.context/CONTEXT.md`.

## Enforcement is native

A universal interception hook would be a second policy layer that can disagree with the harness, so Meridian compiles rules into each harness's own mechanism; where a harness lacks a scoped rule but has a hook, the rules compile into that hook. A sandbox is not equivalent to tool denies, and an unset sandbox means the harness default: inferring it from the tool list would conflate two separate controls.

## Mars owns native names

Permission spellings and callable tool names differ: Claude scopes a write with `Edit(path)` while the callable tool is `Write`. Deriving one from the other can silently leave a gap, so Mars supplies both and Meridian never keeps a second table.

## The report describes what reached the harness

The enforcement report is produced with the projection, so it describes what was emitted rather than what was intended. The lost-restriction invariant in meridian-cli's root `AGENTS.md` governs what must be shown. Profile rules bind primaries as well as spawns. Native user settings and passthrough flags can widen enforcement; they are reported, not stripped. Claude's user, project and local settings stay in force because removing them would also discard their hooks, so their allows can widen an allowlist; the report shows it.

## Codex shell rules are an accident guard

Scoped shell rules on Codex go through a Meridian `PreToolUse` hook. It guards against accidents, not adversarial bypass: wrappers, substitutions, scripts and other execution tools can evade a rule unless explicitly matched. A shell parser was rejected because it adds complexity without closing those paths. Codex silently skips untrusted hooks and fails open on hook errors, so Meridian verifies the hook is trusted and enabled before the first turn and turns a hook crash into a refusal.

## `approval: auto` uses the harness's own mechanism

Codex's `--approve-for-me` was rejected: it conflicts with sandbox selection and isn't available on the app server. Claude's auto mode can widen a confined profile through classifier decisions, including network access and writes outside the working directories; neither Claude's classifier nor Codex's reviewer is a security boundary (`docs/upgrading.md`).

## Headless ask

A spawn has no human to answer `ask`, so a supported `ask` is refused rather than routed to the parent; a primary keeps `ask`. OpenCode lowers `ask` to its native `deny` in spawns, so the model sees OpenCode's refusal, and a tool removed from the catalog leaves no attempt to journal. A custom refusal reason would need a mediation hook or an upstream seam (harness `.context/TODO`).

> [!FLAG] **Needs human review**: whether OpenCode spawns should keep lowering `ask` to `deny`, or get a mediation hook so the refusal carries Meridian's reason, is undecided. Flagged 2026-11-07.

## A list grants; it doesn't restrict

A tools list grants the named tools; it doesn't mean "only these". A map with `'*': deny` expresses a whole-profile allowlist. Imported Claude agent lists keep their restrictive meaning, because that is what they meant where they came from. Shell rules differ: a profile whose shell rules are only scoped allows is a shell allowlist, so ordinary read commands need explicit permission too; read-only agents have been denied basic inspection this way.

## Permission sets add across layers

Tool grants and denies, and network host sets, add across configuration surfaces. A package profile can't replace a consumer's deny set: a dependency must never erase the consumer's restriction.

## Platform grants

Meridian adds the control-plane grants it needs to operate. A child needs `meridian spawn done` and `spawn rearm` to signal completion, so profile rules can't deny those two; read and launch grants yield to an equal or narrower profile restriction. The sandbox side is in [sandbox projection](sandbox-projection.md).
