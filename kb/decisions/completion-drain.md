# Completion Drain

A finished harness turn is a completion candidate, not proof that the work is finished. These decisions say when a child closes, what it reports, and who decides how long it runs. Mechanism: [drain plans](../architecture/drain-plans.md) and [completion coordination](../architecture/completion-drain-coordination.md). Why each row of the hold's table sits where it does: `src/meridian/lib/streaming/.context/completion-and-entry-points.md`.

## Open while work runs {#d-open-while-work-runs}

A spawn stays open while work runs, including a background command while the model is idle. A turn end with something running holds the spawn with no command needed. With nothing running it closes: at once where the harness reports its own work, and after one confirmation elsewhere. A primary stays open until the human ends it and has no `done`.

A child that has to wait starts the job in the background with a `timeout` that covers it and ends its turn. Claude stops a background command at its `timeout`, 30 minutes by default and 2 hours at most, so a longer job has to run detached with a watcher the child restarts. A subagent doesn't park for hours: it reports and closes, and its owner resumes it with `--continue` or spawns another.

## The ask is skipped where the harness reports its work {#d-skip-ask-native}

A harness with a native task source gets no confirmation turn at a turn end. Today that is Claude. The gate is `native_task_source()`, not a harness ID, so a new source needs no edit. Codex and OpenCode keep the ask because a background shell is invisible there, and the ask is the only check for it.

On Claude the ask told Meridian nothing it couldn't already see. It cost 3.3% of Claude child usage over 15 days and 5 to 17 s per child. The trade is that a premature stop is no longer caught: 10 of 2,469 children rearmed with nothing visible running and 108 more did other work when asked, so the true rate sits between 0.4% and 4.8%. `--continue` recovers those.

The ask had also been covering for three things, so three guarantees replace it: a parent is woken for an unread child's report before it closes, `done` and `rearm` work during a turn, and the report follows the last turn instead of staying on an early "still waiting".

## Waiting has one number {#d-one-waiting-number}

The idle limit is the only waiting window. It is the same for every model, with no per-model or per-provider table: model and provider limits change every few weeks, and no build decision rests on them or on tracking them. `timeouts.resident_deadline_seconds` overrides it through the usual precedence.

At the limit the child gets one ask, naming what runs and what closing stops. A reply closes it and is its report. `rearm` stays. No work within 300 s is `timed_out`; work in that time closes the window and restarts the idle clock. A `rearm` is short by default, with the next check 270 s after the child goes idle; `--long` puts it at the idle limit, for a wait Meridian can't see such as remote CI. Visible blockers clearing ask at once in both forms. A reason is required so a parent can see why a child stayed, and a bare `rearm` warns for one release.

A partial clear stays silent. When a spawn is waiting on two things and one finishes, the hold sends nothing: Claude already tells the spawn when its own background task ends, and a parent waiting on two children is still waiting, so a message would only interrupt it.

## No work limit {#d-no-work-limit}

Meridian doesn't cap how long a child works. The parent controls that and can cancel a child that has gone on too long, and `--timeout` stays the opt-in absolute ceiling. A cap on active-turn time, with a soft stop and rearm grants, was designed and cut before it was built. The idle limit covers a stalled child; a child doing real work is the parent's call.

## No keep-warm pings {#d-no-keep-warm}

A waiting child takes one cold wake, a turn that has to rewrite the expired prompt cache, or reports and closes for its owner to resume. Pings would save at most about 30 cents a window, and they would add a frame filter, a blocker snapshot and TTL plumbing to the hold, plus a rule for everything a ping must not do.

## Parents wait by ending their turn {#d-parents-end-turn}

Spawned parents are told to end their turn, and the hold wakes them with each finished child's report path. Primaries keep `spawn wait`, because their 1-hour cache keeps that wake warm.

| A spawned parent waiting 6 hours, Opus at 141k | Cost | Basis |
|---|---|---|
| Foreground wait at the 10-minute cap | $22 to $29 | 36 wakes at a measured $0.62 to $0.81 |
| `spawn wait --yield-after-secs 270` | About $3.40 | 80 warm wakes at a measured $0.043 |
| Turn ended, child reports at the first ask | About $0.60 | One cold wake, estimated |

No spawned parent has actually waited 6 hours; these are models at a measured wake price. The limit is that a parent which ended its turn can't react mid-wait, so a child's question reaches it when the child closes.

## A close takes what it started {#d-close-cancels-descendants}

`done` and a reply to the ask close through one path. Both cancel running descendants and stop local background tasks, and the ask names what they will stop. A job that must outlive its child has to run detached. One rule for both closes replaced a split between publishing a ready tree and cleaning up a blocked one, and a closed child can't read what it left running anyway.

