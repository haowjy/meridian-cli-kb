# Decisions: Meridian's Access Inside a Harness Sandbox

A profile's `sandbox:` limits what an agent may write. Meridian's own work still has to
run under that limit: the spawn takes its project lock, its `meridian` commands read and
write state, and a child spawn must outlive the shell call that launched it. Profile authors
should not have to know any of this. Meridian grants what its control plane needs wherever the
harness can express the grant, and `--dry-run` reports each grant, or its absence, per harness.

**Status (as of 2026-10-23):** the Claude and Codex grants, delegation floor, and profile
`network` projection are combined on `feat/sandbox-network-projection`, not yet merged to
`main`. The implementation is gated only on the separate pre-merge pin to the first Mars
release that carries the field (expected 0.18.0). Each decision below says which.

**Where the mechanism lives.** Platform command grants and the rule that owns them:
[tool-permissions.md](tool-permissions.md#d-permission-platform-rules). Which paths a
sandbox may see: [architecture/sandbox-projection.md](../architecture/sandbox-projection.md).
Per-harness compiles: `src/meridian/lib/harness/projections/.context/CONTEXT.md`.

## What each harness gets

| Grant | Claude | Codex | OpenCode, Pi |
|---|---|---|---|
| Project lifetime lock usable | Yes: bind pre-creates the exact file; POSIX locking opens it read-only | Yes by the same mechanism, including `read-only` | n/a: no sandbox |
| Meridian commands allowed (completion, read-only, spawn) | Exact and prefix `Bash(...)` allows | The `meridian spawn` prefix through a user-wide execution-policy rule; other command grants are not expressible | Native shell or extension allows |
| Child spawn leaves the shell sandbox | Yes, for a standalone `meridian spawn` in a spawn-capable profile | Yes, for a standalone `meridian spawn` in every session using the Codex home | n/a: no sandbox |
| Nested child authority | A Meridian-launched caller cannot select a less-confined child or weaken an authored, enforced network deny | Same when the caller was launched by Meridian; a direct Codex session has no floor | Same floor, without a sandbox escape |
| Compound launch refused | Yes on Linux, by PID-namespace comparison | Yes on Linux, by the same comparison | n/a: same namespace |

A compound launch is a `meridian spawn` inside a shell line that also contains `&&`, a pipe,
`$()`, or a leading `VAR=` assignment. A spawn-capable profile is one that declares
`subagents`. The three grant sets are defined in
[tool-permissions.md](tool-permissions.md#d-permission-platform-rules). `--dry-run` states each grant on the `sandbox:` and
`platform:` lines.

## D-sandbox-state-roots: pre-create the exact project lock {#d-sandbox-state-roots}

**Decided:** every spawn and session takes a shared lifetime lock in
`~/.meridian/projects/.locks/<project-id>.lock`, where `<project-id>` is the project's ID.
The lock sits beside the project runtime root (`~/.meridian/projects/<project-id>/`), not
under it. Launch binding pre-creates that exact file before the harness starts. On POSIX,
lock acquisition opens an existing lock read-only, so a sandboxed `meridian` process can take
the lock without receiving write access to the shared `.locks` directory.

**Why:** the first implementation projected the entire sibling directory because ordinary
lock acquisition created a missing file in append mode. That fixed the sandbox failure but
let a child create lock names for other projects. Splitting creation from acquisition removes
the shared writable root: trusted launch binding creates the current project's file, while
the nested control process only needs the existing inode. This follows the fixed rule that a
launch may see the current project's runtime state but does not receive a writable
cross-project state tree.

**Rejected:**

- *A per-harness `allowWrite` rule for the lock directory.* A Claude-only rule would fix one
  sandbox while retaining a broader write grant than lock acquisition requires.
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

**Consequence the profile author must accept:** the excluded `meridian` process itself
bypasses the sandbox completely. The [delegation floor](#d-delegation-floor) prevents a
Meridian-launched caller from selecting a less-confined child's sandbox or approval mode,
and preserves an authored network deny only when the caller's harness effectively denies
network. It does not re-sandbox the escaped control process. That process still reads trusted
launch inputs outside the sandbox. The user-facing boundary and residual risks are in
`docs/agent-profiles.md` in the meridian-cli repository.

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
`/proc/self/ns/pid` link in `_MERIDIAN_LAUNCHER_PID_NAMESPACE`. A `meridian spawn` create,
continue, or fork compares that value with its current namespace and refuses when they differ.
Every nested Meridian launcher replaces the value with its own namespace. The error says to
run `meridian spawn` as a standalone shell command. `meridian spawn done`, `rearm`, help, and
dry run are not refused by this namespace check.

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
were refused with exit 1 and no child; standalone Claude, Codex, and Pi nesting succeeded. A
real Codex workspace-write probe also exercised flags before and after the prompt, background,
model, agent, prompt-file, continue, fork, and a harness shortcut; every launch form reached
the startup refusal before sandboxed state access. Source: `work:harness-sandbox-network`.

## D-delegation-floor: a child is never less confined than its caller {#d-delegation-floor}

**Decided:** every harness process launched by Meridian receives a launcher-owned floor with
its resolved sandbox, approval mode, optional network deny, and spawn capability. Nested
create, continue, fork, and their dry runs enforce that floor through the shared spawn-creation
boundary. A caller without spawn capability is refused. A child is refused when its sandbox or
approval mode is weaker than the caller's, when it would weaken a recorded network deny, when
native passthrough arguments could replace Meridian's resolved flags, or when `--env` could
replace the process or harness config loader, executable search path, or harness home. The
exact blocked environment-key set is owned by `docs/agent-profiles.md` in the meridian-cli
repository rather than duplicated here.

Sandbox modes run from most to least confined as `read-only`, `workspace-write`,
`danger-full-access`, then `default`; because a harness default is unknown, it compares weaker
than every explicit mode. Approval runs from most to least confined as `confirm`, `default`,
`auto`, then `never`/`yolo`. Network is different: the boundary record contains `deny` only
when the caller authored `network: deny` and the harness's effective network is also deny.
Default allow, authored allow, and an unenforced deny create no network floor; a missing key
(and the legacy value `allow`) means no floor. A refusal tells the user to set `network: deny`
on the child profile. Each child launch replaces the inherited value with its own resolved
floor, so descendants compare against their immediate caller rather than the root of the
spawn tree.

**Why:** Claude's exclusion and Codex's execution-policy rule move the entire standalone
`meridian spawn` command outside the caller's native sandbox so its detached worker survives.
Before the floor, that escape also let a constrained caller ask Meridian to launch
`danger-full-access`, lower approval to `never`, or pass loader and harness-native overrides.
The escape solves process lifetime only if the unsandboxed control process preserves the
caller's authority. Monotonic confinement is portable across harnesses; inferred roles and
per-harness escape details are not.

**The check precedes writes.** With a floor present, spawn creation resolves policy read-only,
enforces the floor, and only then materializes runtime state. This includes continue and fork.
The ordering is part of the boundary: an earlier implementation reached the spawn-state lock
first, so Codex `workspace-write` reported `EROFS` instead of the policy refusal. The
import-cheap checks for spawn capability, directly requested native passthrough, and sensitive
environment keys run before source-state reads. Continue and fork check replayed native
arguments as soon as a pure authority read recovers them. Every refusal still precedes identity,
state locks, telemetry, legacy import, continue sweeps, and spawn rows.

**No launcher floor means no clamp.** Top-level user commands and shells not launched by
Meridian have no floor and keep their existing behavior. Read operations, `spawn done`, and
`spawn rearm` never consult it. This is deliberately different from trying to infer authority
from an ambient harness marker: Meridian only promises the floor for processes it launched.
Consequently, Codex's user-wide rule can let a direct, non-Meridian Codex session escape for a
standalone spawn without receiving this guarantee.

**Residual trust boundary:** a `workspace-write` caller can still edit inputs the escaped
process reads outside the sandbox, including project hook configuration, shell startup files,
or an earlier `meridian` executable on `PATH`. The direct `--env` and native-argument checks
close the portable override surfaces; they do not make writable project and shell inputs
untrusted. The network floor deliberately records only authored policy that the harness
classifies as effective. It does not turn a portable default or an unenforced request into inherited
authority.

**Rejected:** using declared `subagents:` membership as the boundary, because that still allows
a declared child to be less confined and couples authority to role topology; and accepting the
escape with documentation alone. Membership enforcement remains the separate #742 question.

**Evidence:** Claude `workspace-write` parent p9139 launched a `read-only` explorer, continued
it, and refused danger-full-access, yolo, a danger profile, and `LD_PRELOAD`. Continued parent
p9142 forked under the same floor. Codex read-only p9145 retained its user-wide rules file but
was refused because its floor recorded no spawn capability. Danger-full-access parent p9146
launched an equally confined child unchanged. The final implementation, including network,
is on the unmerged combined branch at `b7393fe4c`. Source: `work:harness-sandbox-network`.

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

**Implemented on the unmerged combined branch** (2026-10-23, `b7393fe4c`; first
supported Mars release expected to be 0.18.0):

| `network` | Claude | Codex | OpenCode, Pi, Cursor |
|---|---|---|---|
| `allow` | Under `read-only`/`workspace-write`, `sandbox.network` gets `allowedDomains` `*`, `localhost`, `127.0.0.1`, `allowLocalBinding=true`, and `allowAllUnixSockets=true` | `workspace-write`: explicit `network_access=true`; `read-only`: unsupported, reported | Already unrestricted |
| `deny` | Under `read-only`/`workspace-write`, `sandbox.network.strictAllowlist=true` | `workspace-write`: explicit `network_access=false`; `read-only`: native offline behavior | Not enforced, reported |

Claude's effective network is the declared portable mode while its constrained Bash sandbox
is enabled, and allow when that sandbox is off. Approval mode is not a network fact. The deny
report is still partial on Linux because processes in the same sandbox can communicate over
private TCP loopback; Claude settings may merge additional domains, and a spawn-capable
profile's excluded `meridian spawn` command runs outside the sandbox. Allow's Linux host-service
hint has one owner in the Claude adapter and appears only under the same sandbox/allow facts.

Codex's workspace-write projection always spells the boolean, so deny overrides a permissive
ambient `config.toml`. The deprecated profile-native
`sandbox_workspace_write.network_access` override is resolved once during permission compile
for its warning release: a matching boolean reports native, a conflicting boolean wins and
reports partial, and a non-boolean warns rather than being interpreted as allow. The next
release reserves the key and deletes that one compile-time override.

A current Mars bundle carries `provenance.network_source`; an authored value remains
`allow`/`deny`, while `network_source: unset` uses the one effective default accessor and
becomes allow. A bundle without `network_source` is from Mars before 0.18: Meridian warns that Mars before 0.18 does not carry `network`, projects no portable network setting, and leaves the
harness's native default in place. Missing provenance is therefore not the same state as a
current bundle's unset value.

`danger-full-access` cannot enforce deny on any harness. Codex `read-only` cannot enforce
explicit allow. Claude Linux allow reaches host services through its proxy only when clients
bypass `NO_PROXY` (for curl, `--noproxy ''`). OpenCode, Pi, and Cursor share one unenforced
network implementation and one default report path rather than parallel harness blocks.

**Supersedes** the 2026-10-22 rejection of a first-class `network` profile field in
[bundle-field-consumption.md](bundle-field-consumption.md). Current-Mars unset changes both
Claude `read-only`/`workspace-write` and Codex `workspace-write` from their native network-off
defaults to allow, so the user-facing upgrade note explains `network: deny` as the way to
preserve offline behavior. The exact Mars dependency bump is a separate pre-merge step.

## Open questions

- **`subagents:` membership is not enforced at spawn creation.** A spawn-capable profile can
  launch any registered agent. The delegation floor prevents that child from being less
  confined, but does not restrict the choice to the caller's declared list. Membership remains
  separate issue #742.
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
PID-namespace probes (`spawn:p9096`–`spawn:p9107`). The gate A security review
(`spawn:p9112`) found the missing delegation boundary; live floor verification is recorded for
`spawn:p9139`–`spawn:p9147`. Claude host-localhost and proxy behavior comes from probes recorded
in the same work item's research notes. The exact-lock correction is unmerged meridian-cli
commit `9517c8e25`; the floor is `d5e7b8781` plus bypass correction `f45c88b5f`.
Phase 3 and its gate fixes are on the combined unmerged branch through `b7393fe4c`; the
correctness and maintainability gates were `spawn:p9242` and `spawn:p9243`. See
`work:harness-sandbox-network/verification.md#gate-b-fixes-final-verification--2026-10-23`
for final commands, old-Mars behavior, live network checks, and executable restoration.
