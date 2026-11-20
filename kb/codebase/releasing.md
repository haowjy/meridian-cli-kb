# Releasing

The release steps and compatibility policy are in meridian-cli `docs/releasing.md` and mars-agents' release guidance; this page keeps the reasons.

## One changelog writer

The release job writes `CHANGELOG.md` from merged PR bodies. Branches editing a shared Unreleased section conflicted, and a rebase could put notes under a version that had already shipped. A malformed merged PR body stops the release before it writes; fix the body and rerun. Edit a PR body only from a fetched copy you checked isn't empty: `gh pr edit --body-file` on a file that a failed `gh pr view` left empty wipes the body, and with it the release notes.

## Guarding against duplicate releases

Merges can land while a release runs, so serializing releases alone doesn't prevent duplicates. A queued release skips when an earlier one already contains its triggering merge and covers its release kind; a stable release covers an RC request, not the reverse. Checks are advisory, and the release job validates collected notes before writing; this guard was chosen over a merge queue. A missing or unrecognized `release:*` label releases an RC, so a forgotten label never publishes stable.

## Breaking changes ship together

Collect minor and major changes on one `release/v<X>.<Y>.0` branch: separate labeled merges would push users through several breaking upgrades. Part PRs into that branch neither release nor contribute notes. Before a release branch merges, compare it with `main` as a whole, with paired benchmarks on copies of a real and a scaled runtime, workflow output comparisons and a review of every behavioral change; reviews of parts against the release branch can't reveal regressions against `main`.

In mars-agents the agent leading a release merges its release PR to `main` without asking, once the whole-change review is resolved and CI is green: every boundary-crossing part was approved before it started. In meridian-cli Jimmy says when a release branch is complete, because other minor changes may still be heading for it.

After rebasing a part branch, compare it with its pre-rebase head (`git range-diff`), not only the conflicts: a rebase once left a `#[cfg(unix)]` attribute on the wrong item in mars-agents, and only the Windows CI check caught it.

Release notes are collected from the latest release commit, RCs included, so a stable release's notes cover changes since the last RC rather than repeating every RC.

## Releasing Mars and Meridian together

Meridian pins an exact Mars version, and a producer and consumer can each pass their own gates while their combination fails. So a change crossing the two lands and releases in dependency order: release the Mars producer change, then update Meridian's pin and release the consumer. A dual-command compatibility path to hide a mismatched pin isn't added. Before calling a change a pin bump, confirm the consumer's migration is published and usable, check the version the consumer actually installs, and run the new binary against a representative downstream project. Search for consumers where their declarations live; a negative search of a summary or parent manifest can miss them. `git merge-tree --write-tree` prints a tree ID even on conflict, so check its exit status.
