# Research: OpenCode Config and Prompt Semantics

Upstream OpenCode behavior that decides how a launcher can put a system prompt,
an agent and a model in front of the model. Verified live on OpenCode 2.0.29
and 1.19.3 against a local logging provider that recorded the actual provider
request, and read from the tagged sources (`packages/core/src/...` for v2,
`packages/opencode/src/...` for v1). Several of these contradict the config
schema and `--help`, so recheck them on a new major version before relying on
them. How Meridian uses them:
[D-opencode-config-agent](../decisions/opencode-prompt-delivery.md).

## Prompt channels

| Channel | 1.19.3 | 2.0.29 |
|---|---|---|
| `instructions` paths in config (inside or outside the project) | loaded on HTTP, TUI and `run` | **ignored.** The schema keeps the key, but the instruction loader discovers only global and ancestor `AGENTS.md` |
| Message `system` field | applied (`POST /session/{id}/prompt_async`) | no such field; the prompt body is `{text}` |
| Config agent prompt | `agent.<id>.prompt` | `agents.<id>.system`; a legacy `agent.<id>.prompt` is normalized to it |
| Ambient `AGENTS.md` | loaded | loaded |
| `CLAUDE.md` | not loaded | not loaded |

An agent prompt replaces only OpenCode's built-in base prompt. `AGENTS.md`,
the skill listing and the environment block still follow it on both versions.
OpenCode 1.0.29 does not expand `{file:}` inside `agent.<id>.prompt`; 1.19.3
does.

## `{file:}` and `{env:}` substitution

- Substitution runs on the **raw config text, before JSON parsing**, on every
  config document including `OPENCODE_CONFIG_CONTENT`. The file's content is
  trimmed and JSON-escaped in place.
- The token ends at the first `}`. Quotes and backslashes are not syntax: the
  path text is taken literally, so a path containing `}`, `"`, `\` or a control
  character cannot be addressed.
- JSON escapes are not decoded first. A non-ASCII path written as `ü` is
  looked up literally and fails; the config must be serialized as raw UTF-8.
- A missing file fails the **whole** config: `/api/config` and `/api/agent`
  return HTTP 500 with `ConfigInvalidError`.
- Single pass: `{file:}` or `{env:}` text inside the substituted file arrives
  untouched. An 80 KB file loads intact.
- v2 also expands `~/` in the path.

## Agent selection

- `default_agent` selects the agent only for a session with none. A v2 session
  stores its agent and keeps it on resume; a changed `default_agent` does not
  apply. v1 resolves `default_agent` for each message that names no agent.
- An explicit agent wins everywhere it can be passed: v2 session create
  (`agent`), the v2 switch-agent route (`POST /api/session/{id}/agent`), the v1
  message `agent`, and `--agent` on `opencode run` and the v1 TUI.
- Same-name agents merge field by field across config documents (project
  files load before `OPENCODE_CONFIG_CONTENT`); fields a later document omits
  survive, and v2 appends permission rules. A launcher that reuses a user's
  agent name inherits that agent's model and permissions.
- A v2 session whose stored agent is absent from the current config:
  `opencode run --session <id>` fails with `Agent not found`; the TUI opens it
  and switches to `build`. v1 runs it as `build`.

## Model and effort

- v2 ignores an agent's `model`. A request uses the session's model, then the
  top-level config `model`.
- v2 parses `provider/model#variant` and agent `model.variant`, but provider
  requests carry no variant or reasoning field: there is no working effort
  channel.
- Session create and prompt use different model shapes: `{model: {id,
  providerID}}` for create, `{model: {modelID, providerID}}` for prompt.

## v2 CLI surfaces

- The bare TUI (`opencode` with no subcommand) has `--prompt`, `--session` and
  `--server`, and no `--model`, `--variant` or `--agent`. Passing `--model`
  exits with `Unrecognized flag: --model in command opencode`. Its only model
  channel is config `model`.
- Without `--standalone`, the bare TUI is a client of the shared
  `opencode serve --service` process, which was configured by whoever started
  it; the launching environment (`OPENCODE_CONFIG_CONTENT`, `OPENCODE_DB`)
  never applies.
- `opencode run` takes `--model provider/model#variant` and `--agent`.
- The v2 TUI submits `--prompt` on `--session`; the v1 TUI does not.

## Permissions

v2 effective permissions are the agent's rules merged with the session's, in
that order: a session `allow` overrides an agent `deny`. An agent `deny`
removes the tool from the catalog.

## Sources

Probes `spawn:p8760`, `spawn:p8761`, `spawn:p8762`, `spawn:p8766`,
`spawn:p8768` and the upstream source reading in `work:long-view-review`
(`opencode-profile-findings.md`, `upstream-opencode-config-semantics.md`,
`opencode-fix1-verification.md`). Upstream: `anomalyco/opencode` tags
`v2.0.29` and `v1.19.3`.
