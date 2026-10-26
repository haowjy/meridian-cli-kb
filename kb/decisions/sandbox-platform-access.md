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

**Status:** shipped in meridian-cli v0.14.0 (2026-10-25), which requires Mars >= 0.18.0
for the network bundle object. The grants, escapes, compound-launch refusal and child
guardrail came in PR #750; network projection, allowlists and host-set launch flags in the
#752 stack. The unmerged Mars schema v1 branch deletes the network policy this page relies
on; see [Schema v1 network conflict](#schema-v1-network-conflict).

**Who decided what.** The owner ruled that Meridian owns its CLI's permission surface. On the
network the owner set the direction in their own words and agreed to the lead's proposals,
with hedges: sandboxed shells offline by default but `danger-full-access` online, a way to
set a project default or a host allowlist, adding it to Mars and Meridian now, and merging
settings across configuration surfaces (quotes in
[D-network-field](#d-network-field) and
[D-permission-set-merge](tool-permissions.md#d-permission-set-merge)). Those are current
choices the owner agreed to, not invariants. Everything else on this page is the implementing
lead's current choice, made after probes and reviews: the child guardrail, and every
network-side choice beneath the owner's direction (the three-mode field shape, limiting the
merge rule to permission sets, the bundle object, how each harness projects a host set, and
keeping user harness config out of the guardrail). The owner was asked about the guardrail
and about user harness config and has not answered; none of these lead choices is
owner-approved.

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
| Nested child authority | A Meridian-launched constrained caller cannot select a less-confined child or a network policy outside the caller's projected host policy | Same when the caller was launched by Meridian; a direct Codex session has no floor | Same floor, without a sandbox escape |
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
> option, the easiest to loosen; the rejected alternatives below remain open. The network
> comparison, and the choice to leave user harness config out of it
> ([below](#guardrail-network-inputs)), are equally unapproved.

**Current choice:** a caller Meridian launched under `read-only` or `workspace-write` carries
a guardrail in its caller record: sandbox, spawn capability, and the caller's projected
network policy. The network value round-trips the exact allow and deny host sets. Nested
create, continue, fork and their dry runs refuse:

- any launch when the caller lacks spawn capability;
- a child whose sandbox is weaker than the caller's (`--yolo` included);
- a child whose projected network is not contained by the caller's: the child must keep
  every caller deny, even when the caller allows everything else, and a caller with a host
  allowlist requires the child's allow set to be a subset by canonical host-string identity
  (wildcards are not expanded for containment). A child that allows nothing is contained by
  any caller;
- harness-native passthrough arguments that could replace Meridian's resolved policy.

Approval, workspace roots and `--env` values are not compared. Top-level, `default` and
`danger-full-access` callers carry no floor. A refusal reports both compact host policies so
the caller can remove extra allowed hosts, preserve required denies, or select a stricter
child profile.

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
network floor, so an upgrade does not retroactively constrain an already-running caller.

**Rejected:** declared roles (enforce `subagents:` membership, issue #742, and forbid widening
flags; a declared child could still be less confined), accepting and documenting the escape,
and an earlier adversarial frame that also compared approval, workspace write roots and loader
environment keys.

### Network inputs: projected policy only {#guardrail-network-inputs}

**Current choice:** the network compared for both caller and child is a pure, harness-owned
function of inputs Meridian owns completely: the sandbox, the resolved network policy, and the
profile's native-config override. One function per harness returns both the native settings
the launch emits and this projected policy, so the launch and the guardrail cannot disagree.
Claude user settings and Codex `config.toml` files (user and project, including parent
directories) can widen what the harness actually enforces. Meridian detects the widening keys
it knows about on a best-effort basis and names the file in a dry-run and launch warning. That
warning never changes the projected policy, the report line or the guardrail. Meridian does
not edit or override users' harness files
([D-claude-settings-files-kept](tool-permissions.md#d-claude-settings-files-kept)). The keys
checked are listed in meridian-cli `src/meridian/lib/harness/AGENTS.md`.

**Why:** the guardrail's job is the one stated above: stop an agent widening a child through
Meridian's CLI. The user's harness config applies to caller and child alike, so it gives the
agent no new CLI lever, and an agent that rewrites files is #747's problem. Feeding that config
into the guardrail turned it into a *prediction* of harness enforcement, and that prediction
needs an open-ended model of how each harness loads config. Each review round found a layer
the model missed: a Claude user proxy port bypasses the domain filter and its denies;
Codex user domains apply only while its proxy runs; a trusted project's `.codex/config.toml`
widens the map; Codex also reads `.codex/config.toml` from parent directories. Every missed
layer failed open. A whole-system review called this frame rot and the lead accepted the
verdict.

**Rejected:**

- *Exact modelling of user config* (compute the widened host sets the harness will enforce).
  It tracks harness internals that change every release, and each missed layer fails open.
- *Detect, then assume open* (any widening key makes the effective network allow-all). It used
  one value for two jobs that need opposite errors: a child must be over-approximated and a
  caller's floor under-approximated. One user `allowedDomains` entry turned an offline
  caller's floor fully open, which let it launch an online child.

The option taken is the "warn only" one the lead offered the owner as recommended; the owner
has not answered.

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

## D-network-field: modes, host sets and sandbox-derived default {#d-network-field}

**Current choice (owner-agreed, 2026-10-24):** `network` is `allow`, `deny`, or `allowlist`,
with a separate allow set and deny set of hosts. It governs the agent's shell only: built-in
web tools stay under `tools: web`. A project default and a launch flag exist alongside the
profile field. Leaving the mode unset follows the sandbox:

| Sandbox | Unset `network` means |
|---|---|
| `read-only`, `workspace-write` | offline, projected exactly like an authored `deny` |
| `danger-full-access` | online: full access includes the network |
| unset or `default` (harness-native) | Meridian projects nothing; the harness decides |

The owner rejected the earlier default with "'no profile in the packages wants its shell
offline' - not true, there's tons of reasons to have it offline", and gave the audiences: "I
generally want to allow network access to most of my agents, some, like explorer i dont want
network access, but then I think most companies if they used this, wouldn't want network
access by default". On the lead's first proposal (unset is offline everywhere): "but... it
depends, danger full access does mean network shoudl be alloweed". On a project default:
"maybe there should be a way to set default net work on, or an allowlist of domains or
something in mars.toml? or meridian.toml? or somewhere else?", then "I think we should add in
this stuff to mars + meridian etc." The three-mode shape, the host grammar and the flag
names are the lead's.

**Why sandboxed means offline:** a sandbox is a confinement request, and there are many
reasons to keep a confined shell offline: exfiltration by a prompt-injected agent reading
untrusted content, accidental side effects (pushes, API calls, local services), supply-chain
installs, and reproducibility. The default serves both audiences the owner named: the
owner's leads, coders, reviewers and probers run under `danger-full-access` and stay online,
explorer stays offline with no edit, and the sandboxed roles that need the network declare
it. "No network keys" is not enough for offline: Claude with no network keys under `--dangerously-skip-permissions` reached the network, so
unset projects the same keys as `deny` (Claude `strictAllowlist`, Codex
`network_access=false`).

**Rejected:**

- *Unset means `allow`* (the 2026-10-23 choice). It rested on "no packaged profile wants its
  shell offline", a premise the owner rejected, and it fails open for the adopters who most
  need the sandbox.
- *Unset follows the harness default* (offline on Claude and Codex, online on OpenCode and
  Pi). The same profile would be online on one route and offline on another.
- *Unset treats `read-only` and `workspace-write` differently.* It ties two independent axes.
- *An explicit `network: default` value.* Policy layers take the first set value, so a
  higher layer's `default` would be a second spelling of the default that could reopen a
  profile's `deny`.
- *Host lists inside `network`* (`network: [github.com]`) *with a project list that
  replaces lower lists.* Superseded by separate host sets that combine across surfaces
  ([D-permission-set-merge](tool-permissions.md#d-permission-set-merge)).
- *A project host list as a ceiling profiles can only narrow.* A new resolution rule, and the
  consumer already controls overlays. Revisit if companies need packages that cannot widen.
- *The project default in `meridian.toml`.* A second policy owner outside Mars's resolution,
  bundle and provenance. The default lives in Mars `[settings] default_network`.

**Where the rules live.** Mars resolves the mode by precedence and the host sets by the
permission-set merge rule. Profile keys, the settings and user-config keys, the host grammar
(exact host, IP, `*.x` subdomains only, `**.x` apex plus subdomains, following Codex) and the
CLI flags are in Mars `docs/config/agent-profiles.md` and `docs/config/mars-toml.md`. An
`allowlist` with no hosts resolves to `deny` with a Mars warning. Meridian adds `--network`,
repeatable `--network-allow` and repeatable `--network-deny` to `spawn`, the primary launch
and `bootstrap`, and passes them to Mars. The CLI host flags only add; a narrower one-off list
uses `--network allowlist` with an overlay or profile.

### The bundle object {#network-bundle-object}

**Current choice:** `execution_policy.network` is one object written by one Mars resolver:
`{"mode": "allow"|"deny"|"allowlist"|null, "allow": [hosts], "deny": [hosts]}`. Mars drops
inert sets (both lists under `deny`, the allow list under `allow`). `mode: null` means unset;
the host sets still travel because a project `network-deny` applies whatever the mode
resolves to. Meridian parses the object once, at the bundle boundary, into
`Allow(deny)` or `Hosts(allow, deny)`, where `Hosts(∅)` is deny. A mode-unset value stays
on the bundle result until Meridian's own sandbox precedence (`primary.sandbox`) is final,
and one function resolves it; the durable execution policy holds only resolved values. The
report, the native projector and the guardrail all read that one value.

**Rejected:** a mode string plus a separate `network_domains` field (two fields that can
pair invalidly), and a single tagged value in the bundle (`"deny"` or `{allow, deny}`), which
cannot express an unset mode that still carries host sets. One resolver writing the object
keeps the no-invalid-pairing property. An old Mars bundle without
`provenance.network_source` is a hard composition error naming the required Mars version and
telling the user to upgrade Meridian (which prefers its bundled Mars over `PATH`). Warning
and falling back to harness defaults was rejected: both fail safe, and the error leaves one
code path instead of two.

#### Schema v1 network conflict {#schema-v1-network-conflict}

> [!FLAG] **Needs human review**
> The two sides disagree and the owner has not ruled. Mars's schema v1 work (branch
> `feat/mars-schema-v1`, planned for mars-agents 0.20.0, unmerged) deletes Mars's network
> policy (commit `75bfd5f`, its decision D12) on the premise that Meridian discards it.
> Shipped Meridian enforces profile `network: allow | deny` (#752) and refuses any launch
> bundle without `provenance.network_source` (`bundle_adapter.py`). A Mars binary built
> from that branch emits no network object, so `meridian spawn` and primary launches fail
> against it. One side has to move: restore the network policy in schema v1, or have
> Meridian read the profile's `network` itself and drop the bundle requirement. Until then
> no Meridian release can pin a schema v1 Mars, which also holds back
> [provided skills](launch.md#installing-bundled-skills-through-mars-decided-not-shipped).

### Projection {#network-projection}

| Projected policy | Claude | Codex | OpenCode, Pi, Cursor |
|---|---|---|---|
| `Allow(∅)` | Under `read-only`/`workspace-write`, allow all destinations and local services | `workspace-write`: `network_access=true` | Already unrestricted |
| `Allow(deny)` | As above plus `deniedDomains` | Network proxy with `"*"="allow"` plus deny entries; local services stay reachable, so partial | Denies not enforced, reported |
| `Hosts(allow, deny)` | Strict allowlist with `allowedDomains` and `deniedDomains`; `**.x` becomes `x` plus `*.x` | `workspace-write`: network proxy with one inline domain map; local services stay reachable, so partial | Not enforced, reported |
| `Hosts(∅)` | Strict allowlist with no external hosts | `workspace-write`: `network_access=false` | Not enforced, reported |

Codex `read-only` stays offline and reports a non-empty allow request unsupported.
`danger-full-access` cannot enforce restrictions and reports that; Mars's earlier warning for
`deny` under `danger-full-access` was deleted because reporting what cannot be enforced is
the projection's job, and with a project default it would fire on every unsandboxed launch.
A harness with a sandbox that cannot represent a host list fails closed (offline), never open.
For a finite Claude policy, `allowLocalBinding` and `allowAllUnixSockets` are explicitly false
unless `localhost` or `127.0.0.1` is allowed; either host turns on Claude's coarse switches for
all local binding and Unix sockets. On Linux, host localhost is reached through Claude's proxy
(`curl --noproxy ''`). Per-harness detail is in meridian-cli
`src/meridian/lib/harness/projections/.context/CONTEXT.md`.

The projected policy is a pure, harness-owned function of sandbox, the typed policy and native
config. Approval, tool rules and the user's own harness config are deliberately not inputs;
see [guardrail network inputs](#guardrail-network-inputs).

**Codex passthrough key.** For one warning release, a profile's
`harness-overrides.codex.sandbox_workspace_write.network_access` boolean wins over the
portable field when Meridian can emit it as a `-c` pair; anything else is reported ignored and
the portable value stands. The next release reserves the key. See
[native-config](../concepts/native-config.md).

**Upgrade impact:** a constrained sandbox with no network mode is offline, including a
primary whose `sandbox` comes from Meridian config. Profiles that need shell network declare
`network: allow` or an allowlist. Continued sessions are re-resolved under this default.

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
Because a sandboxed profile is now offline unless it says otherwise, the same follow-up adds
`network: allow` to the sandboxed roles that need the shell network: dev-workflow architect and
web-researcher (after the stopgap revert), creative-writing web-researcher, kb-maintainer,
imagegen and autoresearch-lead. web-prompt-researcher is `read-only`, which Codex cannot put
online. Whether the microct profiles need it is an open question for the owner. Whether other
roles that use `danger-full-access` only for network or spawning should move to
`workspace-write` is a separate, later decision.

## Open questions

- **`subagents:` membership is not enforced at spawn creation.** A spawn-capable profile can
  launch any registered agent. Issue #742.
- **macOS and Cursor** are untested for every grant here. Claude Code's documentation says
  `allowLocalBinding` fixes host localhost on macOS; not verified.

## Harness facts these decisions rest on

Linux, Claude Code 2.1.309, Codex CLI 0.163.0; the domain-proxy POC used Codex 0.164.0.
Recheck when either harness's sandbox or proxy changes.

| Fact | Claude | Codex |
|---|---|---|
| OS sandbox covers | Bash only. Edit and Write follow tool rules; WebFetch and WebSearch run outside | Every shell command, run through a login shell (`bash -lc`) |
| Detached child survives the shell call | No | No in sandbox; yes with full access |
| Per-command escape | `excludedCommands`, simple commands only | Execution-policy `prefix_rule` with `allow` |
| Sandbox PID namespace | Different from the launcher; an excluded command stays in the launcher's | Different from the launcher; an allowed command runs outside it |
| Shell network switch | Domain allowlist through a proxy; `strictAllowlist` for a hard deny | Workspace-write boolean plus a domain proxy map; localhost remains reachable under the proxy |
| Read-only writes | Everything except roots Meridian denies | Nothing; `--add-dir` is ignored |
| User config that widens a launch's network | User `allowedDomains` widen a launch's `strictAllowlist` (denies still win); a user `httpProxyPort`/`socksProxyPort` routes around the domain filter and its denies. Project settings files did not widen | User and trusted-project `config.toml` `features.network_proxy.domains` merge into the launch's map while the proxy runs; Codex also reads `.codex/config.toml` from parent directories |

## Provenance

`work:harness-sandbox-network` (design, plan, research per harness, verification). Owner
rulings: `chat:c8765`, 2026-10-23; owner direction on the network and settings merge:
`chat:c8788` (tech-lead `spawn:p9046`), 2026-10-24; owner quotes mined in `spawn:p9319`. Structure review `spawn:p9047`; Codex rules POC
`spawn:p9048`; Claude escape and Phase 1 runs `spawn:p9060`–`spawn:p9068`; Codex rule and
confirmation probes `spawn:p9087`–`spawn:p9089`; PID-namespace probes
`spawn:p9096`–`spawn:p9107`. Guardrail: gate A security review `spawn:p9112`, second round
`spawn:p9153`/`spawn:p9154`, whole-system review `spawn:p9223`. Network gates:
`spawn:p9242`/`spawn:p9243`, then `spawn:p9246`/`spawn:p9247`. Allowlist design and evidence:
`spawn:p9263`–`spawn:p9266`, Mars implementation `spawn:p9268`, Meridian implementation
`spawn:p9271`. Final gate `spawn:p9275`–`spawn:p9278`; user harness config probes
`spawn:p9281`, review rounds `spawn:p9285`, `spawn:p9299`; whole-system frame-rot review
`spawn:p9298`; report-only implementation `spawn:p9300` (meridian-cli `aed703fec`), Mars
tool-warning rule `spawn:p9301` (mars-agents `97ff6e9`). Mars validation scope:
`spawn:p9058`, `spawn:p9080`.
