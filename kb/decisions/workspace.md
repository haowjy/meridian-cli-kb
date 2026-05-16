# Decisions: Workspace

Decision records for the workspace configuration system — why this architecture, what alternatives were rejected, what constraints drove the choices.

See [../decisions.md](../decisions.md) for the chronological index. Architecture mechanism is documented in [../architecture/workspace/](../architecture/workspace/).

---

## Background: What Workspace Is and Isn't

Workspace is a filesystem scope grant — it controls which directories are projected into harness subprocess scope via `--add-dir`. It is not context. Context is working memory passed via environment variables and system prompt injection. Merging them would allow workspace roots to bleed into agent working memory, turning a filesystem permission into an instruction.

The design mandate that preceded all other decisions:

```
Workspace = filesystem scope.  Gets --add-dir only.  No env vars.  No prompt mentions.
Context   = working memory.    Gets MERIDIAN_CONTEXT_*_DIR env vars + system prompt.
```

---

## D41: Workspace config moved into meridian.toml / meridian.local.toml {#d41}

*2026-05-01*

**Decided:** Workspace config moves from the standalone gitignored `workspace.local.toml` file into a `[workspace]` section inside the existing `meridian.toml` (committed) and `meridian.local.toml` (gitignored) files. `workspace.local.toml` is deprecated with a migration period.

**Why:** `workspace.local.toml` was local-only and gitignored. Teams sharing a standard multi-repo checkout had to configure workspace manually on every machine — there was no way to share the conventional repo layout declaration. Moving into `meridian.toml` lets teams commit the conventional layout once; developers with non-standard checkouts override only what differs in `meridian.local.toml`.

**Alternative rejected:** Keep as standalone gitignored file — can't share team conventions; every developer on a team still configures their machine individually even when the topology is standard.

**Alternative rejected:** Merge workspace into `[context]` — different trust boundaries. Context is working memory with env vars + system prompt injection; workspace is filesystem scope with `--add-dir` only. Merging them would break the boundary.

---

## D42: Named `[workspace.NAME]` entries over anonymous arrays {#d42}

*2026-05-01*

**Decided:** Switch from anonymous `[[context-roots]]` arrays to named `[workspace.NAME]` tables (e.g., `[workspace.frontend]`).

**Why:** Paths vary per machine; names are stable. Named entries enable a two-tier merge model: a local entry with the same name as a committed entry replaces it entirely (path override), while local-only names append without conflict. Anonymous arrays have no natural identity key — path-based keying fails when the whole point is that paths differ per developer. Append-only arrays don't support overrides at all.

**Alternatives rejected:**

| Alternative | Problem |
|---|---|
| Path-keyed override | Paths are what differ between machines — they can't be the merge key |
| Append-only arrays in permanent schema | No override semantics; can't replace a committed path with a local one |
| Anonymous arrays with `enabled = false` for disable | Still no override for different paths; adds a field without solving the merge problem |

---

## D43: No `enabled` field — path existence is the projection signal {#d43}

*2026-05-01*

**Decided:** Workspace entries have no `enabled` boolean. Non-existent paths are excluded from projection. The committed entry convention: "if this path exists on disk, project it."

**Why:** The committed workspace section declares a *convention* — the expected repo layout for a project. A developer who doesn't have a repo checked out produces a non-existent path, which is the natural signal for exclusion. Adding `enabled = false` would introduce a distinction between "explicitly disabled" and "path not found" with no meaningful semantic difference for committed entries. For local entries, the `workspace_local_missing_root` diagnostic already flags broken paths more actionably than a silent flag check would.

**Deferred:** Subtractive local override — disabling a committed root without removing the checkout — is deferred. If user feedback shows real demand, `disabled = true` can be added as a local-only, backward-compatible extension.

**Alternative rejected:** `enabled` bool in the original design — removed because it adds schema complexity without practical benefit given the convention framing.

---

## D44: Source-differentiated missing-path behavior {#d44}

*2026-05-01*

**Decided:** Non-existent paths from committed entries are silently skipped. Non-existent paths from local entries emit a `workspace_local_missing_root` diagnostic finding.

