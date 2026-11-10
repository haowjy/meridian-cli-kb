# Meridian access inside a harness sandbox

Meridian projects its control-plane needs separately from profile permissions. Current harness projections and gaps are in `docs/agent-profiles.md` and `src/meridian/lib/harness/projections/.context/CONTEXT.md` in meridian-cli.

## Project lock

Materialize the exact project lifetime lock before starting a sandboxed child. Granting the shared lock directory would let the child create lock names for other projects.

## Claude spawn escape

A standalone `meridian spawn` can leave a Meridian-projected Claude sandbox when the profile can spawn: the sandbox otherwise kills the detached worker at shell exit. The escaped process is unsandboxed, including for network. The child guardrail is not adversarial confinement.

## Compound launch refusal

Claude's escape matches simple commands only. On Linux a nested spawn compares PID namespaces and refuses before creating state if a compound command remains sandboxed. The check cannot detect this without procfs or the caller record, including on macOS.

## Delegation guardrail

> [!FLAG] **Needs human review.** The guardrail and its network inputs are provisional; whether sandboxed spawners may launch less-confined children remains open.

A sandboxed caller carries its sandbox, spawn capability and projected network policy. Nested launches refuse missing spawn capability, a weaker child sandbox, a child network policy that widens the caller's, and native arguments that could replace resolved policy. Approval, workspace roots and environment values are not compared. The rule prevents widening through the Meridian CLI; it does not confine an adversarial agent. Escaped processes can consume agent-writable configuration and other inputs. [#747](https://github.com/haowjy/meridian-cli/issues/747) tracks confinement.

Enforcing the caller's declared `subagents:` list alone would not prevent escalation: a permitted child can itself have a weaker sandbox. Membership enforcement remains a separate question in [#742](https://github.com/haowjy/meridian-cli/issues/742).

### Network inputs: projected policy only {#guardrail-network-inputs}

The guardrail compares policy projected from Meridian-owned inputs. User harness configuration can widen actual enforcement, and known widening settings are warned about without changing the comparison. Exact modelling of user configuration was rejected because it depends on changing harness internals and a missed layer fails open. Treating every detected widening as allow-all was also rejected: it could turn an offline caller's floor into an online one.

## Codex child spawn

Meridian installs a user-wide Codex rule for spawn-capable, sandboxed launches because Codex has no per-launch rules layer. It lets standalone `meridian spawn` escape the shell sandbox so detached workers survive and read-only children can signal completion. The rule also applies to other Codex sessions using that home. Project rules were rejected because they depend on project trust and Mars regenerates the directory; Codex offers no per-invocation rules option.

## Escaped process inputs

An escaped command must not load an agent-writable replacement executable. The launching build's optional `meridian` shim lives under the user home rather than a projected writable root. Hooks and extensions invoke `meridian` by name, so a variable alone would miss callers. Re-execution uses Python's `-P` to exclude the implicit working-directory import path; `PYTHONPATH` and project configuration remain separate inputs, and this is not adversarial confinement. See `src/meridian/lib/launch/.context/CONTEXT.md` and [#876](https://github.com/haowjy/meridian-cli/issues/876).

## Network policy

Shell network access is separate from built-in web tools. With no explicit mode, projected read-only and workspace-write sandboxes default offline; full access defaults online, and an unset or native-default sandbox leaves the harness in charge. Host allow and deny sets merge as [permission sets](tool-permissions.md#permission-set-merging).

### Bundle {#network-bundle-object}

Mars resolves the portable policy once; its bundle carries mode and host sets together, allowing an unset mode to carry denies.

#### Portable schema {#schema-v1-network}

A portable Mars profile field avoids duplicating policy in harness overrides and permits host-set merging across configuration layers.

### Projection {#network-projection}

Harnesses differ in what they can enforce. Unsupported or partial enforcement is reported; a sandbox unable to express a requested host set fails closed. User harness configuration can widen actual enforcement, so the guardrail compares projected policy and warns about known widening settings.
