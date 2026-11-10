# Hooks

Hooks run side effects on lifecycle and notification events. Event catalog and payloads: meridian-cli `docs/hooks.md`; hook rows and dispatch: `src/meridian/lib/hooks/AGENTS.md`.

## Hook failures always continue

A failure is logged and the lifecycle and later hooks continue. No supported event is a gate, so a configurable fail policy would promise enforcement it can't deliver. An impossible event condition is rejected. Disabled entries for incompatible default events still override lower layers, so narrowing a route can't expose an older action.

## Events name what just happened

Events are past-tense moments named for what happened; notification events use the `notify.` prefix. Permanent aliases would give one moment two names: `work.resume` doesn't describe activating a new item, and `work.switch` doesn't describe `work start`. An event is added only when something consumes it, not because an internal transition exists: every event is a public name to keep stable. Check accepted event names against what real producers emit: an event can pass validation and never fire ([lessons](../lessons/hook-event-contract.md)). Old names load for one minor release ([compatibility](compatibility.md)).

Each fact about an event has one owner: separate publishers and separate definitions of "session exists" made attachment and work-directory events disagree. Work activation is defined in [work items](../architecture/state-system/work-items.md#activation-means-someone-chose-the-item).

> [!FLAG] **Needs human review**: whether a primary launch that attaches a work item should fire `work.activated` is undecided. Flagged 2026-11-07.

## Project resolution

`meridian hooks list`, `check` and `run` resolve the project from the actual cwd rather than the inherited project environment, because a subprocess may be working in another checkout. Notification emission follows the session's project. The two can differ on purpose.

## Git autosync

Git autosync is a builtin hook, never a state authority; its default events are `spawn.finished`, `work.activated` and `work.done`. Adding `spawn.started` would put a blocking sync before every spawn. A background context autosync is planned ([#691](https://github.com/haowjy/meridian-cli/issues/691)).

Autosync merges rather than rebases, and aborts on conflict: local work stays at HEAD and the remote stays available to inspect. A rebase can stop partway through replay and wedge later syncs; a merge has one conflict point. Conflict state is recorded separately, never written into user instruction files. Behavior: `src/meridian/lib/hooks/builtin/`.

Dispatcher ownership and `Hook.remote` are open ([#692](https://github.com/haowjy/meridian-cli/issues/692), [#693](https://github.com/haowjy/meridian-cli/issues/693)).
