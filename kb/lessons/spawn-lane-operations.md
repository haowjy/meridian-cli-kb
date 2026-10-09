# Spawn Lane Operations

Lessons from running implementation and measurement lanes as `meridian spawn` children.
They concern how a lane can fail without the failure being visible in its result.

## Declared Backups Do Not Cover a Mid-Run Auth Failure

**What happened (2026-09-25):** two Codex-routed spawns, a `gpt-dev` fix lane and a
`prober` measurement lane, failed within 35 s. The error was `401 Unauthorized:
Incorrect API key provided: sk-svcac…`, from the Codex responses endpoint. Neither made
changes. Both profiles declared Claude backups, but no fallback happened.

**Why:** declared backups are prelaunch candidates. Mars picks one when the primary
harness or model is unavailable at resolution time
([model policies](../concepts/model-resolution/model-policies.md#fallback-participation)).
A credential rejected after launch is a run failure, not an unavailable harness.

**What worked:** relaunch each lane with an explicit `-m` naming the profile's declared
Claude backup (`-a gpt-dev -m opus`, `-a prober -m sonnet`), resuming with `--from`. The
key was the user's; the tech lead did not touch credentials and told the user instead.

**The lesson:** when several spawns on one harness fail fast with an auth error, stop
relaunching on that route. Pin a declared backup explicitly, and report the credential
problem to the user. Every spawn on that route fails until the key is fixed.

## A Claude-Routed Lane Can End Its Turn Before Its Gate Finishes

**What happened (2026-09-25):** fix lane D ran on the Claude harness (`gpt-dev -m
opus`). It committed its fixes, started the full preflight as a background command,
then ended its turn "waiting" for it. The spawn finished without the preflight result
and without a report. The tech lead reviewed the commits, ran the gates and wrote the
report.

**Why:** in this run, the background command finishing did not bring the headless lane
back for another turn. The spawn's result was whatever had been written before the turn
ended. The exact harness behavior was not investigated further.

**The lesson:** briefs for Claude-routed spawns should say to run gates and long tests
in the foreground, with a timeout that fits the gate, and to write the report before
ending the turn. When a spawn ends without a report, check its commits and rerun the
gates before merging.

## A Lane Commit Can Carry Another Step's Staged Deletions

**What happened (2026-09-26):** PR 3's deletion lane staged its `git rm` deletions
(`state/history.py`, several test modules) before committing its first step, the
dogfood migration. The deletions went into that commit, which then did not build on
its own: `spawn_manager` still imported the deleted module. The harness blocked
`git reset`, `commit --amend` and `restore --source`, so the lane could not split the
commit. The next commit completed the step.

**What worked:** the lane reported it, and the tech lead squash-merged the two commits
(`a567cae4`). The review confirmed that the squash builds and passes on its own.

**The lesson:** when a lane reports a commit that does not build alone, merge the lane
with squash rather than preserving a broken intermediate commit. Briefs for deletion
lanes should say to stage deletions only with the commit that removes their last
importer.

**Provenance:** `work:native-harness-session-identity` (`decision.md` "Codex harness
auth failure → Claude backups" and "Recheck PASS WITH FIXES; V2 partial; lane D";
`evidence/pr2-fix-d-report.md`; spawns `p7149`, `p7150`, `p7151`; for the squash
lesson, `evidence/pr3-p3a-report.md`, `review/pr3-review.md`, `spawn:p7155`).

## Parallel Lanes Can Collide on Test Basenames

**What happened (2026-10-08):** two parallel lanes each added a new test directory
without an `__init__.py`. The notify-core lane wrote `tests/unit/notify/test_service.py`
and the idle-core lane wrote `tests/unit/idle/test_service.py`. Each lane's full suite
passed. Once both were merged, pytest stopped at collection with `import file
mismatch`. Under pytest's default import mode, two test files with the same basename
in non-package directories map to one module name. The merge contained no
conflict, and no reviewer reading either diff could see the problem.

**What worked:** one commit added empty `__init__.py` files to the new test
directories (`b5fc8682`). The tech lead then cherry-picked it into every lane still
running, before their full-suite gates.

**The lesson:** when lanes will add test directories in parallel, the brief should
fix the convention first: every new test directory is a package, or test basenames
must be unique. The first full-suite run on the merged tree is the earliest point
where a collision shows up, so run it before handing the merge to review.

**Provenance:** `work:idle-cache-notify` (`reviews/G1-review.md` "Gates"; spawn
`p7454`; commit `b5fc8682`, cherry-picked as `ec6aa8d9` and merged with `icn/g0fix`
as `520de295`). The gate context is summarized in [Idle Notifications — Review
gates](../decisions/idle-notifications.md#review-gates).
