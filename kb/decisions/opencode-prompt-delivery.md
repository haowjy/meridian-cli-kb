# Decisions: OpenCode Prompt Delivery

How Meridian's composed system prompt (agent profile body, loaded skills,
`# Meridian Agents`, `# Report`) reaches an OpenCode model, and why. The
current mechanism is described in meridian-cli
`src/meridian/lib/harness/.context/CONTEXT.md`, section "OpenCode: Prompt,
Agent and Model Delivery". The upstream OpenCode behavior these decisions rest
on is in [OpenCode config semantics](../research/opencode-config-semantics.md).

**Status:** settled after review on 2026-10-21 (`work:long-view-review`). The
code is on meridian-cli branch `fix/opencode-v2-prompt` (commits
`c0fb10a1a..865cabca1`) and is not merged. Until it merges, `main` delivers
nothing to OpenCode 2 and delivers the prompt twice to OpenCode 1.

## The problem

Probes on OpenCode 2.0.29 and 1.19.3 read the actual provider request through
a local logging provider (`spawn:p8760`, `spawn:p8762`):

| Path | OpenCode 2 before | OpenCode 1 before |
|---|---|---|
| Streaming spawn | no profile, no skills | prompt delivered twice |
| Managed primary | none | twice |
| Black-box TUI fallback | never starts (`Unrecognized flag: --model`) | none |

Meridian had two channels: a temp file listed in `OPENCODE_CONFIG_CONTENT`
`instructions`, and the v1 message `system` field. OpenCode 2 ignores
`instructions` and its prompt API has no `system` field, so no OpenCode 2 path
had a working channel. No evidence shows it ever worked: the v2 transport
discarded `system` from its first commit (`370940cb8`). OpenCode 1 received
both channels at once. The black-box fallback had neither.

## D-opencode-config-agent: deliver the prompt as a config agent projected at bind

**Decided:** bind adds one agent to the launch's `OPENCODE_CONFIG_CONTENT`:

```json
{"agent": {"meridian-<profile>": {"mode": "primary",
  "prompt": "{file:<spawn-log-dir>/system-prompt.md}"}},
 "default_agent": "meridian-<profile>"}
```

The id is `meridian` when no profile is loaded. Every other key of the
inherited config is kept. Bind already owns the child environment that every
launch path inherits (streaming spawn, managed primary, black-box fallback), so
one projection covers all of them and no path keeps its own prompt channel.
`system-prompt.md` is the same file Claude reads through
`--append-system-prompt-file`. The `instructions` temp file, the v1 message
`system` field and `ConnectionConfig.system` are deleted. The projection
channel label is `config-agent`.

**Why:**

- It works live on both versions: OpenCode 2 over HTTP with and without an
  agent, attached TUI, bare TUI and `opencode run`; OpenCode 1 TUI, `run` and
  HTTP (`spawn:p8761`, `spawn:p8762`).
- One v1-shaped entry serves both versions, because OpenCode 2 normalizes
  `agent.<id>.prompt` to `agents.<id>.system`.
- A file reference keeps the environment variable small. Linux caps one
  environment string at 128 KiB, and an 80 KB prompt file loaded intact.
- The `meridian-` prefix keeps the id from colliding with a user agent. OpenCode
  merges same-name agents field by field across config documents, so reusing a
  user's name would inherit their model and permissions.

**Rejected:**

- *A v2 plugin `context` hook that appends to the system prompt.* v2-only, ships
  and loads a plugin, and reaches the fallback only if the plugin loads.
- *Session instruction entries* (`PUT /api/experimental/session/{id}/instructions/entries/{key}`).
  Experimental, session-scoped, and unreachable from the black-box fallback.
- *Repairing `instructions`.* OpenCode 2's instruction loader discovers only
  `AGENTS.md`; fixing that is an upstream change.

### Accepted cost: the built-in base prompt is replaced

An agent prompt replaces OpenCode's built-in base prompt, not the rest of the
system prompt. On OpenCode 2 that base is about 756 characters plus three
tool-guidance lines (output is GFM, `<system-reminder>` blocks are harness
instructions, parallelize tool calls, edit-tool rules). `AGENTS.md`, OpenCode's
skill listing and the environment block still arrive on both versions. A
read, edit, shell and verify task on the same model showed no degradation
against the default `build` agent (`spawn:p8761`, one run each: a correctness
check, not a benchmark). Revisit if OpenCode agents start missing those
behaviors; a profile can carry equivalent lines.

