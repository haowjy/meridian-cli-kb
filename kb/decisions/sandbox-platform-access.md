# Meridian Access Inside a Harness Sandbox

Meridian projects its control-plane needs separately from profile permissions. Current harness projections and their gaps are in meridian-cli `docs/agent-profiles.md` and `src/meridian/lib/harness/projections/.context/CONTEXT.md`.

## Project lock {#d-sandbox-state-roots}

Materialize the exact project lifetime lock before starting a sandboxed child; the child can then acquire it without write access to the shared lock directory. Granting the directory would let it create lock names for other projects.

## Claude spawn escape {#d-sandbox-spawn-escape}

A standalone `meridian spawn` can leave a Meridian-projected Claude sandbox when the profile can spawn. The sandbox kills detached children at shell exit, so the worker otherwise cannot finish. The escape is limited to spawn commands and explicitly projected sandboxes; the escaped process itself is unsandboxed, including for network. The child guardrail below is not adversarial confinement.

## Compound launch refusal {#d-sandbox-compound-refusal}

On Linux, a nested spawn compares PID namespaces and refuses before creating state when a compound shell command remains inside the sandbox. Claude's escape matches simple commands only. Where procfs or the caller record is unavailable, including macOS, the check cannot detect this condition.

## D-delegation-floor: escaped child launches get a CLI guardrail {#d-delegation-floor}

> [!FLAG] **Needs human review.** The guardrail and its network inputs are provisional;
> whether sandboxed spawners may launch less-confined children remains open.

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

Warnings leave the projected policy unchanged.

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

**Evidence:** with the matching line removed from `default.rules`, a
workspace-write parent launched a child that finished, and a read-only agent ran
`meridian spawn done` with exit 0 and received the confirmation turn.

**Rejected:** a project `.codex/rules/` file (Mars regenerates that directory on sync, and
Codex loads project rules only for trusted projects), and a per-invocation `-c` key (Codex
has none for rules).

## D-escape-loads: what an escaped `meridian spawn` runs stays out of the agent's reach (2026-11, in review) {#d-escape-loads}

**Status:** in review, and everything in this section describes the branches, not
main. `fix/launching-cli-bin` fixes issue #764 and `fix/safe-module-launch` adds `-P`;
neither is merged or has a PR yet. Tech-lead current choices. The owner's only word on #764 was "ok" after the lead explained that Pi
extensions run `meridian` by name (`chat:c10170`); they have not seen the shim's shape
or `-P`. Mechanism, on those branches: `src/meridian/lib/launch/.context/CONTEXT.md`
("The Launching Build on the Child PATH") and `src/meridian/lib/platform/AGENTS.md`.

