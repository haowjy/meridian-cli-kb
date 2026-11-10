# Sandbox projection and Meridian access

A sandboxed child gets the current project's runtime and configured roots, plus the narrow control-plane access Meridian itself needs; never the whole `~/.meridian` home. Projecting all of it would give the child sibling projects' state and lock namespaces. Profile permissions are separate ([tool permissions](tool-permissions.md)). Current projections and gaps: meridian-cli `docs/agent-profiles.md` and `src/meridian/lib/harness/projections/.context/CONTEXT.md`.

## Purpose-scoped paths

- **The exact project lock, not the lock directory.** Meridian materializes the project's lifetime lock before starting a sandboxed child, and nested processes open it read-only. Granting the shared directory would let the child create lock names for other projects.
- **Missing context roots are left out.** Codex's bind mount of a missing path can fail the whole execution namespace, including the commands that could diagnose it.
- **Cross-project maintenance is explicit** and gated at the library boundary, since optional global configuration and shared coordination may be unreachable from a child. Autosync is best effort: failing to take a lock is a skip, never a reason to grant the global lock directory. An unreadable implicit user config is treated as absent with a warning; an explicitly requested one fails visibly.

## Spawning from inside a sandbox

A detached `meridian spawn` worker must outlive the shell that started it, but a harness sandbox kills it at shell exit. So when the profile can spawn, the standalone command escapes the sandbox, and the escaped process is unsandboxed, network included. The child guardrail below is not adversarial confinement.

- **Claude** excludes `meridian spawn *` from its Bash sandbox. The escape matches simple commands only, so on Linux a nested spawn compares PID namespaces and refuses before creating state if a compound command is still sandboxed. Without procfs or the caller record (macOS included) it can't detect this.
- **Codex** has no per-launch rules layer, so Meridian installs a user-wide Codex rule for spawn-capable sandboxed launches. It also lets read-only children signal completion, and it applies to every Codex session using that home. Project rules were rejected: they depend on project trust, and Mars regenerates that directory.
- **What an escaped command runs.** It must not load an executable an agent could have replaced. The launching build's optional `meridian` shim lives under the user home, not in a projected writable root; hooks and extensions call `meridian` by name, so a variable alone would miss them. Re-execution uses Python's `-P` to drop the working directory from the import path. `PYTHONPATH` and project configuration remain separate inputs.

## The child guardrail

A sandboxed caller carries its sandbox, spawn capability and projected network policy into its environment. A nested launch is refused when it lacks spawn capability, asks for a weaker sandbox or a wider network policy than the caller's, or passes native arguments that could replace resolved policy. Approval, workspace roots and environment values are not compared. The rule stops widening through the Meridian CLI; it doesn't confine an adversarial agent, since escaped processes can read agent-writable configuration and other inputs ([#747](https://github.com/haowjy/meridian-cli/issues/747)). Enforcing the caller's `subagents:` list alone wouldn't prevent escalation, because a permitted child can itself have a weaker sandbox; membership enforcement is separate ([#742](https://github.com/haowjy/meridian-cli/issues/742)).

The guardrail compares only policy projected from Meridian-owned inputs. User harness configuration can widen real enforcement, and known widening settings are warned about without entering the comparison. Modelling user configuration exactly would track changing harness internals and fail open on any missed layer; treating every detected widening as allow-all could turn an offline caller's floor into an online one.

> [!FLAG] **Needs human review**: the guardrail and its network inputs are provisional; whether a sandboxed spawner may launch a less-confined child is undecided. Flagged 2026-11-07.

## Network policy

Shell network access is separate from built-in web tools. With no explicit mode, read-only and workspace-write sandboxes default offline, full access defaults online, and an unset or native-default sandbox leaves the harness in charge. Network policy is a portable Mars profile field, so it isn't duplicated in harness overrides and host sets merge across layers ([permission sets](tool-permissions.md#permission-sets-add-across-layers)). Mars resolves it once and the bundle carries mode and host sets together, so an unset mode can still carry denies. Unsupported or partial enforcement is reported, and a sandbox that can't express a requested host set fails closed.
