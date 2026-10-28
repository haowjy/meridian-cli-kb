# Decisions: The Hook Event Contract

> **Status:** Shipped in v0.11.0 (PR #688, fixes #687; the permission notice
> event in PR #689). The pre-0.11 names load with a warning until the next
> minor release.

Decision records for which hook events exist, what each one means, and who
owns each fact about them. The current mechanism is in
[Hooks and Plugins](../concepts/hooks-and-plugins.md); the user reference is
meridian-cli `docs/hooks.md`. See [../decisions.md](../decisions.md) for the
chronological index.

Through v0.10.28 the hook schema accepted names no producer fired.
`spawn.start` and `work.start` passed `meridian hooks check`, showed as enabled
in `hooks list`, and never ran. The `git-autosync` builtin listed `spawn.start`
as a default event, so it never synced at spawn start, and nothing reported it.
The names also mixed spawn state transitions (`running`, `finalized`) with
moments a user would recognize. The decisions below replace that vocabulary and
add the guard that keeps a dead name from shipping again.

---

## D-hook-event-names — past-tense moments named for what just happened

**Decision (2026-10-18, owner-approved):** Hook events are named for the
moment that just happened, in the past tense. Events that carry a notification
share the `notify.` prefix.

The lead (the agent running this work) first proposed a narrow fix: make `work switch` fire `work.started`
too. The owner rejected folding creation and choice into one event ("then
shouldn't there be 2 events of work.start and work.switch or work.resume") and
widened the scope ("or maybe there is better wording for all the events...").
The lead then proposed the table below, the past-tense rule and the `notify.`
prefix; the owner approved it with "yeah, lets fix the plugin API since we are
already breaking it" (`chat:c8397`).

| Event | Fires when |
|---|---|
| `spawn.created` | A spawn record is first written, queued or already running |
| `spawn.started` | A spawn enters `running`, on every path into it |
| `spawn.finished` | A terminal spawn state is persisted |
| `work.created` | `work start` creates a new item |
| `work.activated` | Someone chose a work item; see [D-work-activation](#d-work-activation--activation-means-someone-chose-the-item) |
| `work.done` | A work item is archived as done |
| `notify.your_turn` | The idle service sends its "your turn" push |
| `notify.cache_expiring` | The idle service sends its prompt-cache warning |
| `notify.needs_permission` | Claude reports a permission prompt still waiting ([D-idle-permission-ping](idle-notifications.md#d-idle-permission-ping--forward-claudes-still-waiting-signal-without-rebuilding-prompt-state)) |
| `notify.manual` | `meridian notify` runs |

`notify.needs_permission` was built alongside the rename under the name
`session.needs_permission` and shipped in the same release under the new
scheme, so it has no deprecated name.

Each old name maps to one replacement (`NEXT_MINOR_HOOK_CONFIG_BREAKS` in
meridian-cli `lib/hooks/schema.py`); how old names load is
[D-hook-deprecation-window](#d-hook-deprecation-window--old-names-load-for-one-minor-release):

| Old | Replacement |
|---|---|
| `spawn.start`, `spawn.running` | `spawn.started` |
| `spawn.finalized` | `spawn.finished` |
| `work.start`, `work.started` | `work.activated` |
| `session.responded` | `notify.your_turn` |
| `session.cache_expiring` | `notify.cache_expiring` |

`work.started` maps to `work.activated`, not `work.created`, because it fired
on every `work start`, for existing items too.

**No permanent aliases.** An alias gives one moment two names, which is the
ambiguity the rename exists to remove. Old names are accepted only for the
deprecation window, with a warning on every load, and then rejected.

**Rejected:** `work.resume` for the choice event, since a newly created item
isn't resumed, and `work.switch`, since `work start` also activates without
switching. Also rejected: keeping `spawn.start` / `work.start` as permanent
aliases. The owner had asked "who would have event = spawn.start or work.start
...? also... why remove them?"; the answer was that nothing did except
Meridian's own git-autosync default, and the owner went on to approve the full
rename.

**Considered and not added:** `spawn.exited`, `work.renamed`, `work.deleted`,
`work.reopened` and `session.ended`. Nothing would consume them, so each would
be one more accepted name to keep working. Process exit and the three work
changes are already telemetry records, and `session.ended` is
`spawn.finished` with `when.kind = ["primary"]`. Adding an event later is a
small change (see [D-hook-fact-owners](#d-hook-fact-owners--one-owner-for-each-fact-about-an-event)).
Telemetry also keeps its per-outcome spawn records (`spawn.succeeded`,
`spawn.failed`, …), which are finer-grained than `spawn.finished`.

**`spawn.started` covers every path into `running`.** Before the rename, the
hook fired only from `mark_running`'s queued-to-running transition. A spawn
reserved already running (streaming serve) fired `spawn.created` and nothing
else. `SpawnLifecycleService.start` now also fires `spawn.started` when the
first persisted state is `running`.

**Event IDs.** A spawn event's ID is a UUID v5 of spawn ID and event name, so a
second `spawn.finished` delivery after an authoritative terminal rewrite has
the same ID and can be deduplicated. Work and notification event IDs are
random per firing, so two activations of the same item stay distinct.

**Telemetry follows where it is the same moment.** The lead had left
telemetry names alone as outside the hook contract; the owner overrode that
("I think telemetry is fine to change it", `chat:c8397`). Work telemetry
`work.started` became `work.created` plus a new `work.activated` (ids
`work_id`, `chat_id`). The core lifecycle names
`spawn.queued` and `spawn.running` became `spawn.created` and `spawn.started`.
Local `telemetry query` still keeps only process-exit and terminal records for
spawns, as on `main`; see
[Telemetry Event Catalog](../architecture/telemetry/event-catalog.md).

**Hook context schema 3.** The payload's `schema_version` 3 identifies this
contract. It adds `spawn.kind` (`MERIDIAN_SPAWN_KIND`), emits the `spawn` JSON
block whenever any spawn field is known, and carries the persisted terminal
error on `spawn.finished`. `HookContext.from_authority` became `from_roots`.
The branch first versioned the Plugin API as 3.0.0; that string was deleted
before release because nothing read it, and the Plugin API now carries no
version of its own
([D-version-only-where-read](compatibility-versioning.md#d-version-only-where-read--integer-format-versions-where-another-build-reads-delete-the-rest)).

---

## D-hook-deprecation-window — old names load for one minor release

**Decision (2026-10-19, owner; supersedes rejection at load):** The old event
names, `failure_policy`, and `when` conditions an event can never satisfy load
with one warning per row during v0.11, and become config errors in the next
minor release. A row on an old name registers on the replacement event.

This reversed the branch's first design, which failed config load on every old
name ("`'spawn.start' was removed; use 'spawn.started'`") on the grounds that
Meridian had no users to migrate. Before #688 merged, the owner dropped the
project's no-backwards-compatibility rule and applied the replacement policy
to this PR first
([D-compat-policy](compatibility-versioning.md#d-compat-policy--one-minor-deprecation-new-builds-read-old-data-old-builds-refuse-new-data)).

How the window behaves (meridian-cli `docs/upgrading.md` "Hook event names"
is the user reference):

- **Old names.** The row registers on the new event and warns once.
  `meridian hooks check` exits 0 when deprecations are its only findings, and
  `hooks list` marks the row `deprecated`. Rows on `spawn.start` and
  `work.start` never fired before; they now run, and their warning says so.
- **The row keeps its configured name in the payload.** `HookContext.event_name`,
  JSON `event_name` and `MERIDIAN_HOOK_EVENT` carry the old name for a row
  configured with it, so a script that branches on the event name keeps working
  until its config is migrated. This was the lead's call.
- **`failure_policy`** loads, warns and has no effect.
- **`when.status = ["skipped"]`** drops `skipped` with a warning. If no status
  remains, the row is skipped rather than widened to match every status.
- **Impossible conditions on a row that names its event** warn and skip the
  row, so it no longer shadows a lower-file row with the same name and event.

**Mixed-version machines.** v0.11 accepts both name sets; v0.10.x still
rejects the new names. A shared config keeps the old names until
every machine has upgraded.

**Ending the window:** `NEXT_MINOR_HOOK_CONFIG_BREAKS` is the complete
inventory of this compatibility behavior. Flipping its `mode` to `"error"`
turns every entry into a load error.

---

## D-work-activation — activation means someone chose the item

**Decision (2026-10-18, lead):** `work.activated` fires when a user or agent
picks a work item to work on. It does not fire when Meridian writes a
session's work attachment as maintenance. The owner asked for separate
creation and choice events; the boundary below is the lead's, settled after
review rounds.

| Counts as activation | Does not count |
|---|---|
| `work start` on a new or existing item | A child spawn inheriting its parent's work |
| `work switch`, including to an archived item | `work rename` repointing attachments |
| A primary launch attaching `--work`, or work carried over through `--from` | `work clear` |
| Any of the above from a shell with no Meridian session | |

`activate_work()` (meridian-cli `lib/ops/work_attachment.py`) is the only path
that publishes the event. It fires whether or not a session exists, and writes
the session attachment when one does. Inheritance, rename and clear write
through `session_store` directly.

**Why inheritance is excluded:** `git-autosync` defaults to `work.activated`.
If inherited attachment fired the event, every child spawn would pay a blocking
git sync. Work events carry no spawn kind, so a user could not filter that away
with `when.kind`.

**Why a primary launch counts:** the user named the item. It costs one sync per
session, before the harness starts, not one per spawn. A context sync at
session start is also what the planned context-autosync daemon (#691) wants.
This call is reversible if the startup delay proves annoying.

> [!FLAG] **Needs human review**
> The lead asked the owner whether primary launches should fire activation
> ("Say if you'd rather launches didn't fire it", `chat:c8397`) and got no
> answer. The behavior above is the lead's default, not an owner decision.

**Why sessionless start and switch count:** the old `work.started` fired with
no session, so keeping that preserves behavior for shell users and scripts.

**A session exists when its chat record exists.** `session_store.set_work_attachments`
skips a chat with no live record, and `attach_session` reports whether the chat was
attached. The first activation fix asked for a harness session ID,
which a fresh primary does not have before its harness starts, so
`--work` on a new primary silently never wrote the attachment.

---

## D-autosync-default-events — `spawn.finished`, `work.activated`, `work.done`

**Decision (2026-10-18, lead):** An unevented `git-autosync` row registers on
`spawn.finished`, `work.activated` and `work.done`. Moving the work default to
`work.activated` was a line in the rename table the owner approved.

The v0.10.28 defaults were `spawn.start` (dead), `spawn.finalized` and
`work.started`. Moving the dead default to `spawn.started` was rejected: it
would put a new blocking sync in front of every spawn start, for a hook the
owner wants replaced: "create a specific context autosync setting and a
running background daemon for git autosync, hooks are kinda... not the best"
(`chat:c8174`). That daemon is #691. Autosync therefore keeps
never syncing at spawn start, which is what it already did in practice.

The work default moved from `work.started` to `work.activated`. The cost is one
synchronous sync before a primary launch with `--work` or `--from` work;
`docs/hooks.md` states it.

---

## D-hook-failure-policy-removed — hook failures always continue

**Decision (2026-10-18, lead):** Every hook failure logs and continues to the
next hook. The `FailurePolicy` Plugin API type and the observe/post/gate event
classes are deleted. A config row that still sets `failure_policy` loads with a
warning and no effect until the next minor release
([D-hook-deprecation-window](#d-hook-deprecation-window--old-names-load-for-one-minor-release)). The owner
approved deleting the `gate` class and `when.status = "skipped"` ("add them")
and asked for a kind filter in the same turn; removing `failure_policy`
followed in review, once nothing honored it.

**Why:** only `gate` events honored `failure_policy = "fail"`, and no event was
a gate. The field was accepted, documented, and ignored. Leaving it would have
been one more accepted-but-inert surface, the class this work removes.

`when.status = "skipped"` is gone for the same reason: no spawn ends `skipped`.
The value came from `HookOutcome`, the result of a hook run, which had been
conflated with a spawn's outcome. `when.status` values are the Plugin API
`SpawnStatus` (`success`, `failure`, `cancelled`, `timeout`). The mapping from
stored `timed_out` to `timeout` was missing, so timeout filters never matched;
that is fixed.

**Added with it:** `when.kind` filters spawn events by the shared `SpawnKind`
(`child`, `primary`, `streaming`). The Plugin API owns that literal, and core
lifecycle and persisted spawn state import it, so there is one spawn-kind
vocabulary.

**Conditions are checked against what the event supplies.** `when.status`
exists only on `spawn.finished`, and work events supply neither `when.agent`
nor `when.kind`. A row that names its event and asks for a field the event
cannot supply can never match: during the deprecation window it warns and is
skipped, and in the next minor release it fails load. An unevented builtin row keeps its compatible default events enabled and
records the rest as disabled overrides. The disabled entries still replace
lower-layer rows with the same `(name, event)`, so narrowing a project row
cannot expose a user-layer action. The row fails only if no default can match.

---

## D-hook-fact-owners — one owner for each fact about an event

**Decision (2026-10-18, lead):** Each fact about hook events has one owner.

| Fact | Owner |
|---|---|
| Event names; which events carry a notification | `plugin_api/types.py` (`HookEventName`, `NOTIFICATION_EVENTS`) |
| Per-event timeout and available `when` fields | `EVENTS` table of `EventSpec` rows, `lib/hooks/types.py` |
| Whether a `when` field is unsupported | `_unsupported_when_field`, `lib/hooks/schema.py` (validation and default pruning) |
| Spawn events and their telemetry | `SpawnLifecycleService`, `lib/core/lifecycle.py` |
| Work events, their `work_dir`, and their telemetry | `publish_work_event`, `lib/ops/work_attachment.py` |
| Notification events | `lib/notify/emit.py` |
| Whether a session exists | the chat record: `session_store.set_work_attachments` skips a chat without one |

**Why:** activation needed three rounds of review fixes, and each round's
defect had the same cause: two owners deriving one fact. Two meanings of
"session exists" left a fresh primary's `--work` unattached. Two work-event
publishers worked out `work_dir` differently, so `work.activated` for an
archived item named a directory that did not exist. The per-event facts were
spread over an event-class map, a timeout table and name-prefix rules, and the
`when` rule was written twice. The fix was to delete the duplicate owners
rather than patch a fourth instance. An independent whole-system review
(`spawn:p8650`) confirmed the earlier findings shared that cause and that the
consolidation ends it. See
[Repeated Fixes Mean Parallel Owners](../lessons/hook-event-contract.md#repeated-fixes-mean-parallel-owners).

**Adding an event** now takes the `HookEventName` entry, an `EVENTS` row, a
`NOTIFICATION_EVENTS` entry if it carries a notification, and its producer. The
producer-coverage test fails if those disagree; see
[Accepted Names Need a Producer](../lessons/hook-event-contract.md#accepted-names-need-a-producer).

---

## Open and follow-up

- **Next minor release:** flip `NEXT_MINOR_HOOK_CONFIG_BREAKS` to `"error"`.
- **#692:** build hook dispatchers in one factory in `lib/hooks` (removing the
  `core` ↔ `hooks` import cycle and the duplicate in `notify`), build it once
  per command so a bad config warns once, and move work-event publishing below
  `ops` so `launch` stops importing upward.
- **#693:** decide `Hook.remote`, the `git-autosync` `remote` shorthand still
  commented as a legacy field.
- **#691:** the context-autosync daemon may replace the `git-autosync` hook;
  the default events above are an interim choice until then.

**Provenance:** `work:hook-event-names` `decisions.md` D1–D8; owner turns in
`chat:c8174` (2026-10-18) and `chat:c8397` (2026-10-18), mined by
`spawn:p8653` and `spawn:p8692`; reviews `spawn:p8641`, `spawn:p8646`,
`spawn:p8648`, `spawn:p8650`, `spawn:p8664`; probe `spawn:p8643`.
