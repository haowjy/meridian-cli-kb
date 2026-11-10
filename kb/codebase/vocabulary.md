# Codebase vocabulary

- **Context** can mean configured filesystem paths or the composed launch context; the surrounding usage distinguishes them. See [context resolution](../architecture/state-system/work-items.md).
- **Ghost spawn** is ambiguous: a pre-worker stall has a published row; an orphan artifact directory has no row and must not be recreated by late writers. See [spawn lifecycle](../concepts/spawn-lifecycle.md) and [published artifact lifetime](../architecture/state-system/spawn-state.md#published-spawn-artifact-lifetime).
- **Capability** in Mars tool policy names a harness-neutral group of tools. This differs from the package wording and spawn delegation capability in the [shared vocabulary](../vocabulary.md).
- **Sandbox** confines what an allowed tool can touch; tool rules govern which calls may run, and approval governs when to prompt. Harness support differs. See [tool permissions](../decisions/tool-permissions.md).

Implementation types and their fields are defined in the source checkout's colocated guidance. See [harness abstraction](../concepts/harness-abstraction.md), [hooks and plugins](../concepts/hooks-and-plugins.md), and the [shared vocabulary](../vocabulary.md).
