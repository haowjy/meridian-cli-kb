# Decisions: Package Management

Decisions about the `.mars/` store, source selection, targeting architecture,
compiler pipeline, agent and skill emission, the universal skill schema, and
the bootstrap documentation model.

For mechanism, see:
- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — mars as a package manager
- [concepts/package-management/compiler-pipeline.md](../concepts/package-management/compiler-pipeline.md) — reader → compiler → target sync
- [concepts/package-management/self-source-selection.md](../concepts/package-management/self-source-selection.md) — current-package selection, discovery, and ownership
- [concepts/package-management/targeting.md](../concepts/package-management/targeting.md) — native harness dir emission
- [concepts/skill-schema.md](../concepts/skill-schema.md) — frontmatter, lowering, variants
- [architecture/mars-targeting.md](../architecture/mars-targeting.md) — targeting architecture detail
- [architecture/mars-compiler.md](../architecture/mars-compiler.md) — compiler internals
- [architecture/mars-user-scope.md](../architecture/mars-user-scope.md) — user Mars root, root resolution and content layering

---

## Targeting: `.agents/` to `.mars/`

### D50: `.agents/` eliminated; `.mars/` is Meridian's compiled read surface

**Decision:** `.agents/` is removed as a mars target. Meridian reads agents and skills from `.mars/`. Skills are emitted to every enabled harness's native skill directory. Agent emission to native harness directories is conditional on `agent_emission` setting and `MERIDIAN_MANAGED` env var.

**Why:** `.agents/` caused confusion — `.agents/agents/` was Meridian-only, `.agents/skills/` caused overlap with harness skill dirs, and the name implied cross-tool portability that didn't exist. Now Meridian reads from `.mars/`, and each harness's native skill directory gets a copy.

**Alternatives rejected:** Better naming (same confusion), shared layer (recreates overlap), harnesses read `.mars/` directly (requires harness-level config changes).

---

## Compiler Pipeline

### D35: Config-entry provenance stored in `mars.lock`, not a separate manifest

**Decision:** `mars.lock` is extended with an optional `config_entries` section to record which package installed each MCP server / hook entry into which target config file.

**Why:** Single authority for "what did sync install?" prevents coherence risk between two files. One atomic write = one crash-recovery path. The lock already serves this purpose for content items; extending it to config entries is conceptually consistent. `#[serde(default)]` makes the section backwards-compatible — older mars versions ignore it.

**Alternatives rejected:**
- Separate `.mars/config-entries.json` manifest → two authorities for the same question; doubles crash-recovery complexity
- Derive stale entries from the lock's dependency removals → requires knowing which config entries each dependency provided, which is exactly what we're trying to persist

---

### D36: MCP/hook collision precedence matches agent precedence (local wins, declaration order)

**Decision:** MCP server and hook name collisions resolve via: local (`_self`) beats dependency; among dependencies, declaration order in `mars.toml [dependencies]` wins; alphabetical source name breaks ties within the same scope.

**Why:** Users already understand "local shadows dependency" from agent behavior. Applying the same model to MCP/hooks requires zero new concepts. Previous behavior (abort on any collision) was overly strict — same-name-from-different-packages is common and has an obvious resolution.

**Implementation note:** `graph.order` is alphabetical, not declaration order — declaration-order precedence requires reading `mars.toml` directly.

**Alternatives rejected:** Error on all collisions (too strict), merge collisions (semantically ambiguous — MCP servers are atomic units).

---

### D37: Windows hooks keep `bash` interpreter; adjust quoting per platform

Hook lowering keeps `bash` as the interpreter on Windows, adjusting only quoting. Git for Windows ships bash; extension-based dispatch is over-engineered for V0 when all hooks are `.sh` files. *(Low impact — POSIX-first policy makes this largely historical.)*

---

### D38: `LockIndex` for O(1) lock lookups; lock file parsed once per sync

Build a `DestPath → LockedItem` index once per sync run, replacing per-lookup O(n) scans. Parse the lock file exactly once with a version-aware deserializer.

---

### D39: `compiler/config_entries/` extracted as preparatory refactor

Pure code-motion refactor: extract `compile_config_entries()` into a dedicated module between Phase 3 and Phases 4a/5 to reduce merge risk.

---

### D40: Mars integration tests split into focused top-level files

Split `tests/integration/mod.rs` (2,100 lines) into separate top-level test files with shared helpers in `tests/common/mod.rs`. Enables parallel execution and faster incremental builds.

---

## Skill Schema

### D58: Universal skill frontmatter schema (name, description, model-invocable, user-invocable, allowed-tools, license, metadata)

**Decision:** The canonical SKILL.md frontmatter has seven fields: `name`, `description`, `model-invocable`, `user-invocable`, `allowed-tools`, `license`, and `metadata`. A `compatibility` field proposed in early drafts was removed.

A `compatibility` field was rejected — would require Mars to understand harness semantics. `metadata` is pass-through for tooling use.

> **Partially superseded by D71.** `invocation` enum replaced by `model-invocable` and `user-invocable` booleans.

---

### D71: Skill invocability decomposed into two independent booleans: `model-invocable` and `user-invocable`

**Decision:** Replace the single `invocation: explicit | implicit` enum in SKILL.md frontmatter with two independent booleans: `model-invocable` (default: `true`) and `user-invocable` (default: `true`). Remove all legacy alias fields (`invocation`, `disable-model-invocation`, `allow_implicit_invocation`) as hard errors with no deprecation period.

**Why two booleans:** The old enum collapsed two orthogonal dimensions. `invocation: explicit` only ever controlled model-invocability — user-invocability was unmodeled. Hard errors (no deprecation) because no consumer reads `invocation` at runtime; migration is find-and-replace. Presence is determined by the agent profile's `skills:` list, not skill metadata.

**Compilation:** Claude Code supports both natively. Codex supports `model-invocable` only, through a sibling `openai.yaml` (`policy.allow_implicit_invocation: false`) written only when the source explicitly sets `false`. OpenCode drops both. Pi uses `disable-model-invocation`. Cursor has no such field: `model-invocable: false` becomes a Manual rule (`alwaysApply: false` with the description stripped).

---

### D-load-ignores-invocability: `model-invocable` gates discovery, not an explicit load

**Decision:** `model-invocable` decides only whether a model can find a skill by itself in its harness's skill list. A skill that an agent profile lists under `skills.load`, or that a caller attaches with `--skills`, is loaded in full whatever its `model-invocable` value. A skill under `skills.available` is listed by name whatever its value, but the harness's own skill tool won't load a non-invocable one, so that pairing is a contradiction for package review to catch.

