# KB Change Log

Tracks structural changes to this knowledge base — new pages, reorganizations, and content migrations.

---

## 2026-05-22 — Model-resolution KB: remove deprecated Meridian compiler as current runtime authority

### Trigger

Code audit (p2178/p2179) found that several model-resolution KB pages still presented
`compile_launch_params()` / `compiler.py` as the canonical runtime authority, and
referenced a non-existent `resolve_harness_routing()` function. Current code routes
PRIMARY/SPAWN_PREPARE through `_resolve_policy_from_bundle()` → `bundle_adapter.request_and_resolve()`.
The compiler module is marked deprecated in its own docstring.

Also flagged: D73/D74 decisions described the Meridian-owned compiler as authoritative;
`resolve_harness_routing()` was cited as a live function; the 7-source harness cascade
was presented as the current runtime path. None of these match current code.

Human-review discrepancy: task prompt mentioned bundle schema v2 / mars 0.5.0, but
current code has `_SUPPORTED_BUNDLE_SCHEMA_VERSION = 1` and `_MARS_BUNDLE_MIN_VERSION = "0.4.8rc3"`.
No KB changes for those values — `architecture/mars-launch-bundle.md` (which already
reflected v1) left intact. Flagged for human review below.

### What changed

- `concepts/model-resolution/overview.md`: replaced `compile_launch_params()` pipeline
  diagram and two-pass compiler narrative with bundle-based pipeline diagram and
  current entry points; added legacy note for deprecated `compiler.py`
