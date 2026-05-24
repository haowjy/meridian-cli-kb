# Decisions: Spawn CWD / Worktree / Reference Anchor

Decisions from PR #248 (2026-05-22): how Meridian separates project authority from task working directory and reference file resolution during spawn operations.

See [architecture/launch-system.md — Authority/Task Domain Split](../architecture/launch-system.md#authoritytask-domain-split-pr-248) for the mechanism. See [decisions/launch.md — D-control-root-task-cwd-split](launch.md#d-control-root-task-cwd-split-split-execution_cwd-into-control_root--task_cwd) for the earlier PR #210 split that introduced the two-field model.

---

### D-single-reference-anchor: All relative `-f` paths resolve from one anchor (task_cwd)

**Decision:** All relative `-f` paths resolve from a single `reference_anchor`, which equals `task_cwd`. There is no per-file fallback (try task_cwd, then project_root, then cwd).

**Evidence:** Web research surveyed Docker, Terraform, Git worktrees, AutoGen, npm workspaces. Every tool uses a single named anchor per concern — Docker build context, Terraform `-chdir`, Git's `GIT_WORK_TREE`, AutoGen `work_dir`. Per-file fallback heuristics are an anti-pattern across all surveyed tools.

**Rejected alternative:** Per-file fallback (try task_cwd, then authority_root, then cwd). Rejected because: production tools universally avoid it; it creates "it works on my machine" drift; non-obvious resolution order makes debugging painful.

**Rejected alternative:** Keep authority_root as anchor (prior behavior). Rejected because it breaks the mental model — relative paths should be relative to where the agent works, not to the config root.

---

### D-kb-resolves-from-authority: `kb:` prefix resolves from authority_root, not task_cwd

**Decision:** `kb:path` reference paths always resolve against the KB directory derived from `authority_root`. They are independent of `task_cwd`.

**Rationale:** KB is project-level infrastructure — it belongs to the "platform domain" (authority_root). Worktrees have no KB directory. If KB lookup followed task_cwd, any worktree spawn would break `kb:` references.

**Confinement:** The resolved KB path must stay within the KB root directory. `kb:../outside.md` — any path that escapes via traversal — is rejected with a clear error.

---

### D-stale-worktree-hard-error: Stale worktree_path is a hard error, not silent fallback

**Decision:** If a work item has a configured `worktree_path` that no longer exists on disk (deleted, moved), spawn fails with a clear error. It does NOT silently fall back to `authority_root`.

**Rationale:** The user configured worktree isolation intentionally. Silent fallback risks editing the wrong checkout. Only `--no-worktree` explicitly opts out.

**Only escape hatch:** `--no-worktree` flag bypasses the stale check and forces `task_cwd = authority_root`.

---

### D-explicit-work-boundary: Explicit `--work` is a hard selection boundary for worktree resolution

**Decision:** When the user specifies `--work <item>`, only that item is consulted for worktree resolution. If the named item has no worktree_path, `task_cwd` falls back to `authority_root`. The ambient session work attachment is NOT consulted as a fallback.

**Rationale:** Naming a specific work item should give deterministic behavior. Falling through to ambient work creates "which item supplied this path?" confusion that is hard to debug, especially across continue/fork sessions.

**Contrast:** When no explicit `--work` is given, the ambient session work attachment IS consulted (standard priority chain). The boundary applies only to explicit `--work`.

---

### D-at-removed-immediately: `@` removed immediately for `-f`, not deprecated

**Decision:** `-f @path` syntax for KB-relative file references is removed outright with a helpful error suggesting `kb:path`. No deprecation period.

**Rationale:** No external users. No CLI help text advertising `@`. No test coverage for `-f @` paths. The only risk was internal prompt packages referencing `@` in documentation — mitigated by updating source repos before shipping the CLI change.

**Scope:** Only `-f` file reference `@` is removed. The `@` prefix in `context_ref.py` (spawn/session references) and `query.py` (context refs) are different semantics and unchanged.

---

### D-pi-opencode-first-cwd: PI/OpenCode-first adapter cwd policy

**Decision:** The design center for actual-process-cwd behavior is PI/OpenCode (and Codex). These harnesses set the actual process cwd to `task_cwd`. Claude `-p` is legacy/best-effort: process cwd stays at `authority_root`, and a task-cwd instruction block is injected into the system prompt instead.

**Rationale:** PI/OpenCode is the primary subagent harness. Claude subagents are legacy. The fallback instruction path (`LaunchDirectoryContext.requires_task_cwd_instruction`) is the mechanism — it is `True` only when actual process cwd cannot be set to the logical task cwd.

---

### D-managed-vs-manual-worktree: WorktreeMetadata.managed distinguishes ownership semantics

**Decision:** `WorktreeMetadata.managed` (in `__status.json`) distinguishes Meridian-owned git worktrees from manually assigned directories:

- **`managed: true`** — Meridian created the git worktree via `work start --worktree`. Meridian owns lifecycle: removes on `work done`/`work delete` if not shared with other active items. May rename if the work item is renamed.
- **`managed: false`** — user assigned via `work set-worktree`. Meridian treats as read-only assignment: never removes the directory. `work clear-worktree` removes only the assignment.

**Shared paths:** Multiple work items may point to the same `worktree_path`. No uniqueness constraint. Managed cleanup skips removal when the path is shared with another active work item.

**Rationale:** Unmanaged directories (existing checkouts, parent repo paths) must not be deleted by Meridian. The managed/manual split makes ownership explicit in the stored record rather than requiring path-heuristic inference.

---

### D-workspace-projection-task-cwd: task_cwd outside authority_root must be in workspace projection

**Decision:** When `task_cwd` differs from `authority_root`, the launch layer ensures `task_cwd` is included in workspace projection for harnesses that support it. If the harness cannot project the path, process cwd stays at `authority_root` and a task-cwd instruction is injected.

**Rationale:** Harness sandboxes restrict filesystem access to projected roots. An agent working in `task_cwd` needs read/write access there — without projection, its file operations silently fail or are restricted to the authority root.

---

---

Decisions from PR #266 (2026-05-23): worktree auto-provisioning, canonical path policy, temporary managed worktrees, and crash-safe pending records.

---

### D-worktree-flag-means-ensure-canonical: `spawn --worktree` means provision-or-recover, not assert

**Decision:** `spawn --worktree` now means "ensure a Meridian-managed canonical worktree at `<repo>.worktrees/<name>`, creating or recovering it if needed, then launch in it." It is an ensure operation, not an assertion that a path already exists.

**Before PR #266:** `spawn --worktree` required the work item to already have a configured worktree path. It did not create one.

**Rationale:** Requiring pre-configured worktrees created friction — agents had to invoke `work set-worktree` manually before spawning. The ensure model lets `--worktree` act as a single-flag provisioning gate. Implicit `--work <id>` without `--worktree` still falls back to `authority_root` when no worktree is configured.

---

### D-managed-metadata-authoritative: Persisted managed metadata wins over `--repo` on re-ensure

**Decision:** Once a work item has persisted managed worktree metadata (`repo_path` + `name` in `__status.json` with `managed=true`), subsequent ensures always reuse that metadata. `--repo` selects the target repo only on the first managed ensure; it cannot silently switch a managed worktree to a different repo afterward.

**Rationale:** Allowing `--repo` to redirect an established managed worktree would silently create a second worktree in the wrong repo. The stored record is the source of truth for managed worktrees; `--repo` is an initial configuration hint, not an ongoing override.

---

### D-temp-managed-worktrees: Session-scoped temp worktrees when no work item is active

**Decision:** When `work worktree --ensure` is called with no active or explicit work item, Meridian creates a session-scoped temporary managed worktree keyed by `spawn_id` (in a spawn) or `chat_id` (in a primary session). These are tracked in `~/.meridian/projects/<uuid>/worktree-temp/<key>.json` separately from work item state.

**Rejected alternative:** Fail clearly when no work item is active. Rejected because isolation is useful even without a work item — e.g., for one-off spawns that need a clean worktree context.

**Contrast:** The explicit-work-id form (`work worktree <id> --ensure`) does NOT fall through to a temp worktree when the named item is missing — it fails clearly. The no-work-item temp behavior applies only to the zero-argument `--ensure` form.

---

### D-canonical-path-no-managed-override: Managed worktrees use enforced canonical paths

**Decision:** Managed worktrees always live at `<repo>.worktrees/<name>`, enforced by `managed_worktree_path()`. No `--path` option exists for managed worktrees. Custom/arbitrary paths go through `work set-worktree` (manual assignment, `managed=false`). Non-canonical managed paths detected at runtime are treated as drift and fail clearly.

**Rationale:** A canonical path policy makes managed worktrees predictable and discoverable. Manual `set-worktree` covers the legitimate "assign an existing directory" use case.

---

### D-temp-pending-rollback: Temporary worktree creation uses crash-safe pending records with rollback-on-failure

**Decision:** Temporary worktree creation writes a `status=pending` record before invoking `git worktree add`, then sets `status=ready` on success. If provisioning fails (including `skipped_not_git_repo`), the temp record is deleted (rollback-on-failure) rather than left in a failed/pending state.

**Rationale:** A "failed" state in the temp store buys nothing — the user can retry, and a stale pending record would block retry. Rollback-on-failure keeps the store clean. Work item worktrees use the same pending/rollback discipline.

---

### D-worktree-path-sep-normalization: `WorktreeMetadata` paths normalize to POSIX separators at storage boundary

**Decision:** `WorktreeMetadata.path` and `repo_path` fields normalize backslash separators to forward-slash at the Pydantic validation boundary in `work_store.py`. Stored metadata is stable across platforms. The coercion function `_coerce_worktree_metadata()` detects normalization and marks the record for rewrite.

**Rationale:** Windows writes backslash paths natively; reading those on POSIX (or comparing in tests) produces spurious mismatches. Normalizing at the storage boundary ensures one canonical form in JSON regardless of which platform wrote the record.

---

## Related

- [../architecture/launch-system.md](../architecture/launch-system.md) — full mechanism (see Authority/Task Domain Split section)
- [../concepts/reference-resolution.md](../concepts/reference-resolution.md) — -f resolution algorithm and kb: semantics
- [../codebase/work-items.md](../codebase/work-items.md) — worktree commands and managed/manual distinction
- [launch.md](launch.md) — see D-control-root-task-cwd-split for the PR #210 two-field model that PR #248 extends
