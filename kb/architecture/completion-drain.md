# Completion hold

A finished harness turn is a completion candidate, not proof the spawn is done. The hold decides when a candidate may publish: a spawn stays open while visible work runs, even when the model is idle. A primary never holds: it stays open until the human ends it, and has no `done`. The hold's states, decision table, idle limit and signals are specified in the source checkout's `src/meridian/lib/streaming/.context/completion-and-entry-points.md` (Pi: `pi-drain.md`); this page holds the reasons that span subsystems. How the terminal outcome is then written is in [spawn finalization](spawn-finalization.md).

## Evidence, not discovery

The shared coordinator accepts evidence and policy; it doesn't discover work. Persisted descendant evidence is ready, blocked or unknown, and unknown never proves that nothing remains. Each ordinary success needs a fresh read, because a child may have appeared since the last one. That read is still a snapshot: it doesn't atomically stop a child being admitted before the terminal write. A store-wide read failure stays unknown and blocks completion.

Child rows publish as complete directories, staged under a non-row directory, so a reader sees either no row or a row with its prompt and state, and a scanner never mistakes staging for a published child.

Resident, Pi and plain profiles supply different policy and evidence through composition. An inheritance hierarchy would force Pi to override timeout, event, stream-exit and finalization hooks whose order matters, making hook order and mutable state an implicit contract. One state value and one decision table own the hold's precedence; layering an enum over independent flags would leave competing evaluators.

The terminal write comes before asynchronous cleanup, so a stuck cleanup can't hold the result forever and a cleanup failure can't replace a published outcome. Startup recovery finishes cleanup interrupted by a crash.

## Native work evidence replaces the confirmation ask

A harness that reports its own background work gets no confirmation ask at turn end. Claude does; Codex and OpenCode keep the ask because their background shells can be invisible. Skipping the ask saves a turn, and accepts that a premature finish with no visible work may need a `--continue`. Claude's background-task stream feeds the hold, and a harness mod can't replace it: the finish decision and timeout must survive a wedged child, and the mod API didn't expose background shell tasks. An empty task snapshot isn't a finished follow-up turn, since Claude may still owe the result of the turn the task triggered; a malformed snapshot holds completion.

A child with no writable completion-signal channel gets no confirmation ask. A read-only Codex child that can spawn uses the scoped escape rule ([sandbox projection](sandbox-projection.md#spawning-from-inside-a-sandbox)); if its sandbox is unset but Codex defaults to read-only, the signal can still fail, because Meridian doesn't infer that default.

## Waiting has one number

The idle limit is the only waiting window, independent of provider cache limits, which change. At expiry the child gets one ask naming its running work and what closing will stop; a reply closes, `rearm` keeps it open.

- **No work limit by default.** The parent can cancel, `--timeout` is an opt-in ceiling, and the idle limit catches stalled work.
- **No keep-warm pings.** A waiting child takes a cold wake or closes for its owner to resume; pings add lifecycle machinery and recurring cost for a small saving.
- **Parents wait by ending their turn.** A spawned parent ends its turn and is woken with finished children's report paths: repeated foreground waits burn turns and can cost far more. The cost is that a parent can't react to a child before it closes. Primaries keep `spawn wait`. A parent with unread child results is woken before it closes, including for a child that finished during its turn.

## A close takes what it started

`done` and a reply to the expiry ask close through cleanup: running descendants are cancelled and local background tasks stopped. A dead backend cancels descendants too, because a dead child can't read their reports. A failed reply stays failed rather than publishing an older successful candidate. Work that must outlive the child must run detached.

`done` overrides known blockers but not unknown evidence, and waits out a running turn so it can't publish an outdated candidate. A reply to the expiry ask closes whatever the evidence, because the ask named what closing stops. `done` and `rearm` are read at the turn end that creates a candidate, so neither can pre-accept a later turn.

## The report

An explicitly written `report.md` wins. Otherwise Claude's report follows its last real turn; Codex and OpenCode keep the earlier answer when the confirmation reply only says it's done; Pi keeps its confirmation reply. A wake or expiry ask asks for a new answer, which becomes the report. `rearm` keeps the retained report.
