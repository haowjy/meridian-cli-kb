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

## Related

- [decisions/model-resolution.md](model-resolution.md) — Mars alias authority, how aliases flow into resolution
- [decisions/managed-command-references.md](managed-command-references.md) — `managed_cmd()` for MERIDIAN_MANAGED-aware user-facing output
- [launch.md](launch.md) — composition pipeline, harness adapters
- [concepts/package-management/overview.md](../concepts/package-management/overview.md) — package model mechanism
- [concepts/package-management/sync-model.md](../concepts/package-management/sync-model.md) — sync pipeline mechanics
- [architecture/mars-compiler.md](../architecture/mars-compiler.md) — compiler internals
- [architecture/mars-targeting.md](../architecture/mars-targeting.md) — targeting architecture
- [concepts/skill-schema.md](../concepts/skill-schema.md) — skill schema and variant lowering
- [concepts/bootstrap-docs.md](../concepts/bootstrap-docs.md) — bootstrap doc discovery mechanism
