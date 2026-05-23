# architecture/overview — Architecture Domain

Architecture pages explain how Meridian realizes its concepts in code: subsystem boundaries, ownership, invariants, and data flows.

## Pages

- [system-overview.md](system-overview.md) — Subsystem map, entry points, and end-to-end data flow.
- [launch-system.md](launch-system.md) — Launch factory, adapter composition, resolve-before-persist, and launch invariants.
- [state-system.md](state-system.md) — JSONL event stores, atomic writes, locking, reaper reconciliation, and migrations.
- [spawn-finalization.md](spawn-finalization.md) — Terminal write policy, authority lattice, and finalization races.
- [managed-primary-lifecycle.md](managed-primary-lifecycle.md) — Codex/OpenCode managed-primary process roles, passive reaper safety, and explicit cleanup boundary.
- [app-server.md](app-server.md) — REST, WebSocket/SSE streaming, MCP stdio server, and connection management.
- [mars-compiler.md](mars-compiler.md) — Mars compiler module map, config-entry pipeline, collision resolution, and stale cleanup.
- [mars-targeting.md](mars-targeting.md) — Mars targeting architecture: `.mars/` as Meridian compiled read surface, native harness skill dirs, and conditional native agent emission.
- [mars-launch-bundle.md](mars-launch-bundle.md) — Cross-repo launch-bundle system: Mars/Meridian ownership, bundle contract, scaffold slots, harness status, static sync vs runtime differences.
- [mars-routing.md](mars-routing.md) — Mars-internal routing architecture: slug primitive, default harness_order, routing parity, acceptance layer, RouteDecisionReport DTO.
- [mars-model-refresh.md](mars-model-refresh.md) — Catalog `ensure_fresh` and probe refresh: `ModelsRefreshControl`, CLI flags on sync/models/launch-bundle.

## Subtrees

- [chat/](chat/) — Chat runtime, event pipeline, normalizers, acquisition, recovery, commands, dev frontend, and extensibility.
- [telemetry/](telemetry/) — Three-layer observability spine: local persistence, event catalog, reader and query.
- [workspace/](workspace/) — Workspace config schema, resolution, merge semantics, and migration.

## Reading Paths

- New system work: start with [system-overview.md](system-overview.md), then the subsystem-specific page.
- Chat backend work: read [chat/overview.md](chat/overview.md), then the specific page within the chat subtree.
- Mars work: read [mars-targeting.md](mars-targeting.md) for layout, [mars-compiler.md](mars-compiler.md) for internals.
- Decision rationale: follow D-number references into [../decisions.md](../decisions.md) or the domain-specific decision pages under [../decisions/](../decisions/).
