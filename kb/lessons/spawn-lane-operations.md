# Spawn Lane Operations

Lessons from running implementation and measurement lanes as `meridian spawn` children.
Most concern how a lane can fail without the failure being visible in its result;
two concern how a lane reaches what it does not own: processes matched by
command-line text, and the repository's shared stash list.

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

The same holds for model capacity. On 2026-10-16 two Codex `gpt-5.6-sol` coder lanes
(`spawn:p8250`, `spawn:p8252`) died mid-run at model capacity. No backup was tried;
the work was resumed with `--from` on Opus (`work:memory-audit` M8, M12).

**The lesson:** when several spawns on one harness fail fast with an auth or capacity
error, stop relaunching on that route. Pin a declared backup explicitly, and report the credential
problem to the user. Every spawn on that route fails until the key is fixed.

## A Claude Lane's Background Gate Was Killed, and Reported Succeeded

**What happened:** on 2026-09-25 a Claude-routed fix lane started its preflight as a
background command, ended its turn "waiting" for it, and finished with no gate result
and no report. The lesson recorded then was a prompting workaround: tell Claude lanes
to run gates in the foreground. Nobody asked why the harness didn't come back. The
same failure recurred at least four times in meridian-flow (2026-10-12 to 10-17),
each time published `succeeded` with a report saying "the gate is still running",
until the owner flagged it as blocking (#655).

**Why:** Meridian finalized a Claude child at its first `result` and killed its
process group. Claude Code itself would have waited: raw `claude -p` holds for its
background tasks and takes a follow-up turn. The defect was Meridian's, and months
old. Meridian's own spawn guidance tells agents to run `meridian spawn wait` in
native background execution, so it steered Claude agents straight into the bug the
workaround was meant to avoid. The fix is in code:
[D-claude-native-hold](../decisions/completion-drain.md#d-claude-native-hold).

**The lesson:** a prompting workaround for a harness defect is a deferral, not a fix.
When an agent-behavior lesson papers over something the system does wrong, file the
system defect at the same time. Check whether the project's own guidance contradicts
the workaround. The fix shipped in v0.10.21; on an older meridian, a Claude child
that ends while its background work runs still reports `succeeded`.

**Provenance:** `work:claude-bg-quiescence` D1, D2 (investigator `spawn:p8374`,
bisect `spawn:p8382`); the 2026-09-25 lane is cited below.

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

## A Pattern `pkill -f` Killed the Owner's Primaries

**What happened (2026-10-21, 17:31Z):** a coder lane (`spawn:p14102`) cleaned up its
own wait loop with

```
pkill -f 'until grep -qE "passed|failed|error" /tmp/full-tests.txt'
```

Three of the owner's interactive tech-lead primaries, in other tmux sessions, died
with SIGTERM within ten seconds. The lane itself survived and finished.

**Why:** `pkill -f` matches the pattern as an extended regex against every
process's full command line, and shell quoting does not escape `|`. The pattern was
three alternatives, one of them the bare word `failed`. Meridian launches Claude
with the agent profile inline on argv (`claude --agents <json>`), so the profile's
prose is part of the command line. The tech-lead profile contains "A spawn that
failed before doing any work", so every running tech-lead primary matched. The
coder's own profile did not, which is why the killer lived. It was not a Meridian
regression and not the test suite (root cause `spawn:p14106`).

**The lesson:** never select processes by command-line text (`pkill -f`,
`killall`, `pgrep -f | xargs kill`) on a shared host. Every agent's command line
carries prompt text, so any pattern can match a stranger. Stop what you own by
handle: the PID you recorded when you started it, the harness's background-task
id, or the process group you created.

**Status of the fixes:** none scheduled. Two were proposed: cleanup guidance in
the coder and prober profiles, and keeping profile prose off the Claude argv. The
owner treated the incident as a one-off ("if its a one time thing, we can ignore
it"). If it recurs, the argv change is the structural fix; profile guidance alone
is the weakest guard.

**Provenance:** `work:session-browse-redesign` `decisions.md` (incident entry) and
`sigterm-*.md`; root cause `spawn:p14106`; owner ruling in `chat:c13711`.

## `git stash` in a Worktree Popped the Owner's Stash

**What happened (2026-10-27):** a coder lane (`spawn:p9788`), working in its own
worktree of meridian-cli, ran `git stash -u` to set its changes aside, then
`git stash pop`. The tree was clean, so the first command created no stash
entry, and the `pop` applied the owner's own `stash@{0}`, two weeks old and
from an unrelated branch, onto the lane's branch with 24 conflicts. The coder
reset the worktree. Nothing was lost: the tech lead checked that the worktree
was clean and that the owner's three stash entries were all still listed.

**Why:** a worktree has its own index and checkout, but the stash list belongs
to the repository and is shared by the main checkout and every worktree. `git
stash` reports "No local changes to save" and exits 0 on a clean tree, so a
script or agent that pairs push and pop blindly pops someone else's entry.

**The lesson:** never use `git stash` in a shared repository, worktree or not.
Set work aside with a temporary commit on your own branch instead. Every lane
brief in that work item forbade it afterwards. Claude Code now gives the same
warning when a session runs in a worktree; harnesses that don't, and agents
outside a worktree, still depend on the brief.

**Status:** guarded by brief text only; no hook or tool refuses `git stash`.

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
as `520de295`).
