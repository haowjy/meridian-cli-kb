# Release Process

The release steps and compatibility policy are in meridian-cli `docs/releasing.md` and mars-agents' release guidance.

## One changelog writer

The release job writes `CHANGELOG.md` from PR bodies. Branches editing a shared Unreleased section conflict and can accidentally place notes under an already released version after a rebase. A malformed merged PR body stops the release before it writes; fix the body and rerun.

## Release guard

A queued release skips if an earlier release already contains its triggering merge and covers its requested release kind. Merges can land while a release is running, so serialization alone does not prevent duplicate releases. A stable release covers an RC request; the reverse does not.

## Unlabeled merges

An absent or unrecognized release label defaults to an RC so a missed label does not silently publish stable.

## Advisory checks

Checks are advisory and the release job validates collected notes before writing. A merge queue was rejected in favor of the release guard.

## Breaking changes

Version bump size follows compatibility, not the amount of work: nonbreaking features and fixes are patch releases.

## Integration branch

Collect minor and major changes on one release branch. Separate labeled merges would force users through multiple breaking upgrades; part PRs do not themselves release or contribute notes to the main-branch renderer.

## Release parity

Compare a release branch with main as a whole: paired benchmarks on copies of a real and a scaled runtime, workflow output comparisons, and a review of every behavioral change. Part reviews against the release branch do not reveal regressions against main. Report intended changes and resolve regressions before merging.

## PR body

The PR body should let a reviewer understand the entire change, including examples, impact and comparable benchmarks. The repository template owns the current structure.

## Stable notes after RCs

The notes walk starts at the latest release commit, including RCs. Consequently a stable release's notes cover changes since the last RC rather than repeating everything in preceding RCs. Whether stable releases should instead collect from the last stable release remains open.
