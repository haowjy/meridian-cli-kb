# Claude

Claude sessions share a store for a config root. Selecting the newest file or a similar directory slug can select another concurrent conversation; Meridian uses the recorded store and exact ID ([native session binding](../state/native-session-binding.md)). Messaging a Claude primary goes through its cross-session inbox ([messaging](../spawns/messaging.md#claude-primaries-use-the-session-inbox)).

## Reading transcripts

Tracked reads use the recorded store and ID, independent of the caller's working directory. A missing store is unbound rather than a reason to search ambient roots.

## TUI trampoline

A `/tui fullscreen` successor inferred from prompt history is diagnostic only. A nearby unrelated chat can satisfy the same prompt check, so it cannot prove ownership or replace the entry binding. Resuming Claude in print mode to probe it was rejected because it performs model work and is not deterministic.

## Launch-owned SessionStart journal and current chat

Claude's launch-owned observations distinguish the entry session from later TUI switches. A switch updates the current chat without rebinding the entry; a later continuation selects that current chat and checks its exact transcript. An observed session may not have a readable transcript yet. A disabled hook, full journal or killed launcher can leave a switch unobserved; the last accepted observation remains authoritative. See the source checkout's `src/meridian/lib/launch/native_run.py`.

The journal reader accepts only the launch nonce and child PID, preventing a nested Claude from publishing into its parent's run. It advances its cursor after each durable observation: advancing the entire batch first could lose a switch, while replaying already published observations could transiently rewind the current chat.

## The spawn's prompt-cache TTL follows its profile's `mode`

Meridian sets `CLAUDE_CODE_PROMPT_CACHE_TTL=5m` for a non-interactive spawn whose profile is `mode: subagent`, or that resolved no profile. Claude counts `claude -p` as a main conversation, so on a subscription a short-lived agent would otherwise write the pricier one-hour cache; no quota saving has been measured, and the price is a rebuild for a child that waits past five minutes.

A `mode: primary` spawn gets no override and keeps whatever its environment or Claude resolves, an hour in subscription usage. These roles orchestrate for hours and idle past five minutes waiting on children, and a wake after the TTL re-caches the whole context: one 2.4-hour orchestrator spawn rebuilt it 13 times, 3.25M tokens. The TTL keys on `mode` rather than a second profile field because `mode` already marks the long-lived roles ([mode as lifetime](../launch/launch-system.md#mode-carries-lifetime-not-only-skill-listing)). Keepwarm pings can't help a spawned orchestrator either: idle automation runs only for primaries ([idle notifications](../spawns/idle-notifications.md#primaries-only)), so the cache lifetime is the only lever there.

Claude reapplies its `settings.json` environment after launch, so an entry there decides the value in either case. Settings and precedence: meridian-cli `docs/harness-integration.md#claude-prompt-caching`; provider lifetimes: [retention research](../research/prompt-cache-retention.md).

## No profile field reaches a native Claude subagent

Claude launches its own subagents, under `CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL`. A profile that Mars emits as a native agent never passes through a Meridian launch ([native agent emission](../mars/package-management.md#targets)), so no profile field sets its cache lifetime: a long-lived native subagent keeps the five-minute default and rebuilds after an idle gap. Routing the role through Meridian is what makes a launch-time policy reach it.