### Accepted cost: Meridian-made sessions name an agent plain OpenCode lacks

An OpenCode 2 session created by Meridian stores agent `meridian-<profile>`.
Outside Meridian, where the launch config is absent, `opencode run --session
<id>` fails with `Agent not found`; the TUI opens the session and switches to
`build`. OpenCode 1 runs it as `build`. Avoiding this means overriding the
user's `build` agent, which brings back the model conflict that
D-opencode-model-off-agent removes.

## D-opencode-explicit-agent: select the agent on every session, not only through `default_agent`

**Decided:** the agent id is decided once, on `ResolvedLaunchSpec.agent_name`,
which is set exactly when the appended prompt is non-empty (the same condition
under which bind emits the agent and its file reference). It is sent wherever a
session is opened or resumed: v2 session create, the v2 switch-agent route on
resume, every v1 message, and `--agent` on `opencode run` and the v1 TUI.
`default_agent` stays in the config for sessions the user opens later in the
TUI.

**Why:** `default_agent` applies only to a session that has no agent. A resumed
session keeps the agent it was committed to, so `--continue` of a session
created under `build` delivered no prompt (review `spawn:p8764`; fixed and
re-verified on both versions in `spawn:p8768`).

**Known gap:** the OpenCode 2 bare TUI has no `--agent` flag, so a black-box
fallback that *resumes* a session committed to another agent still gets no
prompt. Fresh fallback sessions take `default_agent`. Tracked in meridian-cli
`src/meridian/lib/harness/.context/FUTURE`.

## D-opencode-model-off-agent: the model stays at top-level config and on the session

**Decided:** Meridian's agent carries no `model`. A fresh session gets the
model from top-level config `model` and, where the API allows, from the session
create or first message. A resumed session keeps its committed model unless
`--model` was given, in which case v2 switches it through the session model
route. The OpenCode 2 bare TUI rejects `--model`, so its only model channel is
the config key.

**Why:** OpenCode 2 ignores an agent's `model`: the session model wins, then the
top-level config model (`spawn:p8761`). Putting the model on the agent would
look correct and do nothing on v2. With no model on Meridian's agent, an
inherited agent can no longer pin a model that conflicts with the requested
one, so the one-restart conflict path in
[the OpenCode native commitment decision](model-resolution.md#d76-harness-specific-model-ids-via-runnablepath)
is unreachable and is deleted. An independent probe with a user
`default_agent` pinned to another model, in both `OPENCODE_CONFIG_CONTENT` and
project `opencode.json`, served Meridian's agent and the requested model
(`spawn:p8766`).

**Rejected:** carrying model and permissions on the agent. The model half does
nothing on v2; the permission half is an open owner decision (below).

## Deliberately out of scope

> [!FLAG] **Needs human review.** Profile tool and permission policy is not
> applied to OpenCode interactive primaries on either version: they defer to the
> native TUI, and `bash`, `edit` and `write` stayed available under a profile
> that denies them (`spawn:p8760`). Meridian's config agent could carry the
> profile's `permission` rules, but that changes what a user can do in their own
> TUI. Owner call; noted in meridian-cli `src/meridian/lib/harness/.context/FUTURE`.

OpenCode 2 has no working effort channel: config parses `provider/model#variant`
but provider requests carry no reasoning field (`spawn:p8762`).

## Provenance

`work:long-view-review` (`opencode-profile-findings.md` holds the probe
matrices and the lead's decision; `opencode-v2-root-cause.md`;
`pr-body-opencode-v2-prompt.md`). Probes `spawn:p8760` (before state, both
versions), `spawn:p8761` (raw config-agent feasibility), `spawn:p8762` (v1
channels, v2 TUI model), `spawn:p8766` (independent edge cases),
`spawn:p8768` (resume fix). Reviews `spawn:p8764`, `spawn:p8770`,
`spawn:p8771`, `spawn:p8773`, `spawn:p8774`.
