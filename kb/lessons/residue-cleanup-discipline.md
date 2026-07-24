# Residue Cleanup Discipline

When a system changes what it emits, the removal path must still recognize
what prior versions actually wrote. This lesson was learned twice in two
weeks and has the same root cause both times.

## The Pattern

Mars writes managed entries into target config files (`settings.local.json`,
`hooks.json`, `opencode.json`). When mars changes the shape, key names, or
format of those entries, the removal sweep that runs before re-adding must
recognize the OLD shape -- not just the current one.

Every instance followed the same failure mode: the removal path was
updated to match the new output format, so it no longer found entries
written by the prior version. Those entries became orphans: mars-created
but mars-unrecognizable.

## Instance 1: SessionStop orphan (mars-agents v0.10.6)

Mars <= v0.10.5 wrote Claude hooks under the key `SessionStop` (a
nonexistent event name). The v0.10.6 fix corrected the event to
`SessionEnd` but the removal path also switched to looking for
`SessionEnd` keys. Prior `SessionStop` entries were orphaned in
`settings.local.json` until manually cleaned.

## Instance 2: Native hook migration (mars-agents PR #133)

Two instances in a single PR:

**Sweep ordering.** The stale-key sweep ran after the new native-key
bindings were written. It matched by hook name in the command path, so
it found and deleted the just-written native bindings. First sync after
migration emitted empty event arrays. Fix: sweep before write.

**Legacy Codex format.** The test for `codex_hooks.json` legacy cleanup
crafted test data in the modern nested-object format. Real pre-v0.10.6
`codex_hooks.json` entries were kebab-case strings in arrays. The
removal sweep's `let-else` pattern preserved non-object entries, so the
legacy strings survived. Fix: failing-first regression tests using the
real prior-version shape; sweep extended to handle string-format entries.

## The Discipline

1. **Verify against the real prior-version shape.** Read actual config files
   written by the prior release, or reproduce them from the prior code. Do
   not construct test data from the current code's output format -- that
   tests the new path, not the migration.

2. **Removal-only sweeps for orphaned formats.** When deleting a write path,
   keep the corresponding removal path for one release. The removal sweep
   is cheap; the orphan is permanent.

3. **Sweep before write.** Stale-entry removal must run before new entries
   are written, or name-matched sweeps can delete the new entries.

4. **Regression tests with real shapes.** Each removal-only sweep needs a
   test that constructs the exact prior-version artifact shape and verifies
   the sweep cleans it. The test should fail first (red-green discipline)
   to prove it exercises the bug.

## Related

- [mars-compiler-cleanup.md](mars-compiler-cleanup.md) -- earlier Mars cleanup lessons
- [../decisions/package-management.md](../decisions/package-management.md#d90-native-hook-passthrough-replaces-universal-event-vocabulary-mars-agents-pr-133-2026-07) -- D90 decision
