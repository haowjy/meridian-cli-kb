# Decisions: Package Management

Decisions about the `.mars/` store, targeting architecture, compiler pipeline, agent and skill emission, the universal skill schema, and the bootstrap documentation model.

For mechanism, see:
- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — mars as a package manager
- [concepts/package-management/compiler-pipeline.md](../concepts/package-management/compiler-pipeline.md) — reader → compiler → target sync
- [concepts/package-management/targeting.md](../concepts/package-management/targeting.md) — native harness dir emission
- [concepts/skill-schema.md](../concepts/skill-schema.md) — frontmatter, lowering, variants
- [architecture/mars-targeting.md](../architecture/mars-targeting.md) — targeting architecture detail
- [architecture/mars-compiler.md](../architecture/mars-compiler.md) — compiler internals

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

**Decision:** The canonical SKILL.md frontmatter has six fields: `name`, `description`, `model-invocable`, `user-invocable`, `allowed-tools`, `license`, and `metadata`. A `compatibility` field proposed in early drafts was removed.

A `compatibility` field was rejected — would require Mars to understand harness semantics. `metadata` is pass-through for tooling use.

> **Partially superseded by D71.** `invocation` enum replaced by `model-invocable` and `user-invocable` booleans.

---

### D71: Skill invocability decomposed into two independent booleans: `model-invocable` and `user-invocable`

**Decision:** Replace the single `invocation: explicit | implicit` enum in SKILL.md frontmatter with two independent booleans: `model-invocable` (default: `true`) and `user-invocable` (default: `true`). Remove all legacy alias fields (`invocation`, `disable-model-invocation`, `allow_implicit_invocation`) as hard errors with no deprecation period.

**Why two booleans:** The old enum collapsed two orthogonal dimensions. `invocation: explicit` only ever controlled model-invocability — user-invocability was unmodeled. Hard errors (no deprecation) because no consumer reads `invocation` at runtime; migration is find-and-replace. Presence is determined by the agent profile's `skills:` list, not skill metadata.

**Compilation:** Claude Code supports both natively. Codex supports `model-invocable` only. OpenCode drops both. Pi/Cursor support `model-invocable` via `disable-model-invocation`.

---

### D59: Native skill projections always refreshed on sync with divergence warnings

**Decision:** `mars sync` re-emits every native harness projection (e.g., `.claude/skills/<skill>/SKILL.md`) unconditionally — no content-hash optimization to skip unchanged files. If a native projection has diverged from what sync would produce, Mars emits a divergence warning.

**Why:** Skipping unchanged projections would leave stale native files after format/lowering changes. Always-refresh keeps the invariant: after sync, native dirs are byte-for-byte identical to lowering output.

---

### D60: Skill variant specificity ladder: model-token+harness > canonical-id+harness > harness > base

**Decision:** When multiple variant files exist for a skill, selection follows a 4-step exact-match ladder:
1. `variants/<harness>/<requested-token>/SKILL.md`
2. `variants/<harness>/<canonical-id>/SKILL.md`
3. `variants/<harness>/SKILL.md`
4. `SKILL.md`

Exact match only — no glob or fuzzy matching.

**Why exact match:** Fuzzy matching would be non-deterministic with multiple globs. Two model-identity levels exist because authors may write either form (`variants/claude/sonnet/` or `variants/claude/claude-sonnet-4-5/`).

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
portable Meridian semantics. Mars resolves them into the bundle's `tools` field
(`{allowed: [...], disallowed: [...], mcp: [...]}`). Both `allowed` and `disallowed` are
preserved simultaneously — neither side is dropped. Meridian projects both sides per
harness.

**The c859 bug:** Prior Meridian Claude projection only emitted `--allowedTools` when a
wildcard `*: deny` was present. Mixed profiles without a wildcard deny silently lost the
allowlist. Fix: emit `--allowedTools` whenever `tools.allowed` is non-empty; emit
`--disallowedTools` whenever `tools.disallowed` is non-empty. No dependency between the
two decisions.

**Why separate from harness passthrough:** Tool policy is portable (Claude, OpenCode have
allow/deny surfaces). Encoding portable tool policy as target-native passthrough would require per-harness
duplication and lose portability.

---

### D84: Cursor experimental, Pi future-first-class, Gemini out of scope

**Decision:**
- **Cursor:** Supported as experimental launch-bundle target. Bundle carries
  `provenance.harness_stability: "experimental"` and a warning. Projection is
  best-effort; contract may change.
- **Pi:** Future first-class Meridian-owned harness. Pi contract is still being
  developed. Not part of the launch-bundle first slice. Tool policy should be part
  of the future Pi contract.
- **Gemini:** Not a current Mars/Meridian target. Excluded from launch-bundle and
  harness passthrough requirements.

**Why Cursor experimental vs first-class:** Cursor's config surfaces are actively
evolving. Committing to a stable projection contract prematurely creates maintenance
debt.

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

## Related

- [decisions/model-resolution.md](model-resolution.md) — Mars alias authority, how aliases flow into resolution
- [launch.md](launch.md) — composition pipeline, harness adapters
- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — package model mechanism
- [concepts/package-management/sync-model.md](../concepts/package-management/sync-model.md) — sync pipeline mechanics
- [architecture/mars-compiler.md](../architecture/mars-compiler.md) — compiler internals
- [architecture/mars-targeting.md](../architecture/mars-targeting.md) — targeting architecture
- [architecture/mars-launch-bundle.md](../architecture/mars-launch-bundle.md) — launch-bundle system
- [concepts/skill-schema.md](../concepts/skill-schema.md) — skill schema and variant lowering
- [concepts/bootstrap-docs.md](../concepts/bootstrap-docs.md) — bootstrap doc discovery mechanism
- [concepts/native-config.md](../concepts/native-config.md) — harness override passthrough concept
