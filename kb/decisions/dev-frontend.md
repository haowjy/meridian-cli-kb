# Decisions: Dev Frontend (`meridian chat --dev`)

These decisions govern the unified development setup for running `meridian chat --dev` with a live backend and hot-reload frontend. They are namespaced as **DF-D1** through **DF-D7** to avoid colliding with older domain-local decision IDs in [../decisions.md](../decisions.md).

Related: [Architecture: Dev Frontend](../architecture/chat/dev-frontend.md) | [Research: Vite, Portless, and Funnel](../research/vite-portless-funnel.md)

## Decision Map

| ID | Decision | Full rationale |
|---|---|---|
| DF-D1 | Auto-detect portless by default; raw Vite is fallback; `--no-portless` opts out. | [DF-D1](#df-d1-auto-portless-is-the-default) |
| DF-D2 | Network sharing is always explicit via `--tailscale` or `--funnel`. | [DF-D2](#df-d2-exposure-is-always-explicit) |
| DF-D3 | Route collisions fail closed; no auto-force or auto-prune. | [DF-D3](#df-d3-no-auto-force-no-auto-prune) |
| DF-D4 | Clear inherited `PORTLESS_*` env before spawning portless. | [DF-D4](#df-d4-scrub-inherited-portless-env) |
| DF-D5 | Raw Vite uses explicit allowed-hosts; portless owns its own allowed-host behavior. | [DF-D5](#df-d5-allowed-hosts-contract) |
| DF-D6 | Raw Vite and portless are separate `FrontendLauncher` strategies. | [DF-D6](#df-d6-frontendlauncher-abstraction-strategy-pattern) |
| DF-D7 | Exposure is modeled as `local \| tailscale \| funnel`. | [DF-D7](#df-d7-three-level-exposure-model) |

---

## DF-D1. Auto-portless is the default

`meridian chat --dev` auto-detects portless. This preserves the current dev workflow for contributors who have portless installed and expect a stable HTTPS URL. `--no-portless` is the explicit escape.

**Rejected:** explicit `--portless` opt-in — would break existing workflows.

## DF-D2. Exposure is always explicit

Network sharing must never change based on what binaries happen to be in `PATH` or because `PORTLESS_TAILSCALE=1` was exported in the shell. Exposure is a security boundary; opt-in must be visible in help text.

## DF-D3. No auto-force, no auto-prune

Route takeover can kill the wrong session. The safe default is fail closed with recovery instructions. `--portless-force` is the explicit escape.

The original implementation auto-retried with `--force` — that was identified as a security bug and removed in the refactor. Research confirmed: portless `--force` sends SIGTERM to the owning process (verified against v0.12.0 source), not just a metadata update.

## DF-D4. Scrub inherited PORTLESS env

A partial blocklist grows stale. Full-clear-then-set is the only approach that keeps the CLI surface contract honest across portless version changes.

Meridian sets only the env it controls after clearing inherited portless state: `VITE_API_PROXY_TARGET` and `VITE_WS_PROXY_TARGET`.

## DF-D5. Allowed-hosts contract

`allowedHosts: true` bypasses Vite's host-header check entirely and is unsafe (CVE-2025-24010 / GHSA-vg6x-rcgg-rjx6). For raw Vite: `RawViteExposure.allowed_hosts` carries an explicit list. For portless: Meridian does nothing — portless's HTTPS default plus `__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS` cover all cases.

Research basis: verified against Vite 6.0.9 security changelog and Vite 6.1.0 PR #19325 (`__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS` native env var).

## DF-D6. FrontendLauncher abstraction (strategy pattern)

Raw Vite and portless differ in readiness detection, URL derivation, subprocess structure, and error classification. Strategy-pattern launchers keep `DevSupervisor` lifecycle code free of mode branches and make each launcher independently testable.

## DF-D7. Three-level exposure model

Funnel changes public internet reachability — a qualitatively different trust level from tailnet-only. It must be represented explicitly in the normalized policy so allowed-hosts derivation, audit logging, and error messages can distinguish all three cases correctly.

## Validation Basis

These decisions are reflected in `src/meridian/lib/chat/dev_frontend/` and guarded by the unit/integration/smoke tests listed in [../architecture/chat/dev-frontend.md](../architecture/chat/dev-frontend.md#recommended-tests). External security and upstream behavior assumptions are summarized in [../research/vite-portless-funnel.md](../research/vite-portless-funnel.md).