**Why:** Authors need a way to give one agent a skill no other agent sees. `model-invocable: false` plus `load` does that: the owning agent always has the text, and no other agent pays a description line for it or loads it because the description looked relevant. If invocability also gated explicit loads, that role-specific skill would have to be model-invocable, and every agent would carry its description on every turn.

**Where it lives:** Mars builds the launch bundle's `skills.loaded` and `skills.available` without reading invocability (`src/build/prompt.rs`, "model-invocable gates global discovery, not explicit profile references"). Meridian composes every bundle-loaded skill ([D-skill-single-path](launch.md#d-skill-single-path-loaded-skills-are-composition-documents-on-every-harness)). The load/available split arrived in mars-agents `d1d48d1` (2026-05-30); the owner restated the rule as something authors rely on on 2026-10-17. See [how skills reach an agent](../concepts/skill-loading.md).

**Revisit if:** a harness starts honoring its native invocability flag for content Meridian composes into the prompt, or Meridian ever routes loaded skills through a native skill list again.

---

### D59: Native skill projections always refreshed on sync with divergence warnings

**Decision:** `mars sync` re-emits every native harness projection (e.g., `.claude/skills/<skill>/SKILL.md`) unconditionally — no content-hash optimization to skip unchanged files. If a native projection has diverged from what sync would produce, Mars emits a divergence warning.

**Why:** Skipping unchanged projections would leave stale native files after format/lowering changes. Always-refresh keeps the invariant: after sync, native dirs are byte-for-byte identical to lowering output.

---

### D60: Skill variants are selected by harness only

**Decision (narrowed 2026-10):** A skill body is selected by harness alone. Mars uses `variants/<harness>/SKILL.md` if that file exists, and the base `SKILL.md` otherwise. Model-level variants are not supported: no part of the system picks a body by model token or canonical model ID.

**Superseded design.** The original record specified a four-step exact-match ladder: `variants/<harness>/<requested-token>/SKILL.md`, then `variants/<harness>/<canonical-id>/SKILL.md`, then `variants/<harness>/SKILL.md`, then `SKILL.md`. The two model-identity steps existed because authors might write either form (`variants/claude/sonnet/` or `variants/claude/claude-sonnet-4-5/`). Only the harness and base steps were ever built into the Mars launch bundle (`src/build/prompt.rs`, since mars-agents `ecbcff4`, 2026-05-17). Meridian's filesystem `SkillRegistry` kept the full ladder, but by 2026-10 only replay of old launch snapshots reached it, and it was deleted when Mars became the sole skill resolver ([D-mars-sole-skill-resolver](#d-mars-sole-skill-resolver-meridian-never-resolves-skills)).

**Why narrowed:** No prompt package ships a `variants/` directory, and no one has asked for a model-specific skill body. Rebuilding model-level selection in Mars before anyone needs it was rejected.

**Where it lives:** the launch bundle (`harness_skill_variant_path` in mars-agents `src/compiler/variants.rs`, called from `src/build/prompt.rs`) and `mars sync` for native projections. Exact match only, no glob or fuzzy matching.

> [!FLAG] **Implementation lags the decision.** Mars still indexes
> `variants/<harness>/<model>/` directories and warns when one lacks a
> `SKILL.md`, and `compile_prompt_surface` still accepts the model token and
> canonical ID only to discard them. Neither selects anything. Removing them is
> Mars cleanup.

**Revisit if:** a package author needs a skill body that differs by model within one harness. Build the selection in Mars's launch bundle, not in Meridian.

---

## Bootstrap Documentation

### D61 (bootstrap): Two-tier bootstrap doc discovery (skill resources + package-level)

**Decision:** Bootstrap docs live at two tiers: skill-level `resources/` (co-located with `SKILL.md`, visible to standalone harness users AND Meridian) and package-level `BOOTSTRAP.md` (Meridian-only via `meridian bootstrap`).

**Why two tiers:** Skill resources provide standalone visibility — a user with Claude Code who never runs Meridian can still discover skill setup docs because they live next to `SKILL.md`. Package-level docs cover cross-skill and project-scope concerns that don't belong in any single skill directory and don't need to be harness-discoverable.

**Alternatives rejected:**
- Single package-level bootstrap → loses standalone visibility
- Embed setup docs in SKILL.md body → wastes system-prompt tokens at runtime on content only needed during setup

---

### D62 (bootstrap): `meridian bootstrap` uses normal agent resolution, not a dedicated bootstrap agent

**Decision:** `meridian bootstrap` reads and surfaces markdown docs using normal agent resolution (profile loading, harness detection, model selection). The earlier BD-C1 design (dedicated bootstrap agent to format and present docs) was rejected.

**Why:** A dedicated bootstrap agent added a bespoke code path for a doc-surfacing problem. The bootstrap command is a document reader — it collects markdown and presents it. Normal agent resolution handles the session context question (which agent/profile/harness/model launches the bootstrap session) without a custom agent. Simpler, more predictable output.

---

## Sync Model

### D77: `upgrades_available` counts direct dependencies only

**Decision:** The "upgrades available" count shown in `mars sync` output is restricted to packages that appear in `effective.dependencies` (direct deps). Transitive-only packages with newer versions available are excluded from the count.

**Problem that triggered this:** `mars sync` reported N upgrades available, but `mars upgrade --bump` did nothing — or reported 0 outdated packages. The UX was contradictory: a hint that promises actionable work produces no action.

**Root cause:** `sync` counted all `graph.nodes` (direct + transitive). `outdated` and `upgrade --bump` only iterated `config.dependencies` (direct deps). Transitive packages with newer versions were counted as "upgradeable" but were never actionable through any user command.

**Solution:** Filter the count to packages whose name is a key in `effective.dependencies`. Transitive-only packages with newer versions are excluded because no user command can act on them.

**Alternatives rejected:** shared count function (premature coupling), actionable transitive upgrades (breaks lockfile semantics), split count display (adds complexity for non-actionable info).

---

## Init Command

### D78: Filesystem scan for mars content discovery, not JSON model coupling

**Decision:** After `mars add` completes, the Python `run_init_flow()` reads materialized content by scanning `.mars/` subdirectories dynamically (`_scan_mars_content()` in `init_ops.py`), not by deserializing a Pydantic model that mirrors the mars Rust `JsonReport` struct.

