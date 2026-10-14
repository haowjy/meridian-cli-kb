# Decision: Idle TTL Detection

This decision covers transcript-store resolution and provider TTL mapping for
idle notifications: Claude, Pi, and OpenCode 2.x.

**Status:**

- Claude and Pi (D1–D5) shipped in v0.9.8 (PR #568, merge `ad4f98e6e`, release
  `da1250a0f`). Verified on the installed 0.9.8 with scratch state: a Claude
  session that had moved directories now detects `ttl=3600` exactly and gets
  push, warn and compact; on 0.9.7 the same session gave `ttl=None`, push only.
  Pi `openai-codex` gets 1800 s estimated and all stages; Pi `xai` gets push
  only.
- OpenCode provider TTL (D7) is implemented on `fix/opencode-idle-v2`, draft
  PR #573, not merged. Released builds still carry the 300 s OpenCode default.
- The user confirmed D5 and D7 (1800 s on the ChatGPT backend) on 2026-10-14.

## D-idle-ttl-detection — Detection reads launch records, never a cwd

**Problem it fixed:** idle compaction shipped in PR #550, then fired once in two
days. An unknown TTL makes `timeline.schedule` drop warn and compact, and
detection returned unknown on almost every real session:

- **Claude.** The mod sent `--cwd $.session.cwd()`, and `claude_idle` built
  `~/.claude/projects/<slug(cwd)>/<id>.jsonl` from it. `$.session.cwd()`
  follows every Bash `cd`. Claude files the transcript under the slug of the
  *launch* directory. Any session whose agent changed directory missed its own
  transcript, even though every assistant row recorded `ephemeral_1h` tokens.
- **Pi.** The detector knew only `anthropic` and `openai`. Real primaries use
  `openai-codex`, `xai` and `deepseek`. All ten sampled Pi idle state files had
  `ttl=None`.

