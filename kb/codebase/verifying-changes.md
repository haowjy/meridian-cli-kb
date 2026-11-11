# Verifying changes

How to get evidence that a Meridian change works, beyond the test suite. Test-writing rules are in [testing](testing.md) and the source checkout's `tests/AGENTS.md`.

## A check is evidence only if it can fail

A check counts only when it reaches the intended behavior and its oracle can fail. Establish the right root, artifact and candidate count, and confirm that setup or an earlier gate didn't consume the evidence or reject the fixture first: a regression test can pass at an earlier gate without reaching the failure it claims to exercise. A negative search proves absence only when its scope could contain the fact. Record the command, cwd, exit status and revision for every check: a pipeline can mask an upstream failure, and a validator can succeed in the wrong tree or on no files. Use `setsid --wait` when a detached command's exit status matters.

Run a representative real command at the integration boundary; fakes miss host lifecycle, timing and cross-language consumers. Compare a failure with the same scenario on the base before calling it a regression. Run the suites on both sides of a cross-language contract (Python core, the TypeScript Pi extension and Claude mod). A per-item bound is not an aggregate or concurrency bound: exercise large backlogs with constrained resources and check which items remain owed.

For fake-only launch checks, deny process and network access before application imports, isolate homes and stores, and fake both catalog and bundle calls: mocking only the expected bundle request misses an earlier catalog or native invocation. Mirror CI prerequisites locally. A built bundle can be stale even when source, package and installed hashes agree, so rebuild from the target revision and check a behavior only the new code has.

## Run unreleased builds away from real state

A build under test can write state an installed older build can't read, even during a nominally read-only operation. Use copied state and an isolated environment from the first command. Inherited project, chat, harness and depth variables can redirect a scratch run to the real project, and a scratch home alone doesn't isolate them or a shared harness binary: start state-mutating probes with a clean environment and scratch home, and disable self-updates for shared harness binaries. Keep scratch homes, copied runtimes and databases under `/tmp`, outside autosynced work directories, which would stage and push them.

- **Isolate `uv tool` installs fully.** Both the tool and executable directories need isolating, or the install can relink the user's `meridian`. A login shell can rebuild `PATH`, so call the build by absolute path and check its behavior.
- **Don't write over `.venv/bin/*`.** uv may hard-link executables to its cache, so writing through one changes other environments. Use a scratch venv with copy link mode, and check inode, link count or content hash rather than the version string.
- **In-harness code runs whichever `meridian` is first on `PATH`.** Plugin hooks, Pi extensions, Mars hooks and the model's shell call `meridian` by name; an older build first on `PATH` once failed a newer session's `spawn show --no-observe` with "Unknown option". Launches now pin the child `PATH` to a shim for the launching build ([sandbox projection](../permissions/sandbox-projection.md#spawning-from-inside-a-sandbox)). Reproduce with the build's binary, not `uv run`, which puts the worktree's `.venv/bin` first and hides the skew.
- **Probe a harness in a live session.** A probe read an old `~/.claude/shell-snapshots` file and concluded interactive Claude drops a prepended `PATH`; a live check showed it keeps it. A scratch harness config hides user hooks, so when a mod observes input another hook can rewrite, probe with representative hooks too.
- **Upgrade paths need a really old build and old state.** Create the state with the older build, identify it by behavior rather than version string (an unreleased build can report the release's version, and cached tool environments can select it), and test overlap while an old process still runs.
- **Worktrees share configuration and the stash.** Use a separate clone for disposable Git configuration, or pass identity per command, and set work aside with a temporary commit on an owned branch rather than the shared stash.

## Reviews

- **Review the composition.** Separately reviewed lanes leave gaps at their boundaries; a green merge and passing lane reviews don't establish combined behavior. Parallel lanes can also collide on test basenames, which pass separately and fail collection together under pytest's default import mode.
- **Exercise prior-release artifacts** through the new command, including unreadable input, partial failure and retry after repair. Omission can't mean deletion when it can also mean the input was unreadable or unprocessed.
- **A recurring finding class points at a boundary.** When the same class recurs, or repeated fixes expose different copies of one fact, examine the owners of that fact and consolidate them instead of adding another exception.
- **State the threat model before an adversarial review.** An adversarial reviewer assumes the strongest attacker it can picture, and a lead who forwards its findings adopts that attacker. Without a stated model, three review rounds hardened the Codex shell hook against an agent attacking its own rule: it grew from 637 to 1,138 lines and still couldn't stop a REPL or `find -exec`; stating the real threat cut it to 219 lines ([tool permissions](../permissions/tool-permissions.md#codex-shell-rules-are-an-accident-guard)). A finding outside the model goes back unfixed unless it shows the model is wrong.
- **Keep the verdict out of the brief.** Ask reviewers for evidence, constraints and tradeoffs; a reviewer given the desired verdict can echo it against its own evidence.
- **A prompting workaround for a harness defect is a deferral, not a fix,** and can contradict the product's own guidance.
