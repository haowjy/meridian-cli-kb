# Composition Pipeline

Content is classified by meaning before harness-specific projection. Keeping system instructions, task context and the user's request distinct lets each adapter select its native channels without making composition depend on harness identity. The raw request remains distinct from rendered context so replay does not compound previously composed content. See [launch decisions](../architecture/launch-system.md) and the launch source guidance.

## Agent Inventory Prompt

Mars renders the agent inventory and delegation preference into the launch bundle. Meridian embeds it rather than maintaining a second inventory renderer. See [launch decisions](../architecture/launch-system.md).
