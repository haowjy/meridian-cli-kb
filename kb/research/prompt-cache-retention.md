# Prompt-Cache Retention by Harness

How long each harness's provider keeps the prompt cache warm while a session
sits idle. Checked 2026-10-08 against Codex CLI `rust-v0.161.0`, OpenCode
`v1.18.34`, Pi 1.1.0 and the OpenAI and Anthropic prompt-caching docs. Pi's
`openai-codex`, `xai` and `deepseek` providers, and the ChatGPT backend, were
checked 2026-10-13; the backend's idle window was probed 2026-10-14.
Provenance: `work:idle-cache-notify` (`spawn:p7423`) and
`work:idle-ttl-detection` (`spawn:p7724`, `spawn:p7824`).

**The common assumption that every harness keeps about an hour is wrong.**
By default only Claude Code sessions can be assumed to get 1 h, and even there
the TTL varies per session. Pi and OpenCode default to 5 minutes. Codex gets
30 minutes on GPT-5.6+ through the public API. Through the ChatGPT backend,
OpenAI publishes no TTL.

| Harness | Idle retention by default | Can the user extend it? | Source |
|---|---|---|---|
| Claude Code | 1 h or 5 min, per session. Each transcript cache write records `ephemeral_1h` or `ephemeral_5m`, so the TTL can be read from the transcript instead of assumed. | Claude Code settings; not verified here | Claude Code transcripts; planning session `chat:c7275` |
| Pi | **5 min** (short retention) on Anthropic. Other providers differ; see [Pi providers](#pi-providers). | `PI_CACHE_RETENTION=long` → Anthropic `cache_control.ttl: "1h"`, OpenAI `prompt_cache_retention: "24h"`. Not on the `openai-codex` provider: Pi 1.1.0's `openai-codex-responses.js` sends only `prompt_cache_key` and ignores the variable. | Pi `docs/environment-variables.md`; `packages/ai/test/cache-retention.test.ts`; installed Pi 1.1.0 (`chat:c7347`) |
| Codex CLI | Whatever OpenAI's default is for the model and endpoint. Codex sends only `prompt_cache_key`, with no retention field (`codex-rs/core/client.rs:946-963`). | **No.** No config key in `config.schema.json` | `openai/codex` at `rust-v0.161.0` |
| OpenCode | Anthropic: **5 min**. It sends `cache_control: {type: "ephemeral"}` without a `ttl` (`packages/llm/src/cache-policy.ts:40-41`). OpenAI-compatible routes get no inline cache hints and rely on provider defaults. | No built-in config key. The internal `CachePolicy.ttlSeconds` needs caller or plugin code. | `anomalyco/opencode` at `v1.18.34` |

## OpenAI retention, which Codex inherits

This is the **public OpenAI API** contract (API-key auth):

| Models | Default | Option |
|---|---|---|
| GPT-5.6 and later | `prompt_cache_options.ttl: "30m"`: the only value and the default; entries stay eligible at least 30 min after a write or reuse | none |
| Earlier models (`gpt-5.5`, `gpt-5.4`, `gpt-5.1-codex*`, `gpt-5-codex`, `gpt-4.1`, …) | Depends on the organization: `24h` without Zero Data Retention, `in_memory` (about 5–10 min idle, up to 1 h) with ZDR | `prompt_cache_retention: "24h"`: usually about 30 min, up to 24 h; no separate cache-write charge |

Source: [OpenAI prompt-caching guide, cache lifetime](https://developers.openai.com/api/docs/guides/prompt-caching#cache-lifetime).

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
- No official document publishes an idle lifetime for the backend's prefix
  cache. Observed reports of retention values are not a contract.
- Pi's 5-minute WebSocket idle expiry on this provider is connection pooling,
  not cache lifetime.

An earlier version of this page left "ChatGPT-auth versus API-key retention"
open, and the idle design treated GPT-5.6's 30 minutes as the Codex fact.
The endpoint split above refutes that mapping. Meridian's 1800 s for this
backend is an estimate, not a provider guarantee. The user confirmed it on
2026-10-14 for the GPT-5.6+ models in use here, and it stays marked estimated.

#### Measured idle window (2026-10-14)

`spawn:p7824` primed parallel sessions on both ChatGPT-backend routes, one
session per idle gap, all with the same 21.6 KB first prompt. After the gap it
sent one short resend per session and read the cached-token count on that
resend. A hit means about 1,000 or more cached tokens. Gaps: 5, 15, 25, 35, 45
and 60 min. The 5-min pair was first resent at about 2 min by mistake; those
results are excluded and the replacement pair ran at about 6 min.

| Gap | Pi `openai-codex` (`gpt-6-astra`), cached / total | OpenCode `openai` (`gpt-6-luna`), cached / total |
|---:|---|---|
| ~6 min | 5,504 / 5,679, hit | 10,752 / 11,624, hit |
| 15 min | 0 / 5,692, **miss** | 1,536 / 11,624, partial hit |
| 25 min | 5,504 / 5,685, hit | 10,752 / 11,624, hit |
| 35 min | 5,504 / 5,683, hit | 10,752 / 11,624, hit |
| 45 min | 0 / 5,683, **miss** | 10,752 / 11,624, hit |
| 60 min | 0 / 5,700, **miss** | 10,752 / 11,624, hit |

- **Pi `openai-codex`:** every prime read 0 cached tokens, so the sessions did
  not visibly share a cache entry. Hits at 6, 25 and 35 min, misses at 15, 45
  and 60. This fits the 1800 s estimate, and compaction at about 25 min would
  have landed warm in this run. It is not a boundary: the 15-min miss is
  unexplained (eviction or routing are guesses, not findings).
- **OpenCode `openai`:** confounded, so it bounds nothing. The primes
  themselves read 1,536–10,752 cached tokens: all six sessions hit one
  prefix-keyed cache entry, and each resend refreshed it for the next. The
  hits through 60 min measure time since the last touch by any session, which
  was 15 min or less. They are not a 60-minute window.
- **Outcome:** nothing contradicts 1800 s, and nothing confirms it as a
  provider value. Both routes keep `estimated=true`.

A clean re-measurement needs a *distinct* prefix for each gap's session, or
serial runs, so that no resend can refresh another gap's entry.

Evidence: `work:idle-ttl-detection` `evidence/probe-openai-cache-ttl.md`, raw
session exports and launch logs in `evidence/openai-cache-ttl/`. Earlier
observational data from real Pi sessions (90% hits after 20–30 min idle) is in
`evidence/openai-codex-observed-cache.md`.

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

- **Idle notifications.** These facts set the defaults: Codex
  `ttl_seconds = 1800` (estimated). OpenCode 2.x maps the session's provider:
  `openai` → 1800, estimated, everything else unknown (PR #573, unreleased;
  released builds still default OpenCode to 300, push only). Claude reads the TTL from
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
- **Spawn-wait yield.** The unified 3000-second yield default assumes about an
  hour of cache for every harness. These facts contradict that assumption for
  Codex (30 minutes on the public API, unpublished on the ChatGPT backend),
  OpenCode, Pi spawns, and Pi primaries on short retention or a non-Anthropic
  provider. See
  [Spawn Wait Barrier](../concepts/spawn-wait-barrier.md#harness-aware-yield-defaults).

Recheck this page when Codex, OpenCode or Pi releases a new version, when a
provider changes its cache-lifetime docs, or when OpenAI documents the ChatGPT
backend's cache.
