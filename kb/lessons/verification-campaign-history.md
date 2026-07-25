# Verification Campaign History

This page preserves the chronology and incident detail from the review campaigns that produced the reusable rules. It is historical evidence, not the current verification procedure. Start with [Verification and Review Discipline](verification-and-review-discipline.md) and [Review Convergence Gate](review-convergence-gate.md).

<details>
<summary>Campaign chronology and incident record</summary>


Patterns from the mars-agents hook-fragment convergence gate (11 review
rounds on PR #138, 7 further rounds on the consolidated PR #147 including
the recovery-seam phase, 2026-07). Each appeared multiple times in
independent instances, grounded in specific evidence below.

## A capability token proves a check happened, not that this call is the thing checked

`WritePermit` carried `target_root` and `surface` as fields. No consumer read
them. Adapter parameters were named `_permit`. The token had the shape of a
capability without its substance: it proved *some* check happened somewhere,
not that *this* write was the operation that was checked.

The round-10 attack demonstrated the gap: once `(".opencode", Hook)` was
`Unconfirmed`, an attacker could take the vacuously-confirmed
`(".opencode", Mcp)` permit, hand it to `write_file_hook_outputs`, and the
replacement wrote. `RemovalToken` was `Copy` and pairless, so one token
authorized any number of removals on any target.

The fix made the operation derive its identity from the capability: `ConfigWrite`
has private fields, its sole constructor `bind_config_entries` consumes the
permit with surface-mismatch rejection, and `into_parts` derives the target
directory from it. `WritePermit` has no `Copy` or `Clone` derives. The
round-10 attack became a compile error, proven by `trybuild`.

**The general form:** when a report claims a type-level guarantee, verify that
the discriminating field is consumed, not merely present. Unused accessors on
a type whose job is to be consulted are evidence of a defect, not dead code
to sweep.

## A regression test that proves its case through an earlier gate is worse than a missing test

The round-9 regression test
`malformed_opencode_legacy_sweep_cannot_replace_file_hook` established its
case by asserting that *preflight* rejected malformed JSON. Execution stopped
before reaching the runtime suppression path the test was named for. The test
was green, the suite reported coverage, and the runtime behavior had no test
at all.

The replacement test (`#[cfg(unix)]`, valid `opencode.json` inside a read-only
`.opencode`) passed preflight and reached the real failure path. Its
fails-first output showed the replacement content landing where the original
should be.

**Cross-check for any regression test:** which gate stops execution, and is it
the gate under test? A test that names a behavior but exercises only the gate
before it reports coverage that does not exist.

## Verification that examines nothing reports success

Three instances in one review cycle:

1. `meridian mermaid check` silently passed an HTML file it could not parse as
   Mermaid. The check succeeded because it found nothing to check, not because
   the content was valid.

2. A post-sync consistency oracle exempted the exact failure mode it existed to
   detect. The `"failed to remove"` clause in its exemption set switched the
   checksum check off precisely where content-replacement-after-failed-removal
   happens. That is the only condition under which the bug can occur, and the
   oracle was excusing it.

3. A git command ran in the wrong repository because shell cwd persisted from a
   prior step. The command succeeded against a clean working tree that was not
   the one under review.

**The pattern:** verification that reports success without having examined the
thing is worse than no verification, because it also removes the impulse to
look. The oracle is the most pernicious form: it is designed to catch a
specific class of defect and carries a carve-out that exempts that exact class.

## A diff shows what changed, not what executes

Twice in the same review cycle, the reviewer concluded a defect was
branch-introduced by reading `git log -L` on the relevant region, and was
corrected by running both binaries (main and branch) against identical
fixtures.

First instance: commit `ca8a166` visibly replaced `discover_mcp_items(...)?`
with a warn-and-continue `match`. The conclusion was that the branch had caused
MCP data loss by continuing past a parse failure. The two-binary probe showed
main also continued past the parse failure -- the `?` never fired because the
parse error was swallowed below that call. The behavioral delta was the warning
printing twice instead of once.

