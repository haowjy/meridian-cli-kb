# Prompt Packages Overview

Package manifests and READMEs own current dependencies and inventories. See [package management](../../concepts/package-management/overview.md) for resolution and targeting.

## Prompter Is Local, Not a Foundation

meridian-base and meridian-dev-workflow declare meridian-prompter as a local dependency: authors can use it in those checkouts, but consumers do not receive it unless they declare it. Prompter depends on base for selected skills.

Dependency pins can exclude a new minor release, so a base minor requires checking and updating dependent packages. A package that drops content now supplied by Meridian must require the Meridian version that supplies the replacement; otherwise its consumers lose that capability. See [bundled skills](../../decisions/launch.md#d-bundled-meridian-skills-meridian-owned-operations-guidance-versions-with-the-cli).

- [meridian-base](meridian-base.md)
- [meridian-dev-workflow](meridian-dev-workflow.md)
- [meridian-prompter](meridian-prompter.md)
