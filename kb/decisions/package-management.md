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

**Why `.agents/` was confusing:**
- `.agents/agents/` was consumed only by Meridian — no harness auto-discovers it
- `.agents/skills/` was shared by some harnesses but caused skill overlap problems (same-name skill appearing in both `.agents/skills/` and the harness's own directory)
- The name implied cross-tool portability that didn't exist for agents

**What changed:** Meridian reads from `.mars/agents/` and `.mars/skills/`. Each harness's native skill directory (`.claude/skills/`, `.codex/skills/`, etc.) gets a copy. Agent emission to harness native directories is off by default in Meridian-managed contexts (`MERIDIAN_MANAGED=1`).

**Alternatives rejected:**
- Keep `.agents/` with better naming → same confusion, just renamed
- Emit skills only once to a shared layer → recreates the overlap problem with a different name
- Make harnesses read from `.mars/` directly → harnesses don't auto-discover `.mars/`; would require harness-level config changes outside Meridian's control

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

**Critical implementation finding:** `graph.order` in the dependency graph is **alphabetical**, not declaration order. Declaration-order precedence requires reading `mars.toml` directly, not trusting graph traversal order.

**Alternatives rejected:**
- Error on all collisions → too strict; breaks common same-name scenarios
- Merge collisions (combine env vars, args) → semantically ambiguous; MCP servers are atomic units

---

### D37: Windows hooks keep `bash` interpreter; adjust quoting per platform

**Decision:** Hook lowering keeps `bash` as the interpreter on Windows, adjusting only the quoting convention per platform. No dispatch-by-extension or WSL requirement.

**Why:** Hooks are shell scripts that Mars doesn't own — only the invocation. Git for Windows ships bash, which is the pragmatic assumption. Dispatch by script extension (.sh → bash, .cmd → cmd.exe, .ps1 → powershell) is over-engineered for V0 when all hooks are `.sh` files.

**Alternatives rejected:**
- Extension-based dispatch → over-engineered; adds runtime complexity for no current benefit
- Use `sh` instead of `bash` → hooks may use bash-isms
- Require WSL → too restrictive

---

### D38: `LockIndex` for O(1) lock lookups; lock file parsed once per sync

**Decision:** Build a `DestPath → LockedItem` index once per sync run (replacing per-lookup O(n) scans). Parse the lock file exactly once with a version-aware deserializer (replacing version probe + full parse).

**Why:** The lock lookup pattern appeared in `diff.rs`, `target_sync/mod.rs`, and `lock/mod.rs` — a loop-inside-loop structure with O(n) lookup produces quadratic cost on large projects. Both are localized, low-risk single-file changes.

**Deferred:** Agent file re-reads, repeated tree walks, frontmatter caching, and model catalog indexing — higher-complexity cross-module changes deferred to a future work item.

---

### D39: `compiler/config_entries/` extracted as preparatory refactor before Phase 4a/5

**Decision:** Extract `compile_config_entries()` into a dedicated `compiler/config_entries/` module (pure code motion) between Phase 3 and Phases 4a/5.

**Why:** Phase 4a (collision resolution) and Phase 5 (stale cleanup) both modify config-entry compilation. Two concurrent feature changes to one complex function increases merge risk. The extraction creates clean module boundaries so each phase's changes are independently reviewable. The refactor itself is behavior-preserving — zero behavior change, pure code motion.

**Alternatives rejected:** Modify in place during Phase 4a → higher merge risk; harder to review. Full compiler restructuring → out of scope for a cleanup work item.

---

### D40: Mars integration tests split from a 2,100-line monolith into focused top-level files

**Decision:** Split `tests/integration/mod.rs` into separate top-level test files (`tests/init_and_add.rs`, `tests/sync_behavior.rs`, etc.) with shared helpers in `tests/common/mod.rs`.

**Why:** Cargo treats each file under `tests/` as an independent test crate, enabling parallel execution and faster incremental builds. A `tests/integration/` subdirectory serializes all tests in it. The 2,100-line monolith also produced large opaque diffs and was hard to navigate.

**Alternatives rejected:** Submodules under `tests/integration/` → serialized execution, larger incremental builds. Keep monolith with section comments → still 2,100 lines, large diffs.

---

## Skill Schema

### D58: Universal skill frontmatter schema (name, description, model-invocable, user-invocable, allowed-tools, license, metadata)

**Decision:** The canonical SKILL.md frontmatter has six fields: `name`, `description`, `model-invocable`, `user-invocable`, `allowed-tools`, `license`, and `metadata`. A `compatibility` field proposed in early drafts was removed.

**Why `compatibility` was removed:** A `compatibility` field declaring which harnesses a skill supports would require Mars to understand harness semantics and would either block valid installs or produce false precision. Bootstrap docs can explain compatibility in prose with nuance.

**`metadata` is pass-through:** Agents can read `metadata`; harnesses and Mars ignore unknown frontmatter fields. This lets skill authors embed tooling metadata (authorship, versions, changelog URLs) without schema changes.

> **Partially superseded (2026-05-02) by D71.** The `invocation` enum field was replaced by two independent booleans: `model-invocable` and `user-invocable`.

---

### D71: Skill invocability decomposed into two independent booleans: `model-invocable` and `user-invocable`

**Decision:** Replace the single `invocation: explicit | implicit` enum in SKILL.md frontmatter with two independent booleans: `model-invocable` (default: `true`) and `user-invocable` (default: `true`). Remove all legacy alias fields (`invocation`, `disable-model-invocation`, `allow_implicit_invocation`) as hard errors with no deprecation period.

**Why two booleans:** The old enum collapsed two orthogonal dimensions into one field. Every harness (Claude Code, Codex, OpenCode, Pi, Cursor) independently controls (1) whether the model can see and self-load a skill and (2) whether the user can trigger it via `/name`. `invocation: explicit` only ever controlled model-invocability in practice — user-invocability was not modeled at all.

**Why hard errors, not deprecation:** Mars-agents had not shipped a release when this change was made. No consumer reads `invocation` from `.mars/skills/` at runtime (Meridian only reads it at compile time). Migration is find-and-replace in source packages followed by `mars sync`. A deprecation period with silent fallback would have masked author mistakes.

**Presence is not a skill property:** Whether a skill is loaded at boot is determined by the agent profile's `skills:` list, not by skill-level metadata. No `presence:` field was added.

**Compilation model per harness:**
- Claude Code — supports both booleans natively (zero lossiness)
- Codex — supports `model-invocable` only (via `allow_implicit_invocation`); `user-invocable` dropped with lossiness entry
- OpenCode — drops both booleans
- Pi, Cursor — support `model-invocable` via `disable-model-invocation`; `user-invocable` dropped

**Codex emit condition:** Codex emits `allow_implicit_invocation` only when the source frontmatter explicitly set `model-invocable` (tracked by `had_model_invocable_field`). This preserves prior behavior where Codex didn't emit the field unless the author intended it.

---

### D59: Native skill projections always refreshed on sync with divergence warnings

**Decision:** `mars sync` re-emits every native harness projection (e.g., `.claude/skills/<skill>/SKILL.md`) unconditionally — no content-hash optimization to skip unchanged files. If a native projection has diverged from what sync would produce, Mars emits a divergence warning.

**Why:** Skipping unchanged projections would leave stale native files after frontmatter format changes or harness lowering changes. Staleness would only surface as subtle harness behavior differences. Always-refresh makes sync idempotent and keeps the invariant checkable: after sync, native dirs are byte-for-byte identical to lowering output.

---

### D60: Skill variant specificity ladder: model-token+harness > canonical-id+harness > harness > base

**Decision:** When multiple variant files exist for a skill, selection follows a 4-step exact-match ladder:
1. `variants/<harness>/<requested-token>/SKILL.md`
2. `variants/<harness>/<canonical-id>/SKILL.md`
3. `variants/<harness>/SKILL.md`
4. `SKILL.md`

Exact match only — no glob or fuzzy matching.

**Why exact match:** Fuzzy matching would make variant selection non-deterministic when multiple globs could match the same harness+model. Exact match makes the selected variant predictable and auditable. Two priority levels for model identity (requested token vs. canonical ID) exist because authors may write either form (`variants/claude/sonnet/SKILL.md` or `variants/claude/claude-sonnet-4-5/SKILL.md`).

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

**Solution:** Filter the count to packages whose name is a key in `effective.dependencies`:

```rust
let upgrades_available = if request.options.frozen {
    0
} else {
    graph
        .nodes
        .values()
        .filter(|node| {
            effective.dependencies.contains_key(&node.name)
                && matches!(
                    (&node.resolved_ref.version, &node.latest_version),
                    (Some(resolved), Some(latest)) if latest > resolved
                )
        })
        .count()
};
```

**Files changed:** `mars-agents/src/sync/mod.rs` — `upgrades_available` computation.

**Alternatives rejected:**

- **Extract a shared "count outdated" function used by both sync and outdated** — reasonable refactor, but overkill for a 10-line fix to a consistency bug. The two paths have different concerns (display count vs. actionable list); sharing code would couple them prematurely.
- **Make transitive dep upgrades actionable** (`upgrade --bump` acts on transitives too) — breaks lockfile semantics. Transitive deps are resolved by the dependency graph algorithm, not user-configured. Letting users bump them directly bypasses the version resolution contract and could introduce incompatibilities.
- **Split the count** (show "N direct, M transitive upgrades available") — adds display complexity; users only care about actionable upgrades. The extra context would prompt questions that have no actionable answer.

**Principle applied:** Smallest structural fix that restores consistency between what the hint claims and what a command can deliver.

---

## Init Command

### D78: Filesystem scan for mars content discovery, not JSON model coupling

**Decision:** After `mars add` completes, the Python `run_init_flow()` reads materialized content by scanning `.mars/` subdirectories dynamically (`_scan_mars_content()` in `init_ops.py`), not by deserializing a Pydantic model that mirrors the mars Rust `JsonReport` struct.

**Why filesystem over JSON model:**
- The mars JSON report shape is an implementation detail of the Rust `JsonReport` struct, not a published contract. Coupling Python to it means Python changes whenever mars adds a new content category, renames a field, or restructures the report.
- Filesystem scanning automatically picks up new content types (new `.mars/` subdirectories) without any Python changes.
- The `.mars/` directory layout IS the stable contract — it is what mars materializes for consumers to use. Meridian already reads agents/skills from `.mars/` at runtime; reading the same tree for counts is consistent.

**Implementation detail:** Skills are stored as directories (containing `SKILL.md`), not files. The scan captures both files and directories to count each correctly. Internal dirs (`cache`) are excluded via a `_SKIP_DIRS` set.

**Scope:** Returns `dict[str, list[str]]` — content type (subdir name) to sorted list of item stems. New content types appear automatically with no schema change.

**Alternatives rejected:**
- Parse `mars add --json` output for content counts → couples Python to an internal Rust struct; breaks on mars report shape changes
- Define a Pydantic `MarsJsonReport` model → same coupling problem, plus schema drift when mars evolves

---

### D79: Honest field naming — `packages_requested` not `packages_resolved`

**Decision:** The field holding the user-supplied package specifiers in `InitResult` is named `packages_requested`, not `packages_resolved`.

**Why:** "Resolved" implies the full resolved dependency set, including transitive dependencies installed by mars. This field holds only what the user passed to `--add` — the input specifiers. "Requested" is honest about the scope. The resolved set is internal to mars and not surfaced by `InitResult`.

**General principle:** Field names should describe what the data actually contains, not what you wish it contained. A field named for its aspirational content is a latent mislead — it tells the next reader that the data is more complete than it is.

---

### D87: Convention-based source discovery and explicit hidden foreign import

**Decision:** Mars discovers package items with one bounded convention walk. The walk starts at the rooted package directory, descends only through non-hidden directories up to `MAX_DISCOVERY_WALK_DEPTH = 5`, and recognizes directories named `agents`, `skills`, and `bootstrap`. A package-root `SKILL.md` remains the flat-skill fallback. After scanning, Mars grounds the result to the shallowest discovered package layer across all item kinds. Any duplicate `(kind, name)` discovered inside one source raises `DiscoveryCollision`, whether the duplicate came from convention scanning, manifest declarations, or both.

**Why:** Package import should be predictable from package conventions, not from a harness-specific guess about where a tool might store generated output. A single bounded walk lets real packages live below monorepo subdirectories without turning every nested fixture, example, or vendored package into importable content. Shallowest-layer grounding keeps a package's own layer authoritative when examples or vendored layouts also contain `agents/` or `skills/` folders.

**Hidden directories are not default sources:** Dot-prefixed directories such as `.claude/`, `.codex/`, `.cursor/`, `.opencode/`, `.git/`, and `.mars/` are local execution, cache, control, or generated output surfaces. The walk skips them by rule rather than carrying a per-harness blocklist. A foreign hidden layout is still importable, but only by rooting the dependency there explicitly, for example `subpath = ".claude"` with `dialect = "claude"`.

**Rejected alternative:** Auto-scan hidden harness containers such as `.claude/agents` and `.claude/skills` from the source root. Rejected because it treats generated/local harness state as source, imports output from packages that never opted into Mars packaging, and requires a growing harness-container heuristic. That heuristic was removed rather than preserved for backwards compatibility.

**Mechanism reference:** `mars-agents/src/discover/mod.rs` owns the convention walk, layer grounding, dot-dir skip, and `DiscoveryCollision` check. `mars-agents/src/local_source.rs` routes project-local items through the same discovery contract under `.mars-src/`.

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

**Why:** Mars has access to the full compiled agent/skill graph at build time; it cannot
see per-spawn content because that content is Meridian's runtime concern. Prompt files,
context files, and session history are session-scoped — they change per spawn, not per
agent version. Mars knowing about them would couple the build system to spawn runtime
semantics.

**Rejected alternative:** Mars receives the prompt file and produces a fully concrete
launch command. Rejected: the prompt file is confidential per-spawn content; Mars is a
shared CLI tool and should not receive it. Also, prompt injection (goal, prior session)
is spawn-lifecycle policy that belongs in Meridian.

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

**Why:** Making Mars understand every harness's config surface would require Mars to track
every harness version's config schema — unsustainable. The escape hatch gives profile
authors direct access to the harness surface for cases with no portable equivalent. If a
knob proves universally useful, promote it to a first-class top-level field later.

**Rejected alternative:** Mars validates specific known passthrough key names per
harness. Rejected: would require Mars to track every harness version's config schema,
coupling Mars to harness implementation details.

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

**Invariant enforced by shared code:** For any given `(model_token, settings.targets,
harness_availability)` input, both commands produce the same `(harness_id, confidence)` outcome.
The `routing::evaluate_candidates()` function is the single authority — neither command has its
own harness selection logic.

**Practical consequence:** `mars models resolve <token>` is a reliable preview. If it resolves
to claude with high confidence, the launch bundle will route to claude. No silent divergence.


---

### D88: Cross-source naming collisions auto-rename both colliders

**Decision:** When two dependency sources install an agent or skill to the same destination path, both colliders are suffixed with `__{source_name}` instead of one winning or the sync failing. Explicit `rename` in mars.toml takes priority over auto-rename.

**Why:** Multi-package installs commonly share item names (e.g. `meridian-dev-workflow` and `creative-writing-skills` both defining `agents/web-researcher.md`). Hard-erroring blocks legitimate use; first-wins creates an ordering dependency on mars.toml declaration order for collision outcomes. Rename-both is unambiguous — the user always knows which package an item came from — and needs no precedence rule for the collision itself.

**Gating:** Auto-rename applies only to same-kind (Agent|Skill) groups where every collider has a distinct source. Same-source collisions, mixed-kind groups, and hook/MCP/bootstrap collisions remain hard errors. Hooks use a separate first-wins policy at the target-lowering layer (D36).

**Frontmatter rewriting:** After rename, one unified `RenameIndex` + `apply_renames` pass rewrites each agent's `skills:` and `subagents:` frontmatter in a single content update. Resolution: same-source copy wins; if the agent's source still owns an unrenamed copy, the ref is left alone (no retargeting); otherwise fall back to mars.toml declaration order. This replaced two separate passes (`rewrite_skill_refs` + `rewrite_collision_refs`) that could rewrite an agent's content twice.

**Prune-before-rewrite:** Unmanaged-collision pruning runs before the rewrite pass so ref resolution sees post-prune state. Without this, `source_has_unrenamed_item` evaluated against pre-prune items and produced dangling refs when an unrenamed item was later removed.

**Regression history:** Auto-rename-both was the original behavior. Commit `ac12ea4` (subpath source refactor) incidentally removed it; the hard error blocked multi-package installs. PR #125 restored it with the unified pass and prune-before-rewrite ordering.

**Alternatives rejected:**
- Hard error on cross-source collision → blocks legitimate multi-package installs sharing item names
- First-wins by declaration order → creates an ordering dependency for collision outcomes; the losing package silently disappears
- Merge collisions → agents/skills are atomic units; merging two `agents/web-researcher.md` files is semantically meaningless

---

### D89: Config-rename-dangle diagnostic for renamed-away config references

**Decision:** After any rename (explicit or collision auto-rename), sync checks config-side name references against installed names and warns when a referenced name was renamed away and no installed item still carries it. The warning code is `config-rename-dangle`.

**What it checks:** `[settings.meridian.fanout].agents` entries, `[agents.<name>]` overlay keys (from config + local), and `[skills.<name>]` overlay keys. One `warn_config_dangles_after_rename` function covers all three, driven by both rename sources against the post-prune installed set.

**Why:** A rename can leave config references silently dangling. Without the diagnostic, `[settings.meridian.fanout].agents = ["web-researcher"]` would silently stop firing after a collision renames the agent to `web-researcher__pkga`. The warning lists the new installed names so the user can update the config reference.

**Fires only when the original name is gone:** If another source still provides the unrenamed name, the reference is left alone — the fanout/overlay still has a valid target. This avoids false positives when one source's rename doesn't affect the whole installed set.

**Alternatives rejected:**
- Auto-rewrite the config references → config is user-authored; mars shouldn't mutate it silently
- Hard error → too strict; a dangling config reference is a warning, not a corrupt state
- No diagnostic → silent breakage; the user discovers it when the fanout silently stops working


---

### D90: Native hook passthrough replaces universal event vocabulary (mars-agents PR #133, 2026-07)

**Decision:** `hook.toml` declares harness-native event names in per-target `[targets."<target>"]` tables. Mars validates each event against a static per-target allowlist (`TargetAdapter::known_hook_events()`) and passes names through verbatim to target config files. The universal event vocabulary (`UniversalEvent`, `classify_for_target`, all five per-target translation tables, `LossinessKind`, `codex_hook_matcher`, legacy `codex_hooks.json` format support) is deleted.

**Owner ratification:** "I've never used the compile feature at all" and "yeah, I think we just passthrough... the vision was I would write once and then we would lower down but..." The original write-once/lower-down vision was abandoned deliberately after audit evidence.

**What the audit found:** Of 12 event-name mappings that emit config, 6 were wrong: Claude `session.end` compiled to the nonexistent `SessionStop`; Codex `session.end` compiled to `Stop` (fires per-turn, not session end); all 4 OpenCode mappings were fabricated names written to a schema with `additionalProperties: false`. Of the 3 real hooks in existence, all were single-target with harness-native payload handling in their scripts.

**What mars keeps:** Hook discovery across package dependency trees, deterministic ordering, script staging, atomic config merge/removal under per-target lock ownership, provenance tracking in `mars.lock`. Mars invariants hold: never delete files it didn't create, tmp+rename writes, resolve-first semantics.

**What mars stops doing:** Inventing event names. Lock keys change from `hook:<universal-event>:<name>` to `hook:<NativeEvent>:<name>`. Targets without declarative command hooks (OpenCode, Pi) produce hard errors instead of silent drops. The `unchecked = true` escape hatch per target table opts out of allowlist validation for events newer than the mars binary.

**Codex `SessionEnd` excluded from allowlist:** Runtime probe (spawn p5698, Codex 0.144.4) proved Codex silently tolerates but never fires `SessionEnd`. Including a known-dead event would be the exact silent-dead-hook class this work eliminates. Codex allowlist is 10 events; re-add tracked in mars-agents#132. Claude allowlist is 29 events.

**Migration:** Old `hook.toml` schema is a hard error with a migration hint. No compatibility shim: no real users beyond three meridian-base hooks, all migrated. One-release removal-only residue sweeps strip prior-version artifacts from OpenCode (`opencode.json` hooks) and Codex (`codex_hooks.json` entries). Deferred: delete residue sweeps in next release (mars-agents#130); Cursor hook writing (mars-agents#131).

**Release sequencing:** Mars release (schema flag-day) -> meridian-base `mars version patch --push` (migrated hook.tomls) -> meridian-cli pin bump. No intermediate compatibility window.

**Alternatives rejected:**

| Option | Why not |
|---|---|
| Hybrid (universal core + native escape hatch) | Keeps drifting translation tables alive for a portable core with zero users; adds precedence rules; gives authors two ways to say the same thing |
| Expanded universal vocabulary with generated/validated tables | Portability remains fake: payloads, matchers, and output protocols stay native; maintaining a validated intersection of five vocabularies (29/11/21/29/33 events) is standing tax for an unused property |
| Per-target hook directories (`hooks/claude/...`) | Same authoring model but loses shared script, `order`, `visibility`, and single discovery unit |
| Drop hooks from mars entirely | Config-writing machinery is real value: managed merge/removal into `settings.local.json` and `hooks.json`, script staging, lock ownership |

---

### D91: Native hook fragments replace command synthesis (2026-07)

> **Supersedes D90's schema.** D90 established the principle (native passthrough, per-target allowlists, `unchecked` escape hatch). D91 replaces the residual command-synthesis machinery with full native fragments: the author writes the harness's own hook shape verbatim, and mars stops owning any hook entry schema at all.

**Decision:** `hook.toml` shrinks to identity and routing (name, visibility, order, target-to-fragment map). Per-target fragment files (`claude.json`, `codex.json`, `cursor.json`, or `.ts` for file-mode targets) carry the harness's native config shape verbatim. Mars validates event keys against the existing per-target allowlists, substitutes `${MARS_HOOK_DIR}` with the absolute installed hook directory path, merges entries into target config files (or places file-mode fragments whole), copies the complete hook directory into `<target>/hooks/<name>/`, and records the exact emitted JSON entries in `mars.lock` for structural removal.

**What mars stops doing:** Synthesizing hook entries. D90 still had `[action] kind = "script" path = "run.sh"` and mars assembled `bash '<path>/run.sh'` commands with platform-specific quoting. The author now writes the complete command string (`"bash \"${MARS_HOOK_DIR}/run.sh\""`) and every harness-specific field (handler `type`, `matcher`, `timeout`, `args`, `if`, `once`, `async`, `shell`, `statusMessage`, `failClosed`). Mars never parses or generates any of these.

**Two fragment modes:**

| Mode | Targets | What mars does |
|---|---|---|
| MergeJson | Claude, Codex, Cursor | Validates event keys, substitutes `${MARS_HOOK_DIR}` in all JSON string values, merges entry arrays per event into the target config file |
| File | OpenCode, Pi | Substitutes `${MARS_HOOK_DIR}` textually, places the file at `plugins/mars-<name>.ts` or `extensions/mars-<name>.ts` |

**Ownership model: lock-recorded emissions.** For each `hook:<Event>:<name>` config-entry key, `mars.lock` stores the exact emitted JSON entry array (post-substitution). Removal matches entries in the current config file by structural JSON equality. User-edited entries no longer match and are preserved. File-mode fragments are ordinary target outputs with `OutputRecord` tracking through `surface_ownership`. This structurally prevents the residue-cleanup class: future emission-shape changes are automatically recognizable because the lock carries the actual bytes, not a shape assumption.

**`${MARS_HOOK_DIR}` substitution rationale:** Runtime probes (p5711) proved hook cwd is the launch directory on both Claude and Codex, and Claude's `$CLAUDE_PROJECT_DIR` follows the launch directory too. Codex has no project-root env var at all. Relative paths and env-var-prefixed paths both break when the user launches from a subdirectory. Sync-time textual substitution with absolute paths is the only form that works on every probed harness.

**Cursor merge-mode:** Mars owns the `{"version": 1, "hooks": {...}}` wrapper in `.cursor/hooks.json`. The 21 camelCase event names were confirmed against the cursor-agent 2026.07.16 binary validator. Cursor's flat entry shape (command entries directly in the event array, no inner `"hooks"` nesting) is the author's responsibility to write correctly in the fragment.

**File-mode ownership fix:** File-mode fragments were initially tracked by a parallel `hook-file:<name>` scheme that bypassed `surface_ownership.rs`. This produced the exact ownership violations the contract was written to prevent: untracked user files silently overwritten, hook removal deleting files the lock did not own. Fixed by making file fragments ordinary target outputs. See [../lessons/residue-cleanup-discipline.md](../lessons/residue-cleanup-discipline.md) Instance 3.

**Two-surface ownership contract.** Mars manages two distinct ownership surfaces: path ownership via `OutputRecord` keyed on `(target_root, dest_path)` for copied files and file-mode fragments; config-entry ownership via `ConfigEntryRecord` with `emitted_json` for merge-mode entries. Byte equality is never ownership. When a removal step fails, the retention seam (`surface_ownership/retention.rs`) derives retention and write suppression from a single outcome per `(target_root, surface)` pair. The wrong states are unrepresentable: `Confirmed` has no field for retained records; every replacement write requires a `WritePermit` that only the outcome table issues. Six consecutive review rounds found ownership bugs while per-call-site retention decisions disagreed with each other; the redesign replaced all three decision sites with one outcome table. See [architecture/mars-compiler.md](../architecture/mars-compiler.md#two-surface-ownership-model).

**Copy-paste leniency:** If a fragment's top level is exactly `{"hooks": ...}` (optionally plus `"version"` or `"description"`), mars unwraps it. No harness has an event named `hooks`, `version`, or `description`, so the unwrap is unambiguous.

**Codex trust churn accepted:** Index-keyed trust means adding a new package can shift indices and silently skip affected hooks until re-trusted via `/hooks`. Deterministic stable ordering minimizes churn; the residual prompt is Codex's model working as designed. No append-only ordering special case.

**Migration:** `events`/`matcher`/`[action]`/target-`path` in `hook.toml` triggers a hard error naming the file with a fragment-migration hint. The v0.11.0 command-path removal sweep is restricted to lock records lacking structural emission data and joins the #130 one-release deletion ledger. Three meridian-base hooks migrated: `context-autosync` (gained per-event `timeout: 30` on `SessionEnd`), `deny-generic-agent` (gained explicit `matcher: "Agent"`), `deny-interactive-prompts` (unchanged behavior).

**Alternatives rejected:**

| Option | Why not |
|---|---|
| Coexist: keep `events`/`matcher` as sugar over generated fragments | Two ways to author; adapters keep entry-synthesis alive next to passthrough; every new harness field re-raises "add to sugar or force fragments?" |
| TOML-inline fragments | Kills copy-paste-from-harness-docs; deep TOML nesting of Claude's shape is worse to read and diff |
| Extend mars schema field-by-field (add `timeout`, `type`, per-event matcher) | Recreates the vocabulary treadmill one level down; mars permanently behind every harness release |
| Marker field in emitted entries (`"_mars": ...`) for ownership | Depends on every harness tolerating unknown fields forever; lock-recorded emissions need no foreign bytes |

---

### D92: Shape-A recovery seam -- halt before compilation when hook surfaces are unreadable (2026-07)

**Decision:** Recovery commands (`upgrade`, `override`, `remove`, `repair`) persist their intent mutation (pins, overrides, dependency graph changes to `mars.toml` / `mars.local.toml`) and **halt with exit 2 before the compiler** when any resolved source has an unreadable hook surface (removed-schema package). Strict `sync` remains the sole materializer and only ever runs against a fully readable graph. The "frozen" hook category is deleted from the sync pipeline entirely.

**Why:** The sync pipeline assumes the staged tree is total desired intent. Two implementation attempts to thread a "cannot currently interpret, must not touch" exception through the pipeline each failed with rising severity, not falling:

- **Round 4 (omission-as-absence):** omitting unreadable hooks from staging read as removal intent to the desired-state engine. `mars remove a` deleted unrelated dependency b's installed hooks, bindings, and lock records.
- **Round 5 (freeze-as-carried-state):** `frozen_hook_sources` flowed through resolution, but three of N persistence sites did not honor it. `PlannedAction::Skip` copied at the target boundary (repair overwrote user edits, finalize mutated frozen checksums). Omitted desired paths were not reserved (a readable hook could claim a frozen hook's destination). Provenance was carried for transitive but not direct dependencies (the override escape hatch self-blocked).

Every stage of the sync pipeline is a persistence site; an exception must be enforced at all of them or it corrupts state. Six pipeline stages needed freeze awareness; five more were named by the round-5 review as eventually needing it. A permanent concept in every current and future persistence site, built for a one-release migration window, was disproportionate.

**What Shape A deletes:** `DiffEntry::Frozen`, `PlannedAction::Skip` frozen lowering, frozen-source hook omission during target construction, `frozen_hook_retention` and every frozen branch in config-entry compilation, frozen partition and exact-record carry-forward in lock construction, `RemovedHookSchemaPolicy` and the policy-conditional staging detection. Commits `5fc4148` and `783437b` are substantially reverted.

**The seam.** The gate sits at the existing reader/compiler boundary in `sync::execute` -- between `reader::read` (load config, resolve, stage, classify hook surfaces) and `compiler::compile` (target, diff, plan, apply, target-sync, sweeps, lock). `unreadable_hook_surfaces` on `ResolvedGraph` has exactly **one consumer**: the gate. Nothing downstream reads it.

**Why one-shot convergence is barely lost.** In the dominant real case (one legacy source), `mars upgrade` resolves a migrated release, the post-mutation graph becomes fully readable, and the full pipeline runs in the same invocation -- including v2-to-v3 promotion and the #130 sweeps. `override` and `remove` behave the same way: the mutation itself makes the graph readable. Shape A gives up one-shot convergence only in the multi-legacy-source case, where one-shot was never achievable under any shape (each escape-hatch command fixes one source).

**User-facing contract.** Exit 0 iff the full pipeline ran (project converged). Exit 2 with a halt report otherwise. The halt report states what was persisted, each remaining blocker as `package@version` with its unreadable hook names, and the suggested next command. JSON output carries a `recovery_halt` object. Exit 2 after a successful intent write follows the `git rebase` conflict pattern: state advanced, work remains, the message says what.

**Corrupt-lock repair contract.** Repair treats a corrupt lock as empty in memory under the sync flock; the on-disk corrupt bytes are preserved until a successful full run finalizes via atomic tmp+rename. A corrupt lock is evidence, not garbage.

**Vocabulary repair.** "Frozen" had two unrelated meanings: `--frozen` (lockfile immutability mode, `check_frozen_gate`, `MarsError::FrozenViolation`) and `frozen_hook_sources` (recovery freeze). Shape A deletes the second meaning. `HookSurfaceState::Frozen` renames to `Unreadable`.

**What survives from rounds 4-5:** `RecoveryPolicy` on `SyncRequest` with strict default and explicit per-command opt-in (`e9992b9`); `compiler::hooks::uses_removed_schema` as the single classifier; contextualized source-package errors with no staging-path leaks (`04f6b04`); transitive override capability (`a18c0c6`) plus declared-identity preservation (`41b540d`).

**Alternatives rejected:**

| Shape | Why not |
|---|---|
| Omission-as-absence (round 4) | Staging absence reads as removal intent; destroyed unrelated dependencies' installed state |
| Freeze-as-carried-state (round 5) | Every pipeline stage is a persistence site; the exception leaked at 3 of N sites on the first attempt, with 5 more stages named as eventually needing awareness. Permanent complexity for a one-release migration bridge |
| Recovery commands write targets while blind | Round 5 proved repair-while-blind overwrites user edits and corrupts frozen checksums. The #130 sweeps and v2 promotion run inside the materialization pipeline and always eventually execute -- as the same invocation when the graph becomes readable, or as the final `mars sync` |

---

## Engine Version Constraints

### D93: Hard-constraint-with-fallback engine requirements (2026-07)

**Decision:** Packages declare minimum engine versions via optional `requires-mars` and `requires-meridian` fields in `[package]`. When the running engine does not satisfy a dependency's requirement, the resolver excludes that version and walks down to the newest compatible candidate. When no candidate is compatible, resolution fails before any mutation. Escape hatches: `--ignore-requires-mars` and `--ignore-requires-meridian`.

**The model:** pip's `Requires-Python` — hard filter with emergent fallback. Every ecosystem studied (pip, Cargo, Bundler, npm, Go) filters or ranks by runtime compatibility; the pip/Bundler hard-filter model serves the "newest version that works" intent most cleanly. Mars packages compile into agent-facing trees; a wrong compile is worse than a stop, which rules out Cargo's soft-preference and npm's advisory-only defaults.

**Two engines, one mechanism.** `requires-mars` checks against the compiled-in `CARGO_PKG_VERSION`; `requires-meridian` checks against the `MERIDIAN_VERSION` environment variable exported by `mars_passthrough.py` in meridian-cli. Both use the same parse, check, and walk-down code path (`src/resolve/requires.rs`). When `MERIDIAN_VERSION` is absent (standalone mars, or old meridian), `requires-meridian` checks are silently skipped. When present but malformed, resolution fails with a typed error — but only when the manifest actually declares `requires-meridian`; fieldless manifests never parse the environment.

**Bare version rewrite.** A bare version like `"0.12"` is rewritten to `>=0.12.0` before parsing. Without this, semver's default caret interpretation (`^0.12`) would reject mars 0.13 — a trap every author would hit. Full ranges (`>=0.12, <2`) are accepted.

**Prerelease handling.** Prerelease and build metadata are stripped from the running version before matching. Mars `0.12.0-rc.1` satisfies `>=0.12.0`, matching how RC builds are used in practice.

**Exclusion state carries failure provenance.** Each excluded `(source, version)` pair records which engine requirements failed and why. When all candidates are exhausted, the unsatisfiable error names each candidate's requirement and the running engine version, not a generic semver conflict. Exclusions persist across resolver restarts — the restart algorithm's version overrides consult the exclusion map, so a restarted pass never re-proposes an already-excluded version.

**Diagnostics derive from the final graph.** The `engine_fallbacks` sync report is reconciled against the completed `ResolvedGraph` after the restart loop settles. Moot fallback records (sources dropped by a restart) are pruned; `selected_version` is derived from final graph nodes. Two incremental-refresh attempts failed review before this structural fix: accumulating diagnostics during resolution passes produces stale or phantom entries whenever a restart changes which sources appear in the final graph.

**Consumer's own package.** The consuming project's `[package].requires-mars` / `requires-meridian` is checked once during `load_config` (hard error, no fallback possible — there is no other version of the consumer's own manifest).

**Lock interaction.** No lock schema change. When a locked version becomes engine-incompatible (mars downgraded after lock), the resolver treats it like a constraint-violating locked version: warns (`requires-mars-lock-fallback`), enters the walk-down, and rewrites the lock to the older selection. `--frozen` refuses the fallback as a `FrozenViolation`.

**Single-candidate sources.** RefPin, path, and untagged (HEAD) sources have no alternative candidate to walk down to. An incompatible manifest there is a hard error; the escape hatch applies.

**Migration.** Existing packages (no field) resolve exactly as before. Old mars binaries ignore the new fields via serde defaults. Protection is forward-only: the first mars release with this feature is the floor below which declarations are invisible. Adoption discipline for prompt packages: declare the field the first time a package uses a feature newer than the current floor, in the same commit as the feature use. No blanket backfill.

**Alternatives rejected:**

| Alternative | Why not |
|---|---|
| Soft preference (Cargo MSRV model) | "Install latest anyway with a note" is the silent breakage the feature prevents. Cargo moved away from this toward fallback-by-default in edition 2024. |
| Advisory warning only (npm default) | Same failure mode; npm's own trajectory is a retreat from it (engine-strict, pickManifest ranking). |
| Synthetic dependency (Bundler model) | Elegant in a general backtracking resolver; in mars's bottom-up+restart resolver it entangles engine state with `SourceName` registry identity for no capability gain. |
| Engine auto-upgrade (Go model) | Mars downloading newer mars is a distribution/security problem beyond the resolver; the unsatisfiable error's upgrade hint captures the useful part. |
| `[engines]` table section | A table for two keys, importing vocabulary that fits mars awkwardly. Flat `requires-mars` and `requires-meridian` under `[package]` follow the Cargo precedent. |
| Out-of-band version metadata (tag annotations, side index) | Infrastructure mars does not have, solving a cost the global cache already bounds at mars's scale. |
| Recording `requires-mars` in the lock | Redundant: the manifest is re-read from cache every resolution; a lock copy could only go stale. |

**Source study evidence (pip, Cargo, Bundler, npm, Go):** pip's `package_finder.py` drops incompatible links in `LinkEvaluator.evaluate_link` (fallback is emergent: incompatible versions are invisible). Cargo's `version_prefs.rs` uses `msrv_compat_count` as sort rank (a sort, not a filter). Bundler appends `Gem::Dependency.new("Ruby\0", required_ruby_version)` to each spec. npm's `npm-pick-manifest` uses `engineOk` as both a filter and a tiebreak ladder. Go auto-downloads newer toolchains and hard-errors on older ones. Full source study in the work directory (`work/mars-version-constraints/design.md`).

---

## Related

- [decisions/model-resolution.md](model-resolution.md) — Mars alias authority, how aliases flow into resolution
- [decisions/managed-command-references.md](managed-command-references.md) — `managed_cmd()` for MERIDIAN_MANAGED-aware user-facing output
- [launch.md](launch.md) — composition pipeline, harness adapters
- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — package model mechanism
- [concepts/package-management/sync-model.md](../concepts/package-management/sync-model.md) — sync pipeline mechanics
- [architecture/mars-compiler.md](../architecture/mars-compiler.md) — compiler internals
- [architecture/mars-targeting.md](../architecture/mars-targeting.md) — targeting architecture
- [architecture/mars-launch-bundle.md](../architecture/mars-launch-bundle.md) — launch-bundle system
- [concepts/skill-schema.md](../concepts/skill-schema.md) — skill schema and variant lowering
- [concepts/bootstrap-docs.md](../concepts/bootstrap-docs.md) — bootstrap doc discovery mechanism
- [concepts/native-config.md](../concepts/native-config.md) — harness override passthrough concept
