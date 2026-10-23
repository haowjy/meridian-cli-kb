# Decisions: Meridian's Access Inside a Harness Sandbox

A profile's `sandbox:` limits what an agent may write. Meridian's own work still has to
run under that limit: the spawn takes its project lock, its `meridian` commands read and
write state, and a child spawn must outlive the shell call that launched it. Profile authors
should not have to know any of this. Meridian grants what its control plane needs wherever the
harness can express the grant, and `--dry-run` reports each grant, or its absence, per harness.

**Status (as of 2026-10-23):** Claude grants: implemented, not yet merged to `main`. Codex
child spawns and the profile `network` field: not done, and no release is set for either.
Each decision below says which.

**Where the mechanism lives.** Platform command grants and the rule that owns them:
[tool-permissions.md](tool-permissions.md#d-permission-platform-rules). Which paths a
sandbox may see: [architecture/sandbox-projection.md](../architecture/sandbox-projection.md).
Per-harness compiles: `src/meridian/lib/harness/projections/.context/CONTEXT.md`.

## What each harness gets

| Grant | Claude | Codex | OpenCode, Pi |
|---|---|---|---|
| Project lock directory writable | Yes: projected as a root, so `--add-dir` (observed: `task-dir` and a lock acquisition succeed under `workspace-write`) | Yes under `workspace-write`, through the same projected roots (not live-tested) | n/a: no sandbox |
| Meridian commands allowed (completion, read-only, spawn) | Exact and prefix `Bash(...)` allows | Not grantable: Codex has no shell-command allow rule; a planned execution-policy rule is in [D-codex-child-spawn](#d-codex-child-spawn) | Native shell or extension allows |
| Child spawn leaves the shell sandbox | Yes, for a standalone `meridian spawn` in a spawn-capable profile | Not yet (planned, below) | n/a: no sandbox |
| Compound launch refused | Yes | No: Codex sets no sandbox marker that Meridian reads | n/a: no sandbox |

A compound launch is a `meridian spawn` inside a shell line that also contains `&&`, a pipe,
`$()`, or a leading `VAR=` assignment. A spawn-capable profile is one that declares
`subagents`. The three grant sets are defined in
[tool-permissions.md](tool-permissions.md#d-permission-platform-rules). `--dry-run` states each grant on the `sandbox:` and
`platform:` lines.

## D-sandbox-state-roots: the project lock directory is projected {#d-sandbox-state-roots}

**Decided:** every spawn and session takes a shared lifetime lock in
`~/.meridian/projects/.locks/<project-id>.lock`, where `<project-id>` is the project's ID.
The lock sits beside the project runtime root (`~/.meridian/projects/<project-id>/`), not
under it. The directory is added to the projected roots, so a sandboxed `meridian` command
can take that lock. Launch binding (the step that assembles a launch's context before the
harness starts) creates the directory first.

**Why:** the runtime root does not cover the lock, because the lock lives beside it. Without
this grant, a sandboxed agent fails with `Read-only file system` on
`~/.meridian/projects/.locks` and cannot launch children. The lock file is created on demand,
so only its directory can be granted in advance.

**Why binding creates it:** a missing bind source aborts Codex's whole sandbox namespace.
The directory is therefore created, not filtered out the way a missing context root is
([sandbox projection](../architecture/sandbox-projection.md#context-root-existence-filtering)).

> [!FLAG] **Needs human review.** The directory holds every project's lifetime lock, not
> only this project's. A sandboxed agent can therefore create or hold another project's lock
> file. [Sandbox projection](../architecture/sandbox-projection.md) rejects projecting
> `~/.meridian/locks` for this reason. No decision record weighs that risk against this
> grant. The project owner records the ruling here. Source: `work:harness-sandbox-network`.

**Rejected:**

- *A per-harness `allowWrite` rule for the lock directory.* Projected roots already reach
  every harness that projects them; a Claude-only rule would fix one sandbox.
- *Projecting `~/.meridian/projects/`.* That is the whole-tree grant the
  [fixed projection constraint](../architecture/sandbox-projection.md#fixed-projection-constraint)
  forbids.
- *Projecting `~/.claude`.* A sandboxed Claude run in a normal workflow, with no `~/.claude`
  grant, completed without a read-only error, so no root was added. Add it when a sandboxed run fails with a read-only error on
  `~/.claude`.

## D-sandbox-spawn-escape: a standalone spawn launch leaves Claude's sandbox {#d-sandbox-spawn-escape}

**Decided:** a spawn-capable profile's standalone `meridian spawn …` command is excluded
from Claude's Bash sandbox (`excludedCommands: ["meridian spawn *"]`). The whole command
runs unsandboxed and its detached child survives the shell call.

**Why:** Claude's sandbox runs commands in their own PID namespace and kills detached
children when the call ends. A child spawn is a detached worker, so inside the sandbox it
dies with the shell. Excluding the launch is the only way its worker finishes. The excluded
process also drops Claude's `SANDBOX_RUNTIME` marker (observed in a probe).

**Why only `meridian spawn` and only spawn-capable profiles:** `meridian *` would have let
any sandboxed profile run `config set`, `work delete` or `mars …` unconfined. The grant is
limited to the launch form because only a launch needs to outlive the shell. A profile has
spawn capability when it declares `subagents` and does not set `spawn: false`
(`has_spawn_capability`). That flag is computed from the profile at prepare time and carried
on the prepared launch, because continue and fork relaunch from a saved snapshot that has no
profile, and a flag computed at bind would be lost there.

**Consequence the profile author must accept:** an excluded command bypasses the sandbox
completely, including `network: deny` once that field ships
([decided, not shipped](#d-network-field)). A spawn-capable profile can therefore launch a
child with full filesystem and network access regardless of its own `sandbox:`. The
user-facing docs, `docs/agent-profiles.md` in the meridian-cli repo, state this.

**Matching is by command shape.** Claude matches only a simple command: one command with no
`&&`, pipe, `$()` or `VAR=` prefix. A compound line (`cd x && meridian spawn …`, a pipe,
`$()`, or `VAR=1 meridian …`) does not match, and its launch stays sandboxed. See
[D-sandbox-compound-refusal](#d-sandbox-compound-refusal).

**Evidence:** on Claude Code 2.1.309, using the haiku model, a spawn-capable architect under
`workspace-write` launched a child from one standalone Bash call, and the child finished and
reported. The same profile's compound form was refused with the error text in the next
section. Spawn IDs are in provenance.

## D-sandbox-compound-refusal: a compound launch fails before it starts {#d-sandbox-compound-refusal}

**Decided:** `meridian spawn` create and continue refuse to run when Claude's sandbox marker
(`SANDBOX_RUNTIME=1`) is set. The error says to run `meridian spawn` as a standalone shell
command. `meridian spawn done` and `meridian spawn rearm` are not refused: they must work in
compound form, through the state roots above. Dry-run is not refused.

**Why:** without the refusal, a compound launch starts a worker that the shell's exit
kills, and the parent sees no error. A failure before any state is written is the only
signal the agent gets, and the message names the rule it must follow.

> [!FLAG] **Needs human review.** The current check reads only `SANDBOX_RUNTIME=1`, so it
> covers Claude alone. A different check was planned and was not shipped as of 2026-10-23: the
> launcher records its PID namespace (`/proc/self/ns/pid`) and refuses a launch whose namespace
> differs. Codex sets no generic marker, so only the namespace check would reach Codex. Open
> choice for the project owner: ship the namespace check, or record the marker-only check as
> the decision. Source: `work:harness-sandbox-network`.

## D-codex-child-spawn: Codex children cannot launch yet {#d-codex-child-spawn}

**Current:** under Codex's workspace-write sandbox a child spawn cannot launch. Two causes
were found. The sandbox kills the detached worker. Mars (the package manager Meridian calls
to build launch bundles) also fails on its wait-timeout path: its `SIGCHLD` self-pipe write
gets `EPERM` under Codex's restricted network. The `--dry-run` report says child spawns cannot
launch under Codex's sandbox. Codex's read-only sandbox cannot write, so `spawn done` cannot
run there either, and the completion confirmation turn (the follow-up prompt that asks a child
to confirm its report) is skipped
([D-completion-channel](completion-drain.md#d-completion-channel)).

**Decided, not shipped:** before a sandboxed Codex spawn launches, Meridian writes a
Meridian-owned `$CODEX_HOME/rules/meridian.rules` containing one execution-policy rule,
`prefix_rule(pattern=["meridian","spawn"], decision="allow")`. The file name is unrelated to the
`codex_rules` bundle field, which Mars no longer emits. A probe with `codex exec` 0.163.0 showed that rule
lets a workspace-write child launch and survive, and lets a read-only agent run `spawn done`.
The probe used a rule saved by an interactive approval, not Meridian's file, so the
Meridian-written file is untested.

**Cost:** Codex has no per-launch rules layer, so the rule applies to the user's interactive
Codex sessions and to every sandboxed Codex agent, not only spawn-capable ones. The
user-facing docs must say so when this ships.

**Rejected:** a project `.codex/rules/` file (Mars regenerates that directory on sync, so a rule
written there would be lost, and Codex loads project rules only for trusted projects), and a
per-invocation `-c` config key (Codex has none for rules).

## D-network-field: profile `network` is decided, not shipped {#d-network-field}

**Decided (owner, 2026-10-23):** a profile may set `network: allow | deny`. The field governs
the agent's shell only: whether its commands may open connections to the internet, to host
services, or to local sockets. Built-in web tools stay under `tools: web`. When `network` is
unset, Meridian resolves it to `allow`.

**Why `allow` is the default:** every `workspace-write` profile in the shipped prompt packages does work that
needs the network or does not care. A harness-dependent default meant one profile had network
access on one harness and none on another. Authors who want isolation write `deny` and see it
in the profile.

**Why there is no `network: default` value:** Mars (`mars-agents`) resolves policy layers by
the first value set, in the order CLI flag, overlay, profile, then settings. A `default` value
would be a second spelling of `allow`, and a higher layer that wrote it would silently reopen a
profile's `deny`. Unset is the only default state.

**Planned projection:**

| `network` | Claude | Codex | OpenCode, Pi |
|---|---|---|---|
| `allow` | `allowedDomains` `*`, `localhost`, `127.0.0.1`; `allowLocalBinding`; `allowAllUnixSockets` | `workspace-write`: `network_access=true`. `read-only`: unsupported, reported | Always on |
| `deny` | No domains | Off (default) | Not enforced, reported |

Claude reaches host services on Linux only through its proxy, and the sandbox's `NO_PROXY`
makes most clients skip it. Observed: `curl --noproxy ''` with `localhost` in `allowedDomains`
works. Meridian's dry-run reports this as partial.

**Supersedes** the 2026-10-22 rejection of a first-class `network` profile field in
[bundle-field-consumption.md](bundle-field-consumption.md). Today Codex `workspace-write` is offline
by default; the `allow` default would make it online, so the user-facing upgrade note must say so. Meridian
requires the Mars release that carries the field before it projects the field.

## Open questions

- **Whether Codex children can launch without the rules file.** Codex has no per-invocation
  rule carrier, so the answer is the rules file or no Codex children.
- **`subagents:` is not enforced at spawn creation.** A spawn-capable profile can launch any
  registered agent, and the escape above lets a sandboxed launch run that agent unsandboxed.
  Enforcement was deferred to a tracked issue; no tracking issue is recorded in this KB.
- **macOS and Cursor** (a harness Meridian supports) are not tested for any of these grants.
  Claude Code's documentation says `allowLocalBinding` fixes host localhost on macOS; that is
  not verified here.

## Harness facts these decisions rest on

Linux, Claude Code 2.1.309, Codex CLI 0.163.0. Recheck when either version changes.

| Fact | Claude | Codex |
|---|---|---|
| OS sandbox covers | Bash only. Edit and Write are governed by tool rules; WebFetch and WebSearch run outside the sandbox | Every shell command |
| Detached child survives the shell call | No | No in sandbox; yes with full access |
| Per-command escape | `excludedCommands`, simple commands only | Execution-policy `prefix_rule` with `allow` |
| Sandbox environment marker | `SANDBOX_RUNTIME=1` | None generic (`CODEX_SANDBOX_NETWORK_DISABLED` only) |
| Shell network switch | Domain allowlist through a proxy | One boolean, which also gates host localhost and sockets |

## Provenance

`work:harness-sandbox-network`: the owner's design approval, the structure review that
changed the platform-rule shape (`spawn:p9047`), the Codex rules probe (`spawn:p9048`), the
Claude excluded-command probe and live runs of the Claude grants (`spawn:p9060`–`spawn:p9068`),
and the verification log. Claude host-localhost and proxy behavior comes from probes recorded
in the same work item's research notes.
