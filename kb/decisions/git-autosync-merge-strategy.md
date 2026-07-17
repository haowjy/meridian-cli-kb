# Decisions: Git Autosync — Merge Strategy and Conflict Handling

These decisions were made during the git-autosync rework (2026-05) that fixed the
two-machine delete/move scenario. The prior implementation used `git pull --rebase` and
left the clone wedged in rebase-in-progress state on conflict. See
[../concepts/hooks-and-plugins.md](../concepts/hooks-and-plugins.md) for the current
behavior.

---

## D-autosync-1: Merge instead of rebase, local-wins on conflict (2026-05)

**Context:** The original `git pull --rebase` design broke on two-machine delete/move
scenarios. Machine B, after modifying a file that Machine A deleted, would enter a
permanent rebase-in-progress state. Every subsequent sync skipped silently. Agents on
Machine B worked against stale files indefinitely.

**Decision:** Replace `git pull --rebase` with `git merge origin/<branch>`. Change the
default `conflict_policy` from `"leave"` to `"abort"`. On merge conflict: `git merge --abort`
(local-wins), record conflict metadata, append AGENTS.md notice.

**Why merge over rebase:**

1. **No wedged state.** Rebase-in-progress is the exact failure mode that motivated
   this rework. Merge conflict → `git merge --abort` → clean state. One operation, always
   reversible. No `.git/rebase-merge` or `.git/rebase-apply` directories left behind.

2. **Single conflict point.** Rebase replays N local commits; each is a potential conflict
   point. A conflict on commit 3 of 5 leaves a half-applied rebase. Merge resolves
   divergence as one operation — at most one conflict to handle.

3. **Local-wins naturally.** `git merge --abort` restores the pre-merge state with local
   changes as committed HEAD. No `git reset --hard`, no content destruction. The agent's
   work is preserved and inspectable.

4. **Better rename handling.** Git's merge strategy detects renames across the divergence
   and applies modifications to the new path. Many rename/move scenarios that conflict
   under per-commit rebase replay merge cleanly. This directly fixed the delete/move
   scenarios described in the original diagnosis.

5. **History doesn't matter.** This is an agent-synced docs repo. Nobody reads `git log`
   for a linear narrative. Merge commits are harmless noise.

6. **Simpler recovery.** Resolution is `git merge origin/<branch>` and resolve conflicts.
   No special refs, no SHA recovery from conflict markers, no multi-step rebase state
   cleanup.

**Why local-wins over remote-wins:**
The agent on this machine has context about its own changes. It is better positioned to
resolve the conflict by inspecting both sides than to have its work silently overwritten
by remote. Remote state is always available at `origin/<branch>` for inspection.

**Alternatives rejected:**

- **Rebase (prior design):** Linear history has no value in agent-synced docs repos.
  Rebase-in-progress was the original failure mode. N-commit replay multiplies conflict
  points. Remote-wins via `git reset --hard` is destructive.

- **`leave` with timeout-based auto-abort:** A timer that aborts stale rebase/merge
  operations after N minutes. Rejected because merge eliminates the wedge entirely —
  no timer needed.

- **Content-level CRDT merge:** Overkill for near-term. The merge approach handles the
  common cases. Genuinely unresolvable conflicts are rare enough to handle manually.

**Migration:** Replace `git pull --rebase` with `git merge origin/<branch>` in
`git_autosync.py`. Explicit `conflict_policy = "leave"` configs continue to work.

---

## D-autosync-2: Conflict notice in AGENTS.md — superseded (2026-05; removed 2026-07)

**Original decision (2026-05):** After recording a merge conflict, autosync appended a
delimited conflict notice block inside a managed `<!-- autosync-notices -->` section at the
end of AGENTS.md at the sync root. The notice was staged and committed locally.

**Superseded (PR #422, 2026-07):** Per-conflict AGENTS.md notice rewrites were removed.
No agent ingestion contract existed to consume the notices — no harness reads managed
`<!-- autosync-notices -->` sections and no harness treats their content as actionable
instructions. The conflict JSON in `.meridian/autosync/conflicts/<id>.json` was already
the authoritative record that CLI (`meridian sync conflict show/resolve`) and dashboard
read from. The notices modified user-owned files without a consuming contract — a write
without a reader. Conflict detection is now surfaced only through the conflict JSON and
CLI/dashboard display.

---

## D-autosync-3: Artifact ownership split — `autosync_store.py` (2026-05; relaxed 2026-07)

**Decision:** Extract all autosync artifact reads and writes into a dedicated module
`hooks/builtin/autosync_store.py`. It owns the `.meridian/autosync/` file layout and
JSON schemas and is the single path for both write operations (from `git_autosync.py`)
and read operations (from ops modules for CLI/dashboard).

**Rationale:** The ops layer needs to read conflict metadata and sync state without
depending on the hook implementation. Without a shared module, the path construction
and JSON schema would be duplicated, or the ops layer would import from hook code
(wrong dependency direction). `autosync_store.py` is the narrow shared surface.

**Stdlib-only constraint relaxed (PR #422, 2026-07):** The module now imports
`meridian.plugin_api` for writes and `meridian.lib.platform.locking` for the reentrant
transaction lock. The plugin lock API is intentionally non-reentrant; a complete autosync
transaction (remote/local/resolve as one locked sequence) requires reentrancy. These were
pre-approved import options in the concurrency-by-construction plan. The store also owns
the canonical sync-root lock path and the `transaction()` context manager.

**Ownership boundary:**
- Ops modules know *which* directories are sync roots (context resolution)
- `autosync_store` knows *what* artifacts live inside them (layout, schema, read/write, transaction lock)

---

## D-autosync-4: No conflict refs — local state is HEAD (2026-05)

**Context:** The prior design used git refs (`refs/autosync/conflict/<id>`) to preserve
local state before `git reset --hard origin/<branch>` (remote-wins). This created
cross-machine problems: refs aren't automatically fetched, so other machines couldn't
see the preserved local content.

**Decision:** No conflict refs are created. With merge/local-wins, local state is the
current HEAD — already accessible via standard git. Remote state is `origin/<branch>` —
always available after fetch. Both sides are inspectable without special storage.

**Why no refs needed:** `git merge --abort` doesn't destroy local state. HEAD stays
pointed at the local commit. There is nothing to save in a ref because nothing was lost.

**Rejected alternative — conflict refs (prior design):** Needed under rebase/remote-wins
because `git reset --hard` destroyed local content. With merge/local-wins, local content
is never destroyed. The ref is redundant and its cross-machine unavailability was a known bug.
