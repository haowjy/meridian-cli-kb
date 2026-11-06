# Skill Loading

`skills.load` gives an agent a skill from the first turn; `skills.available` points to a skill it can load later. `model-invocable` controls discovery, not an explicit load. This lets a profile load a role-specific skill without advertising it to every model. A non-invocable skill listed as available may be named without being loadable through the harness's discovery mechanism.

Mars resolves skill bodies and harness variants once for the launch bundle. Meridian composes that result; exact continuation reuses its recorded snapshot. Inline delivery supplies the body, while experimental named delivery asks the agent to read the file. See [launch decisions](../decisions/launch.md), [package management decisions](../decisions/package-management.md), and `src/meridian/lib/launch/.context/CONTEXT.md`.

OpenCode 2's black-box fallback resuming another agent's session can still miss the prompt; see [OpenCode prompt delivery](../decisions/opencode-prompt-delivery.md).