**Why filesystem over JSON model:**
- The mars JSON report shape is an implementation detail of the Rust `JsonReport` struct, not a published contract. Coupling Python to it means Python changes whenever mars adds a new content category, renames a field, or restructures the report.
- Filesystem scanning automatically picks up new content types (new `.mars/` subdirectories) without any Python changes.
- The `.mars/` directory layout IS the stable contract — it is what mars materializes for consumers to use. Meridian already reads agents/skills from `.mars/` at runtime; reading the same tree for counts is consistent.

**Alternatives rejected:** Parse `mars add --json` output (couples Python to internal Rust struct), define a Pydantic `MarsJsonReport` model (same coupling, plus schema drift).

---

### D79: Honest field naming — `packages_requested` not `packages_resolved`

**Decision:** The field holding the user-supplied package specifiers in `InitResult` is named `packages_requested`, not `packages_resolved`.

**Why:** "Resolved" implies the full resolved dependency set, including transitive dependencies installed by mars. This field holds only what the user passed to `--add` — the input specifiers. "Requested" is honest about the scope. The resolved set is internal to mars and not surfaced by `InitResult`.

**General principle:** Field names should describe what the data actually contains, not what you wish it contained. A field named for its aspirational content is a latent mislead — it tells the next reader that the data is more complete than it is.

---

### D87: Convention-based source discovery and explicit hidden foreign import

**Decision:** Mars discovers package items with one bounded convention walk. The walk starts at the rooted package directory, descends only through non-hidden directories up to `MAX_DISCOVERY_WALK_DEPTH = 5`, and recognizes directories named `agents`, `skills`, and `bootstrap`. A package-root `SKILL.md` remains the flat-skill fallback. After scanning, Mars grounds the result to the shallowest discovered package layer across all item kinds. Any duplicate `(kind, name)` discovered inside one source raises `DiscoveryCollision`, whether the duplicate came from convention scanning, manifest declarations, or both.

**Why:** Package import should be predictable from package conventions, not from a harness-specific guess about where a tool might store generated output. A single bounded walk lets real packages live below monorepo subdirectories without turning every nested fixture, example, or vendored package into importable content. Shallowest-layer grounding keeps a package's own layer authoritative when examples or vendored layouts also contain `agents/` or `skills/` folders.

**Hidden directories are not default sources:** Dot-prefixed directories are skipped by rule (not a per-harness blocklist). Foreign hidden layouts are importable via explicit `subpath = ".claude"` with `dialect = "claude"`.

**Rejected alternative:** Auto-scan hidden harness containers — treats generated state as source and requires a growing heuristic.

---

## Launch-Bundle System

Decisions about the `mars build launch-bundle` flow: Mars/Meridian ownership split,
harness override passthrough, portable tool-policy preservation, and harness target
status.

For mechanism, see:
- [architecture/mars-launch-bundle.md](../architecture/mars-launch-bundle.md) — cross-repo launch-bundle system, bundle contract, scaffold slots
- [concepts/native-config.md](../concepts/native-config.md) — harness override passthrough concept and per-harness projection

---

### D80: Mars/Meridian launch-bundle ownership split

**Decision:** Mars builds the static scaffold (agent identity, routing, execution policy,
prompt surface, scaffold slots, tool policy). Meridian injects per-spawn dynamic content
(prompt file, context files, goal, prior session history, spawn metadata) and launches
the harness process.

**Why:** Mars sees the compiled graph at build time; per-spawn content (prompts, context, history) is Meridian's runtime concern. Coupling Mars to spawn semantics was rejected — prompt files are confidential per-spawn content.

---

### D-mars-sole-skill-resolver: Meridian never resolves skills

**Decision (2026-10):** Mars is the only skill resolver. It reads the agent profile's `skills` field (a flat list or `{load, available}`), selects the harness variant, and returns the loaded and available skills in the launch bundle. Meridian does not parse profile `skills` and does not look up or load skills from `.mars/skills/` (`meridian doctor` only warns when no skills directory exists). It passes Mars only the skills a caller added with `--skills`, then composes what the bundle returns. Replaying a launch snapshot uses only the skill content embedded in that snapshot's `loaded_skills`. A snapshot without it replays with no loaded skills.

**Why:** Meridian had kept a second, older resolver, and two independent implementations had come to read the same profile and skill tree differently. Meridian's profile parser read only the flat list, so a `{load, available}` profile (the form the prompt packages use) parsed as no skills, and Meridian forwarded whatever it parsed back to Mars as extra skills that Mars had already loaded. Its filesystem `SkillRegistry`, reached only by replaying old snapshots, selected variants by a model-level rule nothing else followed ([D60](#d60-skill-variants-are-selected-by-harness-only)). Neither copy changed what a current launch received: Mars read the profile itself and dropped the duplicate names, and current snapshots never reached the fallback. But a later skill change could land in either copy and leave the other inconsistent.

**Rejected:**
- Teaching Meridian's parser the mapping form: it keeps two resolvers to drift again.
- Keeping the filesystem fallback for snapshots without `loaded_skills`: it would select bodies by a rule nothing else follows, and the project keeps no backwards compatibility. Current snapshots always embed `loaded_skills`.

**Where it lives:** `_resolve_policy_from_bundle` in `src/meridian/lib/launch/policies.py` builds `BundleRequest.extra_skills` from the caller's requested skills only. `replay_launch_policy_snapshot` in `src/meridian/lib/launch/policy_snapshot.py` builds `ResolvedSkills` from `snapshot.loaded_skills`. Deleted: `lib/catalog/skill.py` (`SkillRegistry` and variant discovery), `AgentProfile.skills`, `resolve_skills_from_profile`, the snapshot `skill_paths` field. Unknown `--skills` names still warn without failing; Mars owns the lookup, so the warning no longer names a path. Built on branch `refactor/mars-owns-profile-skills`, cut from `main` independently of PR #661. Together with [D-skill-single-path](launch.md#d-skill-single-path-loaded-skills-are-composition-documents-on-every-harness) it leaves skill delivery one path: Mars resolves, Meridian composes.

**Known gaps (predate this decision):** exact continue doesn't replay the available skills into the workspace, because the snapshot keeps only the rendered inventory text; and a fork passes the source spawn's resolved skill names to Mars as extra skills, which Mars dedupes. Both are tracked in `src/meridian/lib/launch/.context/FUTURE`.

**Revisit if:** Meridian needs skill content Mars doesn't return. Extend the bundle rather than reading `.mars/skills/` again.

---

### D81: Public verb is `build`, not `compile`

**Decision:** The CLI verb for producing the launch scaffold is `build`
(`mars build launch-bundle`). Internal code and design may use "compile" or "lower",
but CLI and user-facing docs use `build`.