**Why these represent fundamentally different situations:**

- **Committed missing path** → developer doesn't have that repo checked out. Normal and expected in partial setups (e.g., a frontend developer who never checks out the ML tooling repo). Silent skip preserves ergonomics.
- **Local missing path** → developer explicitly declared a path on this machine that doesn't exist. Almost certainly a typo or stale config (e.g., the path was moved after `meridian.local.toml` was last edited). A diagnostic is warranted; this is an actionable error.

The distinction keeps partial-checkout ergonomics intact while making broken local overrides visible.

---

## D45: Dedicated workspace loader, not a field on MeridianConfig {#d45}

*2026-05-01*

**Decided:** The workspace loader reads `[workspace]` tables directly from TOML files, bypassing `MeridianConfig` and `_normalize_toml_payload`. It follows the same pattern as the context loader (`_try_load_context_config`).

**Why not on MeridianConfig:**

1. `MeridianConfig` is for scalar operational settings (timeouts, depths, retry policies). Workspace needs structured findings, per-root path resolution, and filesystem evaluation — a different shape of concern that doesn't fit scalar normalization.
2. Pydantic's `extra="allow"` stores extra fields as raw dicts, not typed models. It does **not** coerce named subtables (`[workspace.frontend]`) into `WorkspaceEntryConfig` instances. The type safety that makes named entries viable requires a dedicated `TypeAdapter`.

**Alternative rejected:** `WorkspaceConfig(extra="allow")` mirroring `ContextConfig` — verified to not work. Pydantic `extra="allow"` stores extras as raw dicts, not typed models. This was prototyped and rejected, not merely theorized.

---

## D46: No user-global workspace config {#d46}

*2026-05-01*

**Decided:** Workspace config is repo-local only (`meridian.toml` + `meridian.local.toml`). There is no user-global workspace config (no `~/.meridian/config.toml` workspace section, no user-level equivalent).

**Why:** Workspace expands filesystem access scope for harness subprocesses. A hidden global default would silently grant access to directories across all projects without explicit per-project declaration. This is a trust boundary concern — filesystem scope expansion must be declared at the repo level where it is intentional and reviewable.

**Contrast with context config:** Context paths can have user-level defaults because they don't expand filesystem permissions; they tell the agent where working memory lives. Workspace grants read (and potentially write) access to filesystem directories outside the project root — qualitatively different from working memory hints.

---

## D47: `projected_roots` as a first-class launch-spec field, not `extra_args` smuggling {#d47}

*2026-05-05*

**Decided:** Introduce `projected_roots: tuple[Path, ...]` as a dedicated field on `ResolvedLaunchSpec`. Workspace roots flow through this field — not through `extra_args` as synthetic `--add-dir` flags. Each harness projection converts `projected_roots` into its native mechanism.

**Why `extra_args` was wrong:** The previous pattern injected Meridian-owned workspace paths into `extra_args` as `--add-dir` flags, then Codex streaming projection re-parsed those flags to reconstruct the root list. This created three problems:
1. `extra_args` mixed Meridian policy-owned data with user passthrough — the two became indistinguishable at projection time.
2. Codex app-server projection had to parse `--add-dir` back out of a flat arg list it had already assembled.
3. The Codex **remote TUI attach** command (`codex resume --remote ws://...`) was built without any workspace roots, because `build_tui_command()` had no clean way to discover which args in `extra_args` were workspace roots vs user passthrough.

**The remote TUI attach gap:** When a user attaches the Codex TUI via `codex resume --remote ws://...`, the TUI sends per-turn `sandbox_policy` derived from its own local `PermissionProfile` — which is built from the TUI's CLI flags (including `--add-dir`). Without `--add-dir` on the attach command, the TUI's per-turn overrides narrow the sandbox scope below what the app-server was originally configured with. Codex's remote-TUI permission override behavior is working as designed; it is Meridian's responsibility to pass the same roots to both sides.

**Target harness projection mapping:**

