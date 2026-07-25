# Decision Index

Decision pages own rationale, rejected alternatives, supersession, and revisit
conditions. Architecture pages own current mechanism. This page only routes to
the canonical records; it must not become a second explanation layer.

See [Decision Records](decisions/overview.md) for how to add or update a record.

## Current Cross-Cutting Decisions

| Date / ID | Status | Decision | Canonical record |
|---|---|---|---|
| 2026-07, State Decision 3 | Current | Commit project identity as `[project].id` in `meridian.toml`; keep runtime state user-local. | [State](decisions/state.md#project-identity-in-meridiantoml-no-repo-local-state-decision-3-2026-07) |
| 2026-07, PR #422 | Current | Every store mutation acquires its stable lock, re-reads, mutates, and atomically publishes. | [State](decisions/state.md#concurrency-by-construction-mutate-under-lock-seams-over-convention-enforced-write-tiers-pr-422-2026-07) |
| 2026-07, D91 | Current | Native hook fragments and lock-v3 lifecycle records replace synthesized universal hooks. | [Package management](decisions/package-management.md#d91-native-hook-fragments-replace-command-synthesis-2026-07) |
| 2026-07, D92 | Current | Recovery commands halt before compilation when removed-schema hook surfaces are unreadable. | [Package management](decisions/package-management.md#d92-shape-a-recovery-seam----halt-before-compilation-when-hook-surfaces-are-unreadable-2026-07) |
| 2026-07-17 | Current | Support POSIX; retain native-Windows branches only as untested legacy best effort. | [Design principles](principles/design-principles.md) |
| 2026-06, D-mars-owns-inventory | Current | Mars renders agent inventory; Meridian embeds the result rather than rebuilding it. | [Launch](decisions/launch.md) |
| 2026-05, D-fork-identity-lock | Current | Continue/fork modes preserve or intentionally replace recorded session identity. | [Session initiation](decisions/launch-session-initiation.md) |
| 2026-05, spawn-state-v2 | Current | Per-spawn `state.json` replaces the global spawn event log; session history remains JSONL. | [State](decisions/state.md#spawn-state-v2-per-spawn-statejson-over-global-jsonl-2026-05) |
| D7 / D27 / D28 | Superseded | `.agents/` as Meridian catalog and shared target was replaced by `.mars/` plus native targets. | [Mars targeting](architecture/mars-targeting.md) |

## Domain Records

| Domain | What its records decide | Page |
|---|---|---|
| State | Identity, stores, atomicity, locks, reconciliation, typed rows | [decisions/state.md](decisions/state.md) |
| Launch composition | Prepare/bind, policy replay, inventory, capability gates | [decisions/launch.md](decisions/launch.md) |
| Managed processes | Managed primaries, process scope, cancellation and cleanup | [decisions/launch-process-ownership.md](decisions/launch-process-ownership.md) |
| Waiting and sessions | Wait barriers, goals, output, continue/fork/from | [decisions/launch-session-initiation.md](decisions/launch-session-initiation.md) |
| Harness compatibility | Harness-specific launch history and superseded platform work | [decisions/launch-harness-compatibility.md](decisions/launch-harness-compatibility.md) |
| Startup / health / sandbox | Startup descriptors, doctor, bootstrap and projection policy | [decisions/startup-health-sandbox.md](decisions/startup-health-sandbox.md) |
| Package management | Mars compilation, targeting, ownership, hooks, recovery | [decisions/package-management.md](decisions/package-management.md) |
| Model resolution | Alias routing, candidate policy, prompting ownership | [decisions/model-resolution.md](decisions/model-resolution.md) |
| Workspace | Schema, path resolution, permission projection and migration | [decisions/workspace.md](decisions/workspace.md) |
| Telemetry | Envelopes, sinks, segments, querying and retention | [decisions/telemetry.md](decisions/telemetry.md) |
| Testing | Behavior-owned tiers and evidence gates | [decisions/testing.md](decisions/testing.md) |
| Chat backend | Protocol and command-layer history | [decisions/chat-backend.md](decisions/chat-backend.md) |
| Dev frontend | Portless/Vite exposure and launcher policy | [decisions/dev-frontend.md](decisions/dev-frontend.md) |
| Managed command references | Managed-aware command rendering | [decisions/managed-command-references.md](decisions/managed-command-references.md) |

## Compatibility Entry Points

[decisions/state-and-launch.md](decisions/state-and-launch.md) exists for old
inbound links. New links should target the domain record that owns the choice.
