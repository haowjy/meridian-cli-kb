# Prompt-Cache Retention by Harness

How long each harness's provider keeps the prompt cache warm while a session
sits idle. Checked 2026-10-08 against Codex CLI `rust-v0.161.0`, OpenCode
`v1.18.34`, Pi 1.1.0 and the OpenAI and Anthropic prompt-caching docs.
Provenance: `work:idle-cache-notify`, `spawn:p7423`.

**The common assumption that every harness keeps about an hour is wrong.**
By default only Claude Code sessions can be assumed to get 1 h, and even there
the TTL varies per session. Pi and OpenCode default to 5 minutes, and Codex on
GPT-5.6+ gets 30 minutes.

| Harness | Idle retention by default | Can the user extend it? | Source |
|---|---|---|---|
| Claude Code | 1 h or 5 min, per session. Each transcript cache write records `ephemeral_1h` or `ephemeral_5m`, so the TTL can be read from the transcript instead of assumed. | Claude Code settings; not verified here | Claude Code transcripts; planning session `chat:c7275` |
| Pi | **5 min** (short retention) | `PI_CACHE_RETENTION=long` → Anthropic `cache_control.ttl: "1h"`, OpenAI `prompt_cache_retention: "24h"`. Not on the `openai-codex` provider: Pi 1.1.0's `openai-codex-responses.js` sends only `prompt_cache_key` and ignores the variable. | Pi `docs/environment-variables.md`; `packages/ai/test/cache-retention.test.ts`; installed Pi 1.1.0 (`chat:c7347`) |
| Codex CLI | Whatever OpenAI's default is for the model. Codex sends only `prompt_cache_key`, with no retention field (`codex-rs/core/client.rs:946-963`). | **No.** No config key in `config.schema.json` | `openai/codex` at `rust-v0.161.0` |
| OpenCode | Anthropic: **5 min**. It sends `cache_control: {type: "ephemeral"}` without a `ttl` (`packages/llm/src/cache-policy.ts:40-41`). OpenAI-compatible routes get no inline cache hints and rely on provider defaults. | No built-in config key. The internal `CachePolicy.ttlSeconds` needs caller or plugin code. | `anomalyco/opencode` at `v1.18.34` |

## OpenAI retention, which Codex inherits

| Models | Default | Option |
|---|---|---|
| GPT-5.6 and later | `prompt_cache_options.ttl: "30m"`: the only value and the default; entries stay eligible at least 30 min after a write or reuse | none |
| Earlier models (`gpt-5.5`, `gpt-5.4`, `gpt-5.1-codex*`, `gpt-5-codex`, `gpt-4.1`, …) | Depends on the organization: `24h` without Zero Data Retention, `in_memory` (about 5–10 min idle, up to 1 h) with ZDR | `prompt_cache_retention: "24h"`: usually about 30 min, up to 24 h; no separate cache-write charge |

Source: [OpenAI prompt-caching guide, cache lifetime](https://developers.openai.com/api/docs/guides/prompt-caching#cache-lifetime).

**Unconfirmed:** whether ChatGPT-auth and API-key Codex sessions get different
retention. The CLI builds the same request for both, so any difference would
be server-side policy.

## Anthropic retention, which OpenCode and Pi inherit

The default `ephemeral` cache lives 5 minutes. `ttl: "1h"` extends it to an
hour at a higher cache-write price
([Anthropic prompt-caching docs](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#what-is-the-cache-lifetime)).

## What Meridian does with this

- **Idle notifications.** These facts set the defaults: Codex
  `ttl_seconds = 1800`, OpenCode `300`. Claude reads the TTL from the
  transcript, and Pi's extension derives it from the provider and
  `PI_CACHE_RETENTION`. See
  [Idle Notifications](../decisions/idle-notifications.md#d-idle-config--separate-notify--idle-namespaces-standard-precedence).
- **Pi primaries get long retention.** Meridian sets `PI_CACHE_RETENTION=long`
  for interactive Pi primaries unless the user set it; spawns keep Pi's
  5-minute default. That buys a one-hour Anthropic cache at the higher write
  price. See
  [D-pi-cache-retention](../decisions/idle-notifications.md#d-pi-cache-retention--meridian-launches-pi-primaries-with-long-cache-retention).
- **Spawn-wait yield.** The unified 3000-second yield default assumes about an
  hour of cache for every harness. These facts contradict that assumption for
  Codex on GPT-5.6+, OpenCode, Pi spawns, and Pi primaries on short retention
  or the `openai-codex` provider. See
  [Spawn Wait Barrier](../concepts/spawn-wait-barrier.md#harness-aware-yield-defaults).

Recheck this page when Codex or OpenCode releases a new version, or when a
provider changes its cache-lifetime docs.
