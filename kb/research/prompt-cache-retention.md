# Prompt-Cache Retention by Harness

How long each harness's provider keeps the prompt cache warm while a session
sits idle. Checked 2026-10-08 against Codex CLI `rust-v0.161.0`, OpenCode
`v1.18.34`, Pi 1.1.0 and the OpenAI and Anthropic prompt-caching docs. Pi's
`openai-codex`, `xai` and `deepseek` providers, and the ChatGPT backend, were
checked 2026-10-13; the backend's idle window was probed 2026-10-14.
Provenance: `work:idle-cache-notify` (`spawn:p7423`) and
`work:idle-ttl-detection` (`spawn:p7724`, `spawn:p7824`).

**Cache policy depends on authentication and provider, not a universal idle
TTL.** Project-provided evidence from the owner on 2026-10-20 records indefinite retention
and cache hits on resumption for signed-in Codex and Work. API-key use is
separate: supported Responses API models offer a 30-minute default or a
12-hour option. Neither is the rationale for Meridian's child-completion
window or caller wait checkpoint.

Meridian's fixed 1800-second signed-in Codex idle warning is a known divergence,
tracked separately in [#707](https://github.com/haowjy/meridian-cli/issues/707).
The completion handshake does not change idle-warning policy. Do not infer
third-party clients' guarantees merely from their sharing an authentication
endpoint; their applicability still needs checking.


| Harness | Idle retention by default | Can the user extend it? | Source |
|---|---|---|---|
| Claude Code | 1 h or 5 min, chosen per request bucket (main conversation vs. everything else). Each transcript cache write records `ephemeral_1h_input_tokens` or `ephemeral_5m_input_tokens`, so the TTL can be read from the transcript instead of assumed. | `CLAUDE_CODE_PROMPT_CACHE_TTL` (env, Claude Code 2.1.242+) or the `promptCacheTtl` setting chooses the main-conversation TTL; `claude -p` counts as a main conversation, 1 h by default on subscription usage. Built-in subagents use `CLAUDE_CODE_SUBAGENT_PROMPT_CACHE_TTL`. | Claude Code transcripts; planning session `chat:c7275`; [Claude Code prompt caching](https://code.claude.com/docs/en/prompt-caching#choose-the-ttl-yourself) |
| Pi | **5 min** (short retention) on Anthropic. Other providers differ; see [Pi providers](#pi-providers). | `PI_CACHE_RETENTION=long` → Anthropic `cache_control.ttl: "1h"`, OpenAI `prompt_cache_retention: "24h"`. Not on the `openai-codex` provider: Pi 1.1.0's `openai-codex-responses.js` sends only `prompt_cache_key` and ignores the variable. | Pi `docs/environment-variables.md`; `packages/ai/test/cache-retention.test.ts`; installed Pi 1.1.0 (`chat:c7347`) |
| Codex CLI | Signed-in Codex: indefinite retention on resumption per the owner-supplied OpenAI statement. API-key sessions follow the API contract. | No Codex retention setting was found in the inspected `rust-v0.161.0` schema; API capability is not proof of a CLI setting. | `work:child-completion` K9; inspected Codex source |
| OpenCode | Anthropic: **5 min**. It sends `cache_control: {type: "ephemeral"}` without a `ttl` (`packages/llm/src/cache-policy.ts:40-41`). OpenAI-compatible routes get no inline cache hints and rely on provider defaults. | No built-in config key. The internal `CachePolicy.ttlSeconds` needs caller or plugin code. | `anomalyco/opencode` at `v1.18.34` |

## OpenAI retention, which Codex inherits

This is the **public OpenAI API** contract (API-key auth):

| Models | Default | Option |
|---|---|---|
| GPT-5.6 and later | `prompt_cache_options.ttl: "30m"` by default | `"12h"` in the Responses API; these are minimum lifetimes, not deletion deadlines |
| Earlier models (`gpt-5.5`, `gpt-5.4`, `gpt-5.1-codex*`, `gpt-5-codex`, `gpt-4.1`, …) | Depends on the organization: `24h` without Zero Data Retention, `in_memory` (about 5–10 min idle, up to 1 h) with ZDR | `prompt_cache_retention: "24h"`: usually about 30 min, up to 24 h; no separate cache-write charge |

API lifetime options rechecked 2026-10-21: [OpenAI prompt-caching guide, cache lifetime](https://developers.openai.com/api/docs/guides/prompt-caching#cache-lifetime).

### ChatGPT backend (Codex and Pi's `openai-codex`)

ChatGPT-auth Codex sessions and Pi's `openai-codex` provider call
`https://chatgpt.com/backend-api`, not the public API. OpenCode's `openai`
provider reaches the same backend when it authenticates with a ChatGPT account
(OAuth), as it does on this machine. **The 30-minute GPT-5.6 TTL above is not
established for this endpoint.**

- The backend caches automatically. Cache affinity comes from the Responses
  `session-id` header and `prompt_cache_key` (OpenAI Codex `codex-rs/core/src/client.rs`).
- It rejects the public API's cache controls. The Codex issue tracker records
  `Unknown parameter: 'prompt_cache_retention'` on
  `chatgpt.com/backend-api/codex/responses/compact`
  ([openai/codex#17809](https://github.com/openai/codex/issues/17809)), and
  `prompt_cache_options` is reported unsupported as well.
- The owner-supplied OpenAI statement establishes indefinite retention for
  signed-in Codex and Work. It does not by itself establish the same contract
  for every third-party client using this endpoint.
- Pi's 5-minute WebSocket idle expiry on this provider is connection pooling,
  not cache lifetime.

The former 1800-second estimate must not be used as signed-in Codex's cache
contract. Earlier third-party-client probes did not establish a TTL: the Pi
run had unexplained misses, and the OpenCode run reused prefixes between
sessions, confounding idle duration. They do not override the signed-in
product statement or settle its applicability to Pi and OpenCode.

#### Measured idle window (2026-10-14)

Historical probes are recoverable through `spawn:p7824` and
`work:idle-ttl-detection`. A new third-party-client measurement needs distinct
prefixes or serial runs so one request cannot refresh another gap's entry.
The owner supplied [this discussion of the OpenAI statement](https://www.reddit.com/r/codex/comments/1xb9prz/caching_upgraded_massively/)
in `chat:c8468` (entry 29); the accepted project interpretation is recorded in
`work:child-completion`, `decisions.md` K9. The signed-in product statement was
not independently reverified during this capture; the API options above were. Meridian's operational choice is
[the completion decision](../decisions/completion-drain.md).

## Anthropic retention, which OpenCode and Pi inherit

The default `ephemeral` cache lives 5 minutes. `ttl: "1h"` extends it to an
hour at a higher cache-write price
([Anthropic prompt-caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#what-is-the-cache-lifetime)).

## Pi providers

Pi routes each provider through a shared adapter, so `PI_CACHE_RETENTION=long`
can change the outgoing payload without changing the provider's cache. In the
table, "unknown" means the provider publishes no usable idle TTL. It does not
mean the provider has no cache.

| Provider (model seen) | Caching | Documented lifetime | Effect of `PI_CACHE_RETENTION=long` |
|---|---|---|---|
| `anthropic` | `cache_control` | 5 min, or 1 h with `ttl: "1h"` | Sends `ttl: "1h"` |
| `openai` | Automatic prefix cache | See [OpenAI retention](#openai-retention-which-codex-inherits) | Sends `prompt_cache_retention: "24h"` (usually about 30 min) |
| `openai-codex` (`gpt-6-astra`) | Automatic, ChatGPT backend; Pi sends only `prompt_cache_key` plus affinity headers | Unknown ([ChatGPT backend](#chatgpt-backend-codex-and-pis-openai-codex)) | None; the Codex adapter never reads the variable |
| `xai` (`grok-4.7`) | Automatic prefix cache, through Pi's OpenAI Responses adapter; `prompt_cache_key` from the session ID | **Best effort, no TTL.** Entries "can be evicted at any time due to server load or restarts" ([xAI prompt-caching FAQ, 2026-05-10](https://docs.x.ai/developers/advanced-api-usage/prompt-caching/best-practices)) | Pi adds `prompt_cache_retention: "24h"`, which xAI does not document; no evidence of a 24 h xAI cache |
| `deepseek` (`deepseek-flash`) | Automatic disk cache, on for everyone, through Pi's OpenAI Chat Completions adapter | **Best effort, usually a few hours to a few days**, then cleared ([DeepSeek context caching](https://api-docs.deepseek.com/guides/kv_cache/)) | Pi adds `prompt_cache_key` and `prompt_cache_retention: "24h"`; DeepSeek lists both as "Not supported" on its Responses API, so no DeepSeek TTL follows |

Pi sources: [`providers/xai.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/providers/xai.ts),
[`providers/deepseek.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/providers/deepseek.ts),
[`providers/openai-codex.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/providers/openai-codex.ts),
[`api/openai-responses.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/openai-responses.ts),
[`api/openai-completions.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/openai-completions.ts),
[`api/openai-codex-responses.ts`](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/openai-codex-responses.ts).

## What Meridian does with this

- **Idle notifications.** The existing implementation still uses Codex
  `ttl_seconds = 1800` (estimated), the signed-in policy divergence tracked by #707. OpenCode 2.x maps the session's provider:
  `openai` → 1800, estimated, everything else unknown (PR #573, released in
  v0.9.9; 0.9.8 and earlier default OpenCode to 300, push only). Claude reads the TTL from
  the transcript in the chat's recorded store. Pi maps provider and
  `PI_CACHE_RETENTION` to a TTL: Anthropic 300 or 3600 (known); OpenAI long
  retention 1800, estimated, because the `24h` label is not a lifetime;
  `openai-codex` 1800, estimated, to match the Codex harness on the same
  backend. `xai` and `deepseek` stay unknown, so they get push only. See
  [D-idle-ttl-detection](../decisions/idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)
  and [OpenCode provider TTL](../decisions/idle-ttl-detection.md#opencode-provider-ttl).
- **Pi primaries get long retention.** Meridian sets `PI_CACHE_RETENTION=long`
  for interactive Pi primaries unless the user set it; spawns keep Pi's
  5-minute default. That buys a one-hour Anthropic cache at the higher write
  price. See
  [D-pi-cache-retention](../decisions/idle-notifications.md#d-pi-cache-retention--meridian-launches-pi-primaries-with-long-cache-retention).
- **Claude spawns get short retention.** Meridian forces
  `CLAUDE_CODE_PROMPT_CACHE_TTL=5m` on non-interactive Claude spawns; primaries
  keep Claude's own setting. That avoids the one-hour write price for
  short-lived agents, at the cost of a cold cache after a wait longer than five
  minutes. See
  [Harness adapters](../codebase/harness-adapters.md#claude-launch-env).
- **Spawn-wait yield.** The 3000-second default is an operational caller
  checkpoint, not a cache-lifetime model. Child completion inherits the calling
  harness's setting by default but uses an independent deadline. See
  [Spawn Wait Barrier](../concepts/spawn-wait-barrier.md).
- **When idle compaction pays.** The write premiums above set the cold
  re-read cost w in the compaction break-even, and whether the compaction
  request itself reads the cache sets c (below). Meridian used this for its own
  idle compaction until it was deleted
  ([D-idle-compaction-removed](../decisions/idle-notifications.md#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions));
  the superseded break-even is
  [D-idle-compaction-growth-floor](../decisions/idle-compaction-growth-floor.md#d-idle-compaction-growth-floor--compact-on-growth-past-a-per-harness-break-even).

## Does the compaction request read the cache?

A compaction is cheap only when its request shares the session's cached prefix.
That depends on how the harness builds the request, not on the provider.

| Harness | Compaction request | Measured cache read |
|---|---|---|
| Claude Code | normal request plus an appended instruction | warm: ≈ `0.1·C + 20k` uncached (Haiku probes; 19.3k uncached on a real primary) |
| Codex | V2 remote compaction appends `CompactionTrigger` to the normal prefix | warm: c ≈ 0.114 (n = 160 local rollouts) |
| OpenCode 2.x | session system prompt, tool snapshot and older native messages (all but the newest 15,000 tokens), then the summary instruction | warm: 93–99.8% of input cached (11 rows with usage, `openai` and `opencode-go`) |
| OpenCode 1.x | compaction agent, empty system prompt, no tools, history flattened into one user message | cold: about 0% on every provider (109 rows, 2026-02 → 2026-10) |
| Pi (1.2.0) | own `SUMMARIZATION_SYSTEM_PROMPT`, no tools, conversation in one user message, `cacheRetention: "none"`, fresh routing id | cold: c = 1 on `openai-codex` |

OpenCode 1.x and 2.x store compactions in different tables (`message` versus
`session_message`). Mixing them is how the 1.x cold sample was once read as
2.x. Pi's fix is upstream: earendil-works/pi#10577 adds an opt-in in-context
compaction for verified models (tracked in haowjy/meridian-cli#593). Source:
`work:harness-compaction-cache` (`evidence/opencode-cache-by-provider.md`,
`research/opencode.md`, `research/pi.md`).

## Pi cache warming

Pi 0.86.0 added cost-aware prompt-cache warming (`cacheWarming`: `off`,
`streaming` (the default), or `idle`). A refresh replays the last request with
`maxTokens: 1`. It does not conflict with Meridian's idle timeline today:

- With the default `streaming`, warming stops when the agent run settles.
- Pi warms only models whose catalog declares a `promptCache` lifetime. In Pi
  1.2.0 the OpenAI and `openai-codex` entries declare none, so they are never
  warmed. Anthropic declares 300 s and 3600 s.
- With `idle`, refreshes are scheduled at 90% of the TTL but stop at a fixed
  30-minute idle horizon. Under the `PI_CACHE_RETENTION=long` Meridian sets,
  the first refresh would fall at 54 min, so it never fires.
- A refresh emits none of the events Meridian's extension listens to.

Only a short-retention Anthropic Pi session with `cacheWarming: idle` could
get warmed past Meridian's schedule, which would make its warning premature.
If Pi starts declaring OpenAI lifetimes, or a user adds `modelOverrides.promptCache`,
re-check this. The lightest fix would be for the Pi extension to pass the
effective warming decision into `idle arm`. Source:
`work:harness-compaction-cache` `research/pi-cache-warming.md`.

Recheck this page when Codex, OpenCode or Pi releases a new version, when a
provider changes its cache-lifetime docs, or when signed-in product guarantees
are clarified for third-party clients.
