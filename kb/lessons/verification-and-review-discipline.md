# Verification and Review Discipline

A check is evidence only when it exercises the changed behavior, observes a
non-vacuous oracle, and reports failure from the environment where the product
actually runs. Review convergence is a separate question: recurring findings
of one class mean the seam or invariant is wrong, not that another local patch
is needed.

## Verification Validity

### Prove the check consumed the capability

A token, marker, or callback proves that *some* check ran. It does not prove the
call under test used it. Seed or replace the capability immediately before the
subject call, make that call the sole possible consumer, then assert both the
behavior and consumption. Otherwise earlier setup can spend the evidence and
leave the regression path untested.

### Make the oracle non-vacuous

An empty traversal, zero matched files, ignored exit status, or assertion on an
unrelated layer can all report green while examining nothing. Every verifier
must establish its preconditions: expected root, nonzero candidate count where
appropriate, the intended command, and a propagated failure status.

### Test below the gate that originally blocked the defect

A regression test is not protective when an earlier validation gate rejects the
fixture before the changed logic runs. Use the narrowest legitimate seam that
reaches the defect class, or add an integration case whose input passes all
preceding gates. Confirm the test fails before the fix when making a regression
claim.

### A diff is not runtime evidence

Static inspection proves source shape. It does not prove import order,
registration side effects, filesystem behavior, process races, or actual CLI
output. Pair structural review with a runtime probe at the real seam whenever
those behaviors matter.

### Guard the process before importing launch tests

When a test must not call Mars, a native harness, a model, or the network,
mocking the expected bundle request is insufficient. Launch policy can query
the catalog before the mocked bundle seam. In the native-session R1 review,
unguarded p6941 runs may have invoked installed Mars catalog commands; the
absence of an intended native/model call was not proof of isolation. A p6942
probe also briefly launched an installed OpenCode TUI under a temporary HOME
after a wrong binding patch. It was stopped without an observed model
response, but its external effects are unknown. These are disclosed safety
misses, not qualifying test evidence.

For future fake-only launch verification, install hard process **and** network
denial before pytest or application imports; isolate HOME, XDG, config and
Meridian state; fake catalog/alias lookup **and** bundle resolution; stop the
consumer before native startup; disable plugin autoload and parallel workers;
and count attempted external effects. Synthetic Pi assets may be needed to
reach the intended seam without building or launching Pi. A guard proves the
bounded test interpreter attempted no external effect; it does not qualify
installed runtime behavior. The accepted R1 recheck used this boundary and
reported zero attempts across 146 focused checks (`work:native-harness-session-identity`,
`review/b3b-r1-final.md`, `spawn:p6970`).

### Verify the search scope contains the fact

A negative search result is evidence only when the search target can contain
the fact being searched for. Grepping root config files for declarations that
live inside compiled package internals cannot find them regardless of search
quality. This is not "should have searched harder" -- it is "searched a
location that could not hold the answer."

Before trusting a negative result, verify that the search scope covers the
domain structure where the fact would live. The cost of a false negative in
blast-radius assessment is an undetected breaking change; the cost of
checking the scope is one question.

The [release-sequencing](release-sequencing.md) lesson has a concrete instance:
a blast-radius assessment grepped sibling `mars.toml` files for `[[hooks]]`
and concluded no consumers existed. Hooks live in `hooks/<name>/hook.toml`
inside the resolved package, a location the search never reached.

### Mirror CI prerequisites in the local gate

A test that exercises a build output (the Pi extension bundle run in Node) needs
that output built first. CI built it before every test job; `scripts/preflight.sh
full`, run by the pre-push hook, did not, so a push failed on a checkout that had
never built the bundle. The fix put the locked install and build into the full
preflight ahead of pytest and packaging, and kept those tests strict. A skip on
"bundle not built" would have let a green local gate claim coverage it never ran.
When a CI job gains a prerequisite step, add it to the local full gate in the same
change.

Mirror CI's environment too, not only its steps. CI runners set `CI=true`, which
switches tools to non-interactive defaults; a Git hook has no TTY and no `CI`. The
same locked `pnpm install` that passed in CI aborted inside pre-push with
`ERR_PNPM_ABORTED_REMOVE_MODULES_DIR_NO_TTY` when a stale local `node_modules`
needed recreating. Git reported only "failed to push some refs", which looks like a
remote or auth failure. Make each prompting step non-interactive with the
narrowest switch available. For pnpm that is `--config.confirmModulesPurge=false`:
it still fails on lockfile drift under `--frozen-lockfile`. Don't export `CI=true`
across the gate, because it changes other tools' defaults and leaks into pytest.
(Provenance: `work:noninteractive-prepush-pnpm`: p7317 and p7319 diagnosed it;
p7328 verified the fix with pnpm 10.34.3 through a real `git push --dry-run`.)