**Why:** "Build" is the standard verb for transforming source artifacts to derived
artifacts in CLI tools (cf. `cargo build`, `docker build`). "Compile" implies a
language-level transformation and creates confusion with "compiler" as the term for
the Mars agent profile parser/lowerer.

---

### D82: `harness-overrides.<target>` — exact passthrough, not portable semantics

**Decision:** The entire `harness-overrides.<target>` block is raw target-native
passthrough. Mars validates only the outer mapping shape and serializability. Mars does
NOT interpret key names, enforce key semantics, or use nested keys to replace top-level
Mars fields. The matching block is carried in launch-bundle
`execution_policy.native_config`; Meridian's harness adapter projects it at launch time.

**Why:** Mars can't track every harness version's config schema. The escape hatch gives profile authors direct access; universally useful knobs get promoted to first-class fields later.

---

### D83: Portable tool policy preserved through the full pipeline (mixed allow/deny)

**Decision:** `tools` (map format with allow/deny entries) and `disallowed-tools` are
portable Meridian semantics. Mars resolves them into `tools.rules[]`, one typed entry per
statement: the canonical `capability` carries meaning, while `native` carries the resolved
harness spelling for emission. Meridian reads those two fields in those directions and
keeps no second capability-to-native name table. The derived `allowed`, `disallowed`, and
`mcp` lists remain temporarily for older consumers, but cannot express `ask`, `*`, or the
capability that produced a name.

Mars also emits profile-independent `tools.capability_natives` entries for the `agent` and
`task` capabilities in every bundle. Meridian still owns the decision to impose its
platform rules; it gets the target harness names for those rules from this map. Both fields
share Mars's one native table and status projection, including older-build names such as
Claude's `TaskOutput`. Each rule also carries `native_tools`, the harness tools it
concerns, because a permission spelling can name a different tool (Claude `Edit(src/*)`
for a scoped `write`). The typed rules and `capability_natives` are on unmerged
mars-agents PR #179; the Meridian reader is on unmerged branch `feat/session-permissions`.
How Meridian enforces them: [tool permissions](tool-permissions.md).

**Why portable, not passthrough:** Tool policy is portable across harnesses. Encoding as passthrough would require per-harness duplication.

---

### D84: Cursor experimental, Pi future-first-class, Gemini out of scope

**Decision:** Cursor is experimental (config surfaces evolving), Pi is future first-class (contract in development), Gemini is out of scope.

---

### D85: Static sync and launch-bundle are intentionally different consumers

**Decision:** `mars sync` produces harness-native static artifacts; `mars build launch-bundle`
produces a runtime JSON scaffold for Meridian. These are different build products for
different consumers. Harness passthrough is dropped from static native artifacts
(meridian-only in lossiness matrix) but preserved in the launch-bundle for runtime
projection. This divergence is intentional.

**Why:** Static native artifacts are for harness-native agent discovery (e.g., Claude Code
sidebar). They don't carry harness passthrough because the harness would need to know how to
apply it — that's Meridian's job. The launch-bundle is Meridian's runtime surface; it
carries everything Meridian needs.

---

### D86: Routing parity invariant — `mars models resolve` and `mars build launch-bundle` share `routing::evaluate_candidates()`

**Decision:** The `mars models resolve <model>` command and `mars build launch-bundle --model <model>`
both call the same `routing::evaluate_candidates()` function. The harness they select and the
confidence they assign for a given input are guaranteed to be identical by shared code, not by
convention.

