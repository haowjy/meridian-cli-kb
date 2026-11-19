# Hooks

Hooks run side effects on lifecycle and notification events. Event catalog and payloads: meridian-cli `docs/hooks.md`; hook rows and dispatch: `src/meridian/lib/hooks/AGENTS.md`.

## Hook failures always continue

A failure is logged and the lifecycle and later hooks continue. No supported event is a gate, so a configurable fail policy would promise enforcement it can't deliver. An impossible event condition is rejected. Disabled entries for incompatible default events still override lower layers, so narrowing a route can't expose an older action.

## Events name what just happened

Events are past-tense moments named for what happened; notification events use the `notify.` prefix. Permanent aliases would give one moment two names: `work.resume` doesn't describe activating a new item, and `work.switch` doesn't describe `work start`. An event is added only when something consumes it, not because an internal transition exists: every event is a public name to keep stable. Check accepted event names against what real producers emit: an event can pass validation and never fire, and a test that constructs the event directly can't show a producer exists. A renamed event's old name loads for one minor release ([compatibility](state/compatibility.md)); after that window it is rejected with an error naming the replacement.

Each fact about an event has one owner: separate publishers and separate definitions of "session exists" made attachment and work-directory events disagree. Work activation is defined in [work items](state/work-items.md#activation-means-someone-chose-the-item).

> [!FLAG] **Needs human review**: whether a primary launch that attaches a work item should fire `work.activated` is undecided. Flagged 2026-11-07.

## Project resolution

The manual hook commands and notification emission resolve the project differently on purpose: a subprocess may be working in another checkout from the session that launched it. The rule is in `src/meridian/lib/hooks/AGENTS.md`.

## Context sync is not a hook

Clones of git contexts sync through the `meridian-sync` daemon, not a hook: a hook fires only on lifecycle events, so an edit made outside a spawn waited for the next one ([context sync](context-sync.md)). A `git-autosync` row, `Hook.remote` and `Hook.exclude` still load for one minor release and do nothing ([#693](https://github.com/haowjy/meridian-cli/issues/693)). Dispatcher ownership is open ([#692](https://github.com/haowjy/meridian-cli/issues/692)).