The same prerequisite applies to every fresh worktree and to every bundle
change. `src/meridian/pi_runtime/dist/` is gitignored, so a new lane worktree has no
Pi bundles. Its first `pytest-llm` run fails on missing Pi extension artifacts and
looks like a regression. A bundle source change without a rebuild leaves the
Python tests exercising the old bundle. In the idle-cache-notify work this hit
several lane worktrees (F2b and F3 among them) until the briefs began with `pnpm install --frozen-lockfile && pnpm run build:extensions`.
Put the rebuild in the brief for any lane that runs the full suite, and repeat it
after editing anything under `pi_runtime/extensions/`.
(Provenance: `work:idle-cache-notify`, `chat:c7280`; `evidence/corefix-gates.txt`.)

### Run every adapter suite in a gate

A test suite that no gate runs protects nothing. The idle adapters for Claude and
Pi are TypeScript that runs inside the harness, with their own Vitest and
`claude plugin test` suites. CI built the Pi bundles but ran neither suite. When
the Python core began requiring `--interactive` on every idle call, the Claude mod
still sent it only on `config`. Every Python test passed. The TypeScript tests,
the only ones covering the hook-to-command mapping, never ran and had no
assertion on the flags anyway. G2 found it by hand-probing the CLI.

The fix added `pnpm test` to CI's fast gate after the bundle build. It also made
`claude plugin validate` and `claude plugin test` a required local gate in
`tests/AGENTS.md` for `claude_runtime/` changes, because CI runners have no
`claude` binary. When a contract spans a language boundary, the consumer's
suite must run wherever the producer's does, and it must assert the contract
itself: here, that every recorded call carries the flag.
(Provenance: `work:idle-cache-notify`, `reviews/G2-review.md` C1 and X1; spawn
`p7466`.)

### `pytest-llm` is not the full suite

`uv run pytest-llm` with no arguments runs pytest's default `testpaths` and
stops at the first failure (`--maxfail=1`). Those paths leave out most of
`tests/integration`: `cli`, `ops`, `launch` and others. The wrapper says so
on stderr ("default paths are not the full suite"), and that line is easy to
miss in a spawn's output.

On the notification branch (`work:notify-hook-events`), one review round
turned a CLI-only note into a report warning. Idle logs every report warning
on every fire, so each idle call wrote it to stderr. The idle CLI wire-contract
test (`tests/integration/cli/test_idle_cli.py`) asserts empty stderr and
failed. Coders and reviewers ran `pytest-llm`, saw green, and the regression
survived two review rounds. It surfaced only when the coordinating agent ran
`uv run pytest tests/` after a merge.

Use `pytest-llm` for a quick loop, and run `uv run pytest tests/` before
calling a branch green, after every merge between stacked branches, and in
any brief that says "full suite". A "full suite passed" claim should name the
command it ran.

### Record exit status and working directory

A copied success-looking line is not a validation record. Capture the command,
exit status, working directory, and the revision or artifact under test. A
wrong-CWD validator can inspect an empty or different tree and still exit zero;
a shell pipeline can hide an upstream failure unless pipe status is preserved.
When you detach a command from the terminal to reproduce a Git hook, use
`setsid --wait`. Plain `setsid` forks and exits 0 at once when it is already a
process-group leader, so its status says nothing about the command it launched.
(Provenance: `work:noninteractive-prepush-pnpm`, reviewer p7327; reproduced with
util-linux 2.39.)

### Check a claim about main against main's code

