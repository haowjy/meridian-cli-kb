# Native Session Binding

How a launch decides its native key, when that key is written, how a run's
observations are judged, and how a run's exit or last observed session is recorded. The rule and its rationale
are in the [native session identity decision](../decisions/native-session-identity.md).
Harness specifics: [Pi](pi-native-sessions.md), [Claude](claude-native-sessions.md).
How reads use the key: [native transcript reads](native-transcript-reads.md).

**State:** entry identity shipped for Pi, Claude, Codex and OpenCode in v0.7.0 (PR
#534, merge `3dae2434`); #520, #526 and #531 are closed as superseded. Claude
current-chat tracking (the NativeRun observation seam below) is implemented locally
(`679ab27f`, `cea151fa`) in unmerged PR #559: not released, full verification and a
frozen-source native probe pending. Cursor was not live-probed because the user does not use it; it
has no native identity binding in this implementation.

The design goal of the restructure: one value type, one pure binding rule, one
adapter template, one post-exit hook, and one runner pipeline.

```mermaid
flowchart LR
  subgraph pre["pre-exec: launch/context.py"]
    R["SpawnParams"] -->|"plan_native_identity"| I["LaunchIntent"]
    I -->|"finalize_native_identity"| N["NativeIdentity"]
  end
  subgraph run["each runner: process, streaming, serve"]
    N --> B["bind_entry: NativeRun"]
    B -->|"owned session IDs"| O["NativeRun.observe / note"]
    O --> J["child exited, teardown joined"]
    J --> C["conclude_native_run"]
  end
  C -->|"SessionAttempt.bind"| S[("sessions.jsonl")]
  C -->|"one update_spawn"| P[("spawn row run_boundary")]
  C -->|"record_identity_failure"| L[("runner-lifecycle.jsonl")]
```

## Types

In `lib/core/native_identity.py`:


| Name | Meaning |
|---|---|
| `NativeKey(harness, native_store, session_id)` | A complete binding. Tracked reads, continue and fork require one. |
| `NativeKeyFields` | The partial key a record may hold: a legacy record has no store; a Codex or OpenCode create has no ID until observed. `complete()` returns a `NativeKey` or `None`. `render()` is the one formatter for lifecycle payloads and messages. |
| `LaunchIntent(operation, source_session_id, preforked_session_id)` | What the caller asked for, before any store is known. |
| `NativeIdentity` | What the child is launched with: harness, operation, store (always set), assigned `session_id` or `None`, fork source ID, verified source file. The only identity input to projection, `bind_entry` and `observe_after_exit`. |
| `PostExit(entry_error, entry_observed, exit, trampoline_successor_id)` | What the adapter saw after exit. Pure; the runner pipeline decides. |
| `NativeIdentityError` | Base of every typed refusal. Runners catch only this. |
| `NativeSessionUnavailable(ref, unbound \| missing \| ambiguous_native_file)` | Nothing trustworthy to open. `missing` reports as `native_transcript_missing`. `for_ref()` re-targets the message at the user's ref. |
| `NativeEntryMismatch(expected, observed, reason)` | A readable identity contradicts the key. Carries keys, not strings. `reason ∈ {key, fork_reused_source, source_changed, fork_parent}`. |

`BindSource` is `assigned` (pre-exec), `observed` (an owned signal),
`legacy_import` (the [one-time import](legacy-native-import.md)),
`legacy_pi_recovery` (the content-proven [0.6.7 Pi pass](legacy-native-import.md#legacy-pi-recovery))
or `user_repair` ([`session repair --native`](legacy-native-import.md#session-repair)).

Two key types exist because records legitimately hold partial keys. With a complete
type, "tracked reads need the complete key" becomes a type check:
`record.native_key() is None` means `unbound`.

## Binding

In `lib/state/`:


**`native_binding.bind(prior, attempted) -> Bound | Same | Conflict`** is the one
binding rule. It is pure and fieldwise:
- an empty prior field fills;
- a differing non-empty field is a `Conflict` naming the field;
- no change is `Same`.

A conflict is never applied: the kept key stays. `bind` does not look at the bind
source; `assigned` and `observed` obey the same immutability.

**Who uses it.**
- **Replay:** the pure replay fold, public as `state/session_fold.py`, uses `bind`
  and never logs. `session_fold.by_native_key(records)` inverts the accepted keys
  into `NativeKey → chats`. Incomplete keys are omitted, and aliases keep input
  order.
- **Writes:** writers go through `state/session_binding.py`. `session_bindings()`
  takes the history-mutation lock and the sessions lock, and replays the journal
  once. Each `bind()` checks the chat, its generation and `bind`. The batch commits
  as one durable append.
- **Conflict reporting:** `native_binding.report_conflict` is the only emitter of
  `native_binding_conflict`. Only writers call it, so replay no longer repeats old
  conflicts on every command.

**Replay rules** (byte-equal to the old fold on a copied 7,000-chat journal):
- A conflicting update is dropped whole.
- A start is rewritten with the chat's accepted key before its generation is
  projected.
- `Same` still appends when it carries a startup attempt ID. That update is the
  attempt→native link that model-selection lookups read.

In PR 2 (draft #526), the per-event generation step is also public, as
`project_session_generation(...)`. The metadata index (schema 6) persists the fold's
working state and feeds appended events through it, so its incremental catch-up
calls the same code instead of re-deriving it. Search inverts the fold with
`by_native_key` to map native keys to chats.

**Runner-side writer.** `launch/session_scope.SessionAttempt.bind(attempted, source)`
is the only one:
- it mirrors a `Bound` or `Same` ID onto the spawn row;
- it never raises on `Conflict`, because the store already logged it.

Raising belongs to `NativeRun`: whether a conflict is fatal depends on whether the
signal was the attempt's first.

## Adapter template

In `lib/harness/adapter.py`:


**The two template methods.** `plan_native_identity` and `finalize_native_identity`
are base-class templates that adapters never override.
- **Plan** refuses `refused_identity_flags` in passthrough args. It builds a
  `LaunchIntent` from the run's continue/fork request, or from the pre-forked ID
  (Codex fork materialization).
- **Finalize** runs at the launch-binding site in `launch/context.py`, before argv
  projection, against the final child env:
  1. resolves and pins the store;
  2. verifies the exact source file and its native header for resume and fork;
  3. assigns the session ID.

  Argv and env are projected from its result, so they cannot drift from the bound
  key.

Each harness supplies small primitives:

| Primitive | Contract |
|---|---|
| `native_store_for_launch(...)` | Pure: no env or filesystem writes. Returns an absolute store. |
| `pin_native_store(child_env, store)` | The only env writer; never touches the filesystem |
| `assign_session_id(intent, store)` | Default: resume → source ID; fork → pre-forked ID or `None`; create → `None` |
| `validate_intent(intent)` | Optional (Codex: UUID source) |
| `resolve_native_session_file(session_id, native_store)` | Exact file for a key, header-checked; no project-root fallback |
| `observe_after_exit(identity, entry, …) -> PostExit` | Post-exit observation only; never persists |
| `native_observation_reader(child_env, pid)` | Optional live seam: a cursor-based reader of owned observations, or `None` (default; only Claude supplies one) |
| `continues_in_source_store`, `resolves_untracked_source`, `refused_identity_flags` | Class flags |

| Harness | create ID | resume | fork ID | Store pinned via | `observe_after_exit` |
|---|---|---|---|---|---|
| Claude | minted UUID, `--session-id` | `--resume <id>` after the source's `sessionId` checks | harness-assigned (`--resume <src> --fork-session`) | none (store is `<config root>/projects/<slug(cwd)>`; the source is seeded into it) | none after exit; live SessionStart journal feeds `native_observation_reader` (see [Claude](claude-native-sessions.md#launch-owned-sessionstart-journal-and-current-chat)); trampoline successor remains diagnostic |
| Codex | harness-assigned | exact rollout, `session_meta.payload.id` checked | pre-forked by Meridian before exec | `CODEX_HOME = store.parent` | default (nothing) |
| OpenCode | harness-assigned | exact session row | harness-assigned (blackbox subprocess only; streaming and attach refuse forks) | `OPENCODE_DB = store` (`:memory:` is `unbound`) | default |
| Pi | minted, `--session-id` | `--session <abs verified path>` | minted, `--fork <abs source> --session-id <new>` | `PI_CODING_AGENT_SESSION_DIR = store` (env only; Pi creates the dir) | header re-check plus the session-boundary record (the verified run-boundary exit path) |
| Cursor | no native identity | — | — | — | — |

`continues_in_source_store` is `{resume, fork}` for Codex and OpenCode and `{resume}`
for Pi. A continue then runs inside the recorded store. A recorded store that is not
canonical refuses as `missing` before exec.

## Runner pipeline

In `lib/launch/native_run.py`:


The process runner, the streaming runner and `streaming serve` call only these for
identity. Managed primary attach feeds live IDs into `NativeRun.observe`.

1. **`bind_entry(attempt, spec, harness=…) -> NativeRun`**, once before exec. An
   assigned ID binds as `assigned`; a `Conflict` raises `NativeEntryMismatch`. A
   harness-assigned ID binds together with its store at the first observation, so
   no chat ever holds an ID without a store.
2. **`NativeRun.observe(id)`**: owned signals, live or the post-exit artifact ID.
   Only the attempt's **first** signal can fail the run:
   - if the harness was given a pre-exec ID, the first signal must equal it;
   - a harness-assigned fork's first ID must differ from its source
     (`fork_reused_source`).

   Every signal then binds as `observed`. A later conflicting ID is diagnostic: the
   store logs it once, and the run continues. This is the one drift rule that
   replaced three runner copies that disagreed.
3. **`NativeRun.note(id)`**: the connection's *current* ID after exit. It is always
   diagnostic, because transports overwrite it on legitimate switches. Each
   candidate is bound once per turn.
4. **`conclude_native_run(...)`**, once per turn, **after the child exited and
   teardown was joined**. The first error wins, and later identity steps are
   skipped:
   1. **Candidate first signal:** on PR 1, an ID extracted from artifacts. In PR 2
      it is the turn fold's `first_session_id`, observed from live events
      ([run facts](attempt-facts-and-delivery.md#attempt-folds)).
   2. **Current ID:** the connection's current ID goes to `note`.
   3. **Adapter:** `observe_after_exit`. An adapter-reported entry that differs from
      the run's entry is `NativeEntryMismatch`. Pi's session-boundary record supplies
      its owned exit identity; Claude supplies none here.
   4. **Exit:** with no error and an exit key, `session_store.get_or_create_exit_chat`
      invokes the adapter's `native_exists` callback (exact resolver/header check)
      under the history/session locks before every reuse or create. Of the harnesses
      with an adapter boundary, only Pi reaches this step now. A harness without
      one, including Claude, leaves `run_boundary` `unresolved`.
   5. **One row write:** `run_boundary = RunBoundaryOutcome(status, exit_chat_id,
      trampoline_successor_id)`.
   6. **On error:** `record_identity_failure(...)`, the one lifecycle writer for
      identity refusals.
   7. **Otherwise:** `record_started`, with the accepted entry ID.

### Live observations (Claude)

A second, in-run input runs beside the signals above. After exec, `start_observing`
asks the adapter for a `native_observation_reader`; Claude returns one over its
launch-owned journal, every other harness returns `None` and no thread starts. A
per-run drainer feeds `NativeRun.apply_observation` in journal order and is stopped
and joined on every runner exit path, before `conclude_native_run`.

- **Entry stays immutable.** Only the first fact whose source agrees with the planned
  operation (and the assigned ID, if any) completes the entry. A later fact, or an
  unknown-entry run's first non-establishing fact, goes to
  `session_store.get_or_create_observed_chat`.
- **Atomic chat + pointer.** That call finds or materializes the chat for the exact
  `NativeKey` and publishes the spawn row's `current_chat_id` in one lock epoch. A
  missing spawn creates nothing. In-memory dedup state and the journal cursor
  advance only after it returns, per observation, so a retry never rewinds the
  pointer.
- **Fresh chat, not a copy.** A new observed chat gets the source chat's settings and
  work, a captured launch-policy snapshot, fresh history and generation, and no lease,
  spawn ownership or fork lineage. A reused chat keeps its own.
- **Overlap is warned, not denied.** The switch already happened natively. Meridian's
  own resume and capture guards observe the live owner through
  `session_identity.live_observer_for_chat`.

This is last-observed state, not an exit: `run_boundary` for Claude stays
`unresolved` and `continue_chat_id` prefers `current_chat_id`
([row rule](#spawn-row)). Details and limits:
[Claude native sessions](claude-native-sessions.md#launch-owned-sessionstart-journal-and-current-chat).

**Why teardown comes first.** Exit evidence (Pi's boundary record) is written by the child as it shuts down.
Real Pi showed the failure mode: the streaming runner read the boundary record about
4 s before Pi wrote `quit`. The fix orders the read after `SpawnManager.join_teardown`;
there is no polling or timeout
([lesson](../lessons/native-session-identity.md#terminal-status-is-not-process-exit)).

### One-attempt causality

The native run is not re-armed. A startup, terminal, identity, transport, timeout,
guardrail, cleanup, cancellation, or generic failure finalizes the same harness turn.
Teardown must still join before conclusion so late native evidence is visible and the
owned process scope is not orphaned; quiescence is a cleanup and observation contract,
not permission to replay. See [D-one-launch-attempt](../decisions/launch.md#d-one-launch-attempt-harness-turn-failures-finalize-without-replay).

**Serve.** `streaming serve` concludes like the other runners. The original error
survives, cleanup runs in `finally`, and the conclusion runs once.
`LifecycleLog.for_spawn` builds the spawn-scoped lifecycle log.

**Refusals before the runner** (launch preparation) pass through
`ops/spawn/execute_runner.py`. There one `raise exc.for_ref(source_ref)` re-targets
the message, and `ops/spawn/failure_policy.py` uses the typed `failure_code` as the
terminal error.

## Spawn row

In `lib/state/spawn/model.py`:


- **`chat_id`** is the entry chat, and it is immutable. There is no separate
  `entry_chat_id`.
- **`current_chat_id`:** the last observed chat of a run that has live observations
  (Claude). Absent on rows that never observed a switch and on every other harness.
  It is not in the 0.9.3 schema: a strict 0.9.3 reader rejected it on a copied
  runtime, which is why this ships as a minor release (metadata projection schema 7,
  new namespace).
- **`run_boundary: RunBoundaryOutcome | None`:**
  - `status ∈ {verified, unresolved, mismatch}`;
  - `exit_chat_id` is set exactly when `verified`; a validator enforces this;
  - `trampoline_successor_id` is a Claude diagnostic that never binds.
- **`SpawnRecord.continue_chat_id`** is the one continue rule: a `mismatch` run keeps
  its entry chat; otherwise `current_chat_id` when set; otherwise a terminal run's
  verified exit chat (Pi); otherwise the entry chat. The primary exit hint,
  `--continue pN`/`--fork pN` and `session log pN` all use it.
- **Dogfood rows.** Rows written by the pre-restructure PR 1 dogfood build carry flat
  `entry_chat_id`, `exit_chat_id`, `exit_identity` and top-level
  `trampoline_successor_id` fields (`repository.DOGFOOD_BOUNDARY_FIELDS`). The strict
  schema quarantines them. Since PR 3, `state/spawn/dogfood_migration.py` rewrites them
  once, run by `meridian doctor` and primary-launch background repairs; the read-time
  translator is gone ([why](../decisions/native-only-history.md#dogfood-rows-migrate-once-not-on-read)).
- **`run_boundary_summary(row)`** renders verified entry/exit mappings or explicit
  unverified diagnostics for `spawn show`. Primary completion omits absence-only
  diagnostics; mismatches and possible successors still warn. Presentation never
  changes the boundary status or continue target.

The history-index `SCHEMA_VERSION` is 5 at PR 1's head. It was bumped at each
serialized record shape change so that older builds refuse the index with a typed
rebuild instruction instead of crashing
([lesson](../lessons/dogfooding-pr-builds.md#a-pr-build-must-never-touch-a-real-runtime-root)).
PR 2 bumps it to 6, adding the `session_chats` working set.

## Source key

`SessionRequest.source_native_store`, plus the requested native ID, is the only
description of a resume or fork source. It flows:
1. `ops/reference.py`, from the chat binding or the spawn row;
2. continue replay;
3. the fork request builder;
4. `execute_runner`;
5. the adapter template.

`SessionRequest.source_ref` names the user's ref in refusals. A tracked source with no
recorded store refuses as `unbound`. No launch or ops code carries a harness-specific
source field ([why](../decisions/native-session-identity.md#one-recorded-source-key)).
A spawned exact continue reuses the source chat; fresh and fork launches allocate new
chats.

## Testing

- **Fakes.** Reporting fakes must adopt `spec.native_identity.session_id`; a fixed
  fake ID correctly trips `entry_mismatch`.
- **Fixtures.** Native source fixtures need faithful headers (Claude `sessionId`,
  Codex `session_meta`); a `"{}\n"` stand-in refuses as `missing`.
- **Shims.** The qualifying standard is POSIX `sh` harness shims at the real runner
  seams. They emit real owned events, including negatives:
  - prose;
  - nested keys;
  - no event;
  - a contradictory first frame;
  - an unrelated concurrent session.
- **Refactor gates.** Refactors of the fold or the template are gated on real-data
  equivalence:
  - copied-journal byte equality;
  - 1,000 generated histories;
  - a launch-golden matrix of harness × operation × mode.

**Evidence, by kind.** For Pi, the shipped run-boundary exit path was qualified by real
probes in v0.7.0. For Claude, the earlier built-wheel `/clear` → `/exit`, Haiku-spawn
and 0.9.3 fork probes qualified the superseded final-exit receipt and are historical
only. They also showed that Claude 2.1.296 emits `SessionStart.source=fork`. The
current Claude design has focused fake and process-seam tests plus one isolated
real-Claude 2.1.296 Haiku turn (a `/clear` moved the pointer during the run, `--continue`
followed it, a SIGTERM left the pointer at the last observed chat). That run did not
cover fork, `/compact`, repeated switches or concurrent writers, and its source tree
was changing underneath it. The full gate and a frozen-source native probe were
pending, so this is not final qualification
([lesson](../lessons/native-session-identity.md#green-suites-did-not-find-the-seam-defects)).

## Related

- [Native session identity decision](../decisions/native-session-identity.md)
- [Native-only history decision](../decisions/native-only-history.md)
- [Native transcript reads](native-transcript-reads.md)
- [Legacy native import](legacy-native-import.md)
- [Session state](state-system/session-state.md)
- [Session reference resolution](../decisions/session-reference-resolution.md)

**Provenance:** `work:native-harness-session-identity`:
- `design/pr1-foundation-restructure.md`, revision 3 (reviews `spawn:p7092`,
  `spawn:p7097`);
- phase lanes P0 `spawn:p7100`, P1 `spawn:p7102`, P2 `spawn:p7114`, P4 `spawn:p7104`,
  P3 `spawn:p7116`;
- thermo recheck `spawn:p7120`, alignment review `spawn:p7121`, real-data probe
  `spawn:p7122`, fix pass `spawn:p7126`;
- earlier lanes and reviews are listed on the
  [decision page](../decisions/native-session-identity.md);
- code checked at `08499af0`; final live probe and targeted re-probes are recorded in
  `work:native-harness-session-identity` evidence.
- Claude boundary extension, first design (final-exit receipt, superseded for
  Claude): `work:exit-identity-reporting`, commits `3aa18aae`, `3630a1f0b`,
  `b74dc0b3`, and shared persistence gate `a2713624` (still serves Pi); review
  `spawn:p7571` and structural review `spawn:p7590`; evidence
  `claude-boundary-live-probe.md`, `claude-final-live-probe.md`,
  `claude-fork-clear-final-proof.md`.
- Current-chat tracking: same work item, `DIVERGENCE/02-last-observed-session.md`,
  `current-chat-primary-refinements.md`, `current-chat-final-review-result.md`,
  `current-chat-live-probe-result.md`; commits `19c8ba3e`, `679ab27f`, `cea151fa`
  (unmerged PR #559).
