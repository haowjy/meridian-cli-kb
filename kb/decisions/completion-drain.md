# Completion drain

A finished harness turn is a completion candidate, not proof that the work is finished. The source checkout's `src/meridian/lib/streaming/.context/completion-and-entry-points.md` describes the hold; [drain plans](../architecture/drain-plans.md) and [completion coordination](../architecture/completion-drain-coordination.md) describe its boundaries.

## Open while work runs {#d-open-while-work-runs}

A spawn stays open while visible work runs, even when the model is idle. A primary stays open until the human ends it and has no `done`. A child waiting on a long job can background it and end its turn; work that must outlive the child must run detached.

## Native work evidence {#d-skip-ask-native}

A harness with a native task source does not receive a turn-end confirmation ask. Claude supplies that evidence; Codex and OpenCode retain the ask because background shells can be invisible. Skipping the ask saves a turn but accepts that premature completion with no visible work may require `--continue`.

## Waiting has one number {#d-one-waiting-number}

The idle limit is the waiting window, independent of changing provider cache limits. At expiry the child gets one ask naming work and what closing stops. A reply closes; `rearm` keeps it open. Work in the response window restarts the idle clock; no work ends it as timed out. Partial blocker clearance does not interrupt the child.

## No work limit {#d-no-work-limit}

Meridian does not cap active work by default: the parent can cancel it, and `--timeout` supplies an opt-in ceiling. The idle limit addresses stalled work.

## No keep-warm pings {#d-no-keep-warm}

A waiting child takes a cold wake or closes for its owner to resume. Keep-warm pings add lifecycle machinery and recurring cost for a limited saving.

## Parents wait by ending their turn {#d-parents-end-turn}

Spawned parents end their turn and receive finished children's report paths. Repeated foreground waits consume turns and can cost substantially more. The tradeoff is that a parent cannot react to a child until the child closes. Primaries retain `spawn wait`.

## A close takes what it started {#d-close-cancels-descendants}

`done` and a reply to the expiry ask close through cleanup, cancelling running descendants and stopping local background tasks. A dead backend also cancels descendants: a dead child cannot read their reports. A failed reply remains failed rather than publishing an older successful candidate.

## Completion report {#d-completion-report-contract}

An explicitly written `report.md` wins. Without one, Claude's report follows the last real turn; confirmation-based Codex and OpenCode holds preserve the earlier answer when the confirmation only says done. A wake or expiry ask requests a new answer, which becomes the report. Pi retains its confirmation reply. `rearm` keeps the retained report.

## Completion channel {#d-completion-channel}

A child without a usable completion signal channel receives no confirmation turn. A read-only Codex child with spawn capability can use the scoped execpolicy escape. If sandbox policy is unset but Codex defaults to read-only, the signal may still fail because Meridian does not infer that policy. See [Codex child spawn](sandbox-platform-access.md#d-codex-child-spawn).

## Evidence and `done` {#d-evidence-and-done}

A silent child publishes only with ready evidence. `done` can override known blockers but not unknown evidence, and waits out a running turn: otherwise it could publish an outdated candidate. A reply to the expiry ask closes on any evidence because the ask named what closing stops.

## Signals need no candidate {#d-signals-need-no-candidate}

`done` and `rearm` can be written during a turn, including the first. They are read at the turn end that creates a candidate, so a signal cannot pre-accept a later turn. When both arrive in one read, the later signal wins; a tie stays a rearm.

## The hold is one table {#d-hold-one-table}

One state value and decision table own the hold's precedence. An enum layered over independent flags was rejected because it would leave competing evaluators.

## Claude native work {#d-claude-native-hold}

Claude's background-task stream informs the hold. A harness mod cannot replace it: the finish decision and timeout must survive a wedged child, and the mod API did not expose background shell tasks. See [Claude native holds](../architecture/drain-plans.md#claude-native-holds).
