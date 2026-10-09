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
`p7466`; see [Idle Notifications — Review gates](../decisions/idle-notifications.md#review-gates),
G2.)

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

## Measurement Discipline

Measurements need a recoverable provenance note: command or method, dataset,
environment/date, and source artifact or commit. Report scope with the number.
A historical benchmark is not a current guarantee, and repeated numbers without
a canonical evidence pointer become lore.

For source-size changes, state whether generated files, fixtures, and moves are
included. For performance, distinguish cold/warm runs and dataset size. Other
pages should link to the canonical measurement rather than copying it.

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