**Decision (D1–D2, `chat:c7500`):** core resolves the chat's launch-recorded
`native_store`. `IdleService.arm` reads `MERIDIAN_CHAT_ID` from the session env.
It looks up that chat's record in the project runtime's `sessions.jsonl`, under
`MERIDIAN_PROJECT_DIR`. It then passes the record's absolute `native_store` to
the harness's `detect_ttl`. The `DetectIdleTtl` port carries `native_store`, not
a cwd (PR #573 adds the sensor-reported `provider`; see
[OpenCode provider TTL](#opencode-provider-ttl)). Claude resolves `<native_store>/<live session id>.jsonl` exactly, through
the same header-checked `resolve_claude_native_session_file` that resume uses
([Native Session Binding](../architecture/native-session-binding.md)). `--cwd` is
gone from `meridian idle arm`, the service, the port and both mods.

**Rule:** never pass, derive or fall back to a cwd for transcript lookup. An
agent can `cd`, but the transcript stays where the launch filed it. The rule is
the same as for resume: native resolution uses the recorded store. A miss
returns unknown, not a guess:

- no `MERIDIAN_CHAT_ID`;
- no record, a record for another harness, or an empty or relative store;
- a missing file, a header mismatch, or a bad ID.

A wrong TTL cannot happen, because the exact ID must exist in that store.

**Consequences:**

- After `/clear`, the live session ID is joined to the entry chat's store. This
  works because Claude files the successor under the same project-store parent.
  The regression test arms a successor ID against the entry record. If Claude
  ever files successors elsewhere, detection degrades to unknown, not to a wrong
  TTL.
- A Claude session started outside Meridian that opts into the mod through
  `CLAUDE_CODE_PLUGIN_DIRS` has no `MERIDIAN_CHAT_ID`. It now gets push only,
  unless `ttl_seconds` is configured.
- Every arm without a configured TTL parses `sessions.jsonl`: 0.09 s on 13 MB,
  0.19 s on 30 MB. That is accepted because arm runs off the TUI path. Revisit
  if session logs grow tenfold, or move the read onto the SQLite projection.
- **Sessions running across the upgrade lose idle until restarted.** Removing
  `--cwd` broke the mod-to-CLI call shape, and a running session keeps the mod
  it loaded at start. A 0.9.7 mod calling the 0.9.8 CLI gets
  `{"error":"Unknown option: --cwd."}`, exit 1, and no state is written. The
  Claude mod logs `arm failed` only to its own store, so push, warn and compact
  all stop without any visible error. The 0.9.7 Pi extension also sent `--cwd`
  unconditionally; it was not live-checked. This was accepted under "no
  backwards compatibility" and given to the user as an upgrade note. The general
  mechanism is in
  [Bundled mods outlive the CLI they call](../lessons/harness-integration.md#bundled-mods-outlive-the-cli-they-call).

**Pi provider table (D3–D5).** `pi_idle._PROVIDER_TTLS` maps the provider and
the retention mode to a TTL. No match means unknown.

| Provider | Short retention | Long retention | Why |
|---|---|---|---|
| `anthropic` | 300, known | 3600, known | Pi sends `cache_control.ttl`. |
| `openai` | unknown | 1800, estimated | Pi asks for `24h`, which usually lasts about 30 min. |
| `openai-codex` | 1800, estimated | 1800, estimated | Mirrors the Codex harness default (U4). Pi's Codex adapter ignores `PI_CACHE_RETENTION`. |
| `xai` | unknown | unknown | Best-effort cache that "can be evicted at any time"; no TTL to schedule against. |
| `deepseek` | unknown | unknown | Disk cache that lasts "a few hours to a few days"; there is no pre-expiry point. Compact for context pressure, not cache expiry. |

**D5 resolved:** the user confirmed 1800 s on 2026-10-14 ("GPT-5.6+, which we
usually use, is 30 minutes"). `estimated=true` stays because the ChatGPT backend
publishes no cache contract.

**Why 1800:** research (`spawn:p7724`) found no published TTL for the ChatGPT
backend, and refuted mapping the public API's GPT-5.6 30-minute TTL onto it as
a documented fact
([research](../research/prompt-cache-retention.md#chatgpt-backend-codex-and-pis-openai-codex)).
The Codex harness already runs on the same backend with the 1800 estimate, so
Pi matches it rather than diverging, and `estimated=true` puts `est.` on its
notices. **Risk:** if the backend cache lives under 25 minutes, the compaction
lands on a cold cache. Local Pi session logs fit the estimate: after a 20–30 min
idle gap, 28 of 31 `openai-codex` turns still read the cache (90%), and 78 of 84
after 10–20 min. A controlled probe (`spawn:p7824`) saw Pi `openai-codex` hits
at 25 and 35 min, so compaction at about 25 min landed warm, but it is not a
clean boundary
([measurement](../research/prompt-cache-retention.md#measured-idle-window-2026-10-14)).

**Known limitation: TTL keyed by provider, not model.** Both tables, Pi's
`pi_idle._PROVIDER_TTLS` and OpenCode's `opencode_idle._PROVIDER_TTLS` (read by
`detect_ttl`), resolve the TTL from the provider ID alone. The OpenAI estimates
(Pi `openai-codex`, OpenCode `openai`) assume GPT-5.6 or later. An older model
on those routes inherits the 1800 s estimate although its cache may be shorter;
set `ttl_seconds` for it, since a configured TTL always beats detection. The
user docs say the same (`docs/configuration.md` idle section).

**Rejected:**

- **The session's current cwd** (the shipped behaviour): it follows the agent's
  `cd`, so the lookup misses once the session moves. The user chose "where it
  spawned from".
- **Treating Pi `xai` or `deepseek` as known or estimated:** neither provider
  publishes an idle TTL. A guessed TTL would make an estimate look like a
  provider fact.
- **Leaving Pi `openai-codex` unknown**, which the research recommended: it
  would make Pi on the ChatGPT backend the one route where warn and compact
  never fire, while the Codex harness on the same backend schedules both.

## OpenCode provider TTL

**Status:** draft PR #573, not merged (`work:idle-ttl-detection` D7, phase 2).

**Problem:** fixing the OpenCode 2.x sensor
([D-opencode-idle-v2](idle-notifications.md#d-opencode-idle-v2--opencode-2x-gets-its-own-sensor-chosen-by-the-resolved-transport))
would have restored only the push. OpenCode had a configured default of
`ttl_seconds = 300`. Warn sits at TTL − 15 min and compact at TTL − 5 min, so
neither fits in 300 s. And because a configured TTL wins over detection in
`IdleService.arm`, a default meant no detector would ever be asked. The 300 s
reflected OpenCode's hard-coded 5-minute Anthropic cache, but the OpenCode
primaries actually run here use an OpenAI model (`gpt-6-astra`).

**Decision:**

- The 2.x sensor reads the provider from `GET /api/session/{id}`
  (`data.model.providerID`) at each turn end and sets `IdleEvent.provider`.
  `IdleService.event` passes it through `arm` to the harness's `detect_ttl`.
  Core never interprets the provider name.
- `opencode_idle._PROVIDER_TTLS` maps `openai` → 1800 s, estimated. Every
  other provider, a failed provider read, and every 1.x session (V1 never
  reports a provider) resolve to unknown: push only.
- The OpenCode `ttl_seconds` default is removed. A configured
  `[harness.opencode.idle] ttl_seconds` still beats detection, and `meridian
  config show` prints `null` when none is set.
- The table stays in `opencode_idle.py`, not shared with Pi: Pi's is keyed by
  retention mode as well as provider.

**D7 resolved:** the user confirmed 1800 s on 2026-10-14, with D5.
`estimated=true` stays because the ChatGPT backend publishes no cache contract.
The provider-not-model limitation under the
[Pi provider table](#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)
applies here too.

**Why 1800:** the same reasoning as D5. On this machine OpenCode's `openai`
provider authenticates with a ChatGPT account (the provider rejected
`gpt-5.3-codex-spark` with "not supported when using Codex with a ChatGPT
account"), so it reaches the ChatGPT backend that the Codex harness and Pi's
`openai-codex` use, where no TTL is published. On the public API, GPT-5.6+
publishes 30 minutes. Either way 1800 s, marked `est.`, keeps all three
harnesses on that backend scheduling the same stages. If D5 is overturned, D7
goes with it. The `spawn:p7824` probe bounds nothing on this route: its
sessions shared one cache entry
([measurement](../research/prompt-cache-retention.md#measured-idle-window-2026-10-14)).

**Not mapped:** OpenCode `anthropic` sessions have a known 5-minute cache, but
only the push fits in 5 minutes, so mapping them changes nothing but the
notice countdown. `deepseek`, `opencode-go` and other routes follow D4: no
published TTL, no guess.

**Verified live** on the branch with opencode 2.0.28 (2026-10-14):
`openai/gpt-6-luna` → `ttl=1800 est.` with push, warn and compact scheduled;
`opencode-go/deepseek-v4.1-flash` → push only.

**Rejected:**

- **Keeping the 300 s default:** it blocks detection and schedules nothing past
  the push.
- **Reading the provider from the session DB** (the plan, through
  `opencode_snapshot`): the built version asks the server the sensor already
  talks to, at the moment the turn ends.


## Related

- [Idle Notifications](idle-notifications.md): the schedule and guards that
  consume the detected TTL.
- [Idle Adapter Architecture](../architecture/idle-adapters.md): the current
  adapter contracts and per-harness coverage.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md):
  the provider facts behind the Pi table.
- [Native Session Binding](../architecture/native-session-binding.md): the
  recorded-store resolution rule this decision reuses.

## Sources

- `work:idle-ttl-detection` (`chat:c7500`, branch `fix/idle-ttl-detection`):
  before-state evidence and D1–D5 in `decisions.md`; Pi provider research in
  `research/pi-provider-cache-ttl.md` (`spawn:p7724`); review in
  `reviews/review-1.md` (`spawn:p7725`); probe in
  `evidence/probe-ttl-detection.md` (`spawn:p7726`); D5 cache-hit analysis in
  `evidence/openai-codex-observed-cache.md`.
- Post-merge (`chat:c7524`): `decisions.md` "2026-10-14 post-merge" and
  "Outcome"; installed-build replays in
  `evidence/post-upgrade-20261013-215646-meridian-0.9.7/` and
  `evidence/post-upgrade-20261013-215801-meridian-0.9.8/`.
- Phase 2 (branch `fix/opencode-idle-v2`, PR #573): D7 in `decisions.md`
  "2026-10-14 phase 2"; implementation `9f99c041e` (`spawn:p7772`); live checks
  `evidence/coder-opencode-ttl-live.md` and
  `evidence/probe-opencode-idle-gate.md` (`spawn:p7788`); review
  `reviews/review-3.md` (`spawn:p7787`).
- D5/D7 confirmation and cache measurement: `decisions.md` "2026-10-14 D5/D7
  evidence check" and "2026-10-14 p7824 result"; probe report
  `evidence/probe-openai-cache-ttl.md` (`spawn:p7824`), raw records in
  `evidence/openai-cache-ttl/`.
