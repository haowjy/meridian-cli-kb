# Decision Index

Decision pages own rationale, rejected alternatives, supersession, and revisit
conditions. Architecture pages own current mechanism. This page only routes to
the canonical records; it must not become a second explanation layer.

See [Decision Records](decisions/overview.md) for how to add or update a record.

## Current Cross-Cutting Decisions

| Date / ID | Status | Decision | Canonical record |
|---|---|---|---|
| 2026-09-25, D-native-only-history | Settled; implemented in unmerged combined PR #534 (`08499af0`, against `main`); #520/#526/#531 closed as superseded | Transcript reads, search and run facts use only the chat's native transcript; old runner history is dropped, not decoded (user: option C). Search is a disposable native-keyed FTS5 trigram projection with exact re-verification; `complete` means every in-scope source searched and no cap truncation. The metadata index calls the shared fold step. Run facts come from live per-attempt folds; incomplete facts drop only generic usage. Explicit archive apply captures the exact native snapshot before selection; native ZIPs omit retired runner streams. Browse exposes unbound rows without re-entry; late exact binding runs only in doctor/primary-launch repair. | [Native-only history](decisions/native-only-history.md) |
| 2026-09-24 (extended 2026-09-26), D-native-session-identity | Settled; implemented in unmerged combined PR #534 (`08499af0`, against `main`); #520/#526/#531 closed as superseded | Each cN binds one immutable `(harness, store, id)`; `--continue cN` resumes exactly it or refuses before creating rows. Unsupported forks/resumes and harness mismatches never fall back to resume-in-place or fresh launch. Entry is an exact target bound before exec; one binding rule and one drift rule govern identities. One runner pipeline concludes after teardown and writes one `run_boundary`. Legacy import is once-only, with late exact binding only through repair paths. | [Native session identity](decisions/native-session-identity.md) |
| 2026-09-24, D-native-source-argument-admission | Superseded by D-native-session-identity; never shipped | Adapter-owned raw-argument grammars dropped; only identity-overriding passthrough flags are refused. | [Native source argument admission](decisions/native-source-argument-admission.md) |
| 2026-09-22, D96 | Implemented and verified on feature branch `f7a18fd`; unreleased | Explicit `mars sync --force` takes over an eligible selected canonical self path and records installed ownership; default sync remains protective. Mars 0.14.1 retains the earlier refusal. | [Package management](decisions/package-management.md#d96-explicit-force-takes-over-a-selected-canonical-self-path-2026-09-22) |
| 2026-09-15, D95 | Current; shipped in Mars 0.13.2 | Journal new absent canonical outputs before apply; recover only exact regular bytes bound to the prior lock, and publish ownership only at finalization. | [Package management](decisions/package-management.md#d95-journal-new-canonical-writes-without-checkpointing-ownership-2026-09-15) |
| 2026-09-15, D94 | Current; shipped in Mars 0.13.2 | `[package]` contributes Mars-native agents and skills under `_self`; `.mars-src` wins, dependency renames remain authoritative, occupied-layer fallback stays, and default sync refuses unowned canonical destinations. | [Package management](decisions/package-management.md#d94-package-opts-agents-and-skills-into-_self-2026-09-15) |
| 2026-09-14, D-native-transcript-snapshot | Superseded by D-native-session-identity and D-native-only-history | Earlier stream-plus-snapshot choice. Runner history is neither read nor written; archives carry native snapshots of the bound key, and old runner-history members stay inert bytes. | [History storage](decisions/history-storage.md#d-native-transcript-snapshot) |
| 2026-09-14, D-history-index-initialization | Implemented and verified on feature branch; unreleased | Missing/outdated index setup gets one 15-second gate with post-lock recheck, durable genuine-failure suppression, and explicit manual retry. | [History storage](decisions/history-storage.md#d-history-index-initialization) |
| 2026-09-14, D-managed-startup-gate | Approved on feature branch; unreleased | Managed startup, stop, and failure cleanup share one lifecycle owner; an OpenCode conflict restart stays inside that gate. | [Launch process ownership](decisions/launch-process-ownership.md#d-managed-startup-gate-startup-stop-and-cleanup-have-one-owner) |
| 2026-09-14, OpenCode native commitment refinement to D76 | Approved on feature branch; unreleased | Explicit model selection is committed through launch-local config and native create/prompt shapes; native-agent conflict permits one verified restart. | [Model resolution](decisions/model-resolution.md#d76-harness-specific-model-ids-via-runnablepath) |
| 2026-09-11, D-history-file-authority | Partly superseded 2026-09-25 by D-native-only-history; SQLite-disposable rule current | Retained files and ZIPs are independently readable authority for retained history; SQLite metadata, previews and search are disposable one-way projections. Live content authority is the harness-native transcript. | [History storage](decisions/history-storage.md#d-history-file-authority) |
| 2026-08, D-tui-prompt-toolkit | Current | prompt_toolkit full-screen Application for the session-browse picker; hand-rolled, textual, and fzf rejected on POC + source-study evidence. | [TUI framework](decisions/tui-framework.md) |
| 2026-08, D-bare-continue-browse | Current | Bare `--continue` (no ref) canonicalizes to `session browse` before classification; supersedes the "intentionally excluded" note in D-argv-normalization-sentinel. | [Session initiation](decisions/launch-session-initiation.md#d-bare-continue-browse-bare---continue-canonicalizes-to-session-browse) |
| 2026-08, D-session-reentry | Current | Ops-owned re-entry decision (`Resume \| Fork \| Blocked`); advisory on rows, authoritative at Enter; fork-on-live never double-attaches. | [Session initiation](decisions/launch-session-initiation.md#d-session-reentry-ops-owned-re-entry-decision-resume--fork--blocked) |
| 2026-07, State Decision 3 | Current | Commit project identity as `[project].id` in `meridian.toml`; keep runtime state user-local. | [State](decisions/state.md#project-identity-in-meridiantoml-no-repo-local-state-decision-3-2026-07) |
| 2026-07, PR #422 | Current | Every store mutation acquires its stable lock, re-reads, mutates, and atomically publishes. | [State](decisions/state.md#concurrency-by-construction-mutate-under-lock-seams-over-convention-enforced-write-tiers-pr-422-2026-07) |
| 2026-07, D93 | Current | Engine version constraints: hard-filter-with-fallback for `requires-mars` / `requires-meridian`. | [Package management](decisions/package-management.md#d93-hard-constraint-with-fallback-engine-requirements-2026-07) |
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
| State layer | Identity, stores, atomicity, locks, reconciliation, typed rows | [decisions/state.md](decisions/state.md) |
| History storage | Transcript authority, disposable indexes, retention, restore, and native capture | [decisions/history-storage.md](decisions/history-storage.md) |
| Native session identity | Immutable chat→native key, exact-target entry evidence, one binding and drift rule, run-boundary outcomes, no discovery | [decisions/native-session-identity.md](decisions/native-session-identity.md) |
| Native-only history | Native transcripts as the only conversation source, old-data policy, search projection, run facts, retention | [decisions/native-only-history.md](decisions/native-only-history.md) |
| Native source argument admission | Superseded; identity-overriding passthrough refusal is what remains | [decisions/native-source-argument-admission.md](decisions/native-source-argument-admission.md) |
| Launch composition | Prepare/bind, policy replay, inventory, capability gates | [decisions/launch.md](decisions/launch.md) |
| Managed processes | Managed primaries, process scope, cancellation and cleanup | [decisions/launch-process-ownership.md](decisions/launch-process-ownership.md) |
| Waiting and sessions | Wait barriers, goals, output, continue/fork/from, session browse re-entry | [decisions/launch-session-initiation.md](decisions/launch-session-initiation.md) |
| Session-browse TUI | Picker-specific framework choice | [decisions/tui-framework.md](decisions/tui-framework.md) |
| Harness compatibility | Harness-specific launch history and superseded platform work | [decisions/launch-harness-compatibility.md](decisions/launch-harness-compatibility.md) |
| Startup / health / sandbox | Startup descriptors, doctor, bootstrap and projection policy | [decisions/startup-health-sandbox.md](decisions/startup-health-sandbox.md) |
| Package management | Mars compilation, targeting, ownership, hooks, recovery, engine constraints | [decisions/package-management.md](decisions/package-management.md) |
| Model resolution | Alias routing, candidate policy, prompting ownership | [decisions/model-resolution.md](decisions/model-resolution.md) |
| Workspace | Schema, path resolution, permission projection and migration | [decisions/workspace.md](decisions/workspace.md) |
| Telemetry | Envelopes, sinks, segments, querying and retention | [decisions/telemetry.md](decisions/telemetry.md) |
| Testing | Behavior-owned tiers and evidence gates | [decisions/testing.md](decisions/testing.md) |
| Chat backend | Protocol and command-layer history | [decisions/chat-backend.md](decisions/chat-backend.md) |
| Dev frontend | Portless/Vite exposure and launcher policy | [decisions/dev-frontend.md](decisions/dev-frontend.md) |
