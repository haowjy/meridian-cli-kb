# Cross-Repo Release Sequencing

A breaking change in a producer repo and the consumer migration that answers
it are two separate releases with nothing pairing them. When the producer
ships first, the ecosystem is broken until the consumer catches up -- and the
consumer migration can exist, be correct, and still sit unpushed in a local
working tree.

## The Structural Gap

Meridian's ecosystem spans independent repos with independent release
processes: mars-agents (Rust binary), prompt packages (meridian-base,
meridian-dev-workflow), and meridian-cli (Python). A breaking change in
mars-agents requires a coordinated release of the prompt packages that
depend on it and a pin bump in meridian-cli. D90 documents the intended
sequence: mars release (schema flag-day), then meridian-base push (migrated
hooks), then meridian-cli pin bump. No intermediate compatibility window.

Nothing enforces that sequence. Each repo has its own CI, its own merge
gate, and its own version tags. The human or agent performing the release
must remember the downstream steps and execute them in order. A single
forgotten step -- or a step that is performed locally but not pushed --
breaks the ecosystem silently.

## Instance: mars-agents 0.12.0 and meridian-base 0.10.0

mars-agents 0.12.0 shipped the D91 native hook-fragment schema, which
hard-rejects the old per-target event syntax. The meridian-base migration to
the new schema (commit `0819ee6`) was written, committed, and correct -- it
had been sitting in a local working tree since 24 Jul, unpushed. When
mars-agents 0.12.0 landed on PyPI, every published meridian-base (0.8.9 and
0.9.0) was rejected by the new mars binary:

```
error: source package `meridian-base` version `0.8.9` uses the removed
v0.11.0 hook schema; migrate to per-target native fragment files
```

Anyone upgrading mars was broken. `mars sync` against any project using
meridian-base failed with exit 2, and the recovery command (`mars upgrade`)
was bricked by the same error -- the halt-gate design (D92) working as
intended, but against an ecosystem that had not shipped the migration.

The blast-radius assessment that cleared the mars release said "no
hook-schema consumers -- pin change only." It was wrong, and the way it was
wrong is itself a lesson: the assessment grepped root `mars.toml` files in
sibling repos for `[[hooks]]`. Hooks are not declared there. They live in
`hooks/<name>/hook.toml` inside the resolved package. The search was
structurally incapable of finding the answer regardless of how carefully it
read the results. See [Verification and Review Discipline](verification-and-review-discipline.md)
for the reusable rule.

The runtime smoke test caught it. A spawn running `mars sync` against a
throwaway copy of meridian-dev-workflow hit exit 2 and stopped without
shipping -- the right call, and the reason a wrong assessment did not reach a
merged PR. The static gate on the meridian-cli side (ruff 0, pyright 0, 1395
passed) was fully green on a change that would have broken every project
using meridian-base. See [Verification Campaign History](verification-campaign-history.md)
for this and prior instances of static gates passing over behavioral breaks.

### Resolution

Three packages released in sequence:

| Package | Version | Change |
|---|---|---|
| meridian-base | 0.10.0 | Fragment migration (the unpushed commit) |
| meridian-dev-workflow | 0.13.0 | Constraint widened to `>=0.10.0, <0.11.0` |
| meridian-benchmark-agents | 0.2.0 | Both constraints widened |

Verified end to end: `mars sync` against a copy of meridian-dev-workflow
went from exit 2 (lock stuck at version 2) to exit 0 (lock promoted v2 to
v3 in one run).

## The Discipline

1. **Verify the consumer migration is published, not just committed, before
   or alongside the producer release.** A committed but unpushed migration
   is indistinguishable from a missing one to anyone outside the local
   machine. For flag-day changes with no compatibility window (D90's
   explicit choice), this means the consumer package version must be live
   on its registry before the producer tag is cut.

2. **Assess blast radius at the location where the data lives.** When
   checking whether a breaking change has consumers, search the artifacts
   that contain the declarations, not summaries, manifests, or parent
   directories. A negative search result is evidence only when the target
   can contain the fact being searched for.

3. **Run the downstream binary against a real project before declaring a
   dependency bump "pin change only."** Static gates verify local
   correctness. A cross-repo break lives in the gap between two locally
   correct repos. Only running the actual command against real project
   state catches it.

## Related

- [../decisions/package-management.md](../decisions/package-management.md#d90-native-hook-passthrough-replaces-universal-event-vocabulary-mars-agents-pr-133-2026-07) -- D90 release sequencing statement
- [../decisions/package-management.md](../decisions/package-management.md#d91-native-hook-fragments-replace-command-synthesis-2026-07) -- D91 fragment schema
- [verification-and-review-discipline.md](verification-and-review-discipline.md) -- search scope and runtime probe rules
- [verification-campaign-history.md](verification-campaign-history.md) -- prior instances of static gates missing behavioral breaks
- [residue-cleanup-discipline.md](residue-cleanup-discipline.md) -- the one-release sweep window that depends on the sequencing being correct
