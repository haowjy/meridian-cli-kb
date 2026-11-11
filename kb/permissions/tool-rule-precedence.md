# Tool rule precedence and project-wide denies

How a tool call is decided when several rules match it, and how a project-wide deny joins an agent's own rules. User-facing behavior: Mars `docs/config/mars-toml.md` ("Project-wide tool rules"), meridian-cli `docs/agent-profiles.md` ("Project-wide tool rules") and `docs/upgrading.md`. How harnesses enforce rules at all: [tool permissions](tool-permissions.md).

> [!NOTE]
> Project-wide denies and the map form of `[agents.<name>.tools]` are on `feat/project-tool-rules` in mars-agents (Mars 0.29.0) and meridian-cli, not yet merged. Delete this note once both are on `main`.

## The chosen rule: strictest match, narrowness never matters

For a tool call, take every rule whose key matches it. Any `deny` denies; otherwise any `ask` asks; otherwise any `allow` runs. With no matching rule, `'*'` decides, and without `'*'`, `approval` does. How narrow a key is never matters. This is Claude Code's own order and Codex's rules engine's, written in Mars's `tools:` grammar.

The reason: a rule should mean the same on every harness and be readable from its keys alone. Before it, precedence was decided in four places (Mars's `resolve_by_specificity`, and Meridian's Pi, Codex and OpenCode projections) with three meanings of "narrower": fewer native tools, scoped beats unscoped, longer literal. Claude has no "narrower" at all, so one profile gave different answers per harness: `bash(git *): deny` with `bash(git status *): allow` denied `git status` on Claude and allowed it everywhere else. "Narrower wins" was ruled out outright because the harnesses don't share it.

OpenCode's ordered list (last match wins) was rejected for the place project rules would have to sit in it. With agent rules last, any profile's `bash: allow` overrides a project-wide scoped deny; with project rules last, a project-wide allow overrides every profile deny.

What the rule costs, accepted without exception:

- "Only these Bash commands" is a whole-policy allowlist (`'*': deny` plus the allows, and `skill: allow` on Claude); `bash: deny` beside scoped allows turns Bash off.
- An exception inside a deny can't be written; write narrower denies. Claude never honored it anyway.
- "No edits, but writes are fine" can't be written, because `edit` covers `write`. Two of four harnesses can't tell them apart, and a Claude write replaces a whole file, so the form never protected existing files.

If confining one tool while others follow `approval` turns out to matter, the smallest fix is a second default key for "calls to this tool no other rule matches"; it keeps the rule intact because it applies only when nothing else matches.

