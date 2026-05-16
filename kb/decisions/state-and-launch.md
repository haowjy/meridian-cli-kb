# Decisions: State and Launch Map

This page used to hold the full state, launch, startup, health-check, and sandbox decision set. It was split on 2026-05-05 because the combined page had grown past 5,000 words and mixed several independently evolving concepts. Keep this file as the stable entry point for older links and route readers to the focused pages below.

## State Layer

State storage, dual-root layout, UUID read/write behavior, JSONL event stores, and crash-only recovery now live in [state.md](state.md).

## Launch Pipeline

Launch composition, resolve-before-persist, finalization ownership, semantic IR projection, and config precedence now live in [launch.md](launch.md).

## Harness Identity and Environment

`MERIDIAN_HARNESS` one-hop propagation, wait-yield harness selection, and `MERIDIAN_HARNESS_COMMAND` removal now live in [launch.md](launch.md#harness-identity-and-environment).

## Spawn Wait Barrier

No-argument wait discovery, yield checkpoints, and background-spawn barrier language now live in [launch.md](launch.md#spawn-wait-barrier).

## Doctor and Health Checks

Doctor/reaper authority, active-spawn warning semantics, and prune advice behavior now live in [startup-health-sandbox.md](startup-health-sandbox.md#doctor-and-health-checks).

## Startup Pipeline

Descriptor-driven command startup, pure `resolve_*` vs mutating `ensure_*`, process-seam telemetry install, and CLI output protocol now live in [startup-health-sandbox.md](startup-health-sandbox.md#startup-pipeline).

## Sandbox Projection Policy

Child sandbox projection constraints, doctor tier split, the superseded chat root-only gate, autosync lock skips, and implicit user-config degradation now live in [startup-health-sandbox.md](startup-health-sandbox.md#sandbox-projection-policy). Current bounded nested chat behavior is recorded in [chat-backend D32](chat-backend.md#d32).

## Related Architecture

- [../architecture/state-system.md](../architecture/state-system.md) — state-system mechanism
- [../architecture/launch-system.md](../architecture/launch-system.md) — launch-system mechanism
- [../architecture/startup-pipeline.md](../architecture/startup-pipeline.md) — startup-pipeline mechanism
- [../architecture/sandbox-projection.md](../architecture/sandbox-projection.md) — sandbox policy mechanism
