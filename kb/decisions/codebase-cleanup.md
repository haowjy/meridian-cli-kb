# Decision: Codebase Cleanup Program

Decisions about the scope and edges of the [cleanup program](../codebase/cleanup-program.md)
that followed the 2026-10-13 thermo-nuclear review. The program page owns the
diagnosis, phase plan and status; this page owns why the lines were drawn where
they are.

## D-cleanup-scope: phases 1–2 now, 3–5 tracked; stacked on the pyright branch

**Decision (user, 2026-10-13):** Run phases 1 and 2. Defer phases 3–5, the standalone items,
and bugs B5 and B9 (whose fixes belong to those phases) to issue
[#569](https://github.com/haowjy/meridian-cli/issues/569). The cleanup branches
start from `fix/pyright-strict` (PR #570), not `main`, and ship as stacked PRs:
`chore/cleanup-p1` → `chore/cleanup-p2`, each retargeted as its base merges.

**Why this cut** (the program's framing; the user chose "Phase 1 + 2" from the backlog's options): Phases 1 and 2 are low-risk and enabling. Phase 1 is bug fixes
and deletions with proof of no readers. Phase 2 builds the shared config and IO
seams that phases 3–4 would otherwise have to work around. Phases 3–4 carry four
behavior changes that each need sign-off (fork unification, the
`SpawnActionOutput` wire format, agent-mode bootstrap, primary honoring
`request.env`), so they wait until those are decided.

**Why stack on the pyright branch:** Phase 2 restructures the same files that
branch had just typed (`json_shapes`, settings, the process-scope sidecar).
Basing on `main` would have guaranteed a conflict at merge. When `main` moved
(v0.9.8), it was merged into `fix/pyright-strict` rather than rebased, because
six cleanup lanes had already branched from it.

**Rejected:** Taking all five phases now (the backlog's other option) — the
phase-3/4 behavior changes are not yet signed off. Basing on `main` — certain
conflicts with the typing work.

## D-b7-via-546: the spawn wait contention failure is fixed by PR #546

**Decision (2026-10-13):** B7 (`meridian spawn wait` failing with "History index has
unresolved sources (7 files)" while six spawns started; a retry succeeded) is
not fixed in the cleanup program. PR #546 owns the fix; B7 closes when it merges.

**Root cause** (`spawn:p7749`): no-argument wait discovery queries the history
index. Its bounded catch-up treats a dirty marker whose source lock is busy, or
whose turn comes after the two-second deadline, as unresolved, and
`HistoryIndex.query()` then raises `HistoryIndexIncomplete`. The seven "files"
were valid markers held by concurrent startup writes. Failing closed is correct:
answering from a stale SQLite projection could omit a just-created spawn, which
would break files-as-authority. The bug is that the UX boundary turns a
transient, replayable condition into a hard failure.

**The fix in #546:** a typed `HistoryIndexBusy` for pending-source coverage,
retried within the wait budget, plus session-scoped discovery. Corruption,
schema, generation and authority failures are still not retried.

**Rejected:** Accepting stale discovery on contention (violates
files-as-authority). Transplanting the #546 commit into a cleanup lane (broader
than the bug, and already in review on its own PR). The structural follow-up —
separating projection coverage from query policy — is state#12, a standalone
item in #569.

## D-spawn-archive-deleted: delete the orphaned overlay, do not keep it

**Decision (2026-10-14):** The spawn-archive overlay — `lib/spawn/archive.py` (briefly moved
to `lib/state/archived_spawns.py`), `SpawnApplicationService.archive()`, the
`spawn.archived` telemetry event, its lock and its tests — was deleted in phase 1.

**Why:** Phase 1 removed the archived app-server extension specs. One of them,
`ARCHIVE_SPAWN_SPEC`, was the only caller of `archive()`. Afterwards nothing
wrote the archive set, nothing read it, no listing applied its "hidden from
default listings" filter, and the telemetry event could never fire. In the same
phase another lane had mechanically moved the module and written fresh docs
describing it as live behavior — dead code with new documentation is the worst
kind, because agents load the docs and reason from them (`spawn:p7763`).

**Rejected:**

| Alternative | Why rejected |
|---|---|
| Keep the moved module and its docs | Describes a visibility overlay that no command applies |
| Keep it behind a flag until a caller returns | There is no consumer to gate; a future spawn-archive feature should be designed against today's listing and history index, and git history has the old code |

## D-auto-init-inert: `auto_init_cwd` is a TODO, not a feature

**Decision (2026-10-14):** The descriptor flag `bootstrap_plan.auto_init_cwd` is documented as
inert and tracked as a must-do in `cli/.context/TODO`, not presented as behavior.
Phase 1's changelog line claiming B3 fixed auto-init for `spawn done` /
`spawn rearm` was removed.

**Why:** Since #388 (July 2026), `resolve_cli_project_root()` reports
`established=True` for every root, so the bootstrap branch that consumes the
flag is unreachable. The phase-1 reviewer checked in a clean environment: in a
fresh directory, `spawn done`, `spawn rearm`, `spawn cancel` and `work switch`
each still write `meridian.toml`, because `prepare_for_runtime_write()` creates
the project identity, not auto-init. B3's descriptor fix is still correct (the
commands now classify exactly), but it changes nothing a user can observe, and a
changelog entry claiming otherwise would mislead.

**Open:** Delete the flag, or restore a real established-project check (the
unused `cwd_has_project_id()` is the literal-marker test such a check would use).

> [!FLAG] **Needs human review.** Choosing between deleting `auto_init_cwd` and
> restoring an established-project check decides whether write commands may
> create a project in any directory. Today they do.

## Revisit when

- #546 merges: B7 is closed; drop it from the program's open list.
- A phase-3/4 behavior change is signed off: record it here before its phase starts.
- Someone needs spawn archiving again: design it fresh against current listings
  rather than restoring the deleted overlay.

## Sources

`work:pyright-strict` (decisions D8, D10, D13, D16, D17), `spawn:p7749` (B7
root cause), `spawn:p7763` (phase-1 integration review), `spawn:p7769`
(review fixes), PR #546, issue #569.