Second instance: the actual branch-introduced defect (F5, a remove-then-write
ordering window in hook bindings) was in a different function entirely, one
the diff-reader had not suspected.

**The rule:** confidence from reading a hunk is not evidence. A diff shows what
changed in the source; only running the binary shows what changed in behavior.
When the question is "did this branch introduce this defect," the test is two
binaries and one fixture, not one diff.

## Repeated findings of the same class in one function signal a wrong seam, not wrong code

Blocking-finding trend across eleven gate rounds: 8, 5, 4, 1, 2, 1, 0,
2, 2, 0. Ownership-record bugs appeared in six consecutive rounds (4 through
9), all in one function (`compile_config_entries`), all answering the same
question differently at each call site: *when a removal step for a target
fails, what is retained and which writes are suppressed?*

Each round's fix was correct in isolation. Each fix opened or exposed the next
defect. The pattern: if a fix to one call site opens a defect at another, and
the defects are the same class, the decision is made per-call-site where it
should be per-seam.

Round 9 stopped patching and escalated to a redesign brief. The replacement
(`surface_ownership/retention.rs`) made the wrong states unrepresentable rather
than checked: `RemovalOutcome::Confirmed` has no field for retained records,
and every replacement write requires a `WritePermit` that only the outcome
table issues. Both properties fall out of the types; call-site discipline is
no longer required.

**When to stop patching:** the trigger is repeated findings of the same class
within one function or module, each fix correct and each introducing the next.
That is not a defect sequence; it is a missing invariant. A redesign brief
costs one round. Continuing to patch costs N more rounds, each with the same
diagnostic overhead and the same probability of introducing the next bug.

## A hash that follows symlinks grants ownership to whatever the symlink points at

