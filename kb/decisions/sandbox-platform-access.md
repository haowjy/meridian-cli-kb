# Decisions: Meridian's Access Inside a Harness Sandbox

A profile's `sandbox:` limits what an agent may write, and its `network:` limits whether
the agent's shell may connect out. Meridian's own work still has to run under those limits:
the spawn takes its project lock, its `meridian` commands read and write state, and a child
spawn must outlive the shell call that launched it. Profile authors should not have to know
any of this. Meridian grants what its control plane needs wherever the harness can express
the grant, and `--dry-run` reports each grant, or its absence, per harness.

Three owners divide the work. The **profile author** (Mars schema) states what the role may
do: `sandbox`, `network`, `tools`. **Meridian** adds what any spawn needs to operate inside
Meridian, invisibly to the author. The **harness projection** turns both into native
settings and reports what it cannot enforce. The owner asked for this split: "what stuff we
need to pass in by default so prompt authors don't have to learn too much about meridian".

**Status (2026-10-23): unmerged.** The grants, escapes, compound-launch refusal and child
guardrail are on meridian-cli PR #750 (`feat/sandbox-network`). Profile `network` projection
and the network part of the guardrail are on the stacked `feat/sandbox-network-projection`,
which also needs the Mars `network` field (mars-agents PR #182, first release expected 0.18.0)
and a Mars pin bump before it merges.

**Who decided what.** The owner ruled two points: `network` defaults to allow ("yes, default
allow") and Meridian owns its CLI's permission surface ("yes, thats honestly better").
Everything else on this page is the implementing lead's current choice, made after probes
and reviews; the owner has not addressed it. The child guardrail in particular was put to the
owner as an uncertain call and has had no answer.

**Where the mechanism lives.** Platform command grants and the rule that owns them:
[tool-permissions.md](tool-permissions.md#d-permission-platform-rules). Which paths a
sandbox may see: [architecture/sandbox-projection.md](../architecture/sandbox-projection.md).
In meridian-cli: per-harness compiles in `src/meridian/lib/harness/projections/.context/CONTEXT.md`,
the caller record and its checks in `src/meridian/lib/core/caller_boundary.py`.

## What each harness gets

| Grant | Claude | Codex | OpenCode, Pi |
|---|---|---|---|
| Project lifetime lock usable | Yes: real-launch materialization pre-creates the exact file; POSIX locking opens it read-only | Yes by the same mechanism, including `read-only` | n/a: no sandbox |
| Meridian commands allowed (completion, read-only, spawn) | Exact and prefix `Bash(...)` allows | Only the `meridian spawn` prefix, through a user-wide execution-policy rule; other command grants are not expressible | Native shell or extension allows |
| Child spawn leaves the shell sandbox | For a standalone `meridian spawn` in a spawn-capable profile, when the sandbox is one Meridian projected (`read-only` or `workspace-write`) | For a standalone `meridian spawn` in every session using the Codex home, once any spawn-capable constrained launch has written the rule | n/a: no sandbox |
| Completion signals (`spawn done`, `rearm`) under `read-only` | Yes | Only through the spawn rule; a read-only profile without spawn capability skips the confirmation turn | n/a |
| Nested child authority | A Meridian-launched constrained caller cannot select a less-confined child or weaken an authored, enforced network deny | Same when the caller was launched by Meridian; a direct Codex session has no floor | Same floor, without a sandbox escape |
| Compound launch refused | Yes on Linux, by PID-namespace comparison | Yes on Linux, by the same comparison | n/a: same namespace |

A compound launch is a `meridian spawn` inside a shell line that also contains `&&`, a pipe,
`$()`, or a leading `VAR=` assignment. A spawn-capable profile declares `subagents` and does
not set `spawn: false`. The three grant sets are defined in
[tool-permissions.md](tool-permissions.md#d-permission-platform-rules). `--dry-run` states each
grant on the `sandbox:`, `network:` and `platform:` lines. Every spawn's injected guidance
tells the agent to run `meridian` commands as standalone shell calls; it sits in the
always-on block because children without spawn capability still run `spawn done`.

## D-sandbox-state-roots: pre-create the exact project lock {#d-sandbox-state-roots}

**Decided:** every spawn and session takes a shared lifetime lock in
`<user home>/projects/.locks/<project-id>.lock` (`~/.meridian/` unless `MERIDIAN_HOME` moves
it). The lock sits beside the project runtime root, not under it. Runtime materialization
creates that exact file after pure bind and before the harness starts. On POSIX, lock
acquisition opens an existing lock read-only, so a sandboxed `meridian` process can take the
lock without write access to the shared `.locks` directory. Dry-run performs neither write.

**Why:** the first implementation projected the entire sibling directory because ordinary
lock acquisition created a missing file in append mode. That fixed the sandbox failure but
let a child create lock names for other projects. Splitting creation from acquisition removes
the shared writable root: trusted runtime materialization creates the current project's file,
while the nested control process only needs the existing inode. A launch may see the current
project's runtime state but never receives a writable cross-project state tree.

**Rejected:**

- *A per-harness `allowWrite` rule for the lock directory.* A Claude-only rule would fix one
  sandbox while keeping a broader write grant than lock acquisition requires.
- *Projecting `~/.meridian/projects/`.* That is the whole-tree grant the
  [fixed projection constraint](../architecture/sandbox-projection.md#fixed-projection-constraint)
  forbids.
- *Projecting `~/.claude`.* A sandboxed Claude run in a normal workflow, with no `~/.claude`
  grant, completed without a read-only error, so no root was added. Add it when a sandboxed
  run fails with a read-only error on `~/.claude`.

## D-sandbox-spawn-escape: a standalone spawn launch leaves Claude's sandbox {#d-sandbox-spawn-escape}

**Decided:** when Meridian projects a `read-only` or `workspace-write` sandbox for a
spawn-capable profile, Claude's settings exclude `meridian spawn *` from the Bash sandbox
(`excludedCommands`). The whole command runs unsandboxed and its detached child survives the
shell call.

**Why:** Claude's sandbox runs commands in their own PID namespace and kills detached
children when the call ends. A child spawn is a detached worker, so inside the sandbox it
dies with the shell. Excluding the launch is the only way its worker finishes. The excluded
process stays in the launcher's host PID namespace.

**Why only `meridian spawn` and only spawn-capable profiles:** `meridian *` would have let
any sandboxed profile run `config set`, `work delete` or `mars …` unconfined. Only a launch
needs to outlive the shell. Spawn capability is computed from the profile at prepare time and
carried on the prepared launch, because continue and fork relaunch from a saved snapshot that
has no profile.

**Why only sandboxes Meridian projected:** under `sandbox: default` (or no `sandbox`),
Meridian passes Claude no sandbox settings and leaves the choice to the user's own Claude
config. Such a caller is not floored by the [guardrail](#d-delegation-floor), so if the user
had turned Claude's sandbox on, an exclusion would let the agent escape it with no check on
the child. Meridian adds no exclusion there. Its nested launches get the compound-launch refusal instead; the fix is
to set `sandbox:` in the profile.

**Consequence the profile author must accept:** the excluded `meridian` process bypasses the
sandbox completely, including `network: deny`. The [guardrail](#d-delegation-floor) stops a
constrained caller from asking for a weaker child; it does not re-sandbox the escaped process
or make its inputs trustworthy. The user-facing boundary and residual risks are in
`docs/agent-profiles.md` in the meridian-cli repository.

**Matching is by command shape.** Claude matches only a simple command. A compound line
(`cd x && meridian spawn …`, a pipe, `$()`, or `VAR=1 meridian …`) does not match, and its
launch stays sandboxed. See [D-sandbox-compound-refusal](#d-sandbox-compound-refusal).

**Evidence:** on Claude Code 2.1.309 with haiku, a spawn-capable architect under
`workspace-write` launched a child from one standalone Bash call, and the child finished and
reported. The same profile's compound form was refused.

## D-sandbox-compound-refusal: a compound launch fails before it starts {#d-sandbox-compound-refusal}

**Decided:** every Meridian launcher writes a JSON caller record to the child's
`_MERIDIAN_CALLER_BOUNDARY` environment variable; its `pid_namespace` field is the
launcher's `/proc/self/ns/pid` link. A nested `meridian spawn` create, continue or fork
compares that value with its own namespace in the operations layer, before it writes any
state, and refuses when they differ. The error says to run `meridian spawn` as a standalone
shell command. `spawn done`, `rearm`, help and dry run are not refused by this check. Each
nested launcher overwrites the record with its own.

**Why:** a sandbox that kills detached children does so by giving the shell its own PID
namespace, so the comparison tests that failure condition directly, on every harness at once.
Codex sets no general sandbox marker (only `CODEX_SANDBOX_NETWORK_DISABLED`), so a check on
Claude's `SANDBOX_RUNTIME=1` and per-harness markers was rejected. Without the refusal, a
compound launch starts a worker the shell's exit kills, and the parent sees no useful error.
The CLI entrypoint has no copy of the check: duplicating spawn-argument parsing there was
deleted as a second grammar, since the operations layer already refuses before state access.

**Platform limit:** if the launch environment has no record or procfs does not expose the
current namespace, Meridian skips the comparison. This is the expected path on macOS. The
mechanism does not claim to detect a sandbox there.

**Evidence:** the host and a Claude excluded command shared `pid:[4026531836]`; sandboxed
Claude and Codex commands each had their own namespace. Live compound calls under both
harnesses were refused with exit 1 and no child; standalone Claude, Codex and Pi nesting
succeeded. A Codex workspace-write probe covered every launch form (flags before and after
the prompt, background, model, agent, prompt file, continue, fork, a harness shortcut).

## D-delegation-floor: escaped child launches get a CLI guardrail {#d-delegation-floor}

> [!FLAG] **Needs human review.** Provisional: not owner-approved. Whether a sandboxed
> spawner may launch a less-confined child is the owner's call. This is the conservative
> option, the easiest to loosen; the rejected alternatives below remain open.

**Current choice:** a caller Meridian launched under `read-only` or `workspace-write` carries
a guardrail in its caller record: sandbox, spawn capability, and a network deny floor only
when the caller authored `network: deny` and its harness enforces it. Nested create, continue,
fork and their dry runs refuse:

- any launch when the caller lacks spawn capability;
- a child whose sandbox is weaker than the caller's (`--yolo` included);
- a child whose effective network is allow under a deny floor;
- harness-native passthrough arguments that could replace Meridian's resolved policy.

Approval, workspace roots and `--env` values are not compared. Top-level, `default` and
`danger-full-access` callers carry no floor. The refusal names the remedy: a child that omits
`network` or allows it must set `network: deny`; a child that already says deny must use a
harness and sandbox that enforce it.

**Why:** Claude's exclusion and Codex's rule move a standalone `meridian spawn` outside the
caller's sandbox so its worker survives, which would otherwise let any sandboxed spawner run
`meridian spawn --sandbox danger-full-access …`. Meridian's `meridian-privilege-escalation` skill
tells an agent blocked by a sandbox to retry with a less-confined sandbox, so this is the
retry agents are taught. The guardrail stops escalation through the CLI. It is
deliberately **not confinement against an adversarial agent**: the escaped process reads
inputs a sandboxed agent can write (the background-worker request, session records that name
`CODEX_HOME`, project `meridian.toml` hooks, workspace config, other spawns' control state).
A whole-system review proved a guardrail-compliant child could still run an agent-written hook
as the full user. Each tighter round found the next forged input, so the claim was narrowed
rather than chased. Issue #747 owns real confinement and holds the input table.

**Compatibility:** the parser ignores unknown keys, so a newer build's record still parses. A
record without `pid_namespace` or `sandbox` is invalid and refused; a missing `spawn` counts
as no spawn capability. A missing network key, and the older persisted value `allow`, mean no
network floor: the default is now allow, and reading a missing key as deny would refuse the
children of every caller launched before an upgrade.

**Rejected:** declared roles (enforce `subagents:` membership, issue #742, and forbid widening
flags; a declared child could still be less confined), accepting and documenting the escape,
and an earlier adversarial frame that also compared approval, workspace write roots and loader
environment keys.

## D-codex-child-spawn: Codex uses one user-wide spawn rule {#d-codex-child-spawn}

**Current choice:** before a real Codex launch that is spawn-capable and whose sandbox is
`read-only` or `workspace-write`, Meridian ensures that the launch environment's
`$CODEX_HOME/rules/meridian.rules` contains exactly:

```text
# This file is owned and rewritten by Meridian.
prefix_rule(pattern=["meridian", "spawn"], decision="allow")
```

It byte-compares existing content, replaces atomically only when the file differs, and never
edits another rules file. A write failure does not stop the launch; the permission report says
the spawn command remains sandboxed. Dry-run reports the rule without writing it.

The rule lets a standalone `meridian spawn ...` run outside Codex's shell sandbox. It fixes
both causes of nested failure: the sandbox kills the detached worker, and Mars's wait-timeout
path gets `EPERM` writing its `SIGCHLD` self-pipe under Codex's restricted network (writable
roots do not fix that). It also lets `meridian spawn done` take Meridian's state lock under
`read-only`, so a spawn-capable read-only Codex agent gets the ordinary completion-confirmation
turn. A read-only Codex profile without spawn capability (explorer, cold-reader,
session-miner) does not have the rule written for it and still skips that turn
([D-completion-channel](completion-drain.md#d-completion-channel)).

**Cost:** Codex has no per-launch rules layer. Once written, the rule applies to the user's
interactive Codex sessions and every Codex agent using that home. The dry-run report and
user-facing docs state that scope.

**Evidence:** with the matching line removed from the owner's own `default.rules`, a
workspace-write parent launched a child that finished, and a read-only agent ran
`meridian spawn done` with exit 0 and received the confirmation turn.

**Rejected:** a project `.codex/rules/` file (Mars regenerates that directory on sync, and
Codex loads project rules only for trusted projects), and a per-invocation `-c` key (Codex
has none for rules).

## D-network-field: profile `network`, default allow {#d-network-field}

**Decided (owner, 2026-10-23):** a profile may set `network: allow | deny`. It governs the
agent's shell only: whether its commands may connect to the internet, to host services, or to
local sockets. Built-in web tools stay under `tools: web`. Unset resolves to `allow`. The owner
started from "I think architects should be able to have internet access tho (to do research)"
and approved the default with "yes, default allow".

**Why `allow` is the default:** every `workspace-write` profile in the shipped prompt
packages does work that needs the network or does not care. The rejected default, "whatever
the harness does", meant one profile was online on OpenCode and Pi and offline on Claude and
Codex. Authors who want isolation write `deny` and see it in the profile.

**Why there is no `network: default` value:** Mars resolves policy layers by the first value
set (CLI `--network`, overlay, overlay model policy, profile, profile model policy, settings
model policy). A `default` value would be a second spelling of `allow`, and a higher layer
that wrote it would silently reopen a profile's `deny`. Unset is the only default state.

**Mars validates only `network` strictly.** A non-string value such as `network: false` is an
error at every input (frontmatter, model policy, overlay, CLI). The same fail-open still
exists for `sandbox`, `approval` and `effort`: a non-string value silently becomes unset.
Fixing the whole class in this change grew a second enum validator and deprecation code in
about fifteen places, so it was cut to a follow-up: one execution-enum table in Mars config
that every input reads, with a one-release warning for the older keys. Native Codex lowering
reports `network` as Dropped, so a Codex agent invoked outside Meridian keeps Codex's default.

**Projection** (on `feat/sandbox-network-projection`):

| `network` | Claude | Codex | OpenCode, Pi, Cursor |
|---|---|---|---|
| `allow` | Under `read-only`/`workspace-write`, `sandbox.network` gets `allowedDomains` `*`, `localhost`, `127.0.0.1`, plus `allowLocalBinding` and `allowAllUnixSockets` (both ignored by Claude at the top level) | `workspace-write`: explicit `network_access=true`. `read-only`: an authored `allow` is unsupported and reported; unset stays offline and reports native | Already unrestricted |
| `deny` | Under `read-only`/`workspace-write`, `sandbox.network.strictAllowlist=true` | `workspace-write`: explicit `network_access=false`, so a user's `config.toml` cannot open it. `read-only`: offline | Not enforced, reported |

Effective network is one harness-owned function of sandbox, authored network, whether the
bundle carries network, and native config. The permission report and the guardrail both use
it. Approval and tool rules are deliberately not inputs. Claude's `strictAllowlist` is what
makes that possible: probes showed a deny that only listed no domains still let traffic out
under some approval modes, while `strictAllowlist` (or `deniedDomains: ["*"]`) blocked it. The
rejected alternative made effective network a compile output checked at bind, which would have
split the guardrail across two checks.

`danger-full-access` cannot enforce deny on any harness. Claude's deny is partial on Linux
because processes in one sandbox share a private loopback, Claude's own settings can add
domains, and an excluded `meridian spawn` runs outside. Claude's Linux allow reaches host
services only through its proxy, and the sandbox's `NO_PROXY` makes most clients skip it
(curl needs `--noproxy ''`); the injected guidance says so when it applies.

**Old Mars keeps old behavior.** A current bundle carries `provenance.network_source`. A
bundle without it comes from Mars before 0.18: Meridian warns, projects no network setting,
and leaves the harness default, so an authored `deny` cannot fail open on an old Mars. The
guardrail treats that unknown state as allow.

**Codex passthrough key.** For one warning release, a profile's
`harness-overrides.codex.sandbox_workspace_write.network_access` boolean wins over the
portable field when Meridian can emit it as a `-c` pair; anything else is reported ignored and
the portable value stands. The next release reserves the key. See
[native-config](../concepts/native-config.md).

**Upgrade impact:** unset now means online for Claude `read-only`/`workspace-write` and Codex
`workspace-write`, which were offline. The upgrade note tells authors to write `network: deny`
to keep them offline. Continued sessions are re-resolved under the new default.

**Supersedes** the 2026-10-22 rejection of a `network` profile field in
[bundle-field-consumption.md](bundle-field-consumption.md).

## Profile follow-up (not done)

The owner approved changing only profiles first ("ok, phase 0 now that just alters the
profiles"). Released in meridian-base v0.11.14 and meridian-dev-workflow v0.15.7:
architect and web-researcher run on `danger-full-access` as a stopgap for network and child
spawns, and session-miner's allowlist gained the read commands it lacked. Once both
meridian-cli branches ship, the plan is to put architect and web-researcher back on
`workspace-write`, drop session-miner's hand-listed `meridian` reads, and delete scoped
`bash(meridian spawn *)`-style allows from lead profiles, which the platform grants now cover.
Whether other roles that use `danger-full-access` only for network or spawning should move to
`workspace-write` is a separate, later decision.

## Open questions

- **`subagents:` membership is not enforced at spawn creation.** A spawn-capable profile can
  launch any registered agent. Issue #742.
- **macOS and Cursor** are untested for every grant here. Claude Code's documentation says
  `allowLocalBinding` fixes host localhost on macOS; not verified.

## Harness facts these decisions rest on

Linux, Claude Code 2.1.309, Codex CLI 0.163.0. Recheck when either version changes.

| Fact | Claude | Codex |
|---|---|---|
| OS sandbox covers | Bash only. Edit and Write follow tool rules; WebFetch and WebSearch run outside | Every shell command, run through a login shell (`bash -lc`) |
| Detached child survives the shell call | No | No in sandbox; yes with full access |
| Per-command escape | `excludedCommands`, simple commands only | Execution-policy `prefix_rule` with `allow` |
| Sandbox PID namespace | Different from the launcher; an excluded command stays in the launcher's | Different from the launcher; an allowed command runs outside it |
| Shell network switch | Domain allowlist through a proxy; `strictAllowlist` for a hard deny | One boolean, which also gates host localhost and sockets |
| Read-only writes | Everything except roots Meridian denies | Nothing; `--add-dir` is ignored |

## Provenance

`work:harness-sandbox-network` (design, plan, research per harness, verification). Owner
rulings: `chat:c8765`, 2026-10-23. Structure review `spawn:p9047`; Codex rules POC
`spawn:p9048`; Claude escape and Phase 1 runs `spawn:p9060`–`spawn:p9068`; Codex rule and
confirmation probes `spawn:p9087`–`spawn:p9089`; PID-namespace probes
`spawn:p9096`–`spawn:p9107`. Guardrail: gate A security review `spawn:p9112`, second round
`spawn:p9153`/`spawn:p9154`, whole-system review `spawn:p9223`. Network gates:
`spawn:p9242`/`spawn:p9243`, then `spawn:p9246`/`spawn:p9247`. Mars validation scope:
`spawn:p9058`, `spawn:p9080`.
