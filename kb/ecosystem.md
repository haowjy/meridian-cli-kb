# Ecosystem

Meridian coordinates agents; [Mars](mars/package-management.md) resolves prompt packages and projects them into harness-native formats. Package manifests and READMEs own current dependencies and inventories.

## Prompt packages

- **meridian-base** supplies shared workers and domain-neutral coordination skills. It has no lead, because choosing a lead's roster is a domain decision. It names capabilities other runtimes can also provide, while Meridian-specific guidance ships with the CLI ([launch system](launch/launch-system.md#skills-and-guidance)).
- **meridian-dev-workflow** builds on base with the development roles and workflow.
- **meridian-prompter** holds the agents and skills for writing, reviewing and testing what agents read; its `/prompt-principles` skill owns prompt craft, and Meridian-specific artifact formats ship with the CLI so harness changes don't change the general doctrine.

Base and dev-workflow declare prompter as a local dependency: authors can use it in those checkouts, but consumers don't receive it unless they declare it. Prompter depends on base for selected skills.

Dependency pins can exclude a new minor release, so a base minor release means checking and updating the packages that depend on it. A package that drops content Meridian now supplies must require the Meridian version that supplies the replacement, or its consumers lose that capability.

## Domain packages

Creative-writing-skills applies the package model to long-form writing. microct-analysis (a study-design, analyst and code-reviewer workflow) and Jupyter Workbench (reusable notebook execution and visualization) live in the voluma repository.

> [!FLAG] **Needs human review**: the microCT measurement and toolkit notes below are domain science, not Meridian knowledge; they have no other durable home yet. Move them to the voluma repository's `workflows/microct-analysis/` docs, or confirm they stay here. Flagged 2026-11-10.

### microCT measurement and toolkit notes

Femoral surface measurements and tibial slice measurements are different domains; treating both as Euclidean distances produced reported errors of 11–58% against published values. IIOC height is a slice count, and frontal-projected femoral width depends on the orientation plane. Amira display lower bounds and SCANCO quantitative thresholds are not interchangeable.

The published method uses manual tibial orientation. PCA alignment is an approximation and needs independent validation, because asymmetric anatomy can rotate its axes. Label-mask resampling needs nearest-neighbor interpolation to avoid mixed labels. An OA6-1RK run that put all six geometric indices within a ±10% gate (largest deviation 2.89%) still had femoral landmarks visibly misplaced when a tibia-aligned frame left the femur about 23.8° skewed; the package's `docs/findings/2026-06-04-femoral-orientation-and-visual-landmarking.md` holds the evidence.

Watershed needs distinct markers: flood-filling seeds through connected, bridged bones produces overlapping markers that can't separate them, while small markers placed from visual evidence work. Examining several planes can expose a separation hidden in one view. Record the coordinate frame and transform with every point: a slice or seed in an oriented volume isn't an original-volume coordinate until transformed.

### Jupyter Workbench

CLI, MCP and Python share one service layer so notebook and session behavior can't diverge between interfaces; domain packages interpret interaction events and the workbench keeps them generic. A persistent kernel runs both computation and visualization, avoiding a second daemon and competing session state. Replay is explicit because re-running prior cells can be expensive, fail or change state, and a recovered kernel may need visualization set up again. Working notebooks are append-only to keep exploratory provenance; derivation produces a cleaner notebook without erasing it, and session mutation is serialized so concurrent clients don't conflict.
