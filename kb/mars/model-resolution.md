# Model resolution

Mars owns model aliases and harness routing; Meridian takes the route from the launch bundle and never guesses a harness from a model name. Package authors can then change model vocabulary without a Meridian release, and a model-name prefix proves neither harness support nor account readiness. Contracts: mars-agents `src/routing/AGENTS.md` and `src/models/AGENTS.md`; meridian-cli `src/meridian/lib/catalog/AGENTS.md` and `docs/configuration/mars.md`.

## Possible, Curated and Selection stay separate

- **Possible** is a view derived from discovered harness and model evidence, not a second persisted catalog.
- **Curated** applies authored display choices to that view. It shapes only the human `models list`, never alias resolution, sync or routing.
- **Selection** chooses the runtime route from support and auth evidence plus authored policy, and never reads Curated.

The commands split the same way: `models list` is the human inventory, `models aliases` the alias definitions, `models catalog` raw provider data, `models resolve` an explanation of alias resolution, and the launch bundle makes the final routing decision. A list entry, an alias's preferred harness or a catalog row alone doesn't establish a runnable route, so machine consumers never read the curated list as an alias inventory. Preview and launch routing share evidence and one candidate evaluator so they can't disagree. A fixed harness is the caller's choice, not proof its model is available, and a confirmed match can outrank a passthrough candidate whose evidence is unknown. An explicit harness narrows routing but grants no permission and establishes no support or auth.

## The native model ID comes from the route

One canonical model can need different native IDs: provider-qualified for OpenCode, bare for Codex. Mars supplies the value for the selected harness, and every native model boundary uses it. Cursor encodes effort in its model slugs, so Mars resolves the full launch slug ([Cursor](../harnesses/cursor.md)).

## A model is optional

With no managed model the harness picks its default, and a stronger harness override can clear an incompatible model. Older snapshots record absence as an empty string; a placeholder token would instead be sent to the harness as an override.

## Policies and fallback

Model and harness selection decide where a launch runs; execution policy decides how. Project-wide operational settings load from configuration files, while per-launch policy resolves separately for each launch (ladder: meridian-cli `src/meridian/lib/config/AGENTS.md` and `docs/configuration/mars.md`). Model policies refine execution settings for a matched model. A matching profile model-policy rule beats the profile's top-level value, or a top-level effort would disable the profile's own per-model rules; overlay scalars sit above both, and CLI and environment overrides beat all of them. Project-specific agent policy belongs in Mars overlays, not in generated agent files that sync overwrites. An empty overlay doesn't erase profile rules.

Fallback covers harness availability before launch, never failures after: an authentication or capacity failure after launch needs an explicit alternative route. Enabled targets bound routing before availability and auth checks. Concrete profile rules supply backup candidates in order, independently of which settings rule matched, so changing the primary model can't hide earlier backups; glob and `no-fallback` rules don't, and an explicit model request disables fallback.

Each harness lowers the resolved effort to what it supports and reports what it can't do. Current adaptive-thinking Claude models kept reasoning in every tested configuration, so presenting an "off" switch as working would mislead ([tool permissions](../permissions/tool-permissions.md)).

## Refresh and caching

Catalog refresh and harness probes are separate concerns under the same flags. `--no-refresh-models` uses cached data but doesn't promise no subprocess runs, since capability or auth checks may still run. `MARS_OFFLINE` differs from skipping probes: offline can make a probe unavailable even with a stale cache. A Meridian dry run asks for cache-only resolution, under its own cache key, so a cache-only failure can't suppress a later refresh. Spawns don't expose Mars's refresh controls as flags.

## Agent profiles and prompting guidance

A profile combines instructions with launch defaults; a caller can override its model and keep its role and skills. Hiding a profile from model-facing discovery doesn't stop anyone naming it on the CLI. Prompting guidance (`meridian mars models prompting`) resolves an agent through the same policy as launch before looking up its model's guidance, because a shallow duplicate resolver could advise on a model that routing later replaces. Agent names are tried before aliases because callers usually ask about the agent they're about to spawn. Guidance is on demand, not added to every launch prompt.
