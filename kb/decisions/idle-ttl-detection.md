# Decision: Idle TTL Detection

This decision covers transcript-store resolution and provider TTL mapping for idle notifications.

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
a cwd. Claude resolves `<native_store>/<live session id>.jsonl` exactly, through
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

**Pi provider table (D3–D5).** `pi_idle._PROVIDER_TTLS` maps the provider and
the retention mode to a TTL. No match means unknown.

| Provider | Short retention | Long retention | Why |
|---|---|---|---|
| `anthropic` | 300, known | 3600, known | Pi sends `cache_control.ttl`. |
| `openai` | unknown | 1800, estimated | Pi asks for `24h`, which usually lasts about 30 min. |
| `openai-codex` | 1800, estimated | 1800, estimated | Mirrors the Codex harness default (U4). Pi's Codex adapter ignores `PI_CACHE_RETENTION`. |
| `xai` | unknown | unknown | Best-effort cache that "can be evicted at any time"; no TTL to schedule against. |
| `deepseek` | unknown | unknown | Disk cache that lasts "a few hours to a few days"; there is no pre-expiry point. Compact for context pressure, not cache expiry. |

> [!FLAG] **Needs human review**: The `openai-codex` 1800-second TTL is a
> tech-lead estimate, not a provider-confirmed lifetime. Flagged 2026-10-13.

The `openai-codex` row is a tech-lead call (D5), not yet confirmed. Research
(`spawn:p7724`) found no published TTL for the ChatGPT
backend. It also refuted mapping the public API's GPT-5.6 30-minute TTL onto that
backend ([research](../research/prompt-cache-retention.md#chatgpt-backend-codex-and-pis-openai-codex)).
The Codex harness already runs on the same backend with the 1800 estimate. So
Pi should match it rather than diverge, and `estimated=true` puts `est.` on its
notices. **Risk:** if the backend cache lives under 25 minutes, the compaction
lands on a cold cache. Revisit if observed cache-hit data disagrees.

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


## Related

- [Idle Notifications](idle-notifications.md): the schedule, guards and adapter
  contracts that consume the detected TTL.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md):
  the provider facts behind the Pi table.
- [Native Session Binding](../architecture/native-session-binding.md): the
  recorded-store resolution rule this decision reuses.

## Sources

- `work:idle-ttl-detection` (`chat:c7500`, branch `fix/idle-ttl-detection`):
  before-state evidence and D1–D5 in `decisions.md`; Pi provider research in
  `research/pi-provider-cache-ttl.md` (`spawn:p7724`); review in
  `reviews/review-1.md` (`spawn:p7725`); probe in
  `evidence/probe-ttl-detection.md` (`spawn:p7726`).