Four instances of the same bug class across three review rounds, all in
mars-agents (tracked as mars-agents#152):

1. `compute_hash(dest)` in the no-op fast path follows directory symlinks.
   Replacing a synced skill directory with a symlink to an identical external
   directory makes the next sync say "already up to date," leave the symlink,
   and keep an `installed` record. Mars claims a managed copy while the
   destination is user-controlled indirection.

2. `directory_trees_content_equal` (the old comparator the fast path
   bypassed) had its own symlink handling, but the replacement
   checksum-only comparator did not.

3. `check_unmanaged_collisions()` -- pre-existing on `main`, not
   branch-introduced. Same class: follows symlinks when deciding whether a
   path is owned.

4. v2 lock promotion accepted `is_file()` only, tombstoning every directory
   output. The fix added structural validation (nested-shape check plus
   directory-manifest hash) that rejects root and nested symlinks before
   hashing.

**The rule:** ownership-deciding comparisons must use
`symlink_metadata`-based structural validation, never content-following
hashes. A content hash answers "what bytes are here" and follows symlinks
to answer it. Ownership needs "what filesystem object is here" -- a question
about structure, not content. The two answers diverge precisely when a
symlink points at a correct copy, which is the case an attacker (or an
innocent user organizing their workspace) would produce.

The shared structural comparator that replaced the fast-path hash rejects
root and nested symlinks, rejects non-directory roots, includes empty
directories, and compares file content hashes -- but only after structural
validation passes.

## The sync pipeline assumes staged tree == total desired intent

This is the structural invariant that drove the round-9 redesign escalation
and the recovery-seam design (D92). Two competent implementation lanes each
attempted to add a "cannot currently interpret, must not touch" exception
category to the sync pipeline. Both failed with rising severity.

The sync pipeline flows through resolve, stage, discover, diff, plan,
target-sync, finalize, retention, and lock. Every stage is a persistence
site. An exception category must be enforced at every one, or it corrupts
state. Omission-as-absence (round 4) missed the desired-state engine;
staging absence read as removal intent, and `mars remove a` destroyed
unrelated dependency b's hooks. Freeze-as-carried-state (round 5)
threaded `frozen_hook_sources` through resolution but leaked at three
persistence sites on the first attempt, with five more stages named by
the review as eventually needing freeze awareness.

The design that worked (Shape A, D92) deleted the exception category
entirely. Recovery commands persist intent only and halt before the
compiler; strict sync remains the sole materializer. The invariant holds
by construction: when anything is unreadable, the only writes are the
user's own intent files, which are not derived from package content.

**The transferable rule:** prefer deleting an exception category over
perfecting it. If every persistence site must honor a distinction, and
two competent attempts each missed a different subset, the distinction
costs more to maintain than the value it provides. The right question is
whether the feature that requires the exception can be decomposed to
avoid it.

## Runtime probe gate step for schema-breaking releases

Three clean review rounds (diff-and-test) approved a branch whose
real-world upgrade path was broken. Old-binary staging content from a
locked v0.8.9 package used the removed hook schema (`event=` singular,
`[action]` block). Config load rejected it, and every recovery
command -- including `mars upgrade`, the escape hatch -- was bricked by
the state it existed to fix. Deleting staging did not help: the archive
cache held the same content and re-materialized it.

The review gate could not have caught this. It reviewed diffs and tests,
not a live upgrade from real old-binary state. The test suite exercised
the new schema thoroughly and the old schema's error path, but no test
constructed state that a prior binary release would actually have written
to disk.

**Standing gate step for schema-breaking releases:** after the
diff-and-test convergence gate closes, probe the release binary against a
copy of a real old-state project. The probe must exercise the full upgrade
path (promotion, sweeps, intent writes, final sync) against artifacts
written by the prior binary, not reconstructed from the current code.

This extends the existing convergence gate process (below) rather than
replacing it. The diff-and-test gate catches code-level defects; the
runtime probe catches state-level defects that live in the gap between
"the code is correct" and "the upgrade from real prior state works."

## Measurement traps: exit codes and cwd drift

Two recurring trap classes across this work item, four occurrences of cwd
drift alone:

**Exit codes through a pipe.** `$?` captures the exit status of the last
command in a pipeline, not the first. `mars repair | tail -5` reports
tail's exit status; if the preceding command exited 2 (a recovery halt),
the pipe masks it and reads 0. Exit codes must be captured without a pipe
(`mars repair; echo $?`), or `PIPEFAIL` must be set.

**cwd drift into the wrong repository.** Shell cwd persists between
calls. `git status`, `gh pr view`, `gh run list`, and `git rev-parse` all
silently answer about whatever repository cwd points to, returning
plausible wrong answers rather than visible errors. Four occurrences on
this work item: `gh pr view 138` queried the wrong repo; push-state
checks ran against `meridian-cli` instead of `mars-agents`; `gh run list`
returned empty because cwd had drifted; `gh pr checks 147` said "Could
not resolve" for the same reason.

Both are instances of the same pattern described in "Verification that
examines nothing" above: a tool returns a real answer about something
other than what was asked. The failure is never a visible error -- it is a
plausible answer to a question that was not intended. The defense is an
explicit identity check (`pwd`, `headSha` vs branch tip, `git remote -v`)
rather than careful reading of the output.

## Convergence gate: the process that closed this review

The 18-round review (11 on PR #138, 7 on the consolidated PR #147
including the recovery-seam phase) used a specific process that drove
severity to zero:

1. **Frozen read-only worktrees.** Each review round ran against a dedicated
   worktree checked out at a specific commit. The reviewer could not see
   work-in-progress changes from a fix lane. This prevented the two
   phantom-finding incidents that occurred when a reviewer and an editor
   shared a checkout.

2. **Findings produce fix lanes, not patches.** Each round's blocking
   findings spawned a separate fix lane with a scoped prompt. The fix lane
   committed to the implementation worktree. A new review round ran against
   the fixed tip in a fresh frozen worktree.

3. **Severity must fall each round.** The blocking-finding trajectory was
   the primary convergence signal: 8, 5, 4, 1, 2, 1, 0 (rounds 1-7 on
   PR #138), then 2, 2, 0 (rounds 8-10 on retention redesign), then 3, 1,
   0 (rounds 1-3 on the consolidated PR), then probe + 2, 3H (recovery
   freeze attempts), then 1H, 0 (post-design rounds 6-7). Sustained
   non-decrease triggers a scope decision -- round 9 escalated to a
   redesign brief after six consecutive rounds of same-class ownership
   bugs; the recovery freeze triggered a second escalation after two
   rounds with rising severity.

4. **Runtime probe for schema-breaking releases.** After the diff-and-test
   gate closes, probe the binary against real old-state artifacts. See
   "Runtime probe gate step" above. This step was added after the recovery
   seam phase proved that diff-and-test alone cannot catch state-level
   upgrade failures.

5. **Cross-model review for correlated blind spots.** Eleven rounds all ran
   on one model family, so their blind spots were correlated by construction.
   A cross-model reviewer from a different family independently returned
   SHIP and found tighter arguments for several invariants. The diversity
   was limited by infrastructure (one alternative harness had exhausted
   credits, another failed pre-init), which is worth recording rather than
   silently letting "two reviewers ran" stand in for "two independent
   perspectives ran."

**One editor per checkout.** The rule that emerged from two phantom-finding
incidents: read-only or doc-only lanes get the frozen review worktree or
their own; only one lane edits a given checkout at a time. A review run
against a shifting tree cannot be trusted on anything it measured.

## CI trigger gap: retargeting a PR does not queue a run

GitHub's `pull_request` event triggers default to `opened`, `synchronize`,
and `reopened`. Changing a PR's base branch fires `edited`, which is not in
that set. Retargeting a PR base therefore does not queue CI, and the push's
`synchronize` may not fire against the new base.

The symptom is invisible: the PR page shows green checks from a prior run
against the old base, and `gh pr checks` reports those results without
indicating they cover a different commit. Only comparing the run's
`headSha` against the current branch tip catches the gap.

**Workaround:** close and reopen the PR after retargeting. This fires
`reopened` and queues a fresh run against the real head. Adding `edited` to
the trigger list would also work but fires on title/body edits too.

This was discovered when PR #147 was retargeted from `feat/hook-fragments`
to `main` for consolidation -- the merged tree sat unverified by CI while
the PR looked healthy.

## Static gate fully green on a shipped ecosystem break

A second confirming instance of the runtime-probe principle, from the
release-sequencing failure rather than a review gate.

mars-agents 0.12.0 shipped a breaking hook-schema change. The meridian-base
migration answering it sat unpushed since 24 Jul. Every published
meridian-base was rejected by the new mars binary.

The meridian-cli pin-bump PR passed its entire static gate: ruff 0, pyright
0, 1395 tests passed. Nothing in meridian-cli's code changed, and nothing
in meridian-cli's test suite exercises real `mars sync` against a project
with hooks. The static gate was incapable of detecting the break.

The blast-radius assessment was itself wrong: it grepped root `mars.toml`
files for `[[hooks]]`. Hooks are declared in `hooks/<name>/hook.toml`
inside resolved packages -- a location the search could not reach. The
negative result was structurally invalid, not merely unlucky.

A runtime probe -- `mars sync` against a throwaway copy of
meridian-dev-workflow -- caught the break and stopped the PR. The same
pattern as the schema-breaking upgrade lockout instance above, extending
it from within-repo schema changes to cross-repo dependency breaks.

The discipline remains the same: static gates verify local correctness;
behavioral breaks at the boundary between repos require running the actual
binary against real project state. See [release-sequencing.md](release-sequencing.md)
for the full incident and cross-repo release discipline.

## Related

- [release-sequencing.md](release-sequencing.md) -- the cross-repo
  release-sequencing lesson from this incident
- [residue-cleanup-discipline.md](residue-cleanup-discipline.md) -- the
  ownership-violation instances that drove rounds 4-8
- [../decisions/package-management.md](../decisions/package-management.md#d91-native-hook-fragments-replace-command-synthesis-2026-07) -- D91 decision, including the retention seam
- [../decisions/package-management.md](../decisions/package-management.md#d92-shape-a-recovery-seam----halt-before-compilation-when-hook-surfaces-are-unreadable-2026-07) -- D92 decision, recovery seam design
- [../architecture/mars-compiler.md](../architecture/mars-compiler.md) -- compiler module map including `surface_ownership/retention.rs`

</details>