[The escape](#d-sandbox-spawn-escape) runs `meridian spawn` outside the sandbox, so
anything that process loads from a path the agent can write runs unsandboxed. Both
changes below close one such route. Neither is confinement: the
[guardrail's limit](#d-delegation-floor) still stands, and #747 still owns that.

**The shim lives under the user home, because every nearer place is writable.** When
`meridian` on a harness's PATH is not the build that launched it, the launcher prepends
a directory holding only a `meridian` shim, `~/.meridian/bin/<hash>/meridian`. A probe
first recommended `runtime_root`, reasoning that #747 already made that root writable
and a shim there made nothing worse. Review reversed it: `runtime_root` and `/tmp` are
projected writable, and both Codex's rule and Claude's exclusion run `meridian spawn`
outside the sandbox, so a writable shim would let a sandboxed agent run anything
unsandboxed. The user-home directory is readable and executable but not writable,
checked with bwrap directly, not inside a harness's own sandbox. If `MERIDIAN_HOME` falls inside the working directory, a projected
root or `/tmp`, the launch warns and leaves PATH alone.

**A shim, not `MERIDIAN_CLI_BIN`.** Hooks and extensions reach `meridian` by name,
including Mars-installed hooks Meridian does not own. A variable fixes only the callers
patched to read it. Exporting both adds a second mechanism with nothing left for it to
fix. A venv `bin/` was rejected too, because it would also change `python` and `pip`
for the agent.

**A dead shim fails loud.** If the interpreter is gone, `meridian` exits 127. Falling
back to the next `meridian` on PATH was tried in the probe and silently ran the old
build, which is the bug this fixes.

**`-P` covers the implicit working-directory entry and not `PYTHONPATH`.** `python -m`
puts the working directory first on `sys.path`, and the background worker's working
directory is the project, so a `meridian/` directory there ran in place of the package.
That was reproduced in a plain shell with the planted file running in the worker. `-P`
closes that entry, on every re-exec through `module_argv` and in the shim. An empty or
relative `PYTHONPATH` entry still resolves to the working directory:
`PYTHONPATH=. meridian --version` ran the planted file in the console script itself,
which never reaches `module_argv`. `-I` was rejected because it drops user-site
installs; `-E` keeps them and is undecided.

**Open, for the owner (#876):** whether environment and project config count as
sandbox-reachable. Other routes remain, none tried in a real sandbox: `PYTHONPATH`, the
hooks in `meridian.toml`, and a project-local `.venv` holding the build. A dev build run
from a workspace writable to its own sandbox has the same exposure through the shim. If
these count, the fix belongs at the `meridian spawn` rule rather than at each route.

**Not verified.** Nobody has run any of this inside a real harness sandbox. Whether
Codex's rule still matches when `meridian` resolves to the shim is expected but
untested, because the rule matches the command the shell saw. The shim pins the
interpreter, not the build, so `uv tool upgrade` under a live session moves it. Login
shells rebuild PATH and drop the shim; Pi's did in the probe, and interactive Claude
2.1.319 keeps it. The shim writes `-P` by hand; it should use `module_argv` once both
branches land.

## Network policy {#d-network-field}

Network policy controls shell access separately from built-in web tools. With no explicit mode, Meridian-projected read-only and workspace-write sandboxes default offline; full access defaults online, and an unset or native-default sandbox leaves the harness in charge. A sandboxed role that needs network must request it. Host allow and deny sets merge as [permission sets](tool-permissions.md#d-permission-set-merge).

### Bundle {#network-bundle-object}

Mars resolves the portable policy once; the bundle carries mode and host sets together so an unset mode can still carry denies. Meridian resolves the sandbox-derived default before projection.

#### Portable schema {#schema-v1-network}

Network stays a portable Mars profile field. Per-harness native overrides would duplicate authors' policy and prevent host-set merging across configuration layers.

### Projection {#network-projection}

Harnesses differ: Codex host policies may still allow local services, and OpenCode, Pi and Cursor do not enforce these restrictions. Unsupported or partial enforcement is reported; a sandbox unable to express a requested host set fails closed. User harness config can widen actual enforcement, so the CLI guardrail compares only Meridian's projected policy and warns about known widening settings.

## Profile follow-up (not done)

Released in meridian-base v0.11.14 and meridian-dev-workflow v0.15.7:
architect and web-researcher run on `danger-full-access` as a stopgap for network and child
spawns, and session-miner's allowlist gained the read commands it lacked. Once both
meridian-cli branches ship, the plan is to put architect and web-researcher back on
`workspace-write`, drop session-miner's hand-listed `meridian` reads, and delete scoped
`bash(meridian spawn *)`-style allows from lead profiles, which the platform grants now cover.
Because a sandboxed profile is now offline unless it says otherwise, the same follow-up adds
`network: allow` to the sandboxed roles that need the shell network: dev-workflow architect and
web-researcher (after the stopgap revert), creative-writing web-researcher, kb-maintainer,
imagegen and autoresearch-lead. web-prompt-researcher is `read-only`, which Codex cannot put
online. Whether the microct profiles need it is an open question. Whether other
roles that use `danger-full-access` only for network or spawning should move to
`workspace-write` is a separate, later decision.

## Open questions

- **`subagents:` membership is not enforced at spawn creation.** A spawn-capable profile can
  launch any registered agent. Issue #742.
- **macOS and Cursor** are untested for every grant here. Claude Code's documentation says
  `allowLocalBinding` fixes host localhost on macOS; not verified.

