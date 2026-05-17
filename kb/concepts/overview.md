# concepts/overview — Concepts Domain

Concept pages define Meridian's durable mental models. Read these before implementation-specific architecture pages.

## Pages

- [spawn-lifecycle.md](spawn-lifecycle.md) — Spawn status machine, heartbeat, and reaper mechanics.
- [spawn-wait-barrier.md](spawn-wait-barrier.md) — Chat-lineage wait sets, yield checkpoints, and barrier discipline.
- [spawn-output-contract.md](spawn-output-contract.md) — Report-first spawn output, transcript pointer behavior, and metadata disclosure levels.
- [state-model.md](state-model.md) — Dual-root state split, JSONL event sourcing, crash-only recovery, and UUID keying.
- [harness-abstraction.md](harness-abstraction.md) — Policy/mechanism split, adapter contract, and harness-agnostic constraints.
- [composition-pipeline.md](composition-pipeline.md) — Semantic IR to harness-specific argv/env projection.
- [hooks-and-plugins.md](hooks-and-plugins.md) — Lifecycle hooks, built-in hooks, shell hooks, and plugin boundary.
- [extension-system.md](extension-system.md) — Unified command definition, extension registry, surfaces, and dispatcher.
- [skill-schema.md](skill-schema.md) — Universal SKILL.md frontmatter, per-harness lowering, skill variants, and exact-match selection.
- [native-config.md](native-config.md) — Native config passthrough escape hatch, shape validation, harness scoping, and per-harness projection.
- [bootstrap-docs.md](bootstrap-docs.md) — Two-tier bootstrap document discovery and `meridian bootstrap` launch behavior.
- [config-precedence.md](config-precedence.md) — Config file merge order and runtime override chain.
- [context-resolution.md](context-resolution.md) — Work, KB, archive, and strategy path resolution.
- [session-initiation.md](session-initiation.md) — Four-mode session initiation model: --continue, --fork, --fork-fresh, --from; identity lock; four-layer content composition; bare flag inference.
- [workspace-projection.md](workspace-projection.md) — Workspace entries, merge semantics, and snapshot projection.

## Subtrees

- [model-resolution/](model-resolution/) — How model names become concrete models: aliases, routing, profiles, policies.
- [package-management/](package-management/) — Mars package sync, `.mars/` compiled store, targeting, and sync model.
