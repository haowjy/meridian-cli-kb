# Hook Event Contract

The current event catalog and payload are in `docs/hooks.md` and [hooks and plugins](../concepts/hooks-and-plugins.md).

## D-hook-event-names — past-tense moments named for what just happened

Events name the moment that just happened; notification events use the `notify.` prefix. Permanent aliases would give the same moment two names. `work.resume` does not describe activation of a new item, and `work.switch` does not describe `work start`. Events with no consumer are not added merely because an internal transition exists.

## D-hook-deprecation-window — old names load for one minor release

See `docs/releasing.md#compatibility-and-versioning` for the deprecation policy and `docs/upgrading.md` for removed hook names.

## D-work-activation — activation means someone chose the item

Activation means a user or agent chose a work item, including a primary attaching it. Inheritance, rename maintenance and clear do not count. Inheritance would otherwise run the default blocking autosync for every child spawn; work events lack a spawn-kind filter.

> [!FLAG] **Needs human review**
> Whether primary launches should fire activation remains open.

## D-autosync-default-events — `spawn.finished`, `work.activated`, `work.done`

The default autosync events are `spawn.finished`, `work.activated`, and `work.done`. Adding `spawn.started` would put a blocking sync before every spawn. A background context autosync is tracked in [#691](https://github.com/haowjy/meridian-cli/issues/691).

## D-hook-failure-policy-removed — hook failures always continue

Hook failures log and continue; no supported event is a gate, so a configurable fail policy would promise enforcement it cannot provide. An impossible event condition is rejected; disabled entries for incompatible default events still override lower layers so narrowing a route cannot expose an older action.

## D-hook-fact-owners — one owner for each fact about an event

Each event fact has one owner. Separate publishers and separate definitions of session existence caused attachment and work-directory disagreement. See [hook event lessons](../lessons/hook-event-contract.md#repeated-fixes-mean-parallel-owners).

## Open and follow-up

Hook dispatcher ownership and `Hook.remote` are tracked in [#692](https://github.com/haowjy/meridian-cli/issues/692) and [#693](https://github.com/haowjy/meridian-cli/issues/693).