**Why:** Before the routing refactor (PR #51), `mars models resolve` used a separate harness
inference path from `mars build launch-bundle`. A user who ran `mars models resolve sonnet` to
preview routing could see a different harness than what the bundle would actually emit. This
made debugging routing mismatches difficult and undermined the usability of `models resolve`
as a preview tool.

**Practical consequence:** `mars models resolve <token>` is a reliable preview — identical routing by shared code, not convention.


---

### D88: Cross-source naming collisions auto-rename both colliders

**Decision:** When two dependency sources install an agent or skill to the same destination path, both colliders are suffixed with `__{source_name}` instead of one winning or the sync failing. Explicit `rename` in mars.toml takes priority over auto-rename.

**Why:** Multi-package installs commonly share item names (e.g. `meridian-dev-workflow` and `creative-writing-skills` both defining `agents/web-researcher.md`). Hard-erroring blocks legitimate use; first-wins creates an ordering dependency on mars.toml declaration order for collision outcomes. Rename-both is unambiguous — the user always knows which package an item came from — and needs no precedence rule for the collision itself.

**Gating:** Auto-rename applies only to same-kind (Agent|Skill) groups where every collider has a distinct source. Same-source collisions, mixed-kind groups, and hook/MCP/bootstrap collisions remain hard errors. Hooks use a separate first-wins policy at the target-lowering layer (D36).

**Frontmatter rewriting:** A unified `RenameIndex` + `apply_renames` pass rewrites `skills:` and `subagents:` frontmatter refs. Same-source copy wins; prune-before-rewrite ensures ref resolution sees post-prune state.

**Alternatives rejected:**
- Hard error on cross-source collision → blocks legitimate multi-package installs sharing item names
- First-wins by declaration order → creates an ordering dependency for collision outcomes; the losing package silently disappears
- Merge collisions → agents/skills are atomic units; merging two `agents/web-researcher.md` files is semantically meaningless

---

### D89: Config-rename-dangle diagnostic for renamed-away config references

**Decision:** After any rename (explicit or collision auto-rename), sync checks config-side name references against installed names and warns when a referenced name was renamed away and no installed item still carries it. The warning code is `config-rename-dangle`.

**Why:** A rename can leave config references (fanout agents, agent/skill overlays) silently dangling. The warning lists the new installed names so the user can update. Only fires when the original name is gone — if another source still provides it, no false positive.

**Alternatives rejected:** Auto-rewrite config (user-authored, shouldn't mutate silently), hard error (too strict), no diagnostic (silent breakage).


---

### D90: Native hook passthrough replaces universal event vocabulary (mars-agents PR #133, 2026-07)

**Decision:** `hook.toml` declares harness-native event names in per-target `[targets."<target>"]` tables. Mars validates each event against a static per-target allowlist (`TargetAdapter::known_hook_events()`) and passes names through verbatim to target config files. The universal event vocabulary (`UniversalEvent`, `classify_for_target`, all five per-target translation tables, `LossinessKind`, `codex_hook_matcher`, legacy `codex_hooks.json` format support) is deleted.

**Why native passthrough:** Audit found 6 of 12 universal event-name mappings were wrong (Claude `session.end` → nonexistent `SessionStop`; all 4 OpenCode mappings fabricated). All 3 real hooks were single-target with native payload handling. The write-once/lower-down vision was abandoned.

**What mars keeps:** Hook discovery across dependency trees, deterministic ordering, script staging, atomic config merge/removal under per-target lock ownership, provenance tracking in `mars.lock`.

**What mars stops doing:** Inventing event names. Lock keys change from `hook:<universal-event>:<name>` to `hook:<NativeEvent>:<name>`. Targets without declarative command hooks produce hard errors instead of silent drops. `unchecked = true` per target table opts out of allowlist validation.

**Migration:** Old `hook.toml` schema is a hard error with migration hint. No compatibility shim. Deferred: residue sweeps (mars-agents#130); Cursor hook writing (mars-agents#131).

**Alternatives rejected:** Hybrid universal+native (two authoring paths), expanded universal vocabulary (fake portability at standing maintenance cost), per-target hook directories (loses shared script/ordering), drop hooks entirely (config-writing machinery is real value).

---

### D91: Native hook fragments replace command synthesis (2026-07)

> **Supersedes D90's schema.** D90 established the principle (native passthrough, per-target allowlists, `unchecked` escape hatch). D91 replaces the residual command-synthesis machinery with full native fragments: the author writes the harness's own hook shape verbatim, and mars stops owning any hook entry schema at all.

**Decision:** `hook.toml` shrinks to identity and routing (name, visibility, order, target-to-fragment map). Per-target fragment files (`claude.json`, `codex.json`, `cursor.json`, or `.ts` for file-mode targets) carry the harness's native config shape verbatim. Mars validates event keys, substitutes `${MARS_HOOK_DIR}` with absolute paths, merges entries into target config files (or places file-mode fragments whole), and records exact emitted JSON in `mars.lock` for structural removal.

**What mars stops doing:** Synthesizing hook entries. Authors write complete command strings and every harness-specific field. Mars never parses or generates hook entry content.

**Two fragment modes:** MergeJson (Claude, Codex, Cursor) merges entry arrays per event into target config. File (OpenCode, Pi) places fragments at `plugins/mars-<name>.ts` or `extensions/mars-<name>.ts`.

**Ownership model:** `mars.lock` stores exact emitted JSON entries (post-substitution). Removal matches by structural JSON equality. User-edited entries are preserved. Mars manages two ownership surfaces: path ownership via `OutputRecord` for files; config-entry ownership via `ConfigEntryRecord` with `emitted_json` for merge-mode. See [architecture/mars-compiler.md](../architecture/mars-compiler.md#two-surface-ownership-model).

**`${MARS_HOOK_DIR}` substitution:** Runtime probes proved relative paths and env-var paths break when launching from subdirectories. Sync-time absolute path substitution is the only form that works across harnesses.

**Migration:** Old `events`/`matcher`/`[action]` in `hook.toml` triggers a hard error with fragment-migration hint.

**Alternatives rejected:** Coexist old+new syntax (two authoring paths), TOML-inline fragments (kills copy-paste from harness docs), extend schema field-by-field (vocabulary treadmill), marker field in entries (depends on harness tolerance of unknown fields).

---

### D92: Shape-A recovery seam -- halt before compilation when hook surfaces are unreadable (2026-07)

**Decision:** Recovery commands (`upgrade`, `override`, `remove`, `repair`) persist their intent mutation (pins, overrides, dependency graph changes to `mars.toml` / `mars.local.toml`) and **halt with exit 2 before the compiler** when any resolved source has an unreadable hook surface (removed-schema package). Strict `sync` remains the sole materializer and only ever runs against a fully readable graph. The "frozen" hook category is deleted from the sync pipeline entirely.

**Why halt-before-compile:** The sync pipeline assumes the staged tree is total desired intent. Two attempts to thread an "unreadable, must not touch" exception through the pipeline both failed — omission read as removal intent (round 4 deleted unrelated dependencies' hooks), and freeze-as-carried-state leaked at 3 of N persistence sites (round 5). Every pipeline stage is a persistence site; an exception must be enforced at all of them or it corrupts state. A permanent concept in every stage for a one-release migration window was disproportionate.

**The seam:** The gate sits at the reader/compiler boundary in `sync::execute`. `unreadable_hook_surfaces` on `ResolvedGraph` has exactly one consumer: the gate. Nothing downstream reads it.

**Convergence:** In the dominant case (one legacy source), `mars upgrade` resolves a migrated release and the graph becomes fully readable in the same invocation. One-shot convergence is lost only in the multi-legacy-source case, where it was never achievable.

**User-facing contract:** Exit 0 iff full pipeline ran. Exit 2 with halt report (what was persisted, remaining blockers, suggested next command). Follows the `git rebase` conflict pattern.

**Alternatives rejected:** Omission-as-absence (reads as removal intent), freeze-as-carried-state (leaked at persistence sites), recovery commands write targets while blind (overwrites user edits).

---

## Engine Version Constraints

### D93: Hard-constraint-with-fallback engine requirements (2026-07)

**Decision:** Packages declare minimum engine versions via optional `requires-mars` and `requires-meridian` fields in `[package]`. When the running engine does not satisfy a dependency's requirement, the resolver excludes that version and walks down to the newest compatible candidate. When no candidate is compatible, resolution fails before any mutation. Escape hatches: `--ignore-requires-mars` and `--ignore-requires-meridian`.

**The model:** pip's `Requires-Python` — hard filter with emergent fallback. Every ecosystem studied (pip, Cargo, Bundler, npm, Go) filters or ranks by runtime compatibility; the pip/Bundler hard-filter model serves the "newest version that works" intent most cleanly. Mars packages compile into agent-facing trees; a wrong compile is worse than a stop, which rules out Cargo's soft-preference and npm's advisory-only defaults.

**Two engines, one mechanism.** `requires-mars` checks against the compiled-in `CARGO_PKG_VERSION`; `requires-meridian` checks against the `MERIDIAN_VERSION` environment variable exported by `mars_passthrough.py` in meridian-cli. Both use the same parse, check, and walk-down code path (`src/resolve/requires.rs`). When `MERIDIAN_VERSION` is absent (standalone mars, or old meridian), `requires-meridian` checks are silently skipped. When present but malformed, resolution fails with a typed error — but only when the manifest actually declares `requires-meridian`; fieldless manifests never parse the environment.

**Bare version rewrite:** `"0.12"` → `>=0.12.0` before parsing (semver's default caret would reject mars 0.13). Prerelease/build metadata stripped from running version before matching.

**Exclusion provenance:** Each excluded `(source, version)` records which requirements failed. Exclusions persist across resolver restarts so restarted passes never re-propose excluded versions. Diagnostics derive from the final `ResolvedGraph`, not accumulated during resolution passes.

**Edge cases:** Consumer's own `requires-*` is a hard error (no fallback). Locked versions that become incompatible enter walk-down with a warning. Single-candidate sources (RefPin, path, HEAD) are hard errors.

**Migration:** Existing packages (no field) resolve unchanged. Old mars binaries ignore new fields via serde defaults. Adoption discipline: declare the field when first using a feature newer than the current floor.

**Alternatives rejected:** Soft preference/Cargo MSRV (silent breakage), advisory-only/npm (same), synthetic dependency/Bundler (entangles engine with SourceName registry), engine auto-upgrade/Go (distribution/security problem), `[engines]` table (overkill for two keys), out-of-band metadata (infrastructure mars lacks), recording in lock (redundant, would go stale).

**Source study:** pip hard-filters, Cargo soft-ranks, Bundler injects synthetic dep, npm filters+tiebreaks, Go auto-upgrades. Full study in `work/mars-version-constraints/design.md`.

---

## Declared-Package Self-Sync

### D94: `[package]` opts agents and skills into `_self` (2026-09-15)

**Status:** Current; merged by `9dd16c7` (PR #164) and released in Mars 0.13.2.

**Decision:** A project with `[package]` contributes its own agents and skills
to sync under the synthetic `_self` lock owner. `.mars-src` remains an
independent higher-precedence self layer. Declared-package content is
Mars-native, while `.mars-src` and dependency dialect rules remain unchanged.
The feature uses the existing lock schema and canonical pipeline rather than a
self-dependency or runtime source lookup.

**Precedence follows installed identity:** Dependency explicit renames and
collision renames run before selected self items overlay matching destinations.
Mars does not deduplicate raw dependency names or change reference rewriting.
This preserves renamed dependency outputs while making the current package's
unrenamed content available in its own checkout.

**Discovery boundary:** Self selection reuses D87's shallowest occupied-layer
rule. The declared-package contribution is filtered to agents and skills only,
but other convention kinds still participate in grounding. A nested
distribution becomes eligible if all shallower convention items disappear;
there is no permanent `cw/` or distribution-directory exclusion. The owner
explicitly chose this fallback over a new discovery policy.

**Flat-root consequence:** A package-root `SKILL.md` uses the declared package
name while retaining `_self` ownership. Its resources are filtered before
traversal to exclude canonical/staging output, standard native roots, and
current or previously owned target paths. Those project-root names are not
reserved inside a flat `.mars-src` resource tree. This prevents recursive
self-copy, but it does not change discovery: a non-hidden generated output can
still become an input on a later run. That general discovery problem is
tracked separately in
[mars-agents issue #161](https://github.com/haowjy/mars-agents/issues/161).

**Ownership decision:** Default sync cannot adopt an unowned canonical
destination, even when bytes match; sync fails before canonical/native apply
with relocation guidance. D96 supersedes only the earlier decision that force
must also refuse: explicit `mars sync --force` is now intended to authorize a
selected canonical self takeover. For managed identical-byte source
transitions, `Update` records the new owner only when disk is unmodified and
carries native claims forward. Existing keep-local, source-wins, and native
force/report behavior remains unchanged.

**Why this shape:** The June 2026 removal of broad package-root scanning fixed
unmanaged collision symptoms but left package authors unable to consume their
own declared content. Restoring self availability through explicit source
selection preserves the ownership repair: inputs are selected before staging,
installed through the normal canonical plan, and refused rather than adopted
when ownership is absent.

**Rejected alternatives:** A self-dependency (recursive/versioned identity for
the current checkout), alternate runtime lookup (bypasses canonical ownership),
automatic adoption or merge (claims user content), permanent `cw/` exclusion
(repository-specific policy), sticky source roots or a new lock schema, and a
new dependency rename/reference algorithm.

---

### D95: Journal new canonical writes without checkpointing ownership (2026-09-15)

**Status:** Current; merged by `9dd16c7` (PR #164) and released in Mars 0.13.2.

**Decision:** Before writing a new absent canonical output, Mars publishes
version 1 `.mars/pending-canonical.json` with the expected output checksum and
source provenance, bound to the exact prior `mars.lock` bytes or its absence.
On retry, matching regular outputs are recovered into the in-memory lock before
self-source selection. `_self` and `mars.lock` v3 do not change.

**Why durable intent is necessary:** Canonical apply and later lock publication
are separate phases. An ordinary apply installed `muse`, then failed on a skill
behind a regular-file parent obstruction, leaving `muse` on disk without final
ownership. The failure reproduced 12/12 across the pre-#103 and current feature
implementations with fresh and established locks; it required neither process
kill nor lock editing. Atomic individual writes do not close this phase-level
gap. This establishes a real mechanism, not the exact trigger of the June
incident, which remains unproven.

**Recovery authority:** The journal is write intent, not an installed claim.
Mars requires the corresponding prior lock plus exact expected bytes and
regular output shape, rejecting ancestor or nested symlinks. Existing published
claims win over finalized-lock journal residue. Missing or invalid evidence
never turns byte equality or `--force` into adoption permission.

Journal identity is physical, not logical: its `outputs` map is keyed by
canonical `DestPath`, with at most current and planned versions for each path.
Read and write use the same path/kind validation, hooks preserve target scope,
and valid custom dependency destinations remain supported. Recovery derives the
logical item key only after verifying a physical output, then merges every
verified path into that item. This lets repeated interrupted moves retain exact
per-path provenance rather than overwriting evidence under one logical key.

**Retry ordering:** Recovered claims remain in memory until finalization. When
a retry also plans an update, its journal retains at most the verified current
and planned versions, covering another failure on either side of that write.
Only finalization publishes `mars.lock`, then removes the journal. This preserves
exact corrupt-lock bytes through repeated failed repair and source updates.
Dry-run and resolution failure do not publish new intent; no-op sync creates no
journal. Frozen mode refuses recovered but unpublished ownership even when every
filesystem action would be `Skip`; reproducibility includes ownership state,
not only output bytes.

An interrupted destination move retains the old canonical claim until removal
is confirmed. The temporary native-emission ownership view and final lock use
the same physical-path removal after carry-forward and upserts. Confirmed old
paths disappear without deleting the logical item's surviving canonical or
native claims, and skipped/kept outcomes cannot resurrect them. A same-path
pending-deletion claim is replaced by the recovered installed claim rather than
duplicated.

**Reserved recovery path:** `.mars/pending-canonical.json` cannot overlap a
canonical output. Before config or output writes, including dry runs, preflight
rejects files, descendants, directories, and effective bootstrap roots that
equal, contain, or fall beneath the journal path. Recovery evidence therefore
cannot be overwritten by a custom canonical rename. ASCII case variants and
trailing-dot/space aliases are reserved on every platform so moving a checkout
to a case-insensitive or Win32 filesystem cannot create a metadata collision.
This is a portable CLI contract; it was not justified by a Windows runtime
reproduction.

**Boundary:** This decision covers new canonical outputs, including but not
limited to `_self`; it does not make the whole sync transactional. Native and
config outputs remain unjournaled under
[mars-agents issue #149](https://github.com/haowjy/mars-agents/issues/149), and
pre-journal crashes still require safe manual recovery.
The design does not claim power-loss durability.

**Rejected alternatives:** Treating matching unowned bytes as ownership
(adopts arbitrary user content), preflighting only the reproduced obstruction
(does not close the general write/publication window), extending `mars.lock` v3
(conflates pending intent with installed authority), and checkpointing the lock
before finalization (destroyed corrupt-lock evidence when later repair failed).
Logical-item journal keys were also replaced: repeated moves need independent
physical claims and provenance for every unfinished path.

---

### D96: Explicit force takes over a selected canonical self path (2026-09-22)

**Status:** Implemented and verified on feature branch `f7a18fd`; not merged or
released. Mars 0.14.1 retains the earlier refusal.

**Decision:** Explicit `mars sync --force` should authorize takeover of an
unowned canonical destination selected for a current-package `_self` item. The
same operation must overwrite the selected path and publish an installed
ownership record. Default sync remains protective and refuses the path.

This is a scoped ownership transition, not automatic adoption. Byte equality
alone remains insufficient, and force does not authorize claiming unrelated
canonical paths, merging user content, or bypassing selection. The policy is
specific to `mars sync --force`; it must not be conflated with `mars version
--force` validation behavior or with linked-target collision handling.

The takeover is limited to a regular file for an agent or a real directory for
a skill. A symlink at the canonical root, an ancestor, or the destination is
never eligible. Skill replacement removes the selected destination tree rather
than following a nested link. The installed claim records the exact output
checksum only after the normal apply pipeline succeeds. Dry-run reports the
prospective adoption without publishing it, and frozen mode refuses until the
adoption is already current.

**Why:** Ownership provenance can disappear while old canonical output remains,
as happened when package commit `930bb37` removed the `architect` `_self` lock
record even though its parent checksum matched the surviving output. Manual
inspection and relocation remain the safe default recovery. Once the user has
explicitly selected force, requiring relocation as the only path adds ceremony
without adding consent or protection; the resulting write must establish the
ownership evidence needed for later updates and removals.

**Recovery boundary:** Preexisting force-adopted paths do not enter the
pending-canonical journal, which remains scoped to new absent outputs. If a
later apply action fails, finalization publishes no ownership; an earlier
written adoption remains unowned and retry requires `--force`. This preserves
the existing source transitions and recovery design rather than extending the
journal into a transaction.

**Release divergence:** The implementation and its regression contract are on
feature branch `f7a18fd` from baseline `347c472`. They are not merged or
released, so Mars 0.14.1 runtime and user-facing documentation must still
describe the earlier refusal.

**Rejected alternatives:** Adopting on normal sync (weakens the safety default),
adopting solely because bytes match (infers ownership from content), recording
ownership without overwriting the selected output (separates authority from the
user-authorized write), and broad force adoption across unselected paths.

---

## User Scope

### D97: User-scope packages live in an ordinary Mars root (2026-10)

**Status:** decided by the user on 2026-10-14 as part of
[D-two-tier-identity](state.md#d-two-tier-identity) (D41, D43, D44 in
`work:pyright-strict`; `work:two-tier-identity`). Shipped 2026-10-14: the Mars
side, `--fallback-root`, in mars-agents 0.16.0 (PR #173); the Meridian side in
meridian-cli PR #575 (merged to `main`, `release:minor`; released in v0.10.0
after #579), which pins `mars-agents==0.16.0`.

**Decision.** User scope for packages is an ordinary Mars project at
`~/.meridian/mars/` (`get_user_home()/mars`, so a scratch `MERIDIAN_HOME`
isolates it). Meridian, not Mars, decides which root a command runs against:

- **`resolve_mars_roots(project_root)`** returns `primary`, `scope` and `user`.
  `primary` is the nearest directory at or above the project root holding
  `mars.toml` (Mars's own walk-up), scope `project`; otherwise the user root,
  scope `user`. `user` is the user root, or none when `primary` already is it.
- **Every Mars subprocess gets `--root <primary>`**, so Mars's own walk is a
  no-op (the root-less `--help` capability probe aside). The one exception is
  `meridian mars init` / `init --link`, which
  targets the literal project root so a nested Mars project can still be
  created in a repository subdirectory.
- **`ensure_user_mars_root()`** runs `mars init --root ~/.meridian/mars` under
  a Meridian lock before any Mars call against the user root, and writes only
  there. With no `targets` and no `managed_root`, the user root routes
  unrestricted, so every installed harness is eligible: the user's "all
  harnesses possible".
- **Configuration comes from `primary` alone**: targets, aliases, models cache,
  bootstrap docs and Mars defaults. A project's target restriction is never
  bypassed by user content. Meridian's own `[primary]` precedence still wins
  over Mars's defaults.
- **Content layers**: project `.mars/` → user `.mars/` → built-in (a launch
  with no profile, plus Mars's fallback-only aliases). Project packages win.
- **Populating user scope:** `meridian mars --user add <source>`, then
  `--user sync`. `--user` injects `--root ~/.meridian/mars`.
- **Mutation gate.** `add`, `remove`, `link`, `unlink`, `sync`, `upgrade`,
  `rename`, `override`, `adopt`, `version`, `repair`, `resolve` and
  `models alias` with scope `user` and no `--user` fail with
  `no project Mars root here; run 'meridian init --link', or use 'meridian mars --user …'`.
  The gate keys on Mars scope, not identity tier, so a repository subdirectory
  still mutates the repository's `mars.toml`. It runs before root injection, so
  Mars's auto-init on `add` / `link` never writes `mars.toml` into a bare
  directory. `version`, `repair`, `resolve` and `models alias` joined the
  original nine after review (`spawn:p7868`): with `--root primary` always
  injected, they would otherwise have reached the user root from a bare
  directory, and `mars version` commits and tags.

**The `--fallback-root` contract (mars-agents).** Mars read agents and skills
from one root, so a user agent was invisible inside a project with its own
`mars.toml`. The smallest change that keeps the project the only configuration
authority is a lookup-only second root:

- `mars build launch-bundle`, `mars agents [list|show]` and
  `mars skills [list|show]` accept `--fallback-root <DIR>`.
- Missing agents and skills come from that root's canonical `.mars/` store.
  Primary content shadows fallback content. Harness skill variants resolve
  beside whichever skill was selected.
- Config, aliases, caches, targets, MCP refs and native manifests come only from
  the primary root, so an alias in a user agent's `model:` resolves against the
  project.
- A nonexistent or manifest-less fallback is ignored, because callers may pass
  the user root before it is initialized. A corrupt or unreadable fallback lock
  warns and skips the fallback; a primary lock error stays fatal.
- The bundle reports `agent_source_path` and each skill's `source_path`, and
  list/show entries carry `scope` (`root` or `fallback`) and `source_path`.
  Meridian treats these paths as authoritative, so skill paths, snapshot replay
  and sandbox projection point at the user root's files. The bundle schema
  version stays 4.

**The pin, and the probe behind it.** The `mars-agents==0.16.0` pin is what
delivers a capable Mars; see the shipping order below. The probe guards every
other `mars` (an older install, a dev build): Meridian passes
`--fallback-root <user root>` only when `mars build launch-bundle --help`
lists it, checked once per executable per process (cached, 10 s timeout). Any
failure means unsupported. The probe replaced a planned version check because
the release version is set by the PR's label, so a comparison would have to
guess; help text is the contract. On an unsupported Mars, a user agent inside a
project with its own `mars.toml` fails with
`found in user packages; add it here: meridian mars add <source>`. A bare
directory never needs the flag, because there `primary` already is the user
root.

**`mars_fallback` reports activation, with three values** (`work:two-tier-identity`
I19). `context` and `config show` print `active` (project scope, Mars supports the
flag), `unsupported` (project scope, it does not) or `n/a` (user scope: the user
root is already `primary`). The first label was two-valued and keyed on the probe
alone, so in a bare directory it said the fallback was supported when nothing
would ever pass it (G4 review).

**Shipping order: Mars first, then the pin** (I20). Meridian installs `mars` from
PyPI through its `mars-agents` pin, and #575 had not changed the pin
(`0.15.0`). With the genuine 0.15.0 the probe is false, so user agents inside a
project with its own `mars.toml` would have failed with the hint. "Merge order
does not matter" was true of the safety net, not of the shipped behaviour. The
order became: merge mars-agents #173 → publish 0.16.0 → bump the pin and
`uv lock` on the Meridian branch → re-gate → merge #575. The earlier smokes had
passed only because a dev build had been copied over `.venv/bin/mars`, which
also rewrote uv's shared cache
([lesson](../lessons/dogfooding-pr-builds.md#copying-over-venvbin-rewrites-uvs-cache)).

**User agents resolve their model against the project** (I16, confirmed with
mars 0.16.0). Because configuration comes from `primary` only, a user agent
reached through the fallback takes its `model:` alias from the project. The gap
is narrow: package aliases such as `luna` are defined in the package's own
`mars.toml` (`meridian-base/mars.toml` `[models.luna]`), so they
exist wherever that package is added. A user-only agent fails only when the
project lacks the package that defines its alias, none of its `model-policies`
fallbacks resolve there, and no `--model` is passed; otherwise Mars warns and
falls back (`opus46` → `opus` on Claude). Meridian appends a hint naming the
cause and the two fixes (add the package, or pass `--model`). Earlier smoke
failures that looked like this gap were test error: Claude-model agents
launched through `meridian opencode`, which can never match them.
*Not adopted (left open):* layering aliases from the fallback root, with project
aliases winning. It would mean a second config load in Mars, which the
`--fallback-root` contract deliberately avoided.

**Rejected alternatives:**

| Alternative | Why rejected |
|---|---|
| `~/.mars` as the user root | Mars has no such notion, and a scratch `MERIDIAN_HOME` would not isolate it |
| Per-agent `--root` switching (run a user agent against the user root) | Uses user targets and aliases, bypassing the project's target restriction |
| Rootless `mars models` | Unneeded: the user root is always ensured before Mars runs against it |
| Root selection only, no layering | Leaves user agents unreachable inside every project with its own `mars.toml`; recommended against as OQ1 and the user took the default "ship the fallback, project packages win" |
| Detecting the capability by version number | The shipped version is set by the release label; `--help` is reliable (lane C report). The pin now fixes the version Meridian ships with; the probe covers every other binary |
| Shipping #575 on the old pin and relying on the probe | Every project-scope launch of a user agent would have taken the unsupported path until a later pin bump |
| `--global` as the flag name | `--global` already means "across all projects" in `doctor --global` |

**Accepted risks.** User packages are global mutable state: `--user upgrade`
changes every ad-hoc place at once, while a project `mars.toml` pins its own.
Mars's content walk-up can find an unrelated ancestor `mars.toml` (for example
`~/mars.toml`); that was already Mars behaviour, and `context` now shows the
resolved roots.

Current mechanism: [Mars user scope](../architecture/mars-user-scope.md).

---

## Related

- [decisions/model-resolution.md](model-resolution.md) — Mars alias authority, how aliases flow into resolution
- [launch.md](launch.md) — composition pipeline, harness adapters
- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — package model mechanism
- [concepts/package-management/sync-model.md](../concepts/package-management/sync-model.md) — sync pipeline mechanics
- [concepts/package-management/self-source-selection.md](../concepts/package-management/self-source-selection.md) — declared-package source-selection mechanism
- [architecture/mars-compiler.md](../architecture/mars-compiler.md) — compiler internals
- [architecture/mars-user-scope.md](../architecture/mars-user-scope.md) — user Mars root, root resolution and content layering
- [architecture/mars-targeting.md](../architecture/mars-targeting.md) — targeting architecture
- [architecture/mars-launch-bundle.md](../architecture/mars-launch-bundle.md) — launch-bundle system
- [concepts/skill-schema.md](../concepts/skill-schema.md) — skill schema and variant lowering
- [concepts/bootstrap-docs.md](../concepts/bootstrap-docs.md) — bootstrap doc discovery mechanism
- [concepts/native-config.md](../concepts/native-config.md) — harness override passthrough concept
