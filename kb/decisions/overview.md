# Decisions Overview

Decision records explain why durable architectural choices were made — what alternatives were considered, what constraints drove the choice, and when the decision should be revisited. Start with [../decisions.md](../decisions.md) for the chronological index, then follow links into domain pages for full rationale.

## Domain Pages

| Page | Coverage |
|---|---|
| [chat-backend.md](chat-backend.md) | Chat pipeline architecture: custom protocol, event model, acquisition, HITL, command layer, structural refactors (D1–D2, D8–D31) |
| [dev-frontend.md](dev-frontend.md) | Unified dev setup for `meridian chat --dev`: portless, Vite, exposure model, env scrub, launcher strategy (DF-D1–DF-D7) |
| [model-resolution.md](model-resolution.md) | Model alias resolution, routing context, profile schema, resolve-once pattern, `MERIDIAN_HARNESS` spawn-local semantics, agent overlays, compiler ownership, candidate-chain semantics (D52–D57, D72–D75) |
| [package-management.md](package-management.md) | Mars compiler, skill schema, `.agents/` elimination, targeting, sync, bootstrap docs, collision resolution, convention-based source discovery, upgrades-available direct-deps fix (D35–D40, D50–D51, D58–D63, D71, D77, D87) |
| [managed-command-references.md](managed-command-references.md) | `managed_cmd()` helper for MERIDIAN_MANAGED-aware user-facing output; correct `mars` vs `meridian mars` command references in hints and errors (D76) |
| [state.md](state.md) | State roots, JSONL event stores, dual-root layout, crash-only design, WorkScope model (named vs ambient), CR1 fix, state v2 migration (foundational undated decisions + D-WorkScope PR #328) |
| [launch.md](launch.md) | Launch pipeline, composition seam, harness identity env, spawn wait barrier, spawn-level goal authority and completion-contract composition (D32–D34, D57, D63-launch, 2026-05 spawn-goal) |
| [startup-health-sandbox.md](startup-health-sandbox.md) | Descriptor-driven startup, bootstrap split, doctor/reaper, sandbox projection policy |
| [state-and-launch.md](state-and-launch.md) | Compatibility map for links that used the previous combined state/launch decision page |
| [testing.md](testing.md) | Test-tier ownership, aggressive deletion safeguards, security-suite exception, rejected alternatives, and fake-executable observation discipline |
| [telemetry.md](telemetry.md) | Three-layer observability: envelope design, storage layout, readers vs. sinks, liveness, retention (D61–D70) |
| [workspace.md](workspace.md) | Workspace config: `[workspace]` in `meridian.toml`, named entries, missing-path behavior, dedicated loader (D41–D46) |
| [git-autosync-merge-strategy.md](git-autosync-merge-strategy.md) | Git autosync: merge over rebase, local-wins conflict handling, AGENTS.md notice exception, artifact ownership split (D-autosync-1 through D-autosync-4) |

## Naming

Domain-specific decisions use a domain prefix (for example `DF-D1`) when the bare D-number would conflict with existing IDs. Bare D-numbers from the original decision record are preserved as heading anchors in domain pages so existing references remain addressable.

When the same bare ID appears in two domains (D61, D62, D63 each appear in both telemetry and bootstrap/launch clusters), domain pages disambiguate with qualifiers in the heading anchor: `#d61-telemetry`, `#d61-bootstrap`.

## Adding New Decisions

1. Write the decision record in the appropriate domain page under `decisions/`.
2. Add an entry to the chronological table in [../decisions.md](../decisions.md) linking to the new anchor.
3. If the decision supersedes an earlier one, update both the old entry (add a superseded note) and the index entry.
