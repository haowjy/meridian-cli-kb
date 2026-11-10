# Verification and Review Discipline

A check is evidence only when it reaches the intended behavior and its oracle can fail. Establish the correct root, artifact and candidate count; confirm setup or an earlier gate did not consume the evidence or reject the fixture before the behavior under test. A negative search only proves absence when the search scope could contain the fact.

Run a representative real command at the integration boundary. Fakes can miss host lifecycle, timing and cross-language consumers. Compare a failure with the same scenario on the base before calling it a regression, and run the actual old build when a design depends on what that build did.

For fake-only launch verification, deny process and network access before application imports, isolate homes and stores, and fake both catalog and bundle calls. Mocking only the expected bundle request can miss an earlier catalog or native invocation. Audit cleanup too: use live handles or cooperative release rather than signalling saved numeric PIDs.

Mirror CI prerequisites locally. A built bundle can be stale even when source, package and installed hashes agree: rebuild from the target revision, then verify both artifact identity and a behavior unique to the new code. Run the suites for both sides of a cross-language contract. Project test commands and their scope are in `tests/AGENTS.md`.

A per-item bound is not an aggregate or concurrency bound. Exercise large backlogs with constrained resources, preserve mandatory identity and outcomes when truncating, and verify which items remain owed. Ordering tests need to reach the actual failing write without an artificial queue item that changes the schedule. A warning path needs both invalid-input cases and a realistic valid configuration that loads silently.

### Record exit status and working directory

Record the command, cwd, exit status and revision or artifact. Pipelines can mask an upstream failure, and a validator can succeed in the wrong tree or on no files. `setsid --wait` is needed when the detached command's exit status matters.

## Measurement Discipline

Record method, dataset, environment, date and artifact. Compare before and after on the same corpus under comparable load, distinguish cold and warm runs, and report correctness with timing. See [contention diagnosis](contention-diagnosis.md).

## Verification Hazards

An ownership check based only on content can follow a symlink to matching external bytes and wrongly claim the link as managed output; verify filesystem structure as well as content. A capability token proves an operation was checked only if the consumer binds its identity to the checked target. A regression test can pass at an earlier gate without reaching the failure it claims to exercise; verify the intended path was reached.

## From Repeated Finding to Redesign

Repeated findings of the same class signal a boundary to reconsider ([review convergence](review-convergence-gate.md)). When repeated fixes expose different versions of the same fact, inspect its owners and consolidate them. Then review the whole system: a projection joining facts from two stores can also diverge even when each store has one owner ([state design](state-design-lessons.md#a-link-that-spans-two-stores)).
