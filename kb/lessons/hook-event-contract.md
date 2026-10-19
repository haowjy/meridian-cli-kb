# Lessons: The Hook Event Contract (#687)

Two lessons from replacing Meridian's hook event names (PR #688, unmerged,
based on v0.10.28). The decisions themselves are in
[The Hook Event Contract](../decisions/hook-event-contract.md).

---

## Accepted Names Need a Producer

**What happened:** through v0.10.28, `spawn.start` and `work.start` were valid
`HookEventName` values that no code ever fired. A row on either one passed
`meridian hooks check`, appeared enabled in `hooks list`, and never ran. The
`git-autosync` builtin listed `spawn.start` among its default events, so the
dead name reached every unevented autosync row, and nobody noticed because
nothing failed.

The same review found four more surfaces in the same class: accepted at load,
inert at runtime.

- `failure_policy` was accepted on every row. Only gate events honored it,
  and there were no gate events.
- `when.status = "skipped"` was accepted. No spawn ever ends `skipped`.
- `when.status = "timeout"` was accepted, but the stored `timed_out` status
  mapped to nothing, so the filter never matched.
- `spawn.running` was produced from only one of the two paths into `running`;
  a spawn reserved already running never fired it.

**Why validation didn't catch it:** a closed `Literal` checks that a name is
spelled right. It says nothing about whether any producer emits that name, and
a hook that never fires produces no error to notice.

**The guard:** meridian-cli
`tests/integration/hooks/test_event_producers.py` drives the real producers
(the spawn lifecycle service including an initially-running reservation; `work
start`, `switch` and `done`, with and without a session; primary work
attachment; the idle service; `meridian notify`'s emitter) into a capturing
dispatcher. It asserts that the set of events fired equals both the `EVENTS`
table and `HookEventName`. A name added to the literal without a producer, or
a producer firing an unlisted name, fails the test. The same test asserts the
non-events: child work inheritance, `work rename` and `work clear` must not
fire `work.activated`.

**The rule:** a config value that loads successfully must change runtime
behavior. When adding a value to a closed vocabulary, add a test that reaches
it from the production seam, not from a hand-built context. Delete an accepted
value whose behavior is gone, rather than leaving it parseable and ignored.

---

## Repeated Fixes Mean Parallel Owners

**What happened:** making `work.activated` correct took three review rounds,
each with new findings:

1. Child inheritance and `work rename` fired activation, because the generic
   attachment writer published the event.
2. A fresh primary's `--work` never wrote its attachment. One "session exists"
   check required a harness session ID, which doesn't exist before the harness
   starts; another check required only the chat record.
3. `work.activated` for an archived item named a `work_dir` that didn't exist,
   because two work-event publishers derived the path differently. Separately,
   the per-event facts lived in an event-class map, a timeout table and
   name-prefix rules, and the `when` rule was written twice; one copy dropped
   lower-layer overrides when a row narrowed its defaults.

Each round patched the instance it found. The third round's review
(`spawn:p8648`) named the shared cause: two owners for one fact. The fix
consolidated instead of patching: one `session_exists` predicate, one
`publish_work_event`, one `EVENTS` table, one `_unsupported_when_field`. An
independent whole-system review (`spawn:p8650`) then found nothing blocking;
its findings were module placement and tidy-ups, filed as #692.

**The rule:** when a feature reaches a third round of fixes, stop patching.
List each fact the findings touched and count its owners. Wherever there are
two, delete one, then review the whole system rather than another targeted
diff. The general process is in
[From Repeated Finding to Redesign](verification-and-review-discipline.md#from-repeated-finding-to-redesign);
the registry pattern is in
[Single-Source Registry Over Parallel Maps](structural-refactor-methodology.md#single-source-registry-over-parallel-maps).

**Provenance:** `work:hook-event-names` `decisions.md` (D4–D6); reviews
`spawn:p8641`, `spawn:p8646`, `spawn:p8648`, `spawn:p8650`.