A dead backend waiting on descendants cancels them too, through the same cleanup. A dead child can't read their reports, so it would otherwise stay `running` to the deadline. That applies the rule for a closing child to a child that died, and it is an interpretation rather than a direct ruling. [#671](https://github.com/haowjy/meridian-cli/issues/671) stays open for the parent paths outside the hold.

Three edge cases are settled the same way:

- A reply whose turn fails keeps `failed` and the retained report. If an expiry ask was pending, descendants are cancelled, because the ask said closing does that. Publishing the older good candidate would hide a failed last turn.
- `done` followed by a crash before the turn ends reads `failed`, because a candidate stands for a finished turn.
- `spawn cancel` of an idle child with a ready report keeps today's result.

## Completion report {#d-completion-report-contract}

`report.md` wins whenever a child wrote one. Otherwise the rule depends on whether the ask stays.

- **No turn-end ask (Claude):** the report is the last real turn's text, and it moves with each later turn, including the reply to a `spawn message`. A wake turn saying "background finished" can replace a fuller earlier reply; that loss is accepted.
- **Turn-end ask (Codex, OpenCode):** the ask says the previous reply is kept, so a short "Done" must not replace it ([#725](https://github.com/haowjy/meridian-cli/issues/725)), and a later turn doesn't move the report without a `rearm`. If the rearm budget is exhausted, later work cannot replace the retained report; `src/meridian/lib/streaming/.context/FUTURE` tracks that.
- **Any harness:** a wake (the message naming a parent's unread finished children), or an idle-limit ask that lands between turns, asks the child to write something new, so its answer becomes the report. The message says so.
- **Pi** retains its confirmation reply.

`rearm` no longer deletes the kept report, on any harness.

## Completion channel {#d-completion-channel}

A child without a usable completion signal channel receives no confirmation turn. A read-only Codex child with spawn capability can use the scoped execpolicy escape; without spawn capability it has no reason to hold for child work. If sandbox policy is unset but Codex defaults to read-only, the signal may still fail because Meridian does not infer that policy. See [Codex child spawn](sandbox-platform-access.md#d-codex-child-spawn).

## Evidence and `done` {#d-evidence-and-done}

A turn-end ask is best effort: a silent child publishes only on `ready` evidence, and known blockers or `unknown` evidence hold it ([what those mean](../architecture/completion-drain-coordination.md#the-coordinator-accepts-evidence-it-does-not-discover-work)). `done` waits out a running turn, and a turn that sits idle for a whole window is asked once and, with no work in the next 300 s, ends `timed_out`, not `succeeded`. That reverses the earlier rule, because publishing over a running turn reports an outdated candidate. Success never publishes over a running turn, on any close.

`done` can override known blockers but not `unknown`. A reply to the ask closes on any evidence, `unknown` included, because the ask told the child what closing stops and the child chose it.

The calling harness's wait setting supplies the default idle limit, separately from the caller's wait checkpoint.

## Signals need no candidate {#d-signals-need-no-candidate}

`done` and `rearm` work during a child's turn, including its first. They used to be discarded when no candidate existed, because a signal file is a mailbox: a stale `done` could pre-accept the next candidate and a stale `rearm` could spend budget. That can't happen now. With no candidate and no rearm, nothing reads the files until the turn end that creates the candidate, so a read finds only that turn's signals. Of two signals in one read the later stands, by file mtime, and a tie stays a rearm.

## The hold is one table {#d-hold-one-table}

Many bugs at one boundary, even ones that aren't regressions, are evidence that the architecture is wrong. The ask lifecycle took three review rounds on #886, each adding a guard to about ten flags and timers and two evaluators that reimplemented one precedence, so it was rebuilt as one state value and one pure `decide(state, facts)` before anything else was added ([#914](https://github.com/haowjy/meridian-cli/pull/914)). An enum over the existing flags was rejected because it would keep both evaluators. A new hold behaviour is a row, a fact or a fold in `hold_state.py`. Pi stays separate.

## Claude native work {#d-claude-native-hold}

Claude's own background-task stream informs the shared completion hold. A harness mod cannot replace that signal: the finish decision and timeout must survive a wedged child, and the mod API did not expose background shell tasks. See [drain plans](../architecture/drain-plans.md#claude-native-holds).

## Lifecycle is a minor cost line

Across all harnesses, usage at API prices rose from about $3,000 a week in late September to $9,800 in mid-October, before the confirmation turn existed. GPT volume drove that rise, and Claude's own peak came later from more children rather than costlier ones. All Claude nudge turns came to 0.9% of the period's usage, and a nudge loop was under half of that. So look elsewhere first for a cost rise, and judge this work by the reports it stopped losing as much as by the money. These are API-price figures: most of that traffic runs on subscriptions and is not a bill.
