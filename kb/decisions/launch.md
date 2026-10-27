# Decisions: Launch Composition and Policy

This page owns decisions about preparing, binding, and composing a launch.
Managed-process ownership, session/wait behavior, and harness compatibility now
have separate records so independently changing concerns remain discoverable.

## Decision Map

| Concern | Record |
|---|---|
| Prepare/bind and composition | This page |
| Managed-primary and process ownership | [Launch Process Ownership](launch-process-ownership.md) |
| Wait, goals, output, continue/fork/from | [Launch Session Initiation](launch-session-initiation.md) |
| Harness and platform compatibility history | [Launch Harness Compatibility](launch-harness-compatibility.md) |

## Launch Pipeline

### Single composition seam: all driving adapters compose through prepare/bind

**Decision:** All three driving adapters (primary CLI, spawn subprocess, CLI streaming-serve) compose through one seam in `launch/context.py`, now the prepare/bind pair below (originally the single `build_launch_context()` factory). No driving adapter re-derives policy, permissions, prompt, argv, or environment.

**Why:** Separate composition paths drift over time. Security properties (permission resolution, environment sanitization) and behavioral properties (prompt assembly, workspace projection) need a single place to audit. The 13 composition invariants are checkable in one code location.

**Alternatives rejected:** Per-adapter composition was the prior approach and caused drift bugs. Shared utility functions without a single factory were considered, but utility functions don't enforce the "sole callsite" property that invariants require.

See [architecture/launch-system.md](../architecture/launch-system.md) — architecture model and driving adapters.

---

### D-from-request-boundary: `--from` is resolved once at the request boundary; the request prompt stays raw (2026-10-21) {#d-from-request-boundary}

**Status:** settled (`work:session-browse-redesign`); shipped in
meridian-cli PR #714 (merge `471dfbc59`, 2026-10-21).

