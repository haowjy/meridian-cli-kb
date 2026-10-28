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

> **Superseded:** D98 replaces the hook half of this decision, and
> [D103](#d103-collision-precedence-and-hook-order-use-shortest-dependency-depth-2026-10-24)
> replaces the MCP precedence with shortest dependency depth first (released in
> mars 0.19.0). Local still wins and declaration order still breaks
> ties at equal depth.

**Original decision:** MCP server and hook name collisions resolve via: local
(`_self`) beats dependency; among dependencies, declaration order in
`mars.toml [dependencies]` wins; alphabetical source name breaks ties within
the same scope.

**Current scope:** That precedence remains the MCP rule. Hooks now follow D98:
selected dependency collisions are errors, and the project-local overlay may
shadow a dependency.

**Why:** Users already understand "local shadows dependency" from agent behavior. Applying the same model required zero new concepts. For MCP, aborting on every collision was overly strict — same-name servers from different packages have an unambiguous winner.

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

**Known gap (predates this decision):** a fork passes the source spawn's resolved skill names to Mars as extra skills, which Mars dedupes. Tracked in `src/meridian/lib/launch/.context/FUTURE`. The other gap, a snapshot that kept only the rendered available-skills text, is closed in meridian-cli v0.16.0 (PR #786), whose snapshot stores the entries.

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
`execution_policy.native_config`; Meridian applies it on Codex and reports it not applied
on other harnesses ([D-native-config-codex-only](bundle-field-consumption.md#d-native-config-codex-only)).

**Why:** Mars can't track every harness version's config schema. The escape hatch gives profile authors direct access; universally useful knobs get promoted to first-class fields later.

---

### D83: Portable tool policy preserved through the full pipeline (mixed allow/deny)

**Decision:** `tools` (map format with allow/deny entries) and `disallowed-tools` are
portable Meridian semantics. Mars resolves them into `tools.rules[]`, one typed entry per
statement: the canonical `capability` carries meaning, while `native` carries the resolved
harness spelling for emission. Meridian reads those two fields in those directions and
keeps no second capability-to-native name table. The derived `allowed`, `disallowed`, and
`mcp` lists remain temporarily for older consumers, but cannot express `ask`, `*`, or the
capability that produced a name; `mcp` holds permission names, never MCP server
definitions. The list form `tools: [a, b]` is deprecated and only grants; `'*': deny` in a
map is the allowlist ([D-permission-list-form-deprecated](tool-permissions.md#d-permission-list-form-deprecated)).

Mars also emits profile-independent `tools.capability_natives` entries for the `agent` and
`task` capabilities in every bundle. Meridian still owns the decision to impose its
platform rules; it gets the target harness names for those rules from this map. Both fields
share Mars's one native table and status projection, including older-build names such as
Claude's `TaskOutput`. Each rule also carries `native_tools`, the harness tools it
concerns, because a permission spelling can name a different tool (Claude `Edit(src/*)`
for a scoped `write`). The typed rules and `capability_natives` shipped in Mars 0.17.0
(mars-agents PR #179); Meridian reads them from v0.12.0 (PR #730).
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

**Gating:** Auto-rename applies only to same-kind (Agent|Skill) groups where every collider has a distinct source. Same-source collisions, mixed-kind groups, and hook/MCP/bootstrap collisions remain hard errors. A project-local hook may shadow a dependency hook; two selected dependency hooks remain a hard error (D98).

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
([lesson](../lessons/dogfooding-pr-builds.md#writing-over-venvbin-rewrites-uvs-cache)).

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

### D98: The target build is the only hook-selection authority (2026-10-24)

**Status:** Owner-settled in `work:harness-plugin-export`; released in mars
0.19.0. Its temporary MCP exception is superseded by D99. Since mars 0.20.4, an
unmanaged hook directory no longer fails the sync; see
[D104](#d104-packaged-mcp-servers-install-through-the-hook-directory-path-2026-10-27).

**Decision:** The target build selects one parsed hook per target-scoped
canonical destination after dependency filters, exported visibility and
collision handling. It carries that hook, its source package and ordering
metadata in `TargetState`. Config-entry preflight, merge-mode lowering and
file-mode lowering consume this value; they do not rediscover hooks or choose a
second winner.

Two selected dependencies providing the same hook remain a hard collision. A
project-local hook overlays a dependency hook and emits the existing shadow
warning. D99 later moved MCP discovery into the same target state while retaining
D36's collision policy in the per-target lowerer.

**Why:** Installation and config-entry discovery previously answered "which
hooks does this project have?" independently. Dependency filters applied only
to installation. A filtered provider could therefore win lowering, fail the
installed-directory check, suppress the provider that was actually installed,
and emit a false warning. Copying filter predicates into config entries would
leave two mechanisms to drift again; looking up the provider from an installed
directory would make selection unavailable to plugin builds, which lower into a
fresh tree with no project install.

`TargetState`'s selected hook is intent, not installation proof. If an unmanaged
canonical destination prevents installation, the selection survives so sync's
post-target-sync directory/hash check still emits the genuine not-installed
warning. Plugin builds instead provide their plugin-root hook path to the same
pure lowering function.

**Rejected alternatives:**

- Duplicate the dependency filter in config-entry discovery — fixes one repro
  while preserving two selection authorities.
- Reconstruct the winner from `mars.lock` or an installed directory — no value
  exists before installation, so plugin builds cannot use it.
- Keep hook collision precedence in `config_entries/resolve.rs` — selected
  dependency collisions already fail during target construction and local
  overlay already chose the winner; another loser-warning path is unreachable.

Current mechanism: [Mars compiler architecture](../architecture/mars-compiler.md#config-entry-selection-and-collisions).

---

### D99: MCP selection joins target state (2026-10-24)

**Status:** Owner-settled in `work:harness-plugin-export`; released in mars
0.19.0. Its MCP collision ranking is superseded by
[D103](#d103-collision-precedence-and-hook-order-use-shortest-dependency-depth-2026-10-24)
(shortest depth first). Since mars 0.20.4, collisions resolve once per target
during target construction, so a server's files, entry and lock name the same winner
([D104](#d104-packaged-mcp-servers-install-through-the-hook-directory-path-2026-10-27)).

**Decision:** Target construction is the only package-discovery and visibility
authority for effectful config entries. Alongside D98's hooks, it discovers each
source's MCP manifests once and carries every eligible parsed MCP item plus its
managed-target scope in `TargetState`. Config-entry preflight and lowering consume
that state instead of rediscovering MCP manifests.

MCP collision handling does not move. Per-target lowering keeps D36's first-wins
behavior and warning text: project `_self` precedes dependencies, and dependency
precedence comes from `mars.toml` declaration order rather than alphabetical
`graph.order`. Candidates with no managed target and discovery failures remain in
target state so environment and discovery warnings occur in the same phases and
order as before.

Hook and MCP manifest loaders parse visibility into one `Visibility` enum. An
unknown value warns and becomes local, preserving its previous package-boundary
effect. Downstream selection compares only with `Visibility::Exported`.

**Why:** MCP had the same split-authority problem D98 removed for hooks: preflight
and compile each scanned the resolved packages and applied visibility separately.
A plugin build needs the flattened MCP set before it has a project installation,
and a second discovery path would let sync and build disagree. Carrying the parsed
candidates in target state gives both consumers the same value without changing
collision or diagnostic behavior.

**Kept open:** A dependency `exclude = [...]` filter continues to admit every MCP
server from that source. Applying exclusion by server name is a separate
user-visible policy decision, not part of this structural change.

**Rejected alternatives:**

- Let preflight and compile continue to discover MCP independently — leaves two
  authorities for the plugin build to reproduce.
- Resolve MCP collisions during target construction — unnecessary for a shared
  selection seam and risks changing established first-wins warnings.
- Interpret `exclude` as MCP names during the move — combines a behavior change
  with a selection refactor and silently changes existing consumers.

Current mechanism: [Mars compiler architecture](../architecture/mars-compiler.md#config-entry-selection-and-collisions).

---

### D100: A package publishes one plugin holding its full resolved graph (2026-10-24)

**Status:** The one-plugin, full-graph rule is owner-settled
(`chat:c9019`); the harness set and release defaults are current choices
(`chat:c9182`, see below). Shipped in Mars 0.19.0 (PR #197). The
OpenCode agent clause below was revised by mars-agents#249 (Mars 0.23.0).

**Decision:** A package with a plugin builds **one plugin** under its own
name, holding exactly what `mars sync` installs in the package's own
checkout: its own agents and skills plus its dependencies', under the
package's `[dependencies.x]` filters, renames and visibility rules.
Dependency items share the building package's namespace: a meridian-base
skill ships in meridian-dev-workflow's plugin as
`meridian-dev-workflow:<skill>`. The plugin declares no plugin
`dependencies`, and the catalogs list no dependency plugins. An author who
wants to ship less narrows the graph with the filters a mars consumer already
uses; there is no plugin-only selection rule. The owner put it as "we just
literally flatten meridian-dev-workflow + meridian-base and ship it to the
plugin (like what cw does right now...???)", meaning creative-writing-skills'
hand-built plugin, which already ships dependency content inside one plugin.

`mars version` rebuilds the plugin at the new version and commits it with
the release; `--no-plugins` skips that, and `mars check` then reports the
output as stale. `mars build plugin` is the same build for development and
CI. Usage: mars-agents `docs/plugin-authoring.md`.

**Harnesses and trees.** One logical plugin is written as up to three
physical trees under `<dir>/` (default `plugin/`):

| Tree | Read by | Why it is separate |
|---|---|---|
| `claude` | Claude Code, Codex, Grok | Codex and Grok install a Claude-format tree unchanged |
| `cursor` | Cursor | Its own manifest, `${CURSOR_PLUGIN_ROOT}` in hooks, its own hook event names and `mcp.json` |
| `opencode` | OpenCode | OpenCode loads only a JavaScript module |

The loader facts behind this split are in
[harness-plugin-loading.md](../research/harness-plugin-loading.md). The
owner widened the scope from Claude-first: "it should be for claude, codex,
opencode, and cursor (cursor plugin support because thats how it works with
grok i think)". Probes then showed Grok reads the Claude tree, not the
Cursor one, so Grok became a fifth harness on the Claude tree. The tech lead
made `[plugin] harnesses` default to all five.
The owner approved that default on 2026-10-26, and decided that a harness
added in a later release joins it too (mars-agents#211): mars exists to make
packages easy for non-technical users and to widen distribution, so an
author who doesn't list `harnesses` gets every supported harness. Listing
`harnesses` opts out. A release that adds a harness must say so in its
release notes.

- **OpenCode ships agents; Codex doesn't.** The first build shipped OpenCode
  skills, MCP and hooks only, on the belief that no OpenCode API adds an
  agent, and the owner accepted the loss: "just having skills mcp and hooks
  but no agents is alright", because "most of the agent definitions should
  actually BE in the skills rather than the agent bodies (agents become
  wrappers around a collection of skills + selected model for the job + i
  guess permissions)". That principle governs how prompt packages are
  written, not mars. A later source read of OpenCode 2.2.0 found that
  `ctx.agent` creates agents, so since mars-agents#249 (Mars 0.23.0) the
  generated `index.mjs` registers them; their model rule is
  [D106](#d106-opencode-plugin-agents-keep-opencodes-default-model-when-the-model-is-ambiguous-2026-10-28).
  Codex plugins still can't carry agents, and the owner chose to wait for
  upstream (mars-agents#256).
- **The OpenCode package is the repo itself**: one root `package.json`
  whose `main` and `files` point into `<dir>/opencode/`, installed with
  `opencode plugin add git+…`, not published to npm. The owner's answer was
  "repo root I think? what does that mean?"; the tech lead chose this form.
- **Cursor ships on its documentation.** The owner has no Cursor account
  ("no, i only have grok"), and Cursor refuses plugin commands without one.
  The generated manifest and catalog pass Cursor's published JSON schemas;
  nothing has run on the binary.
- **Support is published as tables generated from the build's own data.**
  The owner asked for "a few tables in the README of mars listing what is
  supported where". The mars-agents README's component-support and
  verification-level tables render from the same harness and tree tables
  that drive the build's diagnostics, and a test fails when the README
  disagrees. Support state (supported, dropped with warning, impossible) is
  kept apart from verification level (live, install-only, docs-only), so
  Cursor's "supported" can't read as tested.
- **Each harness with a binary was checked live on built plugins**: Claude,
  Codex, Grok and OpenCode installed the built output and ran skills with
  resources, agents where loadable, hooks and MCP. The owner's bar was "we
  just need to make sure it works".
- **Hooks reach a tree only through the fragment written for that
  harness.** Mars never translates one harness's hook events into another's,
  exactly as `mars sync` installs them; a hook with no fragment for a built
  tree is omitted with a warning.

**Diagnostics are warnings unless publishing would be unsafe.** Asked about
failing the build on leftover Meridian and Mars terms, the owner said "it
should just be a warning tho... also... meridian will become an app soon
lol": a plugin may run where neither is installed, so the author should see
the text, but the author decides. The tech lead extended this to every
content and support finding. Errors are kept for configuration, IO, write
safety, publishing metadata, and graphs `mars sync` would also refuse.

The rest follows from the plugin being a published artifact rather than a
project install:

- **Publish-lens resolve.** The build resolves through the package's
  consumer-facing config and never reads `mars.lock`, `mars.local.toml` or
  `.mars/`. Output that depended on untracked or local files could not be
  checked for staleness from a clean clone.
- **Only exported hooks and MCP servers ship.** Hooks and MCP default to
  local visibility, so a package's own development hooks stay home.
- **Models come only from the package's `[models]`**, never from the
  machine's harness catalog, so two machines build the same bytes.
- **The build shares sync's pipeline.** It flattens the graph through the
  same target-state collection, filters, renames and validation that sync
  uses, then lowers through the existing per-harness lowerers. It never
  installs into a temporary project, which would pull in the lock, ownership
  records, host probes and absolute hook paths.

**Contracts other code must respect.** These make a committed plugin safe
to rebuild, check and release. Their mechanism is documented beside the
code in mars-agents `src/build/AGENTS.md`, `src/build/.context/CONTEXT.md`,
`src/config/AGENTS.md` and `src/discover/.context/CONTEXT.md` (on the
`release/v0.19.0` line until it ships).

- **A directory holding a valid `.mars-plugin.json` is generated output,
  never authored input**, at any depth in any package. Discovery, MCP
  discovery and staging skip it. Without the rule, a plugin committed inside
  a package became package input on the next build and for every consumer.
- **Published output never leaves the package.** A symlink whose target is
  outside its package, or a special file, fails the build; an in-package
  symlink is copied as a regular file. Only what will be published is
  walked, so ordinary tooling links (`.venv`, `node_modules`) elsewhere in
  the checkout don't matter. Otherwise a dependency could publish the
  builder's environment or private files.
- **Mars owns `<dir>` whole and at most three files outside it**: the Claude
  catalog, the Cursor catalog and OpenCode's root `package.json`. One
  sentinel at `<dir>/.mars-plugin.json` records file hashes, the input
  fingerprint and resolved dependency versions. Its `root_files` may name
  only those three paths, and any other key is an error: the sentinel is
  committed and rarely reviewed, and trusting its keys let a forged entry
  delete any file. A root file is replaced only when it is absent, matches
  its recorded hash, or already holds the bytes being written; anything else
  is refused. `--out` builds write no root files.
- **Plugin-only metadata never blocks other commands.** `[plugin]` and the
  `[package]` keys `authors`, `homepage` and `license` stay raw when
  `mars.toml` loads; the plugin build validates them once. So a malformed
  author or unknown harness can't break `mars sync` or `mars list` for the
  package or its consumers, and `config::save` writes back what was read.
  When these values were typed at load, every new validation rule became a
  way for publishing metadata to break ordinary config loading.
- **A release rolls back through git.** The commit being released from is
  the snapshot: before writing, every touched path outside `<dir>` must be
  tracked and clean at HEAD, or absent, and rollback never deletes an
  untracked file it didn't create. `mars version` releases the output lock
  before committing, so a pre-commit hook that runs `mars build plugin` can
  rebuild identical bytes instead of hanging.
- **The input fingerprint is the authored view, recomputable offline.** It
  hashes the root package's own sources as staging reads them (generated
  trees and catalogs skipped, text normalized to LF) plus the
  output-affecting `mars.toml` tables, with `version` removed from
  `[package]`. It never resolves dependencies, so `mars check` can call a
  committed plugin stale without the network; the sentinel's dependency
  versions cover dependency releases.

**Why one plugin of the whole graph:** An earlier design built one plugin per
graph package, linked by plugin `dependencies`, and chose which dependency
items to ship by following references in prose. The owner corrected both:
"no, we don't need the meridian-base plugin... right?" The graph is what a
mars user gets, so it should be what a plugin user gets; prose references are
an unreliable second definition of content. Plugin dependencies also fail on
the harness side: Codex ignores them, and a Claude cross-marketplace
dependency installs only when the user has already registered and allowlisted
the other marketplace.

**Not centered on cw.** creative-writing-skills is the only existing hand-built
plugin, and the first design took its needs (extras, standalone variants,
adopting a hand-built tree) as requirements. The owner pushed back on
centering the design on cw. Those features are out; cw can migrate to this
build later.

**Rejected alternatives:**

| Option | Why not |
|---|---|
| One plugin per graph package, linked by `dependencies` | Owner correction; Codex ignores dependencies, and Claude cross-marketplace dependencies need the user to register the other marketplace first |
| Select dependency items by following references | Owner correction: the graph defines content |
| Plugin extras, standalone variants, `--adopt` | cw-specific; a hand-built tree is replaced in a reviewed commit instead |
| One tree for every harness, with Cursor and OpenCode overlays | Cursor's hook token, hook file and agent lowering differ, and OpenCode needs JavaScript anyway |
| Translating Claude hook events into OpenCode events | Guesswork with no `SubagentStop` equivalent, and sync doesn't do it |
| `mars build plugin` as the only entry point | Releases would ship stale plugins |
| `mars sync` builds and enables plugins | Codex needs a user-cache install and Claude an absolute marketplace path, so a project cannot enable a plugin by itself |
| Configured output under `dist/` | Commonly gitignored, and marketplaces install from the tagged commit. `--out dist/…` stays valid for CI |
| A hand-built rollback that keeps the previous tree until the tag | Git already holds the previous state at HEAD; a second copy needed its own crash-safe restore ordering, which is where the bugs were |
| Typing plugin metadata when `mars.toml` loads | Let publishing metadata break sync and list, and made `config::save` lossy |
| Sentinel key inside `marketplace.json` | Claude validates nested objects strictly |
| Reuse the `mars export` name | It already means compile-plan JSON |

> [!FLAG] **Needs human review**
> - **Does a new harness join the default `harnesses`?** Today the default
>   is all five; adding a sixth harness would silently add a tree to every
>   package that relies on it. Open as
>   [mars-agents#211](https://github.com/haowjy/mars-agents/issues/211).
> - **Choices built as recommended without an explicit yes.** The owner
>   was offered the release defaults (`dir = "plugin"`, name and
>   marketplace from the package, all five harnesses, `--no-plugins`), the
>   `[package]` `authors`/`homepage`/`license` keys, mars owning the root
>   files, a `mars check` warning for a `requires-mars` floor below 0.19.0,
>   and shipping dependencies' exported hooks. The replies were "accept
>   what" and "we just need to make sure it works...", which the tech lead
>   took as acceptance. The warnings-not-errors extension beyond term
>   checks is likewise unconfirmed.

**Known gaps** (mars-agents issues): [#198](https://github.com/haowjy/mars-agents/issues/198), OpenCode hook fragments in the
named-export shape `mars sync` documents don't load; [#199](https://github.com/haowjy/mars-agents/issues/199), the OpenCode sync
adapter may write MCP under a key OpenCode doesn't read (found by reading
source, not yet run); [#201](https://github.com/haowjy/mars-agents/issues/201), a dry run leaves an untracked `.mars/`; [#202](https://github.com/haowjy/mars-agents/issues/202), root
files and the output directory publish in two steps, so a crash between them
leaves them out of step until the next build; [#204](https://github.com/haowjy/mars-agents/issues/204), no portable token for MCP
server files shipped in the package; [#210](https://github.com/haowjy/mars-agents/issues/210), structural cleanups in the build
(two tree registries, README prose that can drift from support state).

**Revisit when:** a harness can't be served by one of the three trees (a
Codex-only hook needs a Codex tree); Codex starts loading plugin agents; a
live Cursor run contradicts its docs; or a harness changes how it loads a
shared Claude tree.

**Provenance:** `work:harness-plugin-export`; `chat:c9019` (one plugin of
the graph), `chat:c9182` (harness scope and release defaults); live
verification `spawn:p9533`, `spawn:p9535`; review rounds `spawn:p9511`
through `spawn:p9542`.

---

### D101: Agent skills lower by harness capability (2026-10-24)

**Status:** Current choice. The owner raised the idea and agreed to the
lead's form of it ("yeah, I think load these skills is acceptable",
`chat:c9019`). Merged into mars-agents `release/v0.19.0` through PR #194,
ahead of the multi-harness plugin build; it ships in 0.19.0 through PR #197,
so no published plugin carries the old merged lists.

**Decision:** An agent's `skills: {load, available}` lowers per harness from
one policy in the per-harness agent lowerer, used by both `mars sync` and
plugin builds:

- `load` goes into the native `skills:` field where the harness preloads it
  (Claude and Cursor subagents). `available` never goes there.
- Where native preload can't carry the intent, one generated paragraph is
  appended after the authored body: `available` skills on every harness, `load` skills where no native
  preload exists (Codex, OpenCode, Pi), and `load` skills on Claude as well.
  Its text is "Load these skills before starting, unless they are already in
  your context: /a, /b. Load these skills when the task fits: /c, /d."
- In a plugin, the reference pass qualifies those `/name` tokens like any
  other reference.

Meridian launch bundles are unchanged: they still read `load` and
`available` from the canonical profile.

**Uncertain:** whether models act on the line is barely measured. In one
isolated run, Claude Haiku as session agent saw the line and did not load
the skill; no other model or run has been checked.

**Why:** Before this, Claude and Cursor lowered `load ∪ available` into
`skills:`, so subagents preloaded skills meant to be optional, and Codex,
OpenCode and Pi agents got no skill guidance. Claude needs the line too
because one agent file serves as a subagent, which preloads `skills:`, and as
a session agent, which does not; "unless they are already in your context"
covers both.

**Rejected:** inlining `load` skill text into agents where the harness has no
preload. Every harness Mars targets (Claude, Codex, Cursor, OpenCode, Pi)
already discovers skills natively, so `/name` resolves. Inlined text breaks skill
`resources/` paths, which are relative to the skill directory and which
`${CLAUDE_PLUGIN_ROOT}` does not reach inside agent bodies; it also copies
each skill into every agent and goes stale against it. Inlining stays the
answer only for a harness with no skill mechanism, and none exists today.

---

### D102: Renamed agents and skills answer to their installed name (2026-10-24)

**Status:** Current choice, made by the tech lead at the plugin-build review
gate (`work:harness-plugin-export`). Merged into mars-agents
`release/v0.19.0` through PR #193 and ships in 0.19.0 through PR #197; it
changes `mars sync` output too.

**Decision:** When an explicit `rename` or a cross-source collision
(`coder` → `coder__base`) renames an agent or skill, its frontmatter `name:`
becomes the installed name, and `subagents:` and `skills:` references follow.
Native-agent sync deletes every native agent file it recorded as its own in
an earlier sync and did not emit this time, so the pre-rename `.claude/agents/coder.md` is removed (files Mars
didn't create are never touched).

**Why:** Both collision copies used to keep `name: coder`, and Claude loaded
only one of them. A plugin makes this worse, since every item shares one
namespace. Agent Skills also requires a skill's `name` to match its directory.

**Consequence:** a renamed agent no longer answers to its old name. When two
dependencies both ship `coder`, `meridian spawn -a coder` stops resolving and
callers must use `coder__<source>` or add an explicit rename. User-facing
note: mars-agents `docs/internals/conflicts.md`.

---

### D103: Collision precedence and hook order use shortest dependency depth (2026-10-24)

**Status:** Current choice, made by the tech lead at the plugin-build review
gate. Merged into mars-agents `release/v0.19.0` through PR #193 and ships
in 0.19.0 through PR #197. Supersedes D36's MCP precedence and D99's "MCP
collision handling does not move".

**Decision:** A package's depth is its shortest distance from the consumer:
the project or building package itself (`_self`) first, then direct
dependencies, then their dependencies. Among MCP servers with the same name
for one target, the shallowest package wins; at equal depth, the earlier
`[dependencies]` entry it was reached through wins; alphabetical source name
breaks remaining ties. Hooks use the same depth for ordering. Dependency hook
collisions stay errors (D98). The resolver derives depth once
(`ResolvedGraph::dependency_depths()`); sync, lowering and the plugin build
read it rather than reconstructing it.

**Why:** D36 ranked only `_self` above dependencies and every dependency by
declaration order, so a dependency's MCP server could beat a server from a
package that depends on it. Review found exactly that: a package lost its own
server to its dependency's, in both the plugin build and sync. Depth fixes
that, but only as the *shortest* distance: a first-found depth made a direct
dependency that was also reached transitively count as depth 2 and lose to a
later-declared sibling. Hooks already sorted by depth, and one meaning of
depth for both beats a second sort key.

**Visible effect:** hook order changes only for a direct dependency that is
also reached transitively, and in that case Codex may ask once to re-trust the
moved hooks.

---

### D104: Packaged MCP servers install through the hook directory path (2026-10-27)

**Status:** Current choice. The owner asked for `${MARS_MCP_DIR}` (mars-agents
#204) and approved its two tests ("sure, add it, the mars snyc infinite hang
sounds bad"); the shape is the tech lead's, from two code reviews and three
structure reviews. Shipped in mars 0.20.4 (PR #233); the follow-ups to
selection (#227, #228) shipped in 0.20.5 (PR #238). Mechanism: mars-agents
`src/sync/AGENTS.md`, `src/compiler/AGENTS.md`, `src/build/AGENTS.md`,
`docs/config/mcp-and-hooks.md` and `docs/internals/sync-pipeline.md`.
`work:post-0-19-followups`.

**Decision.**
- `${MARS_MCP_DIR}` is a literal placeholder a package writes in a server's
  `mcp.toml`, like `${MARS_HOOK_DIR}` for hooks. A server whose `command` or
  `args` names it gets its `mcp/<name>/` directory installed beside each target that writes MCP config,
  and its entry carries the installed absolute path. A plugin build maps the
  token to the harness's plugin root. A token-free server is a config entry
  alone, as before.
- Hooks and token-using servers share one install path. `PackageDir`
  (`src/sync/package_dir.rs`) owns every path and lock name for their
  directories; selection keeps one record per target; and mars writes a hook's or
  server's entry for a target only if target sync recorded installing its
  directory there in this run.
- One unmanaged-path rule for both: anything mars didn't create at a hook's or
  server's directory warns, is left alone, and withholds that target's entry.
  `mars sync --force` adopts the unmanaged path. Hooks used to fail the sync with
  exit 2 here.
- The tree hash rejects non-regular files with an error naming the path.
- Two fixes to the same selection code (PR #238, mars 0.20.5): a dependency's include filter
  (`agents`, `skills`, `only_skills`, `only_agents`) never narrows its MCP
  servers; and in a plugin build, the building package keeps its own agent and
  skill names when a dependency uses the same one, so only the dependency is
  suffixed.

**Why.**
- *One install path:* without it, hooks and servers handled the same conflict
  differently (a fail for hooks, a warning for servers), and every later fix
  would have had to land twice.
- *Directories only for token-using servers, only in MCP targets:* the first
  version made every server a directory item in every target. Both reviewers
  traced their findings to that one choice: it copied directories nobody
  referenced, into targets like `.pi` that write no MCP config.
- *Warn and withhold, not fail:* a directory the user put there is theirs. A
  warning that names the path, plus `--force`, lets the rest of the sync proceed
  without overwriting it; the cost is that an unmanaged hook path no longer
  stops the sync.
- *Reject non-regular files:* a FIFO inside a skill, hook or server directory
  hung `mars sync` forever while hashing.
- *Filters never narrow servers:* a filter names agents and skills, never a
  server, yet any include filter silently dropped every server the dependency
  declared (#228). The cost: a newly visible server with the same name as an
  entry the user wrote by hand replaces it, as it already did for unfiltered
  dependencies (#235). `exclude` still can't drop one server (#237).
- *Root package keeps its names:* the building package's own agents and skills
  were renamed whenever a dependency shared the name (#227), so the package's
  own agent lost its name. Sync already shadowed the dependency.

**Rejected:**
- *A directory item for every server in every target:* the first round; see
  above.
- *Keep exit 2 for unmanaged hook paths and add a second rule for servers:* two
  policies on one install path.

---

### D105: Meridian hosts own full-name namespaces under `harness-overrides` (2026-10-27)

**Status:** Current choice; the full names are the owner's call. Shipped in
Mars 0.22.0 (mars-agents#248). meridian-cli reads only the new namespace from
meridian-cli#790, merged into `release/v0.17.0` with Mars 0.23.4 bundled; it
reaches users in meridian-cli 0.17.0 (integration PR #798).

**Decision.**
- `harness-overrides.meridian-cli` and `harness-overrides.meridian-flow` are
  consumer namespaces: Mars reserves them, carries them through, never
  interprets them, and never routes to them, because neither host is a
  launch harness. They sit beside the harness blocks of
  [D82](#d82-harness-overridestarget--exact-passthrough-not-portable-semantics)
  without becoming harness targets.
- meridian-cli's Agent-level runtime knobs live only in its namespace:
  `harness-overrides.meridian-cli.resident-rearm-budget` and
  `harness-overrides.meridian-cli.capabilities` (for example
  `{spawn: false}`). The top-level `resident-rearm-budget` and
  `meridian-capabilities` are `field.removed`: an error naming the new key.
- Mars's tool vocabulary keeps one native tool table per host; Meridian Flow's is
  `natives["meridian-flow"]`.
- The short names `harness-overrides.meridian`, `harness-overrides.flow` and
  `natives.flow` are `field.removed`. `mars migrate` renames them in place,
  keeping comments; when a file has both spellings it merges them key by
  key, the new block wins, and a dropped `permission` or `capabilities` entry
  is reported as `field.restriction-lost`, the warning Mars gives whenever a
  rewrite leaves an Agent less restricted than its file said.

**Why.**
- *A namespace instead of top-level fields:* recognizing the two knobs as
  top-level Meridian-only fields would couple the shared schema to one
  host's runtime settings; the namespace and `[settings.meridian.*]` already
  had that job. No published package used either key.
- *Full names:* the owner asked "can we name the full thing for both", then
  chose "rename everywhere". `meridian` could mean the product family or the
  CLI, so an author reading `harness-overrides.meridian` could not tell which
  host it addressed; `flow` had the same problem for Meridian Flow.
- *meridian-cli reads only the new spelling:* its profile reader looks the
  block up with Python's `dict.get`, which returns nothing for a missing key,
  so a migrated package read by an unchanged reader would silently drop a
  `spawn: false` restriction. The reader change and the bump of meridian-cli's
  bundled Mars version ship in one meridian-cli release.

Mechanism: mars-agents `docs/config/schema-v1.md` and
`docs/config/migrating-to-schema-v1.md`; meridian-cli `docs/agent-profiles.md`
and `docs/upgrading.md`. `work:agent-definition-design`.

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