| Harness | Mechanism |
|---------|-----------|
| Claude | `--add-dir <path>` CLI args |
| Codex subprocess | `--add-dir <path>` CLI args |
| Codex app-server | `-c sandbox_workspace_write.writable_roots=[...]` config flag |
| Codex remote TUI attach | `--add-dir <path>` on `codex resume` (`codex-rs/utils/cli/src/shared_options.rs` confirms support) |
| OpenCode | `OPENCODE_CONFIG_CONTENT.permission.external_directory` env var |

**Rule:** `extra_args` is user-owned passthrough only. `projected_roots` is Meridian-owned policy input. Deduplication and path normalization happen once, before harness projection.

**Implementation note:** `ResolvedRunInputs` and `ResolvedLaunchSpec` both carry `projected_roots`; `build_tui_command()` for Codex receives the resolved spec and mirrors those roots onto the remote attach command.

**Alternative rejected:** Keep `extra_args` smuggling and add a separate re-parsing step for the attach command — this would add code that the `projected_roots` refactor immediately deletes; throwaway intermediate state.

---

## D48: OpenCode workspace projection merges into parent `OPENCODE_CONFIG_CONTENT`, never suppresses {#d48}

*2026-05-05*

**Decided:** When `OPENCODE_CONFIG_CONTENT` is already set in the parent environment, OpenCode workspace projection **deep-merges** new `permission.external_directory` entries into the parsed parent config JSON. The previous behavior — suppressing projection and emitting only a diagnostic — was a bug.

**Why suppress was wrong:** In nested-spawn contexts, worktrees with a pre-set env, or any situation where an outer Meridian process has already configured OpenCode, child spawns could silently lose all workspace root access. The old `workspace_opencode_parent_env_suppressed` diagnostic told the user something went wrong, but didn't fix it. The merge is straightforward (additive JSON object update, never remove existing entries) and is already demonstrated by the `opencode_http.py` instruction-path merge pattern.

**Merge rules:**
1. Parse parent JSON into a dict (fall back to `{}` on parse error).
2. Get or create `permission.external_directory` sub-object.
3. Add `root.as_posix() + "/*": "allow"` entries — additive only.
4. Serialize back to env override.

**The `/*` wildcard suffix is required** by OpenCode's permission schema — it covers files inside the directory, not just the directory inode itself.

**Alternative rejected:** Keep diagnostic-only suppress — leaves nested spawns with no workspace access in inherited-env contexts.

---

## D49: Codex sandbox scope and approval policy are independent; workspace roots alone don't prevent approvals {#d49}

*2026-05-05*

**Decided:** Document explicitly that Codex treats `sandbox_workspace_write.writable_roots` (write scope) and `approval_policy` (when to pause for a human decision) as independent concerns. Observing an `item/fileChange/requestApproval` prompt is **not proof** that workspace roots are missing from projection.

**Why this matters:** A natural diagnosis for "sibling repo writes require approval" is "workspace roots are not projected." This diagnosis is often wrong. Codex may issue approval prompts for writes to fully-writable sandbox paths depending on its `approval_policy` configuration. The two questions are:
1. Are the roots in `writable_roots`? (Projection question — answered by Meridian config + smoke test)
2. Does the current `approval_policy` require human approval for that operation? (Policy question — answered by Codex config)

Conflating them leads to incorrect diagnosis. An `item/permissions/requestApproval` is stronger evidence of missing scope; `item/fileChange/requestApproval` alone is not.

---

## Original design: workspace.local.toml [SUPERSEDED by D41–D46] {#original-workspace}

*2026 — superseded 2026-05-01*

**Original decision:** Additional repo roots for harness subprocesses are declared in `workspace.local.toml` at the state root parent. Invalid workspace config blocks all spawns.

**What carried forward:** The hard gate (invalid config = block spawn, not silent ignore) and the centralized declaration pattern both carry forward into the D41 design. Invalid workspace still blocks launch, and topology is still declared once rather than per-spawn.

**What changed:** Config moved into `meridian.toml` / `meridian.local.toml` with named entries and two-tier merge semantics. The standalone file is deprecated.
