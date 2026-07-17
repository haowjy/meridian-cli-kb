# KB Change Log

Tracks structural changes to this knowledge base — new pages, reorganizations, and content migrations.

---

## 2026-07-17 — cbc-followup knowledge reconciliation (PR #444, closes #427)

### Trigger

PR #444 (cbc-followup, 21 commits over main): lock GC via validated-exclusive
sweep (#427), torn-tail-proof JSONL appends, session cleanup ordering invariant,
bounded first-writer wait for projection reads, spawn_aggregate extraction.
Follow-up to PR #422 (concurrency by construction, v0.3.34).

### What changed

**Stale content fixed:**
- `architecture/state-system.md`: "never unlinked" claims amended with GC-seam
  exception; `launch-boundary` and `gc.lock` added to lock layout; JSONL append
  section rewritten for torn-tail repair; `delete_published_spawn` attributed to
  `spawn_aggregate.py`; `mutate_scope_projection` marked private.
- `concepts/state-model.md`: three "never unlinked" claims updated with GC-seam
  exception.
- `principles/invariants.md`: "Stable Lock Inodes" invariant rewritten for GC
  seam and forbidden pattern.
- `decisions/state.md`: "Never-unlink" decision amended as "Stable lock inodes
  with validated-exclusive GC seam" covering PR #422 + #444; striping rejection
  and forbidden pattern documented.
- `decisions.md` (index): "never-unlinked" phrasing updated; #444 decisions
  added to the State domain row.
- `open-questions/future-work.md`: "Lock-Inode Accumulation (#427)" entry
  deleted (resolved).
- `architecture/spawn-finalization.md`: `delete_published_spawn` moved from
  `repository.py` row to new `spawn_aggregate.py` row in the module map.
- `lessons/thermo-nuclear-audit.md`: "never-unlinked" annotation updated with
  GC seam provenance.

**New content:**
- `decisions/state.md`: three new decision entries — torn-tail-proof JSONL
  appends, spawn_aggregate extraction, generic mutation framework rejected.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-17 — Probe-fix cycle knowledge reconciliation (post-PR #375/v0.3.34)

### Trigger

Follow-up probe-fix cycle on branch `task/pi-probe-fixes`: full real-Pi
re-run exposed issues #432-#435 (fixed), filed #438/#439/#440 (deferred).
Terminal publication architecture changed (publication barrier), Pi exit
classification became category-complete, Pi RPC inject semantics tightened,
fakes gained process-exit fidelity helpers.

### What changed

**Stale content fixed:**
- `architecture/completion-drain-coordination.md`: diagram and "Publication
  precedes cleanup" section replaced with "Publication is a manager-owned
  barrier" describing `_publish_terminal`, `resolve_terminal_outcome`, and
  per-spawn cleanup keying. Invariant 7 updated. Provenance updated.
- `architecture/drain-plans.md`: terminal publication paragraph rewritten
  for the manager-owned barrier (removed reference to drain loop owning
  publication).
- `open-questions/future-work.md`: "Publish-at-Deadline Redesign (#431)"
  rewritten as "Store-Level Publication Ordering (#431)" with resolved items
  marked. Three new deferred entries: typed terminal provenance (#438),
  cleanup telemetry categories (#439), Pi notification telemetry (#440).

**New content:**
- `decisions.md`: new foundational entry "Terminal publication barrier" for
  the `_publish_terminal` / `resolve_terminal_outcome` / per-spawn cleanup
  architecture.
- `architecture/pi-lifecycle.md`: two new drain-correctness safeguards:
  category-complete exit classification via `classify_outstanding_work()`
  and canonical subprocess-exit precedence via `PI_SUBPROCESS_EXIT_ERROR_PREFIX`.
- `codebase/test-determinism.md`: new "Pi Process-Exit Fidelity Helpers"
  section documenting `pi_process_exit_event()` and `write_pi_bash_record()`.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-17 — Post-drain-streaming-cleanup knowledge reconciliation (PR #375)

### Trigger

PR #375 (drain-streaming-cleanup, 21 commits) completed the post-drain-convergence
streaming cleanup: migration scaffolding deletion, drain composition-root
inversion, Pi drain test consolidation, resident rearm budget, Pi timeout
policy documentation, timeout carrier unification, shared reconciliation
decisions extraction, and windows-gate CI demotion.

### What changed

**Stale content fixed:**
- `decisions/model-resolution.md` D72: updated `ExecutionBudget` reference to
  `execution_policy.timeout` (ExecutionBudget was deleted in PR #375; timeout
  is now a single carrier in minutes).
- `concepts/model-resolution/model-policies.md`: same ExecutionBudget reference
  corrected.
- `open-questions/future-work.md`: agent overlay timeout item updated for
  new carrier name.
- `decisions/launch.md` D-win-ci: marked as superseded by POSIX-first; windows-gate
  CI demoted to non-blocking.
- `decisions.md` domain index Launch row: noted windows-gate as superseded.

**New content:**
- `decisions.md`: new foundational entry "Timeout and completion policy" covering
  single timeout carrier, opt-in ceilings, spawn-scoped rearm budget, and
  rejected alternatives.
- `architecture/drain-plans.md`: new sections "Rearm budget" (signal-gated,
  unlimited default, spawn-scoped, exhaustion behavior, Mars schema limitation)
  and "Timeout carrier" (single `execution_policy.timeout`, dual-carrier deletion
  rationale).
- `architecture/completion-drain-coordination.md`: provenance updated.
- `open-questions/future-work.md`: new "Streaming & Completion" section with
  two deferred items: #430 (resident nudge serialization + test seeding) and
  #431 (publish-at-deadline redesign, prefer_drain_outcome conflation,
  streaming_runner extraction).
- `codebase/test-determinism.md`: new section "PiDrainScenario Builder" and
  "Bounded History-Phase Polling" pattern.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-17 — POSIX-first supersedes Windows-first principle

The 2026-07-17 platform decision (POSIX-first: Linux/macOS supported; native
Windows never worked and is not planned) superseded the prior "Windows Is
First-Class" design principle. Doc-only repo sweep landed on branch
`task/docs-posix-first`; this entry reconciles the durable KB so a cold agent
no longer reads Windows as a supported product target.

- `principles/design-principles.md` §12 rewritten from "Windows Is
  First-Class" to "POSIX-First Platform Stance" (current truth; history now
  lives in the decision record, not the principle list).
- `principles/overview.md`: principle descriptor updated (Windows-first →
  POSIX-first platform stance).
- `overview.md`: "Windows is first-class" key property replaced with the
  POSIX-first stance.
- `codebase/guide.md`: platform package descriptor and the "Platform
  coverage required" guidance reframed from "consider Windows semantics" to
  POSIX-first (branches legacy/untested, not to be expanded).
- `decisions.md`: new foundational entry "POSIX-first supersedes
  Windows-first" recording rationale and the supersession.

Architecture/concept/lesson pages describing existing Windows code branches
(locking, process-scope, signals, path resolution) were left intact — they
document concrete reality the POSIX-first decision explicitly permits to
remain (legacy, untested, best-effort), not forward-looking support claims.

---

## 2026-07-16 — Thermo-nuclear audit knowledge capture

### Trigger

The thermo-nuclear audit (issue #389) completed. PR #390 (qi-docs-hygiene)
flagged three KB pages as stale against the current three-adapter,
`__status.json`-era architecture: `architecture/launch-system.md`,
`architecture/state-system.md`, `concepts/extension-system.md`. The audit
also produced durable decisions (rejected alternatives, peer-benchmark
verdicts) worth preserving.

### What changed

**Stale page fixes:**
- `architecture/launch-system.md`: "Four Driving Adapters" replaced with
  "Three Driving Adapters" (REST app path removed — `lib/app/` was
  archived). Mermaid diagram updated. Section 3 (REST App Path) removed;
  streaming-serve renumbered to section 3. Harness list corrected to
  include Cursor in the CWD policy table.
- `architecture/state-system.md`: split state layout corrected. Work items
  now live under the context work root (`<context.work>/<slug>/__status.json`),
  not `.meridian/work-items/`. Added staging directory to spawn layout.
  Work Item Store section updated for directory-based `__status.json`
  pattern.
- `concepts/extension-system.md`: HTTP app server marked as archived.
  `requires_app_server` commands return `app_server_archived`. Surface
  allocation table updated to show only active CLI/MCP surfaces. Core model
  diagram updated. RemoteExtensionInvoker section removed.
- `architecture/system-overview.md`: reduced from "three surfaces + one
  in-process adapter" to "two external surfaces". Mermaid diagrams updated
  to remove `lib/app/`, add Cursor and Pi to harness list, change "four"
  to "three" driving adapters.
- `architecture/overview.md`: app-server.md entry updated to note archived
  status.
- `index.md`, `decisions.md`: stale "four driving adapters" and "JSONL
  event stores" references updated.

**New lesson page:**
- `lessons/thermo-nuclear-audit.md`: two-source adversarial verification
  method, six rejected alternatives with specific evidence (SpawnOwnership,
  exactly-once reaper gating, wholesale TypeAdapter, shared read_json_object,
  ruff TID251, strict raw-event rejection), three defended peer-benchmark
  strengths, one dropped steal.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-16 — Drain-plan boundary extraction

### Trigger

The completion-drain convergence capture expanded the drain-plan and resident
completion material within the terminal-finalization page.

### What changed

- `architecture/drain-plans.md`: new owner for streaming `DrainPlan`
  composition and resident completion behavior.
- `architecture/spawn-finalization.md`: retained terminal-write authority and
  store-level finalization; links to the extracted drain-plan page.
- `architecture/overview.md` and `index.md`: indexed the new architecture page.

### Validation

`meridian kg check .` and `meridian mermaid check .` pass.

## 2026-07-15 — Completion drain convergence design capture

### Trigger

The `work:drain-convergence` design phase settled the shared Pi/resident
completion architecture and completed a Linux/POSIX crash/concurrency probe for
child-row publication.

### What changed

- `architecture/completion-drain-coordination.md`: new cross-cutting target
  architecture covering composition, evidence authority, Pi-private work,
  cleanup and persistence invariants, current divergences, and reversible phases.
- `architecture/atomic-child-row-publication.md`: new publication protocol and
  bounded evidence, including the unsafe sibling-stage layout and untested gates.
- `architecture/pi-lifecycle.md`: explicitly scoped as current checkout behavior,
  linked to the target, and updated for shipped unresolved-candidate tombstones.
- `decisions.md`, `architecture/overview.md`, and `index.md`: indexed the settled
  decision and new pages.

### Validation

Run `meridian kg check .` and `meridian mermaid check .` before committing.

## 2026-06-21 — Rootless commands, established project, test determinism capture

### Trigger

meridian-cli PRs #338 (rootless commands + startup model refactor) and #339 (test de-flake infrastructure). Cross-cutting capture from the session that designed and shipped these changes.

### What changed

- `architecture/startup-pipeline.md`: added `READ_ROOTLESS` to invocation class table; new section "Rootless Commands and Established Project" documenting `resolve_cli_project_root()`, `cwd_has_project_id()`, `exit_no_established_project()`, and the SystemExit-through-except-Exception footgun fix.
- `concepts/state-model.md`: new "Established Project" section with the established-project predicate, `cwd_has_project_id()`, and link to #341 migration direction (`.meridian/id` → `meridian.toml`).
- `concepts/config-precedence.md`: fixed stale project-root discovery description (removed ancestor walk-up steps 3-5 from #335; now correctly describes explicit / env / literal-CWD).
- `decisions/startup-health-sandbox.md`: new sections "Rootless commands" and "Established Project Detection" with decision rationale for #338.
- `codebase/test-determinism.md`: **new page** capturing the test-determinism infrastructure (`AsyncDeterminism` + `FakeClock`, `process_race.py`, advance-past-boundary principle, behavioral vs safety-bound timeout distinction, windows-gate flake pattern).

### Validation

`meridian kg check kb/` → pending (need kg in a project context).

---

## 2026-06-20 — Knowledge-layers KB guidance refresh

### Trigger

meridian-base commit `6fa9cc6` updated the `knowledge-layers` skill with four corrected conventions:
KB structure guidance now says read the local KB `AGENTS.md`; current-truth guidance
now distinguishes concept/architecture/operations pages from decision records;
vocabulary guidance no longer hardcodes a filename; ingest guidance now says update
log only for structural/major changes when the KB guide asks for it.

### What changed

- `research/llm-wiki-pattern.md`: replaced stale `kb-writer` agent references with
  `kb-lead`; reworked "When to Archive" → "When to Remove" section with the
  current-truth/page-type distinction; updated "When to Revise" to explain the
  decision-record exception; cross-referenced `AGENTS.md` for the full KB
  maintenance guide.

### Validation

`meridian kg check .` → 0 errors. `meridian mermaid check research/llm-wiki-pattern.md` → 1 block valid.

---

## 2026-06-20 — Mars `models prompting` canonical policy resolution (commit `1859a16`)

### Trigger
Post-ship KB capture for mars-agents commit `1859a16`: `mars models prompting` agent resolution now uses canonical launch policy path (`resolve_policy` with `PolicyInput`) instead of shallow overlay/profile/default token triage. Output reflects final `policy.routing` after model-policies, harness routing, and model clearing. Adds `--refresh-models` / `--no-refresh-models`; direct model-alias refs keep catalog-backed name resolution.

### What changed
- `decisions/model-resolution.md` — D86 expanded with canonical-policy rationale, shallow-resolver rejection, and `1859a16` correction note
- `concepts/model-resolution/aliases-and-routing.md` — retrieval section updated for launch-policy path, refresh flags, and `policy.routing` output semantics
- `concepts/model-resolution/vocabulary.md` — agent-first resolution term updated
- `architecture/mars-model-refresh.md` — `models prompting` listed as refresh consumer

### Validation
Run `meridian kg check kb` and `meridian mermaid check kb/concepts/model-resolution/` before committing.

---

## 2026-06-13 — Spawn lifecycle / reliability KB capture (PR #328)

### Trigger

PR #328 (`feature/spawn-capability-injection`) shipped six concurrent spawn-lifecycle
improvements: capability-gated spawn injection, reserve-before-prep crash recovery,
codex terminal semantics clarification, resident-done contract, sandbox bind-mount
context-root fix, and meridian-base v0.7.20 profile removal.

### What changed

- `decisions/launch.md`: added D-spawn-capability-gate and D-reserve-before-prep decisions.
- `concepts/spawn-lifecycle.md`: added Spawn Capability Gating section, Reserve-Before-Prep
  Crash Window section, clarified Resident Turn Boundaries (done = residency control, not
  status source), updated invariants.
- `concepts/harness-abstraction.md`: added Terminal Status Semantics section — per-harness
  terminal_outcome table, Codex `succeeded` = turn-completion doctrine.
- `architecture/sandbox-projection.md`: added Context-Root Existence Filtering section —
  codex bwrap missing-path abort, debug-only logging, git clone root exemption.
- `ecosystem/prompt-packages/meridian-base.md`: noted v0.7.20 removal of default
  orchestrator and subagent profiles.
- `decisions.md`, `index.md`: updated cross-references.

### Related inline docs

- `src/meridian/lib/launch/spawn_guidance.py` — capability gate + harness-templated contracts
- `src/meridian/lib/launch/context.py` — `_collect_context_projection_roots` existence filter
- `src/meridian/lib/harness/semantics.py` — `terminal_outcome()` per-harness classification
- `src/meridian/lib/streaming/resident_drain.py` — ResidentDrainCoordinator
- `src/meridian/lib/streaming/completion_nudge.py` — 270s completion nudge
- `src/meridian/lib/state/spawn_signals.py` — crash-only done/rearm signal files

---

## 2026-06-12 — OpenCode parent-session scope capture

### Trigger

Commit `2621f68c` generalized Codex main-thread protection into a shared primary event
scope and applied it to OpenCode's global event stream. Child OpenCode task sessions
remain visible in session logs but no longer complete, fail, clear signals for, or
supply reports for the parent spawn.

### What changed

- `concepts/harness-abstraction.md`: added the Primary Event Scope concept under the
  connection model.
- `codebase/vocabulary.md`: added the Primary event scope term.
- `index.md`, `codebase/overview.md`: refreshed the harness-adapters summary.

### Already-current items verified

- `codebase/harness-adapters.md` already documents `HarnessConnection.primary_event_scope`,
  Codex `threadId` vs OpenCode parent `sessionID`, the child-event persistence rule,
  and the Codex/OpenCode unscoped fallback difference.

### Related inline docs

The meridian-cli worktree capture updated `src/meridian/lib/harness/`,
`src/meridian/lib/streaming/`, and `docs/harness-integration.md` to replace the old
Codex-only wording with the shared parent-scope contract.

---

## 2026-06-06 — Post-#314/#317/#318 knowledge currency refresh

### Trigger

Batch of merged PRs (#314 prompt.py decomposition, #316 mars-agents 0.8.1/v3-only
bundle schema, #317 Codex thread-aware drain, #318 ambient-cwd task_cwd resolution)
left inline docs and KB pages stale. kb-lead reconciliation pass.

### What changed

**Inline docs (meridian worktree, `refactor/promptpy-decomposition`):**
- `launch/.context/CONTEXT.md`: resolution priority list rewritten to match
  `cwd.py` docstring exactly (steps 1, 2, 3, 3.5, 4); removed "or authority root
  fallback" per-step qualifiers that confused with-priority fallback vs
  separate priority level.

**KB — decisions:**
- `decisions/launch.md`: D-model-invocable + D-model-invocable-vs-user-invocable
  rewritten for current truth — Mars owns inventory rendering (bundle
  `prompt_surface.inventory_prompt`), Meridian embeds verbatim, Python-side
  `build_agent_inventory_prompt()` was deleted in #314.
- `decisions/launch.md`: **new** D-mars-owns-inventory — bundle-only inventory
  contract, no Python fallback.
- `decisions/launch.md`: **new** D-headless-claude-deny — headless Claude denied
  by default with 2026-06-15 driver, startup warning on override.
- `decisions/launch.md`: **new** D-agent-copy-key — `settings.meridian.agent_copy`
  key rename, auto-scaffold on init.
- `decisions/launch.md`: D-control-root-task-cwd-split env var updated
  (`MERIDIAN_TASK_CWD` → `MERIDIAN_TASK_DIR`) and system prompt block heading
  (`# Task Working Directory` → `# Source-edit directory`).
- `decisions.md`: summary entries updated for deleted `build_agent_inventory_prompt()`
  and new PR #314 decisions.

**KB — concepts/codebase/architecture:**
- `codebase/vocabulary.md`: "Delegation preference guidance" entry — removed
  reference to deleted `with_agent_inventory_guidance()`; now describes Mars-rendered
  bundle inventory.
- `codebase/harness-adapters.md`: removed `with_agent_inventory_guidance()` reference;
  replaced with Mars-rendered bundle paragraph.
- `concepts/harness-abstraction.md`: removed `with_agent_inventory_guidance()` reference;
  replaced with Mars-rendered bundle paragraph.
- `architecture/launch-system.md`: task_cwd resolution table rewritten — removed
  `--no-worktree`/`--worktree` flag rows, added ambient-cwd priority (step 3.5);
  updated `MERIDIAN_TASK_CWD` → `MERIDIAN_TASK_DIR`; updated `# Task Working Directory`
  → `# Source-edit directory`.
- `architecture/spawn-finalization.md`: `MERIDIAN_TASK_CWD` → `MERIDIAN_TASK_DIR`;
  system prompt heading updated.

### Already-current items verified
- Task CWD instruction injection paragraph in `launch/.context/CONTEXT.md` already
  described #318 contract (MERIDIAN_TASK_DIR, shell cwd is project root, cd or
  absolute paths).
- Codex thread-aware drain already documented in both `harness/AGENTS.md` and
  `harness/.context/CONTEXT.md`.
- v4 `launch_actions` removal already described in `architecture/mars-launch-bundle.md`.

---

## 2026-05-30 — Pi quiescence architecture refresh (PR #297)

### Trigger

PR #297 tightened Pi disk-backed quiescence and split the streaming implementation.
Session review found KB pages still naming retired lifecycle event-file surfaces and stale
Pi launch examples.

### What changed

- `architecture/pi-lifecycle.md`: current-only Pi extension split, disk-backed
  quiescence rule, PR #297 drain correctness constraints, removed historical event-file
  appendix from the live page
- `codebase/harness-adapters.md`, `architecture/launch-system.md`: Pi launch examples
  updated to `managed-bash` + `meridian-spawn-watch`
- `lessons/harness-integration.md`: Pi extension lesson now names disk-state authority,
  `PiDiskWatcher`, and `PiQuiescenceTracker` correctly
- `lessons/pi-rpc-quiescence-impl.md`: narrowed to still-current Windows/test-design
  lessons
- `architecture/pi-runtime/vocab.md`: current env vars and spawn-watch ownership

### Validation

Run `meridian kg check kb` and `meridian mermaid check kb/architecture/` before committing.

---

## 2026-05-23 — Mars model refresh and routing parity (PR #72)

### Trigger

Post-ship KB capture for mars-agents PR #72: `ModelsRefreshControl`,
`--refresh-models` / `--no-refresh-models`, `ensure_fresh` on launch-bundle,
probe refresh matrix, default `harness_order`, routing parity across models CLI
and launch-bundle, Cursor build-time `harness_model` effort resolution.

### What changed

- **New:** `architecture/mars-model-refresh.md` — focused catalog + probe refresh page
- `architecture/mars-routing.md` — Routing Parity (PR #72) section; cross-link refresh page
- `architecture/mars-launch-bundle.md` — `candidate_slugs` diagnostic; catalog refresh before build
- `architecture/cursor-harness.md` — build-time effort resolution; legacy Meridian projector labeled
- `concepts/package-management/sync-model.md` — symmetric `--refresh-models` / `--no-refresh-models` table
- `concepts/package-management/vocabulary.md` — `ModelsRefreshControl` term
- `index.md`, `architecture/overview.md` — navigation for launch-bundle, model-refresh pages
- `open-questions/mars-feature-gaps.md` — gap #7 Meridian refresh-models passthrough stub

### Validation

Run `meridian kg check kb` and `meridian mermaid check kb/architecture/` before committing.

---

## 2026-05-22 — Launch-bundle schema v2 follow-up (routing/scaffold detail pass)

### Trigger

Targeted follow-up after the first schema-v2 update: source audit showed a few
stale or ambiguous claims remained in launch-bundle/routing docs (old scaffold
placeholder names, `model_token` wording at harness boundary, and over-specific
enum-label examples in Mars routing diagnostics).

### What changed

- `architecture/mars-launch-bundle.md`:
  - Split "full Mars v2 schema" vs "Meridian-consumed subset" explicitly
  - Added top-level `agent_body` row and clarified `agent`/`agent_body` optionality
  - Corrected routing semantics:
    - `model_token` documented as selected routing/policy token
    - `harness_model` documented as the harness-boundary model ID when present
  - Listed current Mars routing diagnostic fields as Mars-owned and ignored by Meridian
  - Replaced stale scaffold placeholder table with v2 `scaffold_slots` keys:
    `completion_contract`, `context_prompt`, `user_prompt_file`, `context_files`,
    `prior_session_context`, `spawn_metadata` (all `###SLOT###`)
  - Tightened compatibility text to exact schema v2 + mars >= 0.5.0
- `architecture/mars-routing.md`:
  - Kept SelectionKind/MatchEvidence and RouteDecisionReport conceptual, but removed
    stale hard-coded enum-label examples
  - Clarified diagnostic labels are Mars-internal serialized strings, not Meridian contract

### Validation

Run `meridian kg check kb` and `meridian mermaid check kb/architecture/` before committing.

## 2026-05-22 — Bundle schema v2 / mars 0.5.0 KB update

### Trigger

`bundle_adapter.py` confirmed `_SUPPORTED_BUNDLE_SCHEMA_VERSION = 2` and `_MARS_BUNDLE_MIN_VERSION = "0.5.0"`, resolving the human-review flag left by the 2026-05-22 model-resolution KB pass. The KB still reflected schema v1 and mars 0.4.8rc3.

### What changed

- `architecture/mars-launch-bundle.md`:
  - Updated Bundle Structure header from "version 1" to "version 2, mars >= 0.5.0"
  - Updated `version` row to say currently `2`
  - Updated `routing` row to include `harness_model`
  - Added `routing` object subsection documenting the four Meridian-consumed fields (`model`, `model_token`, `harness`, `harness_model`) and noting that Mars-internal diagnostic fields (`selection_kind`, `match_evidence`, `route_trace`) are ignored by Meridian
  - Updated Compatibility section to reflect v2 as the required schema with exact-match validation
- `architecture/mars-routing.md`:
  - Added "Mars-internal DTO" clarification to the `RouteDecisionReport` section
  - Made explicit that Mars-internal consumers are `build::policy::*` (all Mars modules), not Meridian
  - Added note that Meridian reads only `model`, `model_token`, `harness`, `harness_model` from the bundle routing object
- `log.md` (this file):
  - Removed the stale human-review FLAG from the 2026-05-22 model-resolution pass
  - Removed the stale "Human-review note" from the structural follow-up entry above it

### Validation

Run `meridian kg check kb` and `meridian mermaid check kb/architecture/` before committing.

---

## 2026-05-22 — Model-resolution KB structural follow-up

### Trigger

Post-cleanup review found a few live stale references still lingering in the
decision and related pages:
`resolve_harness_routing()`, `resolve_policies()` as the routing-thread anchor,
and the old candidate-chain helper names in D75. A related link label in
`config-precedence.md` still read like the compiler was current authority.

### What changed

- `decisions/model-resolution.md`: rewrote D52/D53/D55 to point at the Mars
  bundle path / `resolve_launch_policy()` instead of `resolve_harness_routing()`
  or `resolve_policies()` as the routing anchor; replaced D75 helper-list
  references with current module-level references
- `concepts/model-resolution/model-policies.md`: changed "primary compilation"
  wording to "primary launch resolution" and retargeted the related-page link to
  the Mars bundle routing path
- `concepts/model-resolution/agent-profiles.md`: tightened the related-page label
  to "full bundle-based resolution pipeline"
- `concepts/config-precedence.md`: retitled the D73 related link text so it no
  longer reads as current compiler authority

### Validation

- `meridian kg check kb` → 0 errors, 6 warnings (all existing flag blocks)
- `meridian mermaid check /home/jimyao/.meridian/git/haowjy-meridian-cli-kb/kb/concepts/model-resolution` → 1 Mermaid block valid

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

Human-review discrepancy (at this audit point on 2026-05-22): task prompt mentioned
bundle schema v2 / mars 0.5.0, but inspected code then had
`_SUPPORTED_BUNDLE_SCHEMA_VERSION = 1` and `_MARS_BUNDLE_MIN_VERSION = "0.4.8rc3"`.
No KB changes for those values in this pass — `architecture/mars-launch-bundle.md`
(which then reflected v1) was left intact pending follow-up confirmation.

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
