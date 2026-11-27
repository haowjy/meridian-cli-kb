# Ecosystem

Meridian coordinates agents; [Mars](mars/package-management.md) resolves prompt packages and projects them into harness-native formats; meridian-comms carries work between Slack and Meridian sessions. Package manifests and READMEs own current dependencies and inventories.

## Prompt packages

- **meridian-base** supplies shared workers and domain-neutral coordination skills. It has no lead, because choosing a lead's roster is a domain decision. It names capabilities other runtimes can also provide, while Meridian-specific guidance ships with the CLI ([launch system](launch/launch-system.md#skills-and-guidance)).
- **meridian-dev-workflow** builds on base with the development roles and workflow.
- **meridian-prompter** holds the agents and skills for writing, reviewing and testing what agents read; its `/prompt-principles` skill owns prompt craft, and Meridian-specific artifact formats ship with the CLI so harness changes don't change the general doctrine.

Base and dev-workflow declare prompter as a local dependency: authors can use it in those checkouts, but consumers don't receive it unless they declare it. Prompter depends on base for selected skills.

Dependency pins can exclude a new minor release, so a base minor release means checking and updating the packages that depend on it. A package that drops content Meridian now supplies must require the Meridian version that supplies the replacement, or its consumers lose that capability.

## meridian-comms

[meridian-comms](https://github.com/haowjy/meridian-comms) is a Slack bot for handing work to someone's agent sessions and getting the answer in the same thread. Its daemon wakes the sessions a person names. A message that names nobody wakes nobody: it is filed in Meridian's machine inbox, and an optional router offers it to one session that is working or idle on a warm cache. What it does and why: its `docs/design.md`; the inbox's own reasons are on [the machine inbox](inbox.md).

**It uses Meridian only through the public CLI.** It parses the JSON of Meridian's commands (the list is in its design's "Client and daemon" section), learns of a new inbox message from an `inbox.received` hook row the user adds to `~/.meridian/config.toml`, and writes under `~/.meridian/` only by running those commands. A change to that output, to the hook and its `MERIDIAN_INBOX_MESSAGE_ID`, or to the `activity` and `idle` fields of a primary's row in `spawn list --global --json` changes a program outside this repository, which is why such changes need the owner's approval (meridian-cli's root `AGENTS.md`).

**It runs on any Meridian, and without one.** Waking sessions works with any current Meridian; filing and routing need 0.28, the first release with the machine inbox. The daemon asks at start whether Meridian has the `inbox` command, and without it files and routes nothing: a message that woke nobody waits in meridian-comms's own pending list. With no Meridian at all it still reads and posts Slack threads, and every such message waits there. The owner drew the split so that meridian-comms stays usable without meridian-cli, so everything that needs sessions (waking, filing, offering, routing) sits in its Meridian source, the one module that runs `meridian`.

**Meridian owns the inbox and the offer; meridian-comms owns the router.** The inbox is files under `~/.meridian/` that every session on the machine can read with nothing else installed, so a message between two sessions never has to pass through Slack. The router passes `meridian inbox offer` only a message ID and a session, and Meridian sends its own fixed notice, so nothing a sender wrote reaches a command line or the notice. The router asks Jev, TypeSafe AI's decision model served through OpenCode, which answers by picking one of the options it is given; that needs an API key. Meridian calls no model itself, and a router inside it would put a model client and a key on every machine that has an inbox. The daemon also holds the Slack thread, where it reports who was offered the message. The cost: Meridian without meridian-comms has an inbox and no router, unless the user writes an `inbox.received` row that runs `meridian inbox offer`.

**Only the router offers; a person's choice wakes.** A choice reaches the session even when it is cold. An offer is a guess: Meridian refuses an offer to a cold session, to a spawned child, and to any session outside the message's project, and its notice tells the session to do nothing if the message isn't its own. A person who named a session meant it, so refusing it for being cold, or inviting it to ignore the message, would defeat the instruction. Meridian and meridian-comms keep one cost rule: nothing wakes a cold session on a guess, because waking one re-reads its whole context; a person may still choose to.

**The router's second step is meridian-base's `inbox-router` profile.** When Jev picks no session with enough confidence, the daemon spawns that profile in a Meridian project it keeps for it. The agent may run only Meridian's read commands, and its report's last line is the whole contract: `offer <session>`, `none`, or `ask <question>`, which the daemon posts in the Slack thread. meridian-comms carries out the decision and reads any other line as `none`. The profile pins `harness: opencode`, because its model is paid through the machine's OpenCode sign-in. Changing its last line, its tools or its harness changes the router.

## Domain packages

Creative-writing-skills applies the package model to long-form writing. microct-analysis (a study-design, analyst and code-reviewer workflow) and Jupyter Workbench (reusable notebook execution and visualization) live in the voluma repository.

> [!FLAG] **Needs human review**: the microCT measurement and toolkit notes below are domain science, not Meridian knowledge; they have no other durable home yet. Move them to the voluma repository's `workflows/microct-analysis/` docs, or confirm they stay here. Flagged 2026-11-10.

### microCT measurement and toolkit notes

Femoral surface measurements and tibial slice measurements are different domains; treating both as Euclidean distances produced reported errors of 11–58% against published values. IIOC height is a slice count, and frontal-projected femoral width depends on the orientation plane. Amira display lower bounds and SCANCO quantitative thresholds are not interchangeable.

The published method uses manual tibial orientation. PCA alignment is an approximation and needs independent validation, because asymmetric anatomy can rotate its axes. Label-mask resampling needs nearest-neighbor interpolation to avoid mixed labels. An OA6-1RK run that put all six geometric indices within a ±10% gate (largest deviation 2.89%) still had femoral landmarks visibly misplaced when a tibia-aligned frame left the femur about 23.8° skewed; the package's `docs/findings/2026-06-04-femoral-orientation-and-visual-landmarking.md` holds the evidence.

Watershed needs distinct markers: flood-filling seeds through connected, bridged bones produces overlapping markers that can't separate them, while small markers placed from visual evidence work. Examining several planes can expose a separation hidden in one view. Record the coordinate frame and transform with every point: a slice or seed in an oriented volume isn't an original-volume coordinate until transformed.

### Jupyter Workbench

CLI, MCP and Python share one service layer so notebook and session behavior can't diverge between interfaces; domain packages interpret interaction events and the workbench keeps them generic. A persistent kernel runs both computation and visualization, avoiding a second daemon and competing session state. Replay is explicit because re-running prior cells can be expensive, fail or change state, and a recovered kernel may need visualization set up again. Working notebooks are append-only to keep exploratory provenance; derivation produces a cleaner notebook without erasing it, and session mutation is serialized so concurrent clients don't conflict.