**Decision.**
- **Resolve once, at the boundary.** A `--from` ref is resolved into a `ContextRef`
  (chat id, history id, rendered prior context) in exactly two places, both through
  `ops/spawn/context_ref.resolve_context_ref`: `cli/primary_launch.py` for primaries
  and `ops/spawn/prepare.py` for spawns. Requests carry only `context_refs`. Launch
  renders them (`core/context_ref.render_context_refs`) and never resolves a ref.
  A primary's lineage edge is persisted from that same result
  ([D-lineage-conversation-identity](state.md#d-lineage-conversation-identity)).
- **The request's `prompt` is the caller's raw prompt, always.** Every prepare
  composes from the request's inputs (`prompt`, `context_refs`, `reference_files`,
  template variables). The composed text lives only on the output side of prepare
  and bind, never written back into the request.

**Why.** Both rules give one fact one owner. Before, the CLI kept `--from` as
strings, and launch, each harness adapter (`context_from_payload`) and several
fallbacks resolved them again, and recording a primary's `--from` as lineage
would have added one more resolution beside them. The prompt had two meanings, raw or already composed,
told apart by a `prompt_is_composed` flag. The background worker re-prepares the
persisted request; when the request already held composed text, `spawn --bg --from`
and `spawn --bg -f` rendered the prior context and references twice. That bug was
on `main` (v0.11.4) and is fixed by the same change.

**Rejected:** keeping the flag and guarding the worker path. It would leave two
meanings of `prompt` for the next re-prepare path to trip over.

**Compatibility.** A background request written by an older build holds composed
text in `prompt`; if this build's worker picks it up, the context is composed a
second time. That needs an upgrade inside the gap between writing a request and
the worker starting, which is short. The reverse (a 0.11.4 worker reading this
build's request) is untested and could drop a `--from` context, because 0.11.4
defaults the missing flag to "composed" (meridian-cli `docs/upgrading.md`).

**Provenance:** `work:session-browse-redesign` `pr-body-session-browse.md` part 7;
root cause `spawn:p14151`; streaming-harness probe `spawn:p14155`.

---

### Prepare/bind split: expensive resolution separated from cheap materialization

**Decision (2026-05, launch-primary-context-dedupe):** `build_launch_context()` is decomposed into two public phases: `prepare_launch_surface()` (expensive: model/harness/skill resolution, composition, prompt assembly) and `bind_launch_context()` (cheap: env, cwd, spec, argv, permissions given a spawn ID and report path). The existing `build_launch_context()` entry point was preserved as a backward-compat wrapper.

**Amended (2026-10, cleanup phase 1, launch#9):** The wrapper and its production fallback were deleted. `PreparedLaunchSurface` is mandatory for `run_harness_process()`, and the `DIRECT` composition surface is test-only (`tests/support/launch.py`); production policy resolution rejects it. Keeping a second one-call path meant two composition routes to audit, and nothing in production needed it. See [Codebase cleanup](codebase-cleanup.md).

**Why:** The primary CLI path previously called the full factory twice — once for `--dry-run` preview, once for real execution. The split makes the prepare-once/bind-N pattern explicit and enforced.

**`PreparedLaunchSurface`** — public frozen dataclass carrying everything spawn-stable
(resolved request, harness, composition warnings, agent inventory, and a typed empty
capability placeholder). It explicitly excludes runtime-only values (spawn ID, report
path, env, argv, permissions). The exclusion is the invariant — bind-phase code cannot
accidentally depend on preparation-phase values.

**`RuntimeBindings`** — frozen dataclass for bind-phase inputs: `spawn_id`, `report_output_path`, `runtime_work_id`, `chat_id`, `forked_harness_session_id`, `plan_overrides`, `dry_run`.

See [architecture/launch-system.md](../architecture/launch-system.md) — Prepare/Bind Split.

---

### CatalogSession: operation-scoped Mars result cache

**Decision (2026-05):** `CatalogSession` is an operation-scoped collaborator holding a `MarsResultCache`. Created once per launch operation, discarded at operation end. `prepare_launch_surface()` receives one from its caller.

**Why:** Without this, `resolve_policies()` and `prepare_launch_surface()` called `mars models list` independently on every invocation (~100ms subprocess per call, 3–5 redundant calls per spawn).

---

### D-control-root-task-cwd-split: Split execution_cwd into control_root + task_cwd

**Decision (2026-05-13, PR #210):** `LaunchContext`, spawn records, and session start events carry two distinct path fields — `control_root` (project config/authority root, anchored to `meridian.toml`) and `task_cwd` (task's intended working directory, `None` when it coincides with control_root).

**Why:** `execution_cwd` was overloaded — one field served double duty as both config authority and task directory. When a spawn originates from a nested directory these concerns diverge: config authority stays at repo root while the agent operates in the subdirectory.

**Behavior (updated 2026-10, meridian-cli v0.14.3):**
`bind_launch_context()` sets `MERIDIAN_TASK_DIR`, renders the entire single capability
section with the final control root, logical task cwd, work context, and run identity,
then re-projects the composed prompt. It does not compose or append a separate
task-directory block.

**continue/fork authority:** `ClaudeSessionAccessSource` carries `source_control_root` and `target_control_root` instead of `source_cwd`. Legacy records fall back to current launch `control_root`.

**Extended by PR #248 (2026-05-22):** Full authority/task domain architecture with worktree-based task_cwd resolution. See [spawn-cwd-worktree-anchor.md](spawn-cwd-worktree-anchor.md).

---

### Resolve-before-persist for the streaming-serve path

**Decision:** The CLI streaming-serve path composes (`SpawnApplicationService.prepare_spawn()`) before creating the spawn row. The spawn row is created only if resolution succeeds, preventing phantom active rows when resolution fails.

**Now also true of the spawn subprocess path:** `build_create_payload()` composes before the row is reserved. The background worker recomposes after the row exists because it has no in-memory surface; a failure there finalizes the row as a launch failure rather than leaving a phantom.

---

### Layered finalization ownership (R4-R6 refactor)

> Supersedes the earlier `pre_launch_complete` boolean flag approach.

**Decision:** Finalization responsibility is divided across three concentric function scopes:
1. **Runner layer** (`execute_with_streaming()`): Owns terminal finalization from function entry. Uses sentinel locals (initialized to `None`, set during setup) so `finally` handles partial-setup failures.
2. **Helper layer** (`launch_prepared_spawn()`): Shared pre-run setup for both foreground and background paths. Owns `launch_failure` finalization for pre-execution exceptions.
3. **Surface backstop layer**: Last-resort catch ensuring no spawn is left permanently stuck in `queued`.

**Why structural layers over boolean flags:** Each layer is defined by function scope, not flag state. `launch_prepared_spawn()` eliminates code duplication between foreground and background paths.

**`complete_spawn()` idempotency** enables safe layered catches — if the record is already terminal, the call is a no-op. Known issue #153: concurrent finalizers silently drop the second set of metrics.

### D-one-launch-attempt: harness-turn failures finalize without replay

**Decision (2026-09, issue #542):** Each Meridian harness turn gets one
launch-level attempt. Terminal outcomes, transport and startup failures, timeouts,
guardrail failures, identity failures, cleanup failures, cancellation, and generic
subprocess failures finalize that turn. Meridian does not automatically launch the
harness again.

A runtime audit found 22 retrying lineages among 10,247 observed lineages. Twenty-one
lineages failed, and 63 of 64 later attempts failed. Seventeen Claude lineages hid
the causal quota result behind a reused-identity collision; four Cursor lineages
repeated the same authentication failure. The sole recovery was an old Codex run
whose local SQLite runtime failed before session creation. One pre-session recovery
does not justify a cross-harness replay policy, especially when a retry can repeat
model, tool, child-process, or native-create side effects.

Exact typed terminal outcomes remain the causal final result. Cleanup,
cancellation handling, process-scope ownership, and post-exit identity observation
still run for the one attempt; they prevent orphans and preserve evidence rather
than authorize replay. Bounded adapter-local polling may remain when it is
demonstrably idempotent and completes before the turn is submitted, such as safe
readiness checks or owned port reselection.

**Rejected:** a typed safe-startup exception based on unsubmitted-turn,
quiescent-teardown, and unmaterialized-identity proof. That design required retry
policy/configuration, permits, native-run rearming, replay classification, backoff,
and artifact rotation. The proof narrowed the unsafe cases but did not establish a
valuable generic recovery mechanism. See the [launch system](../architecture/launch-system.md#one-launch-level-attempt)
and [native session binding](../architecture/native-session-binding.md#one-attempt-causality).

---

### D-primary-approval: Managed-primary Codex approval routing

*2026-05-05*

**Decided:** Managed-primary Codex sessions use `InteractiveHandler` (passive — surfaces requests as events). Spawn/subagent paths keep `AutoAcceptHandler`. The blanket `_primary_observer_mode` rejection of all server requests is removed.

**Why:** In managed-primary mode, the app-server routes `requestApproval` to the turn owner (Meridian), not the TUI. The TUI cannot answer approval requests; Meridian must handle them via the interactive handler.

**Three defects fixed:** (D-PA1) blanket rejection removed, (D-PA2) managed-primary uses InteractiveHandler, (D-PA3) `_reject_confirm_mode_approval_request()` conditioned on `no_runtime_hitl`.

---

### D-model-invocable: Filter at inventory prompt boundary, not at catalog scan

**Decision (2026-05, PR #208; rewritten 2026-06, PR #314):** `model-invocable: false` in agent profile frontmatter is enforced by Mars at inventory render time. The catalog scan returns all profiles regardless.

**Why (2026-06 rewrite):** Inventory rendering is now exclusively Mars-owned — Meridian passes through the bundle's `prompt_surface.inventory_prompt` verbatim. No Python fallback renderer.

**Original Why (still holds):** `scan_agent_profiles()` has multiple consumers. Filtering there conflates model-facing visibility with general catalog availability. The inventory prompt is the only model-facing boundary.

---

### D-model-invocable-vs-user-invocable: Two separate visibility surfaces

**Decision (2026-05, PR #208):** `model-invocable` controls only the model-facing agent inventory prompt. It does not restrict explicit user invocation via `meridian spawn -a <name>`.

**Why:** A deprecated or internal agent that shouldn't appear in the model's menu may still be explicitly invoked by a user who knows what they're doing.

---

### D-mars-owns-inventory: Mars renders agent inventory; Meridian preserves its semantics

**Decision (2026-06, PR #314):** The agent inventory prompt is bundle-only. Mars renders `prompt_surface.inventory_prompt`. There is no Python fallback renderer.

**Calibration (2026-10, meridian-cli v0.14.3):** Meridian passes the
inventory through verbatim. Stale producer guidance is fixed at its Mars owner (see
mars-agents #190), never by exact-string filtering in the consumer.

**Why single-owner:** A dual-renderer creates divergence risk. Mars already owns the `.mars/` directory schema, profile scanning, and agent-copy; inventory rendering is a natural extension.

**Resume/snapshot persistence:** `bundle_inventory_prompt` on `LaunchPolicySnapshot` so inventory survives without re-calling mars.

---

### D-bundled-meridian-skills: Meridian-owned operations guidance versions with the CLI

**Status:** shipped in meridian-cli v0.14.3–v0.14.6 (PRs #767, #770, #771). meridian-base
0.12.0 deleted the package copies it replaced (`notify`, `session-mining`,
`meridian-privilege-escalation`, and the work-item mechanics of `work-artifacts`).

**Decision:** Guidance whose subject is Meridian itself ships inside the Meridian wheel
(`src/meridian/skills/<name>__meridian/`), not in a prompt package. The set is
`escalation`, `handoff`, `notify`, `sessions`, `work`, and `prompt-authoring` (agent and
skill file formats). Every launch gets a `# Meridian skills` section with one trigger
line per skill, and agents read a skill or one of its resources with
`meridian skills NAME [RESOURCE]`. Package skills keep Mars and harness-native loading.
The colocated contract (frontmatter, naming, trimming rule) is
`src/meridian/skills/AGENTS.md`; user docs are `docs/agent-profiles.md`.

Bundled skills may name the commands and flags of the release they ship in, since they
version with it. They may not depend on package-specific agent or skill names. The
capabilities section stays the short pointer; the skill carries the judgment.

**Why:** Moving Meridian syntax out of reusable packages (see
[D-capability-command-mapping](#d-capability-command-mapping)) removes runtime coupling,
but that judgment still needs an owner. Shipped with the binary, it changes in the same
release as the commands it explains, so a package needs no `requires-meridian` bump to
stay accurate.

**Naming (current choice):** every bundled skill is `<name>__meridian`, copying Mars's
`name__source` spelling for cross-source collisions. Recalling that Mars puts the source
last, the owner settled on the suffix ("oh... isn't it `meridian__notify` ...? or is
meridian added at the end?"). Rejected: reserving the plain names (a generic name like
`notify` would keep blocking package skills of that name); `meridian__notify` (breaks the
Mars convention and sorts by source instead of name); renaming only on collision (the
name would change from one profile to the next).

**Gating (current choice):** a bundled skill's optional `mode: primary` frontmatter shows
its trigger only to profiles whose `mode` is `primary`, including a primary-mode profile
launched as a spawn. `handoff`, `notify`, `work`, `escalation`, and `prompt-authoring`
are primary-only; `sessions` reaches every agent. The `meridian notify` capability line
uses the same gate. When the lead proposed a new `audience` field, the owner pointed at
the field that already existed ("uh... we have an audience field...?", then "we have
mode: primary"). Rejected: the `spawn-only` marker v0.14.3 shipped, which hid triggers
only from agents that cannot delegate; reviewers and `@kb-lead` can delegate but report
to their caller, not the human, so notify and handoff reached them. Also rejected: a new
`audience` field (a second word for `mode`).

**Reading looks like loading (current choice):** `meridian skills NAME` prints the
skill under the same `# Skill:` header and folder line an injected skill gets, through
the shared renderer, so an agent treats a read skill as loaded. The owner's words: "read
form permanent for agents I think... it should basically be the same as activating a
normal skill?" A general `meridian skill <name>` for package skills was scrapped
(meridian-cli #766).

**Lesson:** the first cut of the moved skills trimmed flag syntax and took the judgment
with it: escalation's symptom-to-fix scenarios (try `--network allowlist` before
`--network allow` for a port bind; ask the human before `--yolo`), notify's rule to raise
an uncertain call before more work builds on it, sessions' advice to mine before
compaction. A review against the meridian-base originals restored it in v0.14.5. Compare
a trimmed skill against its original before shipping it.

#### Installing bundled skills through Mars (Mars side shipped, Meridian side inert)

**Status:** partly done. Mars 0.20.0 and later have `mars provide`,
`.mars/provided.lock` and `[package] uses-providers`. Meridian's code that calls
`mars provide` is merged (meridian-cli #779) but never takes effect yet: Meridian pins
Mars 0.18.4, which has no `mars provide`, so every launch reads the skill copies in the
wheel. Provided skills go live when
Meridian's pinned Mars reaches 0.20; see [the Mars 0.20 upgrade](#upgrading-meridians-mars-to-020-split-from-provided-skills).
The colocated contracts are mars-agents `src/provided/.context/CONTEXT.md` and
`docs/internals/lock-file.md` (canonical ownership), and meridian-cli
`src/meridian/lib/bundled_skills.py`.

**Decision (the owner's):** bundled skills install into `.mars/skills/` beside package skills, so there
is one home for skills, Mars stays the only writer of `.mars/`, and a package agent can
list `prompt-authoring__meridian` under `skills:`. Rejected: keeping bundled skills
wheel-only and teaching Mars a host-suffix rule (smaller, but two homes for skills).

The rule behind every edge case is that **a host never changes committed state**:

- `mars provide meridian@<version>=<package dir>` copies the wheel's `skills/` entries
  into `.mars/skills/` and records them in a machine-local `.mars/provided.lock`.
  `mars.toml`, `mars.lock` and harness folders are byte-identical whether or not Meridian
  ever ran, and `.mars/` is gitignored by Mars convention, so teammates on different
  Meridian versions never churn a committed file. A skill a newer Meridian drops is removed
  from `.mars/skills/` on the next provide.
  Provided skills are never projected into harness folders, since a native copy would
  load without Meridian's `mode` gate. Plugin builds never contain them.
- A package that lists `<x>__meridian` declares `[package] uses-providers = ["meridian"]`
  in its `mars.toml`.
  Only a declared provider's names can be satisfied by provided skills, and they warn
  instead of failing where the provider is absent. An undeclared ref is `SkillNotFound`
  on every machine, even one where the skill is installed, so a ref's validity depends on
  the package, never on what the author happens to have installed. Rejected: a suffix rule
  where any unresolved `x__p` warns (a typo shaped `foo__bar` would warn instead of fail);
  the field names `providers` (reads as "what this package provides") and
  `expects-providers`.
- Meridian runs `mars provide` at every primary and spawn launch, before
  `meridian skills NAME`, and before `meridian mars` commands that resolve refs, then reads
  the `.mars/` copy for names Mars reports installed and the wheel for everything else.
  Any other outcome (busy lock, interrupted sync, older Mars without the command, error)
  falls back to the wheel. When Mars refuses the bundle (an invalid skill, a
  `provided.lock` from a newer Mars), Meridian prints Mars's reason, but only after the
  command it ran has succeeded: Mars reports a broken `mars.toml` the same way, and a
  command that fails on it already shows the message. Rejected: refreshing at primary
  launch only, which would let a spawn load an older body than its Meridian.
- meridian-flow, a separate app whose agents have no shell, imports packages as snapshots
  and cannot run `mars provide`, so packages it imports (creative-writing-skills, meridian-base) list no `*__meridian` skills.

Three choices made during implementation are the work lead's, not the owner's (current
choices):

- **`mars provide` answers `up-to-date` only when a re-plan would write nothing.** Its
  fast path runs before taking the sync lock, so parallel spawns and launches during a long
  sync do not all report `busy`, and it hashes each installed tree against the source.
  Existence is not ownership: a sync that crashes after replacing a recorded directory and
  before publishing `mars.lock` leaves the record intact and foreign bytes on disk, and an
  existence check would call that current while Meridian read the foreign copy. Measured
  cost with Meridian's six-skill bundle is about 2 ms per call.
- **A provided skill's availability comes from disk, its ownership from the record.**
  Mars's readers hold two name sets: a ref is satisfied only while the recorded skill is
  actually installed, and a destination named in `provided.lock` counts as Mars-owned, so a
  package item of the same name replaces it on every machine.
- **Gating moves to `metadata.mode` with the Mars 0.20 upgrade.** Bundled skills will carry
  `metadata: {mode: primary}` instead of a top-level `mode:` key, because Mars 0.20 warns
  on unknown top-level skill keys and `mars provide` would report one warning per gated
  skill on every applied run. The skills in the wheel still use top-level `mode:`, which
  Mars 0.18.4 never reads; the change ships in the same release as the Mars 0.20 pin. The
  gate's meaning is unchanged.

**Release (current choice):** the Mars side shipped in mars-agents 0.20.0 together with
schema v1, which rewrote the skill readers, ref resolver and sync planning that provided
skills change. The owner,
agreeing to the lead's proposal: "yeah, I think going on top of 0.20.0 makes sense."
Rejected: landing first in 0.19.0 (schema v1 would have to merge it through its rewrite).
Meridian's side merged before its Mars pin moved; see the split below.

**Provenance:** `work:package-transferability`, `chat:c9186`, `chat:c9297`.

#### Upgrading Meridian's Mars to 0.20 (split from provided skills)

**Status:** not done. The pin, the upgrade notes and the `metadata.mode` change are
prepared on meridian-cli's local branch `feat/mars-020`, unpushed.

**Decision (the owner's, 2026-10-26):** the Mars 0.20 pin ships in its own meridian-cli
PR, not with provided skills. Pinning it hands every Mars 0.19 and 0.20 break straight to
Meridian users who upgrade:

- an overlay `[agents.<name>] approval = "yolo"` and `autocompact` in a `model-policies`
  override make `mars.toml` fail to load;
- a `model-policies` override now beats a profile's top-level effort, sandbox, approval
  and network, so the `coder` profile drops from `xhigh` to `high` or `medium`;
- `mars sync --frozen` fails once after the upgrade, until one plain `mars sync`.

The prompt packages this project publishes (meridian-base, meridian-dev-workflow and the
others) were also still in the pre-v1 schema. Hearing this, the owner
asked "hmmm... uh... should we really be shipping the mars upgrade lol", then agreed to
the lead's split: "yeah split it that way". Meridian therefore carries the provided-skills
code while its bundled Mars stays 0.18.4, and skills keep loading from the wheel.

Before the upgrade PR: migrate the prompt packages to schema v1, and decide the effort
`coder`'s `model-policies` should resolve to. The upgrade notes come from running each
Mars 0.19 and 0.20 break against Meridian, and belong in meridian-cli `docs/upgrading.md`
with the PR.

**Provenance:** `work:package-transferability`, `chat:c9297`.

---

### D-continue-replays-recorded-launch-contract: same-session continue is not live policy recomputation

**Decision (2026-07):** Primary `--continue` and spawn `--continue` replay the recorded launch contract for tracked source sessions/spawns instead of resolving a new launch from current CWD/env/config.

**Behavior:** Continue inherits source work attachment and task directory. Absence of source work is also recorded state: exact continue suppresses the caller's ambient `MERIDIAN_ACTIVE_WORK_*`. When a `LaunchPolicySnapshot` exists, continue reuses the snapshot's model, harness, agent, skills, execution policy, tool policy, terminal surface mode, env, and passthrough args. Explicit agent opt-out is authoritative — replay preserves `agent=None` without falling back to configured defaults.

**Why:** Continue means re-entering the same conversation. Recomputing from current CWD/config/env can change the system prompt, workspace projection, or prompt-cache key.

**Override boundary:** Continue preserves the recorded contract for ordinary replay. `--model` is the explicit target-constrained exception: primary and spawn continuation accept it and warn, then Mars revalidates current targets and exclusions under the same harness. Other policy-changing options remain rejected; changing identity belongs to `--fork`, `--fork-fresh`, `--from`, or a fresh session.

**Session ID authority:** Continue/fork consume `ResolvedSessionReference.authoritative_harness_session_id`. See [session-reference-resolution.md](session-reference-resolution.md).

---

### D-headless-claude-deny: Headless Claude denied by default with 2026-06-15 driver

**Decision (2026-06, PR #314):** Meridian denies headless Claude by default. `[spawn] deny_headless_harnesses = ("claude",)` ships in auto-scaffolded `meridian.toml`. Users who override receive a startup warning.

**Why deny-by-default:** Headless Claude `-p` is an Anthropic API surface with announced deprecation. Meridian cannot fix a broken headless path — the deny default gives users a clear Meridian error with recovery guidance rather than an opaque API rejection.

---

### D-agent-copy-key: Mars `settings.meridian.agent_copy` key renamed; auto-scaffolded

**Decision (2026-06, PR #314):** The Mars config key was renamed to `[settings.meridian.agent_copy] harnesses = ["claude"]` (was `[settings.agent_copy]`). Agent copy is a Meridian concept, so it belongs under `settings.meridian`.

**Auto-scaffold:** New projects get the key written into initial `mars.toml`. Meridian only reads the new key path.

---

### D-skill-single-path: Loaded skills are composition documents on every harness

**Status:** Shipped in v0.10.23 (PR #661, merge `71bec25fe`).

**Decision (2026-10, PR #661):** Both composers (`_build_spawn_composed_launch_content()` and `_resolve_primary_projection()`) always add `compose_skill_prompt_documents(loaded_skills)` to `supplemental_documents`. Projection places them in the system prompt after the agent profile, grouped principle → guardrail → reference. There is no harness-capability gate and no second channel. `system-prompt.md`, which Claude reads through `--append-system-prompt-file`, is written from `run_params.appended_system_prompt`. That is the full prompt the harness receives, including additions made after projection such as the source-edit-directory instruction.

**Why:** Four sites with four conditions had been adding and removing skill text. On Claude, Codex and OpenCode the only one that actually delivered skills was the bind-time re-append in `_reproject_context_env_at_bind`. It rebuilt the appended prompt from scratch, and `system-prompt.md` was written from the projected prompt, which had no skills. Claude agents got no loaded skills from 615dcb330 (2026-05-13) until this PR. A single path removes the whole class of bug where skills arrive on one surface and are missing on another.

**Rejected:**
- Keep native-skill suppression. A harness's native skill list shows only model-invocable skills, so a loaded skill that isn't model-invocable (e.g. `solo-dev`) would be unreachable.
- Keep #661's first-draft `_inject_loaded_skills` for primaries. A dry-run probe on main showed it was a no-op: the bind re-append threw its output away.

**Deleted with it:** `_inject_loaded_skills`, the bind re-append and its `loaded_skills` parameter, `compose_skill_injections`, `RunPromptPolicy.skill_injection_mode`, `HarnessCapabilities.supports_native_skills`.

**Divergence:** composition is not delivery everywhere. Before v0.11.6, OpenCode 2 and the OpenCode black-box fallback never passed the composed system prompt to the model, so agents there got no loaded skills; this predated #661. The fix, shipped in v0.11.6 (PR #710), is [D-opencode-config-agent](opencode-prompt-delivery.md) ([per-harness channels](../codebase/harness-adapters.md#skill-delivery)).

**Supersedes:** D-skill-doc-suppression, below.

---

### D-pi-spawn-native-discovery: Pi spawns keep native skill and context discovery

**Status:** Merged in PR #675. Until it merges, `main` passes `--no-skills --no-context-files --no-prompt-templates` to spawned Pi.

**Decision:** Spawned Pi (`project_pi_rpc.py`) keeps Pi's native skill, context-file and prompt-template discovery, like the Pi primary and every other harness's spawns. It still passes `--no-extensions` with explicit `-e` for Meridian's bundles, because ambient extensions run code and can conflict with the managed ones; `load_all_pi_extensions` opts in.

**Why:** The flags came in with PR #225 (`ae70d398e`, 2026-05-19) under a generic integration-guide rule ("every harness has ambient discovery… for spawns, suppress all of it") that no other harness followed. With native skills off, Pi had no skill tool, so a profile's `available` skills were listed but unloadable. `--no-context-files` also hid the project `AGENTS.md`. Meridian composes loaded skills and never injects `AGENTS.md`, so enabling discovery duplicates neither: Pi's native list adds only names, descriptions and paths, and honors `disable-model-invocation`.

**Origin, fuller:** the owner's 2026-05-16 design (work `meridian-pi-mars-support`) meant `--no-skills` and `--no-context-files` as half of "explicit injection, not discovery": suppress discovery, then pass the agent's skills with `--skill <path>`. Only the extension half (`--no-extensions` plus `-e`) was built. The design also ran spawns in a Meridian-owned agent dir, which PR #225 removed.

**Rejected:** finish that design (keep `--no-skills`, pass `--skill` per available skill). It would scope Pi spawns to the agent definition while every other harness shows all model-invocable skills; per-agent skill scoping, if wanted, is a cross-harness decision. Also rejected: render each available skill's `source_path` in the Available Skills listing so Pi agents can read the file. It works around the suppression instead of removing it, and it costs tokens on every harness.

---

### D-skill-doc-suppression: Skip supplemental_documents for native-skill harnesses (superseded)

**Superseded (2026-10) by [D-skill-single-path](#d-skill-single-path-loaded-skills-are-composition-documents-on-every-harness).** This record stays because it explains how the split happened.

**Decision (2026-05, PR #208, 615dcb330):** When a harness declared `supports_native_skills=True` (Claude, Codex, OpenCode), composition left `supplemental_documents` empty and relied on Mars-delivered native skills, plus a Claude-only `compose_skill_injections()` append channel.

**How the split came about:** In March 2026 Claude Code's `--agent` did not preload a profile's `skills:` (anthropics/claude-code#29902), so `00453e433` added a Claude-only append channel (`compose_skill_injections()` → `--append-system-prompt`). Five days later `2b23522bc` removed the injection to fix duplicate skills, with a comment that Claude loads skills natively through `--agent`; the upstream issue had been bot-closed as a duplicate, not fixed. In April `8f47d2eab` unified delivery: composition always included loaded skills for every harness. PR #208 then re-split it, in the same commit that filtered `model-invocable: false` agents out of the spawn inventory, reasoning that native skill directories already delivered skills and composition duplicated them.

**Why it failed:** The goal was to stop sending skill content twice. Two mistaken assumptions undid it. First, native delivery is not equivalent to loading, because native lists cover only model-invocable skills. Second, a capability-gated second channel would stay in sync with the first. In practice the append channel was overwritten at bind, so delivery fell to an ad hoc bind re-append. The file Claude actually reads never had skills.

---

### Semantic IR + adapter projection for prompt composition

**Decision:** The launch factory produces `ComposedLaunchContent` (harness-agnostic semantic IR), and each harness adapter's `project_content()` maps this to `ProjectedContent` (harness-specific channel assignments). No conditional branches in composition for per-harness routing.

**Why:** Adding a new harness should mean implementing an adapter, not editing composition code. The old `reference_input_mode` capability flag was removed when it proved insufficient.

See [concepts/composition-pipeline.md](../concepts/composition-pipeline.md).

---

### Spawn-field coverage from adapter `handled_fields` (supersedes StrategyMap)

**Decision:** Each adapter declares the launch fields it handles. Coverage validation uses the union of declarations against the typed launch spec. Bundle registration and launch-spec guards fail when a field has no owner.

The earlier `STRATEGIES: StrategyMap` of `FlagStrategy` entries was deleted; `handled_fields` retains its exhaustive-coverage goal without coupling every field to a flag-strategy abstraction.

See [Harness Abstraction](../concepts/harness-abstraction.md).

---

### Config precedence: CLI flag > ENV var > agent profile > project config > user config > harness default

**Decision:** Every config field resolves independently through a fixed precedence chain. A CLI model override also forces harness re-derivation from the overridden model.

**Why:** Different fields have different override sources. Independent resolution lets each field resolve to the highest-precedence source. Without forced re-derivation, `-m codex` with `harness: claude` in the profile silently uses the wrong harness.

---

## Harness Identity and Environment

> **Renamed (PR #388).** D32, D33 and D57 below use the variable's old name. It is now the internal handle `_MERIDIAN_HARNESS`, set in `bind_launch_context()`; the decisions are unchanged. A plain `MERIDIAN_HARNESS` is not read anywhere.

### D32: `MERIDIAN_HARNESS` is a one-hop env var; does not cascade to grandchildren

**Decision:** The composition seam (today `bind_launch_context()`) writes `MERIDIAN_HARNESS = harness.id.value` into every spawned process's environment. NOT in `ALLOWED_CHILD_ENV_KEYS` — does not propagate to grandchildren. Each spawn level gets its own value from its own harness resolution.

**Why:** A grandchild may use a different harness. Inheriting the grandparent's value would silently override the grandchild's resolved harness.

---

### D33: Wait-yield uses the orchestrator's own `MERIDIAN_HARNESS`, not child spawn rows

**Decision:** `spawn_wait_sync()` reads `os.getenv("MERIDIAN_HARNESS")` — the orchestrator's own harness — for yield interval. All harnesses default to 3000 seconds (50 minutes).

**Why:** The yield bounds the orchestrator's own idle/compaction exposure. Reading child spawn rows would instead let a mixed-harness wait set choose another harness's budget. The original cache-preservation rationale is superseded by [D-child-completion](completion-drain.md): checkpointing is operational, not a provider cache-lifetime guarantee.

---

### D57: `MERIDIAN_HARNESS` is spawn-local, not a user-facing policy override

**Decision:** `MERIDIAN_HARNESS` is informational spawn-local. NOT consumed as a policy override by `resolve_policies()`.

**Why:** If treated as a policy override, nested launches would inherit the parent's resolved harness, bypassing their own profile and config.

---

### D63: `MERIDIAN_HARNESS_COMMAND` removed

**Decision:** The env var override mechanism was removed. Harness adapters are the only launch path.

**Why:** Bypassed adapter command assembly including spawn-field coverage and security checks. No documented use case couldn't be served by adapter customization.

---

### D-spawn-capability-gate: Capability-gated spawn knowledge injection (PR #328, 2026-06)

**Decision:** Spawn knowledge (agent inventory + harness-templated spawn usage contract) is injected into the system prompt, gated on the agent profile's `subagents` field. Opt-out via `meridian-capabilities: {spawn: false}`. No positive `spawn: true` override — having `subagents` IS the capability.

**Why:** The spawn usage contract was previously only in the opt-in `meridian-spawn` skill. Across three incidents, the model never loaded that skill — context prioritization buried it. Making the contract always-injected eliminates the opt-in gap. The gate on `subagents` ensures leaf agents don't waste context on spawn instructions.

**Contract as first-class composition block:** Delivered as `spawn_contract_prompt` — a first-class block in `ComposedLaunchContent`, rendered separately from inventory to prevent snapshot round-trip duplication.

### D-capability-command-mapping: Packages name capabilities; launch guidance maps commands (2026-10, v0.14.3) {#d-capability-command-mapping}

**Decision:** Prompt-package prose names portable capabilities — the work
directory, KB, past sessions, notifying the human, or validating diagrams — and
does not teach Meridian syntax. Meridian launch composition injects one compact
`# Meridian capabilities` section that maps each phrase to its command and `-h`
entry point. Shipped in meridian-cli v0.14.3 (#767); meridian-base 0.12.0,
meridian-dev-workflow 0.15.15, creative-writing-skills 0.5.19, and meridian-prompter
0.5.12 rewrote their prose to the capability wording.

Package text follows four rules: name the capability, never the command ("check the
live model catalog", not `meridian mars models list`; "the work directory", not
`$MERIDIAN_ACTIVE_WORK_DIR`); refer to skills as `/name` and agents as `@name`; keep
the judgment and drop text whose only job is syntax; leave frontmatter runtime policy
such as `tools:` allowlists alone, since it is not prose.

The rows live in `src/meridian/lib/launch/spawn_guidance.py`. Work items, past
sessions, link/diagram validation, the source checkout, and the KB are present on
every profile. `meridian notify` appears only for
`mode: primary` profiles (see [D-bundled-meridian-skills](#d-bundled-meridian-skills-meridian-owned-operations-guidance-versions-with-the-cli)).
Delegation, agent inventory, subagent prompting, and model-specific prompting remain
behind the existing spawn capability gate. Behavioral invariants that help alone cannot enforce stay in the
prompt: standalone Meridian shell calls, draining `--bg` spawns, never
background-wrapping a launch, and the confirm-completion handshake.

**Why:** Shared prompt packages are imported into runtimes that may have app tools
instead of Meridian or even a shell. The driving case is meridian-flow, a separate app
that imports creative-writing-skills and meridian-base into agents with no shell. Embedding CLI syntax makes their judgment
non-portable and duplicates command help. Keeping the mapping at Meridian's launch
boundary gives Meridian agents the concrete operation while leaving package prose
usable elsewhere. The consolidation also prevents separately rendered task/work
instructions from drifting or inflating every prompt.

---

### D-reserve-before-prep: Reserve spawn row before preparation for crash recovery (PR #328, 2026-06)

**Decision:** Background spawns write a durable `queued` row **before** preparation. Preparation succeeds → announce (work item, lifecycle events). Preparation fails → reserved row cleaned up, side-effect-free.

**The crash window being closed:** A launcher SIGKILLed during preparation could leave a forever-invisible spawn (work item exists but `spawn list` shows nothing). Reserving the row first leaves a reconcilable `queued` row the reaper handles via `missing_runner_pid`.

**Two-phase reserve/announce:** Reserve writes only the spawn row. Announce creates work item and emits events only after preparation succeeds.

---

## Related

- [Launch architecture](../architecture/launch-system.md)
- [Launch concepts](../concepts/composition-pipeline.md)
- [Decision index](../decisions.md)

### D-continuation-target-constrained-selection: explicit continuation requests remain target-constrained

**Decision (C7 settled):** Exact primary and spawn continuation accept an explicit
`--model` request, and every such request emits a warning. Mars revalidates the
current target and exclusion constraints under the same harness; the literal
canonical/provider pin is retained so an alias cannot drift during continuation.
This is target-constrained selection, not a model-freeze or model-prohibition
rule. Non-routing policy and the original launch snapshots remain unchanged.

After the explicit request is accepted, a later plain continuation uses the
Meridian model selection recorded at `accepted-running`. It does **not** discover
or infer the model from the last model executed by the native harness. A
`SessionAttempt` binds that accepted selection to the same generation and
attempt, while authoritative native-ID callbacks identify the session. A newly
derived ID is only a provisional row hint until the authoritative callback
arrives.

If an append fails, continuation ends as a coordination error; it does not
switch runtime models to recover. Legacy original-generation lookup is
read-only against baseline `c3087ceb`; the original snapshot or partial
historical model is carried through `SessionRequest` and seeded at session
scope under the store lock. Failed continuation never becomes the initial
value. New-protocol missing acceptance or absent tracked original history
requires an explicit model—there is no guessing. Native raw IDs remain
literal, and ambiguous tracked/untracked/mixed ownership requires explicit
`--harness`; known p/c references retain their recorded harness and file
detection does not silently replace it. Native reference is `833dc1f2`.

The remaining C7 work is the spawn native-fork/reference-form audit and
coordinated final gates/PR/release.

### D-primary-identity-delayed-binding: fresh and native-fork primary selection waits for identity

**Decision (2026-09, target-constrained-fallback):** Primary fresh and
native-fork launches do not bind a generated seed or source parent before the
actual harness identity exists. Selection and persistence wait for the
authoritative identity; a generated seed remains provisional and cannot bind
the source parent early. Claude's existing transcript/trampoline identity
detector remains the identity observation mechanism; this increment adds no
model discovery.

If identity binding fails, the failure propagates while adapter cleanup and
session stop still run. This keeps the binding error visible without leaking a
live adapter or masking the original error. This is a partial C7 increment:
streaming-serve recording, OpenCode streaming named-model transport, spawn
native-fork/reference-form audit, and coordinated final gates/PRs remain open.

**Evidence:** `work:target-constrained-fallback` (primary-identity seed/fork
drivers, native-fork fake executable drivers, binding-failure and quiet
transcript controls; reviewer `p6231` closure and runtime `p6225`).

**Evidence:** `work:target-constrained-fallback` (`c7-primary-review.md`,
`c7-primary-probe.md`, `c7-spawn-probe-luna.md`, `c7-spawn-seed-closure-luna.md`).

### D-streaming-opencode-target-constrained-fallback: accepted startup selection is attempt-bound

**Decision (2026-09, target-constrained-fallback):** All three driving drivers record
accepted startup selection through the shared `SessionAttempt`. Streaming-serve
binds a pending selection using the identity callback when available, or the
existing post-run artifact extraction path. Native-spawn fork child isolation
does not record the parent as the child.

OpenCode preserves the native HTTP model contract: session creation uses
`{providerID, id}`, while every invocation's initial prompt—including an
explicit prompt or a plain recorded continuation—uses `{providerID, modelID}`.
Only subsequent resident/injected messages omit the model. Invalid or timed-out
creation does not downgrade to `{}`. Bounded transport-local session readiness
polling is separate from launch orchestration and must remain idempotent and
pre-turn. Meridian does not replay the harness launch after a creation or prompt
failure: one invocation has one create path.

The primary OpenCode named-model resume remains unsupported and has no UI
replacement. Tracked raw native IDs for `spawn --continue` are now accepted.
An explicitly supplied older native ID is retained when it matches the resolved
session; otherwise the matched session's recorded spawn ID is used, with exact
chat/harness checks. Untracked or pruned provenance fails honestly, and
`--harness` disambiguates ownership rather than history. Native primary/spawn/
c-prefixed sequences preserve recorded models.

**Evidence:** `work:target-constrained-fallback` (streaming/OpenCode CLI and
process red/green drivers; `evidence/c7-opencode-evidence-closure-luna.md`,
review approved; fake processes plus real CLI/Mars, not paid native execution).
Codex changes in this increment are fixture snapshots only.
