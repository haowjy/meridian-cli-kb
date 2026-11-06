# meridian-base

Base supplies shared workers and domain-neutral coordination skills. It has no lead: choosing a lead's roster is a domain decision. See the package README for its current inventory and aliases.

Meridian-specific operation guidance versions with the CLI, while base names capabilities that other runtimes can also provide. See [bundled skills](../../decisions/launch.md#d-bundled-meridian-skills-meridian-owned-operations-guidance-versions-with-the-cli) and [capability mapping](../../decisions/launch.md#d-capability-command-mapping).

A profile with only scoped shell allows is an allowlist: ordinary read commands also need explicit permission. This has caused read-only agents to be denied basic inspection commands.
