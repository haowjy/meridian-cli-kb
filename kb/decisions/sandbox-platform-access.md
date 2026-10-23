# Decisions: Meridian's Access Inside a Harness Sandbox

A profile's `sandbox:` limits what an agent may write. Meridian's own work still has to
run under that limit: the spawn takes its project lock, its `meridian` commands read and
write state, and a child spawn must outlive the shell call that launched it. Profile authors
should not have to know any of this. Meridian grants what its control plane needs wherever the
harness can express the grant, and `--dry-run` reports each grant, or its absence, per harness.

**Status (as of 2026-10-23):** Claude and Codex grants are implemented on
`feat/sandbox-network`, not yet merged to `main`. The profile `network` field is not done,
and no release is set for it. Each decision below says which.

**Where the mechanism lives.** Platform command grants and the rule that owns them:
[tool-permissions.md](tool-permissions.md#d-permission-platform-rules). Which paths a
sandbox may see: [architecture/sandbox-projection.md](../architecture/sandbox-projection.md).
Per-harness compiles: `src/meridian/lib/harness/projections/.context/CONTEXT.md`.

## What each harness gets

| Grant | Claude | Codex | OpenCode, Pi |
|---|---|---|---|
| Project lock directory writable | Yes: projected as a root, so `--add-dir` (observed: `task-dir` and a lock acquisition succeed under `workspace-write`) | Yes under `workspace-write`, through the same projected roots (not live-tested) | n/a: no sandbox |
| Meridian commands allowed (completion, read-only, spawn) | Exact and prefix `Bash(...)` allows | The `meridian spawn` prefix through a user-wide execution-policy rule; other command grants are not expressible | Native shell or extension allows |
| Child spawn leaves the shell sandbox | Yes, for a standalone `meridian spawn` in a spawn-capable profile | Yes, for a standalone `meridian spawn` in every session using the Codex home | n/a: no sandbox |
| Compound launch refused | Yes on Linux, by PID-namespace comparison | Yes on Linux, by the same comparison | n/a: same namespace |

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
process remains in the launcher's host PID namespace; a sandboxed command runs in another
namespace.

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

**Decided:** before it launches a harness, Meridian records its own
`/proc/self/ns/pid` link in `_MERIDIAN_LAUNCHER_PID_NAMESPACE`. A `meridian spawn` create or
continue compares that value with its current namespace and refuses when they differ. Every
nested Meridian launcher replaces the value with its own namespace. The error says to run
`meridian spawn` as a standalone shell command. `meridian spawn done`, `rearm`, help, and dry
run are not refused.

**Why:** a sandbox that kills detached children isolates the shell in another PID namespace.
The namespace comparison tests that failure condition directly across Claude and Codex;
harness-specific markers do not. Without the refusal, a compound launch starts a worker the
shell's exit kills, and the parent sees no useful error. The CLI checks on its import-cheap
startup path before Mars loads, then the operations layer repeats the guard for non-CLI
callers. This keeps the corrective message ahead of lower-level sandbox failures and child
state writes.

**Platform limit:** if the launch environment has no recorded value or procfs does not expose
the current namespace, Meridian skips the comparison. This is the expected path on macOS.
The mechanism does not claim to detect a sandbox there.

**Evidence:** the host and a Claude excluded command shared `pid:[4026531836]`; sandboxed
Claude and Codex commands used different namespaces. Live compound calls under both harnesses
were refused with exit 1 and no child; standalone Claude, Codex, and Pi nesting succeeded.
Source: `work:harness-sandbox-network`.

## D-codex-child-spawn: Codex uses one user-wide spawn rule {#d-codex-child-spawn}

**Implemented on the unmerged branch:** before a Codex launch whose resolved sandbox is
`read-only` or `workspace-write`, Meridian ensures that the launch environment's
`$CODEX_HOME/rules/meridian.rules` contains exactly:

```text
# This file is owned and rewritten by Meridian.
prefix_rule(pattern=["meridian", "spawn"], decision="allow")
```

It resolves the Codex home from the final launch environment, byte-compares existing content,
and uses an atomic replacement only when the file differs. It never edits another rules file.
A write failure does not prevent the Codex launch; the permission report says the spawn command
remains sandboxed and includes the error. Dry-run reports the projection without writing it.

The rule lets a standalone `meridian spawn ...` run outside Codex's shell sandbox. It fixes
both observed causes of nested failure: the sandbox would kill the detached worker, and Mars's
wait-timeout path gets `EPERM` writing its `SIGCHLD` self-pipe under Codex's restricted
network. It also lets `meridian spawn done` take Meridian's state lock under `read-only`, so
read-only Codex again gets the ordinary completion-confirmation turn
([D-completion-channel](completion-drain.md#d-completion-channel)).

**Cost:** Codex has no per-launch rules layer, so the rule applies to the user's interactive
Codex sessions and every Codex agent using that home, not only spawn-capable ones. The dry-run
report and user-facing docs state that scope.

**Evidence:** with the matching line removed from the owner's `default.rules`, a
workspace-write `gpt-6-luna` parent launched child p9088, which finished with `child-ok`.
Read-only p9089 ran `meridian spawn done` with exit 0 and received the actual confirmation
turn. These probes used Meridian's generated file; `default.rules` was restored byte-for-byte.

**Rejected:** a project `.codex/rules/` file (Mars regenerates that directory on sync, so a rule
written there would be lost, and Codex loads project rules only for trusted projects), and a
per-invocation `-c` config key (Codex has none for rules).

## D-network-field: profile `network` is decided; projection is implemented {#d-network-field}

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

**Implemented on the unmerged Phase 3 branch** (2026-10-23; first supported Mars release
expected to be 0.18.0):

| `network` | Claude | Codex | OpenCode, Pi, Cursor |
|---|---|---|---|
| `allow` | `allowedDomains` `*`, `localhost`, `127.0.0.1`; `allowLocalBinding`; `allowAllUnixSockets` | `workspace-write`: `network_access=true`. `read-only`: unsupported, reported | Always on |
| `deny` | No domains | Off (default) | Not enforced, reported |

`danger-full-access` cannot enforce deny on any harness. Codex `read-only` cannot enforce
explicit allow; unset remains offline and is not reported as a user-requested gap. Claude
Linux deny is partial: private TCP loopback remains reachable, and `approval: never` can
bypass the proxy's network restriction. Claude Linux allow can reach host services through
the sandbox proxy only when clients bypass `NO_PROXY` (for curl, `--noproxy ''`). The dry-run
report marks these cases partial or unsupported rather than native. Pi's report was covered
by existing projection tests, not a live Pi launch; macOS and Cursor were not exercised live.

Codex `sandbox_workspace_write.network_access` is still a deprecated native-config passthrough
for one release and wins over the portable field during that warning period; reserve it in the
following release. The field is carried through optional bundle/snapshot fields without a
schema bump, so bundles from released Mars versions that omit it remain readable as unset.

Claude reaches host services on Linux only through its proxy, and the sandbox's `NO_PROXY`
makes most clients skip it. Observed: `curl --noproxy ''` with `localhost` in `allowedDomains`
works. Meridian's dry-run reports this as partial.

**Supersedes** the 2026-10-22 rejection of a first-class `network` profile field in
[bundle-field-consumption.md](bundle-field-consumption.md). Today Codex `workspace-write` is offline
by default; the `allow` default makes it online, so the user-facing upgrade note explains
`network: deny` as the way to preserve offline behavior. Meridian requires the Mars release
that carries the field before users can author it.

## Open questions

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
| Sandbox PID namespace | Different from the launcher; an excluded command stays in the launcher's namespace | Different from the launcher; an execpolicy-allowed command runs outside it |
| Shell network switch | Domain allowlist through a proxy | One boolean, which also gates host localhost and sockets |

## Provenance

`work:harness-sandbox-network`: the owner's design approval, the structure review that
changed the platform-rule shape (`spawn:p9047`), the Codex rules POC (`spawn:p9048`), the
Claude excluded-command probe and Phase 1 live runs (`spawn:p9060`–`spawn:p9068`), the
Meridian-owned Codex rules and confirmation probes (`spawn:p9087`–`spawn:p9089`), and the
PID-namespace probes (`spawn:p9096`–`spawn:p9107`). Claude host-localhost and proxy behavior
comes from probes recorded in the same work item's research notes. Phase 3 implementation and
live verification are on unmerged meridian-cli commits `21df98c3f..996a58533`; see
`work:harness-sandbox-network/verification.md#phase-3-verification` for command evidence and
known limits.
