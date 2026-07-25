# Verification and Review Discipline

Five patterns from the mars-agents hook-fragment review (11 rounds, PR #138,
2026-07). Each appeared multiple times in independent instances, grounded in
specific evidence below.

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

## Related

- [residue-cleanup-discipline.md](residue-cleanup-discipline.md) -- the
  ownership-violation instances that drove rounds 4-8
- [../decisions/package-management.md](../decisions/package-management.md#d91-native-hook-fragments-replace-command-synthesis-2026-07) -- D91 decision, including the retention seam
- [../architecture/mars-compiler.md](../architecture/mars-compiler.md) -- compiler module map including `surface_ownership/retention.rs`