**Today the enforcers still rank by narrowness.** Shipping the rule is a second release, not started: the four Meridian compiles apply it, and the code that exists only to implement "narrower wins" is deleted (`resolve_by_specificity` in Mars, the literal-length ranks in the Pi extension and Codex hook, Pi's `write`-over-`edit` rank, OpenCode's specificity sort, and Claude's permission-mode switch for an unscoped deny beside scoped allows). Don't extend any of those. Meridian's command grants then need their own tier, decided before the rules, or a profile `bash: deny` would stop an agent from running `meridian spawn done`; that tier needs a live check on each harness, because a regression there leaves agents unable to report completion. `docs/upgrading.md` gives users notice of the three forms that change, and `mars validate` notes each profile using one (`tool-rule-form-changes`).

## Project-wide rules restrict every agent and widen none

`[settings.tools]` in the user config, `mars.toml` or `mars.local.toml` applies to every agent and to a launch with no agent. A package can't ship `[settings]`, so no installed package adds or removes one. `mars check`, `mars init` and the plugin build pass an empty map: those preview or build a package for other projects, which don't get this project's settings.

**Sources merge by key, most specific wins:** user config < `mars.toml` < `mars.local.toml` < `[agents.<name>.tools]`. Unlike `network-deny` hosts, denies don't add across files: a committed `mars.toml` can replace a user-config deny by naming its exact key, and the owner accepted that. A profile's own `tools:` map is not in this chain: its rules never replace a project-wide rule; both apply. `[agents.<name>.tools]` is the user's own edit of one named agent, so it is the one source that replaces a key on either side; that is how one agent is exempted. No value removes a key, so an exemption is spelled `allow`, which also pre-approves the call.

**A project-wide `allow` will never add a tool to an allowlist agent.** `allow` means both "don't prompt" and "on the allowlist"; a project-wide line is written for the first meaning, and giving it the second would hand a shell to a reviewer written without one. Claude agrees: a settings allow doesn't add a tool to a subagent whose `tools:` omits it. `'*'` is rejected in `[settings.tools]`: a project-wide `'*': deny` would turn every agent into an allowlist, and `approval` already decides unmatched calls.

## Why the first release takes `deny` only

While the enforcers rank by narrowness, a project-wide `ask` can't be settled by keeping or dropping rules. A project-wide `bash(git push *): ask` beside a profile `bash: deny` asks instead of denying on Pi and OpenCode; `bash: ask` beside `'*': deny` adds Bash to the tool list on Claude and Pi; and `bash(git *): ask` beside `bash: deny` plus `bash(git status *): allow` has no correct settlement. A project-wide `task` ask would also remove Meridian's platform `task` deny, which `launch/permissions.py` drops whenever the rules allow or ask for `task`, whatever their origin; when project-wide `ask` arrives, only `profile` and `overlay` rules may do that. The motivating case, a personal `pkill -f` and `killall` deny that holds for every agent, is a deny. Project-wide `allow` and `ask` wait for the precedence change, and `allow`, `ask` and `'*'` fail the load until then.

## Mars settles what the keys can prove

So that a project-wide deny holds now, before the precedence change, Mars's `effective_tools` (the one function behind the launch bundle, native files and the plugin build) sets aside each agent allow or ask that a project-wide deny provably covers; the bundle lists them in `tools.set_aside` with the covering deny, and each rule carries `origin` (`profile`, `overlay`, `project`).

"Covers" (`tool_cover::relation` in mars-schema) must never guess: a wrong answer drops an allow that should apply. It proves cover only for an unscoped key over its capability and the capabilities it covers (`edit` over `write` and `notebook`, `web` over `web_search` and `web_fetch`), an MCP server over its tools, and a Bash literal ending in one `*` over patterns starting with that literal. Any other `*`, any `?` (a wildcard only on OpenCode), and any two different path patterns are unknown: Pi and Claude anchor and normalize paths differently, and Claude's `*` doesn't cross `/`. An unknown pair keeps both rules, and `mars sync` warns once per project, except when `.claude` is the only target, since Claude checks every deny first and Meridian refuses other harnesses there. That unknown pair, on OpenCode, Pi and Codex, is the first release's one hole.

A covered allow is not a lost restriction, so `mars sync` says nothing about it; `mars validate` lists it at `info`. A project-wide deny a target's native files can't carry is a lost restriction and prints by default.

## Continue and fork pick up the denies in force now

A continue replays the launch snapshot ([continue and fork](../launch/continue-and-fork.md)), but a deny added after a session started must hold when it resumes. On replay Meridian asks Mars for the current bundle, replaces the snapshot's `project`-origin rules with the current ones, and drops snapshot rules the current bundle sets aside. A project deny only restricts, so the join can't widen a session. Mars alone decides what a deny covers; Meridian only applies its list. Consequences, each user-visible:

- Continue and fork need a working Mars, and fail without one. A fallback that kept the snapshot's project rules was removed, because the continue path's own agentless Mars call already fails first.
- When the agent no longer resolves, a continue applies the project-wide rules alone, with a warning; a fork fails.
- Removing a deny lifts it on the next continue, but allows set aside at launch don't return: the snapshot never held them.

**Known limit.** `tools.set_aside` is computed from the agent's current profile and overlay, not from the snapshot. A snapshot's scoped allow that the profile no longer has, or that belonged to an agent that no longer resolves, stays beside a newly added covering deny and can beat it on Codex, OpenCode and Pi. Claude checks the deny first. The precedence change ends it; a smaller fix would keep a replayed allow or ask only if the current bundle still carries it.

## An older Mars ignores these keys silently

A Mars before 0.29.0 ignores `[settings.tools]` and map entries in `[agents.<name>.tools]` without a word, so the deny does nothing there. `min_mars_version` is no guard: only `mars validate` checks it, not `mars sync` or `mars build launch-bundle` ([mars-agents #343](https://github.com/haowjy/mars-agents/issues/343)). Mars doesn't warn when `[settings.tools]` sits beside an unset or old `min_mars_version`: the warning would point users at a check `mars sync` doesn't run.

## Pattern matching, not precedence, is the weak point

`bash(pkill -f *)` matches a command starting with `pkill -f`; `pkill -9 -f x`, `sudo pkill -f x` and `sh -c 'pkill -f x'` don't match on Pi or Codex, and Claude unwraps only some wrappers. A broader key such as `bash(pkill *)` covers more. These rules stop accidents ([Codex shell rules](tool-permissions.md#codex-shell-rules-are-an-accident-guard)).

On Claude under `approval: auto` or `never`, an unscoped project-wide `bash` deny removes the tool and every `Bash(meridian ...)` grant with it, so the agent loses `meridian spawn done` and `spawn rearm`; the dry run reports it, as for a profile `bash: deny`. Mars doesn't reject or warn: whether the gap applies depends on harness and approval, both per launch, and Mars doesn't know Meridian's commands. A Claude `PreToolUse` hook letting completion commands through ahead of the deny would close it for profiles too.