- `concepts/model-resolution/aliases-and-routing.md`: removed 7-source
  `resolve_harness_routing()` cascade (function doesn't exist in current code) and
  compiler-era harness-availability fallback section; added current Mars bundle
  routing section; kept AliasEntry/RunnablePath/ModelSelectionContext material
- `concepts/model-resolution/model-policies.md`: replaced compiler-centric
  implementation paragraph (removed `resolve_harness_routing()` cascade reference
  and `compiler.py` canonical claim); updated to bundle-based policy evaluation
- `concepts/model-resolution/vocabulary.md`: updated `Resolved policies` definition
  to reference `resolve_launch_policy()` / Mars bundle path
- `concepts/model-resolution/agent-profiles.md`: narrowed `resolve_model()` sentence
  to catalog-lookup context; updated Harness Availability Fallback section to remove
  non-existent function references (`_try_harness_availability_fallback()` etc.) and
  fixed broken `#harness-availability-fallback` anchor
- `concepts/config-precedence.md`: removed `_resolve_model_policy_overrides()` reference
  (function doesn't exist); replaced with bundle execution policy resolver description
- `decisions/model-resolution.md`: marked D73 and D74 as superseded by Mars
  launch-bundle routing, preserved text for historical context

### Human-review flag

> [!FLAG] **Needs human review** — version discrepancy: task prompt referenced
> bundle schema v2 / mars >= 0.5.0, but current source code has
> `_SUPPORTED_BUNDLE_SCHEMA_VERSION = 1` and `_MARS_BUNDLE_MIN_VERSION = "0.4.8rc3"`.
> `architecture/mars-launch-bundle.md` was left unchanged (it correctly reflects v1).
> If the codebase has been updated to v2/0.5.0 since this audit, update
> `bundle_adapter.py` constants and `architecture/mars-launch-bundle.md` accordingly.

### Validation

- `meridian kg check kb` → 0 errors, 5 warnings (all pre-existing flag blocks)
- `meridian mermaid check kb/concepts/model-resolution/` → 1 block valid

---

## 2026-05-19 — Spawn finalization cross-reference health pass

### Trigger

Structural health check after the spawn-exit-schema-split capture pass exposed one broken KB reference in the new spawn-finalization material.

### What changed

- `architecture/spawn-finalization.md`: removed a broken work-item link target and left the work-item reference as plain text so the page no longer points at a non-existent KB path.

### Validation

- `meridian kg check kb` after the edit still reports `0 errors`.

---

## 2026-05-16 — Session initiation semantics and PR #216 knowledge capture

### Trigger

Work item audit-dev-workflow-scope shipped unified session-initiation semantics (--fork/--fork-fresh split, bare flag inference, primary --from, argv normalization sentinel, four-layer content composition model). No KB pages existed for the four-mode initiation model or the identity-lock and argv-normalization design decisions.

### What changed

**New page:**
- `concepts/session-initiation.md` — Authoritative page for the four-mode session initiation model (--continue, --fork, --fork-fresh, --from). Covers: mode semantics and use cases, decision tree, four-layer content composition model (why Layer 3 is user-turn not system prompt), identity lock (fork vs fork-fresh), bare flag inference, argv normalization sentinel pattern, mutual exclusion matrix, surface coverage table, and implementation seam (`resolve_task_context_inputs`).

**Updated pages:**
- `decisions/launch.md` — Added Session Initiation section (2026-05 PR #216) with four new decisions: D-fork-identity-lock, D-prior-context-user-turn, D-argv-normalization-sentinel, D-from-fork-mutual-exclusion.
- `concepts/composition-pipeline.md` — Added session-initiation.md to Related Pages.
- `concepts/overview.md` — Added session-initiation entry.
- `index.md` — Added session-initiation.md to concepts section.
- `decisions.md` — Added chronological entries and updated Launch row in domain index.

### Validation

Ran `meridian kg check kb` and `meridian mermaid check kb` — see below.


---

## 2026-05-16 — Candidate-aware harness routing (PR #219)

### Trigger

Post-ship KB capture for PR #219 "Candidate-aware model harness routing" (merged 2026-05-16).

### What changed

**New decision records in `decisions/model-resolution.md`:**
- D76: Harness-specific model IDs via `RunnablePath` — `AliasEntry.harness_candidates` and `runnable_paths`; `harness_model_id` threading to bind time.
- D77: Explicit harness = force — `validate_harness_compatibility()` now checks only adapter availability, not model compatibility, for explicit harness selections.
- D78: Invalid override key skip — model-policy rules with all-invalid override keys silently skipped at profile load time instead of kept as phantom no-ops.

**Concept updates:**
- `concepts/model-resolution/aliases-and-routing.md`: added "Harness-Specific Model IDs" section covering `RunnablePath`, `AliasEntry.runnable_paths`, and `harness_model_id_for()`; updated `AliasEntry` and `ModelSelectionContext` code blocks with new fields.
- `concepts/model-resolution/overview.md`: added note in "What Gets Resolved" that `harness_model_id` may differ from `canonical_model_id` at the harness command boundary, with link to aliases-and-routing.

---

## 2026-05-16 — Bootstrap command, release CI fix, PR #184 carrier consolidation

### Trigger

Post-ship KB capture for PR #215 (bootstrap command), PR #213 (release CI semver fix), and PR #184 structural simplification waves.

### What changed

**New page:**
- `operations/bootstrap-command.md` — `meridian bootstrap --add` contract: init steps + guided session launch, difference from `meridian init --add`, rootless startup class rationale, idempotency.

**Navigation updates:**
- `operations/overview.md`: added bootstrap-command.md entry.
- `index.md`: added bootstrap-command.md to operations section.

**Decision update — `decisions/launch.md`:**
- Added Structural Simplification (2026-05, PR #184) section covering Wave 1 (four indirections removed), Wave 2 (session carrier consolidation: `SessionRequest` + `PrimarySessionMetadata` typed carriers replace `**kwargs`), and Wave 3 (LaunchSpec flattened to `ResolvedLaunchSpec`, compat bridges removed).

**Release CI note:**
- Release CI (`release-on-merge.yml`) now computes next version from highest stable `v*` semver tag, not just latest commit tag. Prevents stale lower-version tags from shadowing higher versions. (PR #213, 2026-05-14)

---

## 2026-05-16 — control_root/task_cwd split (PR #210)

### Trigger

Post-ship KB capture for PR #210 "Preserve control root and task cwd across launch" (merged 2026-05-13). The split of `execution_cwd` into `control_root` + `task_cwd` touched spawn/session record schemas and continue/fork authority logic.

### What changed

- `architecture/spawn-finalization.md`: updated `PreparedExecutionHandoff` schema to show `control_root`, `task_cwd`, and `execution_cwd` (legacy alias) as three separate fields with explanatory note.
- `architecture/claude-session-isolation.md`: fixed stale `source_execution_cwd` reference to `source_control_root` in the `--continue` flow diagram.
- `architecture/launch-system.md`: added `## control_root / task_cwd Split` section documenting the two fields on `LaunchContext`, `bind_launch_context()` behavior, env/prompt injection, and continue/fork authority.
- `decisions/launch.md`: added `D-control-root-task-cwd-split` decision record with rationale, alternatives rejected, and behavior description.

### Validation

- `meridian kg check kb/`: run after edits.
- `meridian mermaid check kb/`: run after edits.

---

## 2026-05-13 — model-invocable feature knowledge capture (PR #208)

### Trigger

Post-ship KB capture for the model-invocable agent visibility + native-skill prompt
suppression feature.

### What changed

**New decision records in `decisions/launch.md`:**
- D-model-invocable: filter seam choice (inventory prompt boundary vs catalog scan)
- D-model-invocable-vs-user-invocable: model-facing vs CLI-user surfaces are separate
- D-skill-doc-suppression: supplemental_documents gate on supports_native_skills

**Concept updates:**
- `concepts/composition-pipeline.md`: added "Skill Delivery: Two Channels, One Correct"
  section documenting the supplemental_documents vs --append-system-prompt split, and
  "Agent Inventory Prompt" section documenting the model-invocable filter.
- `concepts/model-resolution/agent-profiles.md`: added `model-invocable` frontmatter
  field documentation with Mars contract gap note.

**Lessons:**
- `lessons/arch-refactor-pitfalls.md`: added `prepare_prompt_payload()` or-chain
  pitfall — silently drops appended_system_prompt when projected_content has system_prompt.

**Open questions:**
- `open-questions/mars-feature-gaps.md`: added gap #6 — Mars does not formally specify
  model-invocable preservation for compiled agent profiles (haowjy/mars-agents#40).

**Index updates:**
- `decisions.md`: updated Launch domain row and added two foundational summaries.

---

## 2026-05-09 — Process-scope cleanup structural pass

### Trigger

Structural health check after process-scope cleanup KB capture. The burst added
process-scope architecture, launch-decision, and future-work material; the
deferred-work details were duplicated between architecture and the general
future-work page.

### What changed

**Structure:**
- Added `open-questions/process-scope.md` as the focused home for PROC-004 and
  PROC-007 follow-up details.
- Replaced the detailed PROC entries in `open-questions/future-work.md` with a
  short pointer, keeping the general future-work page as an index by domain.
- Kept `architecture/process-scope.md` focused on current mechanism and
  architectural summary, with a link to the open-question details.

**Navigation and cross-references:**
- Added the new process-scope open-questions page to `index.md` and
  `open-questions/overview.md`.
- Updated launch-decision and future-work cross-references to point at the new
  focused page.

### Validation

- Link and Mermaid validation rerun after edits; see the maintainer run report
  for command results.

---

## 2026-05-08 — Architecture-refactor KB structural pass

### Trigger

Post-write structural health check after PR #184 architecture-refactor lessons
were captured in the KB. The new lesson page and managed-primary cleanup
refinement needed navigation, bidirectional links, and stale-neighbor checks.

### What changed

**Navigation and cross-references:**
- Added `lessons/arch-refactor-pitfalls.md` to `lessons/overview.md`.
- Linked the PR #184 lesson page from the managed-primary lifecycle,
  launch-decision, health-check, and observability pages where the pitfalls are
  operationally relevant.

**Content health:**
- Updated stale neighboring summaries in `concepts/spawn-lifecycle.md`,
  `decisions.md`, `operations/troubleshooting.md`, and
  `operations/health-checks.md` so they match the refined invariant: Skip
  decisions send no signals; finalize-as-failed decisions may clean up tracked
  managed-primary runtime children when metadata safely identifies them.

### Validation

- `meridian kg check kb/`: 0 errors, existing FLAG warnings only.
- `meridian mermaid check kb/`: all Mermaid blocks valid.

---

## 2026-05-06 — Claude auth persistence and managed-primary reaper safety

### Trigger

Post-implementation KB capture for `primary-lifecycle-and-claude-auth-fixes`.
The implementation changed Claude overlay cleanup semantics and the managed
primary passive reaper boundary; those invariants needed durable documentation
outside the work directory.

### What changed

**Architecture:**
- Added `architecture/managed-primary-lifecycle.md` covering Codex/OpenCode
  launcher/backend/TUI roles, why launcher death is abnormal evidence, passive
  reconciliation limits, explicit `spawn cancel` cleanup, and missing/corrupt
  `primary_meta.json` candidate handling.
- Updated `architecture/claude-session-isolation.md` with auth/config state
  persistence from Claude overlays, known files (`.claude.json`,
  `.credentials.json`), concurrency policy, and best-effort failure behavior.

**Concepts and operations:**
- Updated `concepts/spawn-lifecycle.md` with `orphan_primary` semantics and the
  managed-primary passive-reaper invariant.
- Updated `operations/troubleshooting.md` with diagnosis and cleanup guidance
  for `orphan_primary`.
- Updated navigation in `index.md`, `architecture/overview.md`, and decisions
  pages.

### Validation

- KB link and Mermaid validation rerun after edits; see the run report for
  command results.

---

## 2026-05-06 — Chat normalization repair capture

### Trigger

Post-implementation KB capture for the `codex-dev-tailscale-no-response` work item. The existing chat normalization docs were stale against the repaired harness mappings and did not preserve the live-smoke operational lessons.

### What changed

**Architecture refresh:**
- Updated `architecture/chat/normalization.md` to reflect the repaired normalized chat contract:
  - assistant and reasoning text stay in `content.delta`
  - tool lifecycle is canonical `item.started` / `item.updated` / `item.completed`
  - exactly one `turn.completed` per actual turn
- Rewrote the per-harness mapping summaries for Claude, Codex, and OpenCode to match the current raw event shapes, including Claude aggregated `tool_use` / `tool_result`, Codex `item/agentMessage/delta` plus terminal `agentMessage` fallback, OpenCode part-role tracking and `message.updated` snapshot repair.
- Added the shared completion-dedupe rule to the normalization page and expanded source/test references.

**Pipeline contract refresh:**
- Updated `architecture/chat/event-pipeline.md` so the `content` family no longer claims command or file-change output stream kinds.
- Clarified that tool output stays on `item.*` payloads to avoid frontend double-rendering.

**Lessons capture:**
- Added `lessons/chat-normalization-repair.md` covering the root failure mode, compatibility-adapter framing, completion-dedupe lessons, replay obligations, and live-smoke cautions.
- Updated `lessons/overview.md` and `index.md` to include the new lessons page.

### Validation

- KB links and structure validated after edits; see the work-session run report for command results.

## 2026-05-06 — Codex approval routing and projected roots maintenance

### Trigger

Post-work KB maintenance after `codex-workspace-approval-routing`. The kb-writer update added current knowledge for managed-primary Codex approval routing, Codex remote TUI root mirroring, and `projected_roots`; the KB needed a structural and staleness pass.

### What changed

**Content health:**
- Verified `codebase/harness-adapters.md` already describes the implemented handler-based managed-primary approval flow and Codex remote TUI root mirroring.
- Updated `concepts/workspace-projection.md` and `architecture/workspace/overview.md` so `projected_roots` is documented as the current launch-spec seam, not a pending refactor.
- Updated `architecture/launch-system.md` to clarify that workspace roots flow through `run_params.projected_roots` / `ResolvedLaunchSpec.projected_roots`; `extra_args` remains user passthrough only.
- Updated OpenCode projection descriptions from parent-env suppression to additive `OPENCODE_CONFIG_CONTENT` merge behavior.
- Marked D-primary-approval defect bullets and D48 suppression language as implemented/past-tense while preserving the decision rationale.

**Structure:**
- No page split, merge, or rename needed. The topic already has focused concept, architecture, codebase, and decision pages with cross-links.

### Validation

- Link and Mermaid validation rerun after edits; see the maintainer run report for command results.

---

## 2026-05-03 — v2 full restructure

### Motivation

The v1 KB accumulated three structural problems over time:

1. **Decision entanglement** — architecture pages (especially `architecture/chat-backend.md`) carried inline decision rationale (D1–D29) rather than pure mechanism descriptions. Decisions blended with how-it-works content made both harder to read.
2. **Overloaded pages** — `concepts/config-and-context.md`, `concepts/model-and-agent-resolution.md`, `concepts/package-management.md`, and `architecture/chat-backend.md` each covered 3–6 unrelated topics. Splitting was deferred too long.
3. **Missing ecosystem coverage** — `meridian-web`, prompt packages (`meridian-base`, `meridian-dev-workflow`, `meridian-prompter`), and the mars-agents compiler had no KB pages. Agents were navigating these repos without reference.

### What changed

**Structure — domain reorganization:**
- `architecture/chat-backend.md` + two companion pages → `architecture/chat/` subtree (9 pages)
- `architecture/telemetry-system.md` → `architecture/telemetry/` subtree (4 pages)
- `architecture/workspace-system.md` → `architecture/workspace/` subtree (4 pages)
- `concepts/model-and-agent-resolution.md` → `concepts/model-resolution/` subtree (4 pages)
- `concepts/package-management.md` → `concepts/package-management/` subtree (5 pages)
- `concepts/config-and-context.md` → three focused pages: `concepts/config-precedence.md`, `concepts/context-resolution.md`, `concepts/workspace-projection.md`

**Decisions — extracted and relocated:**
- Inline decision rationale extracted from architecture pages into domain pages under `decisions/`
- New domain decision pages: `state-and-launch.md`, `chat-backend.md`, `model-resolution.md`, `package-management.md`, `telemetry.md`, `workspace.md`
- Existing `decisions/dev-frontend.md` retained and updated
- Root `decisions.md` converted to thin chronological index; each entry links to domain page anchor
- Architecture pages now contain one-sentence links to decisions rather than inline rationale

**Ecosystem — new domain:**
- Added `ecosystem/` domain (14 pages) covering repos that had zero KB coverage
- `ecosystem/meridian-web/`: overview, extension system, chat extension, connection protocol
- `ecosystem/prompt-packages/`: overview, meridian-base, meridian-dev-workflow, meridian-prompter, prompt-principles
- `ecosystem/jupyter-workbench/`: migrated from root `jupyter-workbench/` into ecosystem subtree

**Preserved:**
- All `codebase/`, `operations/`, `principles/`, `research/`, `lessons/`, `open-questions/` pages carried forward intact (minor link fixes only)
- All decision IDs (D-1 through D-70+) preserved as heading anchors in domain pages so existing cross-references remain valid

### Page count

v1: 55 pages across 11 domains  
v2: ~97 pages across 12 domains (+`ecosystem/`)

The increase comes from splitting 6 overloaded pages into 26 focused pages and adding 14 new ecosystem pages.

### Trigger

KB audit identified 11 pages with structural issues (decision entanglement, topic overload). Ecosystem gap analysis identified 4 repos with zero KB coverage. Rewrite scoped to fix the identified gaps; well-structured pages were not rewritten.

---

## 2026-05-05 — Post launch/startup structural health pass

### Trigger

Two implementation tracks shipped in close succession: launch resolution layer consolidation and CLI lazy-import startup optimization. The KB needed a structural check for stale entries, merge conflicts, oversized docs, and cross-reference integrity.

### What changed

**Structure — decision-page split:**
- Split oversized `decisions/state-and-launch.md` into focused pages:
  - `decisions/state.md`
  - `decisions/launch.md`
  - `decisions/startup-health-sandbox.md`
- Kept `decisions/state-and-launch.md` as a compatibility map so older links still land on a routing page.
- Updated `decisions.md`, `decisions/overview.md`, and `index.md` navigation to list the new focused pages.

**Cross-references:**
- Retargeted startup, sandbox, health-check, config-precedence, model-resolution, and package-management links to the new focused decision pages where the target concept now lives.
- Preserved compatibility links for broad legacy references to the combined state/launch page.

**Content health:**
- Removed stale `doctor-cache.json` from the root overview runtime tree after confirming the source tree only retained a pycache artifact.
- Flagged a telemetry bootstrap contradiction: startup docs describe process-seam-only telemetry installation, but current source still has operation-layer `setup_telemetry()` call sites in `src/meridian/lib/ops/spawn/api.py` and no `TelemetryBootstrap` class was found.

### Validation

- Merge-conflict scan: no conflict markers found.
- Link and Mermaid validation rerun after edits; see the maintainer run report for command results.