A finding that says "this is a regression" asserts something about main.
Check it on main before acting on it. On the idle push branch
(`work:notify-hook-events`), a correctness reviewer reported that a
background Claude subagent stuck on a permission prompt now held the push
while the human was needed, and that main never held the push for agents.
The coordinating agent had it live-probed against the branch and built a fix: a permission
prompt releases the held push. The fix drew new review findings, and only
then did anyone read main's mod. Its `armTask` skipped the whole arm while
any subagent ran, so main sent no push either. The behavior was a gap that
already existed, not a regression. The fix was rebuilt once more, drew a
third round of findings, and was withdrawn; the gap went to its own issue
(#684).

The probe was real, and it proved the behavior on the branch, but it could
not show that the behavior was new. A reviewer's premise about main is a
claim about code they may not have read. Before you build on "regression",
run or read the same path at `origin/main`.

### A per-item bound is not an aggregate or concurrency bound

Capping each item (a 64 KiB report) says nothing about how many are opened at once
or how large their concatenation is: 512 capped reports exhausted a 256-descriptor
limit and delivered nothing. Bound the batch (members, total bytes), the concurrency,
and the per-opportunity work, and test with a resource limit lowered below the
backlog. When clipping to fit, truncate only optional detail; never clip identity,
outcome or the way to fetch the full result, or the packet claims a state the reader
cannot see. One packet per inference bounds instantaneous flooding, not cumulative
model cost: reserve every item's mandatory outcome before allocating optional text,
or a greedy first item turns a backlog into one turn per result.
(Provenance: `work:pi-result-delivery`.)

### Matching hashes can all describe a stale bundle

Source, tests, and the built, wheel-embedded and installed artifacts can agree
with each other and still all be old: a restored generated `dist/` after a passing
test run produced exactly that for a Pi extension. Build fresh from the target
commit into a directory named by source SHA, record hashes, and add a positive
behavior check that only the new code can satisfy (for example, a marker the
old bundle lacks). Keep invalidated evidence, labeled invalid, rather than
overwriting it. (Provenance: `work:pi-result-delivery`.)

Packaging does not compile. `uv build` packages whatever JavaScript already sits in
the generated `dist/`, so wheel/sdist/dist hash parity, even against an "expected"
hash read from that same `dist/`, shows the packages agree with each other, not that
they came from the stamped commit. In #561 the audit compiled all four extensions
from clean Git archives of each stated revision and compared: one revision matched
entirely, a later freeze had a stale Bash bundle, and the next freeze had stale Bash
and watcher bundles. A freeze must therefore be one owned sequence: clean source →
explicit all-bundle compile → package → source-built/wheel/sdist parity → clean
source unchanged → a new uniquely named manifest that refuses to overwrite an earlier
one. The stale attempts stay on disk, labeled invalid. Source-only gates that built
current bundles remain valid as source evidence, and an observation made against
specific frozen bytes remains an observation of those hashes, never qualification of
the whole source revision or of later bytes. (Provenance: `work:runtime-efficiency`.)

### A filler in an ordering test can hide the schedule

A test that held a filler queue slot ahead of the failing write proved failure-
before-stage and missed a real race: with a ping-claim write failing, the
supervising `catch` ran one promise hop after the queue advanced, so a terminal
slot queued behind it staged clean success first. Patching that caller's handler
fixed one site but left the rule for the next caller; the durable fix moved
enforcement into the publication boundary, which runs the fatal policy
synchronously in the failing slot. Write ordering tests with no filler slot and no
sleeps, gate the real filesystem operation, assert the first attempted payload,
and include a case where an extra async wrapper surrounds the call.
(Provenance: `work:runtime-efficiency`, #561.)

### Do not expose a completion promise to unrelated milestones

Returning a repair promise from a synchronous failure claim made output draining
inherit a metadata wait, so a foreground abort stayed pending behind an unrelated
held records write. Fix ownership, not conditions: the claim returns nothing, and
callers that need publication name the promise they observe. When a repair
touches a shared queue, test with a real shell and a held write, and check that
test's API assumptions (an intercepted `appendFile` that the code never calls is
invalid evidence). The same barrier survived at launch: both foreground APIs awaited creation
publication before attaching cancellation, and an approving static review plus a
green gate missed it until a public-factory probe held the first records write and
aborted. Reviews are evidence, not authority; a lifecycle claim ("abort never waits
on metadata") needs a runtime control at every entry point, including the first
write. Likewise a test that aborts right after an async call does not show the work
was already enqueued; observe the enqueue itself. Fixture teardown must also respect authority rules: release
fixtures cooperatively through their own handles, never by signalling a saved
numeric PID or process group.

### Force the branch you claim to test, and assert what is left owed

A near-budget notice test with a short optional diagnostic still fit the whole frame
in the space reserved for the truncation notice, so nothing was truncated. Its red
run proved only that a newly added marker was missing, and some controls reported
PASS while the failure text was plainly visible. Size the optional body above the
reservation so the truncation branch is forced, inspect what actually survives
(identity, outcome, error marker, diagnostic prefix), and for the non-admitted
branch assert the item is still tracked, terminal and unconsumed. Absence of a new
string and a PASS label are not evidence of the loss under test.

### Audit inherited probe cleanup itself

A legacy native-probe driver read PIDs from records and, even after the processes
had disappeared, unconditionally sent SIGKILL to the saved PID and process group,
with errors swallowed, while earlier reports said no saved-ID cleanup remained. A
fix in a different ping driver had not made this path safe. Do not reason about what
the swallowed signals did; do not reuse the driver or count its runs as
safety-qualified. Keep the original script and observations, correct the historical
claim, replace the cleanup with captured live child handles and cooperative release
flags (persisted PIDs are diagnostics only), and rerun. Read every driver on a
verification path, not just the one most recently fixed.

### A generalized diagnostic path needs a "loads silently" oracle

The config cleanup refactors generalized the unknown-key walker to every dynamic
section (`02124b70c`). Every test of that path asserted that a bad key warns. None
loaded a realistic valid config and asserted that it produced no warnings. v0.10.0
shipped with a false `Ignoring unknown Meridian config key 'context.<name>'` on
every load in any project with `[context.*]` tables, including this repo (#580,
[decision](../decisions/codebase-cleanup.md#d-unknown-keys-from-pydantic-pydantic-does-the-traversal-for-unknown-key-warnings)).
Negative tests can't see a false positive. When a change widens a warning,
deprecation or validation path, pair the "bad input warns" cases with a valid
config that covers every section shape (list-rooted and dict-rooted dynamic
sections included) and assert that it loads with zero warnings. PR #581's test
loads one valid `[context.*]` table silently next to a bogus one. A whole-config
silent-load test does not exist yet.

The audit that checked the rest of the refactor for similar bugs used two
methods, and both found things the suite had missed:

1. **Load every real config on the machine, in process, from scratch copies.**
   Find every `meridian.toml` (19 on the dev box). Copy each one into a scratch
   directory, and use a copy of the user config as `HOME`. Call
   `load_config_with_sources` in process and capture stdlib `WARNING` records. Do
   this on `main` and on the fix. **Never run the CLI inside another project to do
   this:** since v0.10, a command that writes state mints `[project] id` into an
   unminted `meridian.toml`
   ([project identity](../architecture/state-system/project-identity.md)), so a
   "read-only" sweep can edit other repos. The sweep showed 8 configs hitting #580
   on `main` and 18 of 19 clean on the fix. The last one was a real user error: a
   commented-out `[[hooks]]` header that left hook keys under `[state]`.
2. **Run a differential against the build before the refactor.** Run
   `uv run --no-project --isolated --with meridian-cli==0.9.8 python -I probe.py`
   and the same probe on the new build, over one fixture battery: every TOML block
   in `docs/`, per-section key and type cases, a typo at each nesting level, a
   user/project/local stack, env overrides, and the `config init` scaffold. Diff the
   values, sources, warnings and errors. Two things to know:
   - 0.9.8's loader is `meridian.lib.config.settings.load_config`, because
     `meridian.lib.config.loader` didn't exist before cleanup phase 2. Check the
     import path for the version you're comparing against.
   - `--isolated` matters for the same reason as in
     [upgrade probes](dogfooding-pr-builds.md#probing-the-upgrade-path-needs-a-genuinely-old-build-and-old-state).

   Normalize the expected noise before counting differences: logger names, the
   file path that errors now include, and intended default changes. Raw, 115 of the
   118 fixtures differed; after normalizing, one substantive difference remained,
   and it was an intended fix. A bisect over `git archive` snapshots then tied each
   real difference to its commit. The results:
   - `[notify]` strings are no longer trimmed or checked for emptiness;
   - `[env]` keys are no longer stripped;
   - one bad hook row now breaks `config show`;
   - pre-existing invalid `autocompact` examples in the scaffold and the docs.

   These are tracked in #582 (`spawn:p7912`, `spawn:p7913`).

## Measurement Discipline

Measurements need a recoverable provenance note: command or method, dataset,
environment/date, and source artifact or commit. Report scope with the number.
A historical benchmark is not a current guarantee, and repeated numbers without
a canonical evidence pointer become lore.

For source-size changes, state whether generated files, fixtures, and moves are
included. For performance, distinguish cold/warm runs and dataset size. Other
pages should link to the canonical measurement rather than copying it.

Before widening a timeout or adding retries, measure the uncontended cost and
find the holder; see [contention diagnosis](contention-diagnosis.md).

## From Repeated Finding to Redesign

When the same class recurs in one function or pipeline, stop repairing branches.
Write the total-intent invariant, identify where partial representations become
possible, and move enforcement to a boundary that makes omission impossible.
The [Convergence Gate](review-convergence-gate.md) owns this process, including
frozen-worktree review, severity trajectories, and old-state runtime probes.

## Historical Evidence

The rules above came from two multi-round campaigns involving Mars ownership,
lock-schema recovery, symlink-following hashes, wrong-CWD checks, and CI trigger
behavior. The chronology is retained in [Verification Campaign History](verification-campaign-history.md),
but it is evidence for these rules rather than the reading order of this page.

## Related

- [Review Convergence Gate](review-convergence-gate.md)
- [Source Simplification](source-simplification.md)
- [Residue Cleanup Discipline](residue-cleanup-discipline.md)
