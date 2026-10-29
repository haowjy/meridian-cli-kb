# KB Change Log

Tracks structural changes to this knowledge base — new pages, reorganizations, and content migrations.

---
## 2026-10-29 — spawn identity v2 capture (meridian-cli v0.18.0, `release/v0.18.0`, unmerged to `main`)

New [spawn launches](architecture/state-system/spawn-launches.md) page: one ID
across launches, state v4, per-launch directories, the reopen and its fence,
usage across launches with cumulative-harness seeding, pinned waits,
`owed_launches`, launch-keyed ledgers, bare-ID collect/ack, older builds on one
runtime. New decision [D-continue-reopens](decisions/spawn-identity.md#d-continue-reopens)
with owner and build rulings, departures from D-spawn-identity-v2, and rejected
alternatives; D-spawn-identity-v2's status and PR table updated.
[Derived status](architecture/state-system/derived-status.md) rewritten to
current truth (stored status with one holder write, report-first derive rule,
dead-launch writers, the repair pass, cancel on the lock). Deleted mechanisms
removed from [spawn finalization](architecture/spawn-finalization.md),
[spawn lifecycle](concepts/spawn-lifecycle.md) and
[spawn state](architecture/state-system/spawn-state.md): the authority lattice,
`mark_finalizing`, `queued`/`finalizing` as written statuses, `runner_exit` as a
written field, `signal_canceller.py`, reserve-before-prep. Delivery pages
(child-result delivery, Pi coordination and vocab, wait barrier) moved to the
new ledger names and launch keys. Superseded notes on D-reserve-before-prep,
D-REF-5, the lattice-era state decisions, and the rejected index-candidate
repair (reversed for the background pass). Index-version notes moved to v13.
Lessons: old-build premises, fake-harness timing, CLI output as a schema,
check base before ruling. One correction occurrence added.

## 2026-10-29 — Mars 0.25 portable hooks capture

New [Mars portable hooks](decisions/mars-portable-hooks.md) decision page: why
hooks join Mars's import → model → render pipeline in Claude's syntax, why every
rule lives in `mars-schema`, Codex parser checks, validation by origin with kept
hooks, per-event fragment override, the 0.25/0.26 split, live harness hook facts
with versions, and the tmux-guard quoting hazard. Indexed in `index.md`,
`decisions.md` and `decisions/overview.md`; linked from release process and
package management. [Release process](decisions/release-process.md) checked against
the owner's 0.25 rulings: no change needed.

## 2026-10-28 — post-0.19 follow-ups capture (meridian-cli 0.17.0–0.17.2, Mars 0.23–0.24)

New decisions D106 (OpenCode plugin agents keep the default model), D107 (the user
Mars `config.toml` layer), D108 (Meridian reads what Mars wrote) in
[package management](decisions/package-management.md); D-no-flaky-tests in
[testing](decisions/testing.md); D-minor-means-breaking and D-integration-branch in
[release process](decisions/release-process.md). D100's OpenCode clause revised:
OpenCode plugins now carry agents. Plugin loading research updated for OpenCode
`ctx.agent` and the Codex recheck. Test determinism: wait deadlines, signal timeouts,
and a raised-timeout remedy removed. Stale passages fixed after #795/#796/#797/#805
and Mars 0.24: Codex `network_access`, hook deprecation window, `--include-archives`,
agent-copy detection, MCP file paths, deleted symbols, `[defaults]`/`[primary]`
model keys, old "unmerged" status lines. New vocabulary row separating the three
"user config" locations. Corrections: two classes promoted (release labels, prompt
edits routed to coders), candidates added.

---
## 2026-10-28 — One ref parser and `locate()` (meridian-cli `refactor/ref-resolution`, #800)

New [ref classification](decisions/ref-classification.md) page: D-ref-by-shape and
D-locate-projections. New
[D-owner-on-start](decisions/chat-spawn-ownership.md#d-owner-on-start). Revised
D-chat-names-own-session (every spawn command now asks who is live in a chat; the
known gap and its revisit condition removed), the PR 2 race passage and PR status
table in [spawn identity](decisions/spawn-identity.md), and the reply address in
[primary inject](decisions/primary-inject.md). Status lines that said #801 was
unmerged now say v0.16.4. Pages naming the deleted `session_exists` (hook event
contract, work items) now name the chat record. The **Chat owner** vocabulary
entry, session-state, the `-f` reference page (points to the new page) and a
corrections candidate updated.

---
## 2026-10-28 — Chat→spawn ownership and cross-project messaging (meridian-cli `feat/spawn-message-peers`)

New [chat-spawn-ownership](decisions/chat-spawn-ownership.md) page: D-chat-names-own-session
(a chat ID never names a child; owner ruling) and D-link-at-start (writers link each
generation to its spawn at start; one record-walk reader). New
[D-spawn-message-address](decisions/primary-inject.md#d-spawn-message-address); the
D-spawn-message-frame text no longer says `-C` drops the sender, and its reply names the
chat. Session-state's `session_identity` paragraph, the vocabulary (**Chat owner**) and
the corrections register (found-bug-deferred class promoted from candidate) updated.

---
## 2026-10-27 — Meridian hosts named in full under `harness-overrides` (`work:agent-definition-design`)

New [D105](decisions/package-management.md#d105-meridian-hosts-own-full-name-namespaces-under-harness-overrides-2026-10-27):
`harness-overrides.meridian-cli` and `harness-overrides.meridian-flow` replace the short
`meridian` and `flow` namespaces (Mars 0.22.0, mars-agents#248), and meridian-cli reads
only `meridian-cli` from 0.17.0 (#790). The `resident-rearm-budget` and `capabilities`
passages in [launch](decisions/launch.md), [spawn lifecycle](concepts/spawn-lifecycle.md)
and [drain plans](architecture/drain-plans.md) now name the new key instead of an
unreleased move.

---
## 2026-10-27 — Removed the duplicate principle catalogs (`work:post-0-19-followups`)

Deleted the two principle pages and their now-empty overview after the owner
reviewed all 49 rules against this bar: keep a rule only when a cold agent
cannot see the hazard in code, a real bug or live constraint stands behind it,
and the code holds it today. The review kept 0 rules: 32 duplicated repo-owned
documentation, 16 were enforced by code or wrong/stale, and 1 moved. The
per-spawn cost rule remains detailed in [per-operation cost](architecture/per-operation-cost.md);
repo-wide constraints belong in the meridian-cli root `AGENTS.md` and the
colocated `AGENTS.md` / `.context/CONTEXT.md` files. Inbound references now
point to those owners or to the KB pages that own the surviving rationale.

---
## 2026-10-27 — Deleted the op-layer identity rule (`work:post-0-19-followups`)

Removed the Op Layer Identity Invariant from the former invariants page and
"Op Layer Purity" (old §15) from the former design-principles page;
later principles renumbered. The rule came from the single `cancel-all` fix and never
matched the code: its own mechanism, `runtime_context(ctx)`, falls back to
`RuntimeContext.from_environment()`, and no MCP or REST caller supplies identity, so it
guarded nothing. Owner ruling: delete rather than amend. The live hazard (env deciding what
a store holds) stays where it is stated: meridian-cli `docs/configuration/environment.md`.

## 2026-10-27 — Release process; post-0-19 follow-ups shipped (`work:post-0-19-followups`)

New [release process](decisions/release-process.md) decision page for meridian-cli and
mars-agents: the release job as the only `CHANGELOG.md` writer and the #238 misfile behind
it, the guard that skips a release run whose merge an earlier release already contains,
unlabeled merges cut an RC, no merge queue or required checks by owner choice, and the open
question on stable notes after RCs. The release-sequencing lesson keeps only the #783
lesson and links there. [Resolution algorithm](concepts/package-management/resolution-algorithm.md)
no longer names the removed `discover_source()`/`discover_fallback()`: discovery is split
between `mars-schema` preparation (agents, skills) and `src/discover/` (hooks, bootstrap
docs); stale line ranges replaced by function names. Status sweep now that meridian-cli
#786 (v0.16.0) and mars-agents #233 (0.20.4) and #238 (0.20.5) shipped: about forty
"unmerged" passages rewritten as current behavior, with the released-build alternatives
deleted, and the D98/D99 index rows marked released. [Skill loading](concepts/skill-loading.md)
records the experimental `named` skill delivery (v0.16.1) and the A/B experiment that
decides the default.

---
## 2026-10-27 — Agent overlays live in Mars config; Mars 0.20 status sweep

Rewrote [D72](decisions/model-resolution.md#d72-agentsname-overlay-config-as-the-project-scoped-override-surface)
as superseded in part: `[agents.<name>]` overlays moved from `meridian.toml` to
`mars.toml` / `mars.local.toml` with the 2026-05 Mars routing consolidation, and their
model-policy rules prepend to the profile's instead of replacing them. The
[config precedence](concepts/config-precedence.md) rule-list paragraph now matches (an
empty overlay list no longer suppresses profile rules). Pruned the overlay follow-ups in
[future work](open-questions/future-work.md) that Mars ownership or schema v1 closed.
Marked D-permission-set-merge shipped (Mars 0.18.0), recorded the live Codex check of
effort `none` and `max`, noted the plugin docs shipped in Mars 0.19.0, and added
unreleased-move notes where pages name `meridian-capabilities` or
`resident-rearm-budget` (moving under `harness-overrides.meridian`).

---
## 2026-10-27 — Per-execution work links, stable system prompts, packaged MCP servers (`work:post-0-19-followups`)

meridian-cli PR #786 (closes #768) and mars-agents PRs #233 and #238, all open and
unmerged. New [per-execution links](decisions/session-work-links.md) decision page
(six records) and [D-stable-system-prompt](decisions/launch.md#d-stable-system-prompt);
new [D104](decisions/package-management.md#d104-packaged-mcp-servers-install-through-the-hook-directory-path-2026-10-27)
for `${MARS_MCP_DIR}`. New lessons: [a link that spans two stores](lessons/state-design-lessons.md#a-link-that-spans-two-stores),
the third recurrence of [repeated fixes mean parallel owners](lessons/hook-event-contract.md),
and [`git stash` in a worktree](lessons/spawn-lane-operations.md). Rewrote the pages that
described concrete env paths, env-first work resolution, `MERIDIAN_TASK_DIR` as an
inherited input, and a spawn's goal in the system prompt (launch system, reference
resolution, composition pipeline, session initiation, context resolution, work items,
state-system pages). Removed stale content that predates this work: the work-item
rename intent file (five pages), the managed-worktree machinery in
[work items](codebase/work-items.md) (its six decision records marked superseded by
#282), `CompilerContext`, and the dry-run write claim in the sync model. Status lines
of D36, D98 and D99 now say released. Flagged the op-layer env-read invariant, which
the code does not hold. Three correction occurrences added.

---
## 2026-10-27 — Meridian v0.15.0 runs on Mars 0.20.2

PR #783. In [launch decisions](decisions/launch.md#installing-bundled-skills-through-mars)
provided skills are recorded as live (installed by `mars provide`, wheel fallback,
`metadata.mode` gate), and the Mars 0.20 upgrade section is rewritten as shipped: the
0.20.2 pin and why 0.20.0/0.20.1 were skipped, and the `migrate`/`provide` root guard.
New in [model resolution](decisions/model-resolution.md#d-model-policy-rule-beats-profile-default):
the owner's ruling that a profile's own rule beating its top-level effort is a bug fix,
and per-harness effort lowering (`none`, `max`, Claude thinking evidence, unverified
paths). Precedence ladders in [config precedence](concepts/config-precedence.md) and
[model policies](concepts/model-resolution/model-policies.md) now put overlay scalars above
matched rules; schema v1 override fields replace the old table. The CHANGELOG misfile
procedure in [release sequencing](lessons/release-sequencing.md) is replaced by the
PR-body changelog rule. Composition pipeline, vocabulary, decision index and the
agent-profile example updated to match.

---
## 2026-10-26 — `spawn inject` becomes `spawn message`; sender frame; Claude's inbound gate

PR #777 (`5ad479b0a`). New decision
[D-spawn-message-frame](decisions/primary-inject.md#d-spawn-message-frame): the
`[Meridian]` sender line, the reply command only when the sender can receive
one, and the chat id from the sender's record. The command name is updated
across the KB where it names the CLI; "inject" stays where it names the
mechanism. [Inbound controls](research/claude-code-mods-and-inbox.md#inbound-controls)
gained the 2.1.310 gate facts (`accept`/`hold`/`refuse`, the `mode-mismatch` and
`no-mode-asserted` parity holds, headless expiry, settings precedence) and lost
the claim that an auto-mode receiver delivers bare posts: `defaultMode: "auto"`
counts as bypass and holds them. D-claude-primary-inbox decision 3 now names
auto mode as a hold case. The #498 error text is "no message route yet".

---
## 2026-10-25 — Packages name capabilities; Meridian ships its own skills

`work:package-transferability`, shipped in meridian-cli v0.14.3–v0.14.6 and
meridian-base 0.12.0.
[D-bundled-meridian-skills](decisions/launch.md#d-bundled-meridian-skills-meridian-owned-operations-guidance-versions-with-the-cli)
rewritten as shipped: `<name>__meridian` naming, `mode` gating (the
`spawn-only` marker is gone), reads that print the injected-skill header, the
trimmed-judgment lesson, and the owner's words and rejected alternatives.
[D-capability-command-mapping](decisions/launch.md#d-capability-command-mapping)
gained the package-text rules. Unmerged-branch banners removed from
[composition pipeline](concepts/composition-pipeline.md),
[launch system](architecture/launch-system.md) and the launch decisions.
Package pages updated for meridian-base 0.12.0, prompter 0.5.12 and the widened
pins; the parked meridian-benchmark-agents page deleted (its repo README
records how to revive it). Vocabulary gained *bundled skill* and *capability*;
the corrections register gained the `audience`-field occurrence. Mars-provided
skills are recorded as planned only.

---
## 2026-10-25 — Plugin builds, skills lowering, rename naming, depth precedence

`work:harness-plugin-export`. New decision records
[D100](decisions/package-management.md#d100-a-package-publishes-one-plugin-holding-its-full-resolved-graph-2026-10-24)
(one logical plugin per package holding its full resolved graph),
[D101](decisions/package-management.md#d101-agent-skills-lower-by-harness-capability-2026-10-24)
(capability-aware skills lowering),
[D102](decisions/package-management.md#d102-renamed-agents-and-skills-answer-to-their-installed-name-2026-10-24)
(renamed items carry their installed name) and
[D103](decisions/package-management.md#d103-collision-precedence-and-hook-order-use-shortest-dependency-depth-2026-10-24)
(shortest-depth precedence, superseding D36's MCP ranking). New research page
[harness-plugin-loading.md](research/harness-plugin-loading.md). MCP
precedence tables in the [compiler architecture](architecture/mars-compiler.md)
and [compiler pipeline](concepts/package-management/compiler-pipeline.md)
now rank by depth; [self-source selection](concepts/package-management/self-source-selection.md)
records the plugin-tree discovery skip.

Updated after the multi-harness build settled: D100 now records the
three trees for five harnesses, the owner's words and hedges behind the
harness set, the release-time contracts (output confinement, the three
allowlisted root files, raw plugin metadata at load, git rollback) and the
open default-harness question (mars-agents#211).
[harness-plugin-loading.md](research/harness-plugin-loading.md) gained Grok
and live results: Codex plugin MCP is callable but takes env only through
`env_vars`. D101–D103 statuses name `release/v0.19.0` and PR #197. The
compiler page's plugin section now links mars-agents `src/build/` docs
instead of restating them. Source: `work:harness-plugin-export`,
`chat:c9182`, `spawn:p9533`, `spawn:p9535`.

---
## 2026-10-24 — MCP selection joined target state

`work:harness-plugin-export`. [D99](decisions/package-management.md#d99-mcp-selection-joins-target-state-2026-10-24)
extends D98's single-selection-authority rule to MCP manifests: target
construction now carries parsed, visibility-filtered MCP candidates for
preflight and lowering while per-target first-wins collisions stay unchanged.
The [Mars compiler architecture](architecture/mars-compiler.md) and
[compiler pipeline](concepts/package-management/compiler-pipeline.md) now record
the typed visibility boundary, diagnostic-order compatibility constraints, and
the still-open MCP-name exclusion policy.

---
## 2026-10-24 — Hook selection given one authority

`work:harness-plugin-export`. [D98](decisions/package-management.md#d98-the-target-build-is-the-only-hook-selection-authority-2026-10-24)
records target construction as the only hook-selection authority: preflight and
both lowering modes consume its parsed selection. Its temporary MCP exception was
superseded by D99. [Mars compiler architecture](architecture/mars-compiler.md)
and the [compiler pipeline](concepts/package-management/compiler-pipeline.md)
were corrected to separate MCP precedence from hook collision behavior and to
remove the stale claim that `config_entries/resolve.rs` selects hooks.

---
## 2026-10-24 — Child-result fallback reframed around what the caller saw

`work:cancel-consumes-owed`. The owner clarified that spawn-watch and Pi
delivery stand in for a forgotten `spawn wait`; wait-and-ack-only consumption
was a cost choice, not an invariant. [Child result delivery](concepts/child-result-delivery.md)
and [its decisions](decisions/child-result-delivery.md) now record the rule
implemented on unmerged meridian-cli PR #762 (`9ffcaff7c`): cancelled outcomes
and terminal report bodies printed by `spawn show` / `spawn report show` consume
under the caller. Internal operations, active rows, summary-only show,
status/search/session/context-reference reads, and no-caller invocations do not.
The display-path observation is shared, non-blocking, and best effort.

---
## 2026-10-24 — Shell network decisions reconciled with the final design

`work:harness-sandbox-network`. [Sandbox platform access](decisions/sandbox-platform-access.md)
now records why a sandboxed shell is offline by default and the rejected defaults, the bundle
object and its rejected shapes, and the per-harness projection. The guardrail compares
projected policy only; user harness config that widens the network is a report-only warning,
with exact modelling and detect-then-assume-open recorded as rejected. Every network-side
choice beneath the owner's rulings is marked not owner-approved.
[Tool permissions](decisions/tool-permissions.md) gains D-permission-set-merge (the four
permission sets add across surfaces; why the rule stops there) and drops the stale claim that
an overlay tool list replaces the profile's.
[Dogfooding PR builds](lessons/dogfooding-pr-builds.md) records the third `.venv/bin/mars`
overwrite and the `--link-mode=copy` mitigation.

---
## 2026-10-24 — Child-result delivery checked against merged PRs #743 and #751

`work:read-consumes`. [Child result delivery](concepts/child-result-delivery.md),
[spawn wait barrier](concepts/spawn-wait-barrier.md) and
[its decisions](decisions/child-result-delivery.md) now describe owed no-arg wait
as merged rather than branch-only, and link to the spawn ops `.context/` for the
field-level contract instead of restating it. A new table lists the accepted
duplicate and loss windows, including wait consuming before it prints. Who set
each choice is corrected: the owner set header-only delivery and pure reads,
agreed to the shared owed scope, and has not answered whether Pi should go
header-only (flagged open).

---
## 2026-10-23 — Sandbox platform access reconciled against both branches

`work:harness-sandbox-network`. [Sandbox platform access](decisions/sandbox-platform-access.md)
was checked against PR #750 and the stacked network branch and rewritten: the caller record is
`_MERIDIAN_CALLER_BOUNDARY` in `core/caller_boundary.py`, checked only in the operations layer;
the Codex rule and Claude escape need spawn capability and a sandbox Meridian projected; owner
rulings are separated from lead choices, with the child guardrail flagged; Mars's network-only
validation and the owed profile follow-up are recorded.
[Completion drain](decisions/completion-drain.md) now says read-only Codex without spawn
capability still skips the confirmation turn.
[Dogfooding PR builds](lessons/dogfooding-pr-builds.md) records the second `.venv/bin/mars`
overwrite with the new-inode procedure, and Codex's login shell resolving the installed
`meridian`.

## 2026-10-23 — Sandbox platform access decisions

`work:harness-sandbox-network`. New page [decisions/sandbox-platform-access.md](decisions/sandbox-platform-access.md)
records what Meridian grants a sandboxed launch: the exact project lifetime lock created only
during real-launch materialization, the Meridian-commands platform rule, the Claude and Codex
child-spawn escapes and their limits, PID-namespace compound-launch refusal, and the final gate
A delegation guardrail. The guardrail rejects direct CLI widening for constrained callers but
is not adversarial confinement. It also records the unmerged profile `network` field, including
the final gate B effective-network and deprecated Codex-override decisions. [Tool permissions](decisions/tool-permissions.md) and
[sandbox projection](architecture/sandbox-projection.md) were updated to match. The
2026-10-22 rejection of a `network` field in
[bundle field consumption](decisions/bundle-field-consumption.md) is marked superseded.
The earlier writable projection of the shared project-lock directory was removed.

## 2026-10-23 — Prompt-package pages rewritten against current source

`chat:c8796`. All six pages under [ecosystem/prompt-packages/](ecosystem/prompt-packages/overview.md)
described packages as they were months earlier (meridian-dev-workflow 0.3.4, meridian-base 0.10.0):
planner, EARS, qa-lead, the four-lane review gate, a base orchestrator, removed model aliases.
They now link to each package's README for agent, skill and model inventories and keep only
the reasons behind each package's shape, how the packages depend on each other, retired pieces,
and traps. The overview now covers all ten packages and records that meridian-prompter is a
local-only dependency of base and dev-workflow, and that benchmark-agents, autoresearch and
microct-analysis pin versions older than the current releases. Summaries in `index.md`,
`ecosystem/overview.md`, `vocabulary.md` and `codebase/vocabulary.md` (EARS) were updated to
match. `meridian-base.md#hooks` is unchanged.

## 2026-10-23 — Abandoned control sockets and child-interpreter test isolation

`work:stale-control-sockets` (issues #733, #445; branch unmerged as of this date).
[Project identity](architecture/state-system/project-identity.md) says why the shared
`/tmp/meridian-<uid>` directory leaks sockets after SIGKILL and how the next control server
reclaims them with a datagram probe. [Test determinism](codebase/test-determinism.md) gains
"Patches Stop at the Interpreter Boundary": in-process patches do not reach test-launched
Meridian interpreters.

## 2026-10-23 — Completion report contract, channel-less holds, socket anchor

`work:completion-hardening`. New records in [completion-drain](decisions/completion-drain.md):
D-completion-report-contract (#725's real cause; "latest text wins" rejected) and
D-completion-channel (read-only Codex skips the confirmation turn), both unmerged as of this date.
[Drain plans](architecture/drain-plans.md) and [coordination](architecture/completion-drain-coordination.md)
describe the retained report and the channel-less hold. The read-only Codex open question in
[tool-permissions](decisions/tool-permissions.md) narrows to profiles with no `sandbox`. Control sockets:
[project identity](architecture/state-system/project-identity.md) and the state overview now say
`/tmp/meridian-<uid>` and why (PR #735, v0.12.2). [Claude native sessions](architecture/claude-native-sessions.md)
names `claude_sessions.py`/`claude_settings.py` for PR #736.

## 2026-10-22 — Profile tool permissions shipped

meridian-cli v0.12.0 (PR #730) and Mars 0.17.0 (mars-agents PR #179). Status lines that said
"unmerged branch" or "`main` still reads the flat lists" now say shipped, on
[tool-permissions](decisions/tool-permissions.md), [Mars launch bundle](architecture/mars-launch-bundle.md),
[package management](decisions/package-management.md), [launch system](architecture/launch-system.md),
[harness adapters](codebase/harness-adapters.md), [Pi lifecycle](architecture/pi-lifecycle.md),
[Mars feature gaps](open-questions/mars-feature-gaps.md), the index and the decisions table. The owner
closed the three open questions (Claude settings files are not stripped, the OpenCode headless fallback
is kept, the OpenCode native-denial divergence is accepted): see
`D-claude-settings-files-kept` in tool-permissions.

## 2026-10-22 — Bundle field consumption and list-form deprecation captured

`work:session-permissions`, owner rulings of 2026-10-22. New
[decisions/bundle-field-consumption.md](decisions/bundle-field-consumption.md):
every launch-bundle field is applied, reported or deliberately unread;
`harness-overrides` applies on Codex only; compaction thresholds projected per
harness (#548); `tools.mcp` is permission names and Meridian's `mcp_tools`
projection is deleted; Mars drops `codex_rules`. New
[D-permission-list-form-deprecated](decisions/tool-permissions.md#d-permission-list-form-deprecated)
supersedes list = allowlist. Updated: consumed/unread fields, native-config
strategies and harness table in
[architecture/mars-launch-bundle.md](architecture/mars-launch-bundle.md);
projection section and example in [concepts/native-config.md](concepts/native-config.md)
(Claude/OpenCode strategies removed: never applied); D82/D83 in
[decisions/package-management.md](decisions/package-management.md); autocompact
in [concepts/config-precedence.md](concepts/config-precedence.md); skill `tools`
field in [concepts/skill-schema.md](concepts/skill-schema.md) (canonical skills
do not author `allowed-tools`); #548 note in idle-notifications; indexes.

## 2026-10-22 — observed-spawns.json location captured (unmerged branch)

meridian-cli branch `fix/observed-spawns-move` (PR #729) moves the consumption
record from `pi-bash/<parent>/` to `spawns/<parent>/` with a one-release
read-both and Pi mirror-write transition. New
[D-observed-spawns-location](decisions/child-result-delivery.md#d-observed-spawns-location);
location and transition in
[concepts/child-result-delivery.md](concepts/child-result-delivery.md); path
updates in [architecture/pi-runtime/coordination.md](architecture/pi-runtime/coordination.md)
and [architecture/pi-runtime/vocab.md](architecture/pi-runtime/vocab.md).
D-claude-spawn-watch status updated to shipped (v0.11.9, PR #724).

## 2026-10-22 — Profile tool permissions captured (unmerged branches)

meridian-cli branch `feat/session-permissions` and mars-agents PR #179 make profile
`tools`/`sandbox`/`approval` harness-enforced. New
[decisions/tool-permissions.md](decisions/tool-permissions.md) (eight decisions,
rejected alternatives, open owner questions). Updated: the `tools` contract in
[architecture/mars-launch-bundle.md](architecture/mars-launch-bundle.md)
(`native_tools`, `note`, rules-only reader); the bind section of
[architecture/launch-system.md](architecture/launch-system.md) (replaces the Claude
agent-permission injection section); Pi and Claude agent routing in
[codebase/harness-adapters.md](codebase/harness-adapters.md); permission terms in
[codebase/vocabulary.md](codebase/vocabulary.md) (`PermissionConfig` entry replaced);
the OpenCode permission open question removed from
[decisions/opencode-prompt-delivery.md](decisions/opencode-prompt-delivery.md);
stale symbol and status lines in `principles/invariants.md`, `concepts/native-config.md`,
`concepts/harness-abstraction.md`, `architecture/pi-lifecycle.md`, D83 in
`decisions/package-management.md`, and `open-questions/mars-feature-gaps.md`. Owner corrections
from `chat:c8514` added to [corrections.md](corrections.md).

## 2026-10-21 — Claude spawn watch captured (unmerged branch)

meridian-cli branch `feat/claude-spawn-watch` adds a Claude mod that delivers a
lead's un-waited background results. New
[concepts/child-result-delivery.md](concepts/child-result-delivery.md) (all three
routes and the shared `observed-spawns.json` record) and
[decisions/child-result-delivery.md](decisions/child-result-delivery.md)
(D-claude-spawn-watch). 2.1.307 mod-hook facts added to
[research/claude-code-mods-and-inbox.md](research/claude-code-mods-and-inbox.md);
new lesson in [lessons/harness-integration.md](lessons/harness-integration.md);
cross-links from the wait-barrier, idle-adapter and Pi coordination pages.

## 2026-10-21 — Session browse start-from merged (PR #716)

meridian-cli PR #716 (merge `d09656f5c`) added `n` to start a new session `--from` a
picker row. New [D-browse-start-from](decisions/session-browse.md#d-browse-start-from)
replaces the follow-up bullet; [codebase/session-operations.md](codebase/session-operations.md)
and [decisions.md](decisions.md) updated.

## 2026-10-21 — Session browse redesign merged (PR #714)

meridian-cli PR #714 merged the session browse redesign (merge `471dfbc59`). Pages
checked against `main`: branch-status lines replaced with the merged state; the
time column is named IDLE (time since last turn) in
[decisions/session-browse.md](decisions/session-browse.md),
[codebase/session-operations.md](codebase/session-operations.md) and
[decisions.md](decisions.md); history-index pages now say `main` is at v12, and the
version-collision flag on
[D-session-per-chat-journals](decisions/state.md#d-session-per-chat-journals) now
says identity-v2 PR 4 and #418 B2 need numbers above 12.

## 2026-10-21 — Session browse redesign captured (unmerged branch)

Capture of the settled `meridian session browse` redesign
(`work:session-browse-redesign`, meridian-cli branch `feat/session-browse-redesign`,
not merged).

- New [decisions/session-browse.md](decisions/session-browse.md): picker columns,
  transcript on demand, staleness from idle state, one matcher before `LIMIT`,
  primary-only lineage view, Python before Rust.
- New decisions: [D-lineage-conversation-identity](decisions/state.md#d-lineage-conversation-identity)
  and [D-from-request-boundary](decisions/launch.md#d-from-request-boundary); history
  index v12 flagged against the v11/v12 lane reservations.
- Preview projection marked superseded in [decisions/history-storage.md](decisions/history-storage.md)
  and [decisions/native-only-history.md](decisions/native-only-history.md); preview
  material removed from [portable history](architecture/state-system/portable-history.md),
  [native transcript reads](architecture/native-transcript-reads.md),
  [session-log rendering](codebase/session-log-rendering.md) and
  [session state](architecture/state-system/session-state.md), which also gains the
  chat-owned fields kept across generations.
- [codebase/session-operations.md](codebase/session-operations.md) browse section
  rewritten; `prompt_is_composed` legacy section replaced in
  [composition pipeline](concepts/composition-pipeline.md);
  [continuation model selection](concepts/continuation-model-selection.md) corrected
  (plain continue has preferred the observed last-used model since 2026-09-19).
- Lessons: the `pkill -f` incident in [spawn lane operations](lessons/spawn-lane-operations.md);
  the parallel-owners recurrence in [hook event contract](lessons/hook-event-contract.md).

## 2026-10-21 — OpenCode prompt delivery captured (unmerged branch)

Capture of the settled OpenCode prompt-delivery fix (`work:long-view-review`,
meridian-cli branch `fix/opencode-v2-prompt`, not merged).

- New [decisions/opencode-prompt-delivery.md](decisions/opencode-prompt-delivery.md):
  D-opencode-config-agent, D-opencode-explicit-agent, D-opencode-model-off-agent.
- New [research/opencode-config-semantics.md](research/opencode-config-semantics.md):
  probed OpenCode 2.0.29 / 1.19.3 behavior.
- [concepts/skill-loading.md](concepts/skill-loading.md),
  [codebase/harness-adapters.md](codebase/harness-adapters.md),
  [concepts/composition-pipeline.md](concepts/composition-pipeline.md) and
  [decisions/launch.md](decisions/launch.md): OpenCode delivery gap replaced by the
  config-agent channel, with `main` behavior stated until merge; "needs human review"
  flags on the gap removed.
- [decisions/model-resolution.md](decisions/model-resolution.md),
  [decisions/launch-process-ownership.md](decisions/launch-process-ownership.md),
  [architecture/launch-system.md](architecture/launch-system.md) and the decision
  index: OpenCode model-conflict restart marked superseded.

## 2026-10-19 — v0.11.0 capture: shipped hook contract, permission notice, compatibility policy, idle running work

Post-release pass against `main` at v0.11.0 (`work:hook-event-names`,
`work:permission-ping`, `work:versioning-policy`, `work:idle-other-harnesses`).

- New [decisions/compatibility-versioning.md](decisions/compatibility-versioning.md):
  D-compat-policy (replaces "no backwards compatibility"), D-version-only-where-read,
  D-staged-version-add, D-check-before-deciding, D-index-named-by-inputs,
  D-idle-absent-exception. New vocabulary terms: Deprecation window, Format version.
- [decisions/hook-event-contract.md](decisions/hook-event-contract.md): status
  shipped; ten events with `notify.needs_permission`; new D-hook-deprecation-window
  supersedes rejection at load; Plugin API version string removed. D-hook-event-names
  anchor renamed; inbound links updated.
- New D-idle-running-work-reporters in
  [decisions/idle-notifications.md](decisions/idle-notifications.md): Pi and
  OpenCode 2.x report running work, Codex and OpenCode 1.x don't; the Codex
  self-continue report is recorded as an open question.
  [architecture/idle-adapters.md](architecture/idle-adapters.md) gains the Pi and
  OpenCode 2.x mechanisms, the reconnect re-check, and the over-cap `other` rule.
- "Unmerged" status lines removed from the hook, telemetry, vocabulary, guide,
  work-items, meridian-base, idle and decision-index pages; `session.needs_permission`
  replaced with `notify.needs_permission`.
- History index file names (`history-v<N>-sessions-v<S>.sqlite3`) updated across
  the state-system pages; format-version handling added to session-state,
  roots-and-work-items and telemetry reader pages.
- Also fixed against code: work attachment storage and rename in
  [codebase/work-items.md](codebase/work-items.md), `spawn_model` in the hook
  context table, the plugin_api path in the former invariants page, and the
  no-backward-compat premise in the former design-principles page.
- Corrections register: three occurrences for claimed-without-checking (canaries
  on the wrong path, the goal-mode guess) and one for fixing-only-the-instance.
- [open-questions/future-work.md](open-questions/future-work.md): #700, #702, #701.

---
## 2026-10-18 — Hook event contract captured from PR #688 (`work:hook-event-names`)

PR #688 (unmerged, fixes #687) renames the hook events and deletes
`failure_policy`. Pages describe the branch and say so in their status lines.

- New [decisions/hook-event-contract.md](decisions/hook-event-contract.md):
  D-hook-event-names, D-work-activation (flagged: primary-launch activation
  was never owner-confirmed), D-autosync-default-events,
  D-hook-failure-policy-removed, D-hook-fact-owners. Indexed in
  [decisions.md](decisions.md) and [decisions/overview.md](decisions/overview.md).
- New [lessons/hook-event-contract.md](lessons/hook-event-contract.md):
  accepted names need a producer (the producer-coverage test); repeated fixes
  mean parallel owners.
- [lessons/probe-runtime-placement.md](lessons/probe-runtime-placement.md):
  an inherited `MERIDIAN_PROJECT_DIR` sent a scratch-home probe's `work`
  commands to the real project.
- Renamed events and removed `failure_policy` / event classes in
  [hooks-and-plugins](concepts/hooks-and-plugins.md),
  [work-items](codebase/work-items.md), [vocabulary](codebase/vocabulary.md)
  (new Spawn kind, Work activation, Hook event spec; Hook event class
  removed), [telemetry event catalog](architecture/telemetry/event-catalog.md)
  and [overview](architecture/telemetry/overview.md) (also the missing
  `spawn.timed_out`), [idle-notify-channels](decisions/idle-notify-channels.md),
  [idle-notifications](decisions/idle-notifications.md),
  [idle-adapters](architecture/idle-adapters.md) and
  [meridian-base](ecosystem/prompt-packages/meridian-base.md).
- [corrections.md](corrections.md): one occurrence on the
  instance-only-fix class, two new candidates.

---
## 2026-10-18 — Post-merge check of the idle push and notification pages against v0.10.28 (`work:kb-pass-notify-hook-events`)

The pages from KB PR #1 (`c49e51f`) were checked against the shipped code
(v0.10.28, `d230dc400`). The decision records matched the code; the fixes are
wording left from the branch rounds and older errors on the same pages.

- [idle-adapters](architecture/idle-adapters.md): the permission-prompt
  fragility row no longer reads as if #684 shipped.
- [idle-notifications](decisions/idle-notifications.md): "as on main" and
  "#673 ships" rewritten as before/after v0.10.27; OpenCode `/compact`, not
  `/summarize`; two structural-debt rows now match their FUTURE files.
- [idle-notify-channels](decisions/idle-notify-channels.md): `config show`
  hook summary as shipped; CLI and skill released together (v0.10.28,
  meridian-base v0.11.7); password-file mode check.
- [hooks-and-plugins](concepts/hooks-and-plugins.md): `spawn.start` and
  `work.start` are never dispatched (not aliases), so git-autosync's
  `spawn.start` default never fires; `priority` runs higher first within a
  source; `when.status` values; the actual plugin API surface; autosync
  artifacts under `<runtime_root>/autosync/<hash>/`; builtins take
  `project_root` from `HookContext`.
- `codebase/guide.md`, `codebase/vocabulary.md`: the plugin API carries
  notification types, not delivery.
- Superseded compaction rows marked in [decisions.md](decisions.md); the
  `idle-opencode-v2` index rows mark its compaction confirmation as history.
- `AGENTS.md`: KB PRs merge without asking the owner, and a merged KB PR gets
  one `@kb-lead` pass against the shipped code.

---
## 2026-10-18 — Idle push and notifications merged: PR #685 (#673, v0.10.27) and PR #686 (#670, #666, v0.10.28) (`work:notify-hook-events`)

Status lines in `decisions.md`, D-idle-push-in-flight, D-idle-compaction-removed
and D-notify-hook-events now say merged.

---
## 2026-10-18 — Idle push and notification branches brought to final behavior (#673, #670, #666; `work:notify-hook-events`)

The cloud session's pages described the branch at its last push. A takeover
session split the work into two stacked branches (#673 idle, then #670/#666
notifications) and fixed review findings in four rounds; these pages now
describe the result.

- [D-idle-push-in-flight](decisions/idle-notifications.md#d-idle-push-in-flight--the-push-waits-for-work-that-will-wake-the-session-the-warning-does-not):
  the last-command-segment certainty rule (loops, newlines, redirections,
  comments, line continuations, `lib/cli_args.py`), `spawn wait --timeout`,
  the owed response until a push is delivered, the mod's 16 × 512 cap,
  permission prompts holding the push as on main, the two withdrawn
  prompt-tracking designs, and #684. Verification gap replaced by the live
  probe. Status and the cloud session's user quotes marked session-record.
- [D-idle-compaction-removed](decisions/idle-notifications.md#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions):
  status (ships with #673; user quotes session-record) and upgrade cost.
- [D-notify-hook-events](decisions/idle-notify-channels.md#d-notify-hook-events--notifications-are-hook-events-delivered-by-routes):
  one owner for hook-row validation, fallback keeps valid rows, channel names
  checked at load, `RouteReport` vs `SendReport`, `HookResult.details`, the
  all-`none` note is CLI-only, `hooks run` one-action rule, #178 kept,
  multi-line title split, ntfy partial delivery, `--attach` dedupe.
- [architecture/idle-adapters.md](architecture/idle-adapters.md): running-work
  table and parser, mod caps, Claude live verification, two new fragility rows
  (permission-prompt hold, mod mid-turn push), notify skill location.
- [concepts/hooks-and-plugins.md](concepts/hooks-and-plugins.md): layering now
  `default < user < project < local` (the stale "context config" layer is gone),
  validation owner, `hooks run` one-action rule, `HookResult.details` and
  `supported_events`, route results, the companion exception to the plugin API
  boundary, `hooks` commands under `ignore_env`.
- New lessons:
  [`pytest-llm` is not the full suite](lessons/verification-and-review-discipline.md#pytest-llm-is-not-the-full-suite),
  [check a claim about main against main's code](lessons/verification-and-review-discipline.md#check-a-claim-about-main-against-mains-code),
  [a fresh Claude config dir hides the owner's hooks](lessons/harness-integration.md#a-fresh-claude-config-dir-hides-the-owners-hooks).
  `codebase/guide.md` no longer calls `pytest-llm` the automated suite.
- Index rows updated in [decisions.md](decisions.md) and [index.md](index.md).

---
## 2026-10-17 — Notifications as hook events, #670 (`claude/notify-bash-events-gguu11`)

- New [D-notify-hook-events](decisions/idle-notify-channels.md#d-notify-hook-events--notifications-are-hook-events-delivered-by-routes)
  in `idle-notify-channels.md`, indexed in [decisions.md](decisions.md) and
  [decisions/overview.md](decisions/overview.md). D-idle-leaf-packages and
  D-idle-notice-context no longer describe a channel registry, direct
  `notify.send` calls or `--title`.
- [concepts/hooks-and-plugins.md](concepts/hooks-and-plugins.md): the three
  notification events, default `notify` routes, the `notification` context and
  env, a new section on the built-in `notify` route and command-hook channels,
  hooks without a runtime root, plugin API 2.0.0.
- Plugin API version and notify code locations updated in
  `codebase/guide.md`, `codebase/vocabulary.md` and
  `architecture/idle-adapters.md`.

---
## 2026-10-17 — Idle compaction removed; running work shapes the push (`claude/notify-bash-events-gguu11`)

- [architecture/idle-adapters.md](architecture/idle-adapters.md) now describes
  push and warn only: the compaction actuator column, `Compaction worth`
  section, actuator contracts and compaction fragility rows are gone; new
  [Running work and the push](architecture/idle-adapters.md#running-work-and-the-push)
  covers Claude's `classic.Stop` → `idle arm --background-tasks` and core's
  `wake.py`. `Hosts and actuators` is now `Hosts`.
- [idle-compaction-actuator](decisions/idle-compaction-actuator.md) and
  [idle-compaction-growth-floor](decisions/idle-compaction-growth-floor.md)
  carry superseded banners pointing to
  [D-idle-compaction-removed](decisions/idle-notifications.md#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions).
- Current-tense compaction claims fixed in `idle-notifications.md` (problem,
  invariants, guard list, config, revisit), `idle-notify-channels.md`,
  `idle-opencode-v2.md`, `idle-ttl-detection.md`, `codebase/harness-adapters.md`,
  `architecture/launch-system.md`, `concepts/config-precedence.md`,
  `codebase/test-determinism.md`, `research/prompt-cache-retention.md`, and the
  index rows.

---
## 2026-10-18 — KB brought to `main` after identity-v2 PR 3 merged (#672, v0.10.24; `work:v010-followups` D77–D86)

The previous capture wrote PR 3 as awaiting merge and left pages describing the
guessing reaper. They now describe `main`: status from `read_derived`, a dead
runner as `failed`/`crashed` (or `cancelled` with a cancel request),
`finalize_dead_launch` as the one dead-launch writer, plain `doctor` warning
`dead_spawns_unfinalized`, and `--kill-orphans` persisting.

- **Retired** `architecture/state-system/reconciliation.md`: its decision tree,
  read projection and heartbeat check are deleted. Its one live part, the
  reaper's cleanup claims, moved to
  [derived status](architecture/state-system/derived-status.md#the-reapers-cleanup-claims);
  inbound links repointed there.
- **Rewritten to `main`:** [derived status](architecture/state-system/derived-status.md)
  (banner, continue skips the orphan pass, `cancel-all`),
  [spawn lifecycle](concepts/spawn-lifecycle.md) (crash recovery, error codes,
  invariants), [troubleshooting](operations/troubleshooting.md) (`crashed`
  replaces the `orphan_*` sections), [health checks](operations/health-checks.md)
  (plain doctor never persists), [managed-primary lifecycle](architecture/managed-primary-lifecycle.md)
  (repair and cleanup boundary), [spawn finalization](architecture/spawn-finalization.md)
  (dead-launch outcome replaces the reaper tree),
  [process scope](architecture/process-scope.md) (cleanup triggers, scope
  liveness consumers), [launch system](architecture/launch-system.md) (no
  heartbeat, start gate), telemetry liveness (`is_live`), vocabularies,
  the former invariants page (I-12 is now the runner lock),
  the former design-principles page,
  [state model](concepts/state-model.md), [spawn state](architecture/state-system/spawn-state.md),
  [sandbox projection](architecture/sandbox-projection.md).
- **Status lines:** PR 3 merged in [spawn identity](decisions/spawn-identity.md)
  (plus D84's continue/repair decision), `decisions.md`, `index.md`; #668 merged
  in [primary inject](decisions/primary-inject.md).
- **History kept as history:** [reaper lessons](lessons/state-design-lessons.md#reaper-lessons-history)
  condensed and dated; supersession notes on D-managed-primary-reaper,
  the `mark_finalizing` CAS decision, doctor's delegation to the reaper, and D69
  telemetry liveness.

---
## 2026-10-17 — Mars is the only skill resolver; D60 narrowed to harness variants (`refactor/mars-owns-profile-skills`, `work:long-view-review`)

- New decision [D-mars-sole-skill-resolver](decisions/package-management.md#d-mars-sole-skill-resolver-meridian-never-resolves-skills)
  and index row: Meridian doesn't parse profile `skills` or read `.mars/skills/`;
  it passes only `--skills` extras; replay uses only the snapshot's `loaded_skills`.
- [D60](decisions/package-management.md#d60-skill-variants-are-selected-by-harness-only)
  rewritten and renamed: harness-only selection; model-level variants not
  supported. The owner flag from the previous entry is resolved, and Mars gap 8
  (model variants) is deleted. Mars's leftover model-dir indexing is flagged.
- Removed `SkillRegistry`, `resolve_skills_from_profile()`, `AgentProfile.skills`
  and the legacy replay fallback from [skill-schema](concepts/skill-schema.md)
  (the four-step ladder diagram and table are gone),
  [skill-loading](concepts/skill-loading.md),
  [model-resolution overview](concepts/model-resolution/overview.md),
  [agent-profiles](concepts/model-resolution/agent-profiles.md) (also: duplicate
  profiles never warned), the model-resolution and root vocabularies,
  [launch-system](architecture/launch-system.md),
  [mars-user-scope](architecture/mars-user-scope.md),
  [composition-pipeline](concepts/composition-pipeline.md), package-management
  concept pages, system overview and codebase guide.
- Lesson: the resolution-side twin of the second-channel drift, in
  [arch refactor pitfalls](lessons/arch-refactor-pitfalls.md#a-capability-gated-second-channel-drifts-from-the-first-pr-208--pr-661).

---
## 2026-10-17 — How skills reach an agent: loaded vs available vs model-invocable (PR #661 follow-up, `work:long-view-review`)

- New [concepts/skill-loading.md](concepts/skill-loading.md): a loaded skill (profile
  `load` or `--skills`) is composed in full on every harness whatever its
  `model-invocable`; `available` lists names; native skill lists show only
  model-invocable skills; the named skill header; gaps (OpenCode v2 and
  `opencode run` get no composed prompt, Pi RPC `--no-skills`, model variants).
- New decision [D-load-ignores-invocability](decisions/package-management.md#d-load-ignores-invocability-model-invocable-gates-discovery-not-an-explicit-load)
  and index row. D60 marked: only harness-level variant selection is built
  (flagged for the owner). D71 compilation line corrected (Codex `openai.yaml`,
  Cursor Manual rule).
- [skill-schema.md](concepts/skill-schema.md): `model-invocable` scope, lowering
  matrix for Codex and Cursor, variant selection as built, variant example that
  relied on variant frontmatter.
- [agent-profiles.md](concepts/model-resolution/agent-profiles.md): `skills:`
  flat or `{load, available}`, bundle-based attachment (the `SkillRegistry`
  description was the legacy replay path).
- D-skill-doc-suppression gained how the split came about (#29902 workaround,
  its removal, the April unification, PR #208). Lessons: closed upstream issue ≠
  fixed bug ([harness integration](lessons/harness-integration.md#a-closed-upstream-issue-is-not-a-fixed-harness-bug-29902));
  an observability file a harness reads is a delivery channel, and don't carry
  `model-invocable` from agents to skills ([arch refactor pitfalls](lessons/arch-refactor-pitfalls.md#a-capability-gated-second-channel-drifts-from-the-first-pr-208--pr-661)).
- Per-harness system-prompt channel table in [harness adapters](codebase/harness-adapters.md#skill-delivery);
  launch-bundle `skills` fields; vocabulary entries; Mars gap 8 (model variants).

---
## 2026-10-17 — Derived spawn status: identity-v2 PR 3 (#663, built on `feat/read-derived`, awaiting merge) and the #662 flake (`work:v010-followups` D77–D82)

- New [architecture/state-system/derived-status.md](architecture/state-system/derived-status.md):
  the lock as the only liveness fact, `read_derived`'s three steps, stored vs
  derived status, selection vs liveness, `claim_dead_launch`/`finalize_dead_launch`
  and its callers, lockless legacy rows, lenient quarantine, the deleted guesses,
  behaviour changes, what PR 4 changes. Build status at the top; linked from the
  state-system overview and `index.md`.
- [D-spawn-identity-v2](decisions/spawn-identity.md#pr-3-as-built-read_derived-and-one-dead-launch-finalize):
  new "PR 3 as built" section (D78–D82 decisions, rejected alternatives, owner
  rulings, residuals, provenance); status lines, PR table and `decisions.md` row.
- Lessons: [state design](lessons/state-design-lessons.md#a-deleted-rule-survives-in-other-writers-and-in-fixtures)
  (a deleted rule in a second writer and in fixtures; "every reader" sweeps in
  selectors); [test determinism](codebase/test-determinism.md#fixtures-that-mean-live-must-hold-the-lock)
  (lock-holding "live" fixtures; race tests red against split implementations;
  #662 as memory-ahead-of-disk); [verification](lessons/verification-and-review-discipline.md#compare-against-the-base-before-calling-a-probe-failure-a-regression)
  (base comparison before calling a probe failure a regression).
- Banners on `architecture/state-system/reconciliation.md` (retired since) and
  [spawn lifecycle](concepts/spawn-lifecycle.md) point at the new page. Pages that
  describe `main`'s heartbeat, graces and `orphan_*` codes are left as they are
  until PR 3 merges.

---
## 2026-10-17 — Claude primary inject through the inbox socket: #665, PR #668 open (`work:claude-mod-inject`)

- New domain page [decisions/primary-inject.md](decisions/primary-inject.md) with
  [D-claude-primary-inbox](decisions/primary-inject.md#d-claude-primary-inbox),
  flagged for owner review: the owner's direction was a mod replacing child stdin,
  and the inbox route and children-keep-stdin are tech-lead decisions on probe
  evidence. Rows in `decisions.md`, `decisions/overview.md`, `index.md`.
- New [research/claude-code-mods-and-inbox.md](research/claude-code-mods-and-inbox.md):
  Claude Code 2.1.302 mod behaviour under `-p` and in a TUI, and the inbox socket's
  wire format, timing and inbound controls.
- [D-claude-native-hold](decisions/completion-drain.md#d-claude-native-hold)
  residuals resolved: spawn guidance stays as written (D5); a mod cannot replace the
  stdin pipe, and primaries use the inbox.
- [Claude native sessions](architecture/claude-native-sessions.md#launch-owned-sessionstart-journal-and-current-chat):
  journal record gains `recorded_at` and `messaging_socket` (no token); the inject
  reader and its PID-born-by guard.
- [#498 future work](open-questions/future-work.md#live-primary-injection-control-endpoint-498)
  narrowed to Codex/OpenCode/Pi; [harness abstraction](concepts/harness-abstraction.md)
  records the weaker "posted" contract of `primary_inbox`;
  [primary terminal ownership](architecture/primary-terminal-ownership.md) notes the
  relay is not an inject path.
- `corrections.md`: one occurrence ("gh says 655 is still open") and one candidate.

---
## 2026-10-17 — One skill delivery path: PR #661 (`work:long-view-review/skill-delivery`)

- [decisions/launch.md](decisions/launch.md#d-skill-single-path-loaded-skills-are-composition-documents-on-every-harness):
  new D-skill-single-path (PR #661 open); D-skill-doc-suppression is marked superseded
  and kept, with why it failed (615dcb330, 2026-05-13: Claude lost loaded skills for five months).
- `concepts/composition-pipeline.md`: "Two Channels, One Correct" is replaced by
  "Skill Delivery: One Path". The page also fixes these older stale parts: the IR field
  table (`skill_injection`, `inventory_prompt`, `context_prompt` no longer exist;
  `supplemental_documents`, `available_skills`, `guidance_blocks`, `launch_preamble`
  added), `SYSTEM_INSTRUCTION_BLOCK_ORDER`, Codex/OpenCode projection (system field,
  not inline), G-3 (`system-prompt.md` comes from the appended prompt), and inventory
  gating (spawn capability, not the PRIMARY surface).
- `codebase/harness-adapters.md`: the capability matrix is rebuilt from the
  `HarnessCapabilities` fields that exist. Removed `supports_native_skills`,
  `supports_stream_events`, `supports_programmatic_tools` and the Direct column;
  added Cursor.
- `lessons/arch-refactor-pitfalls.md`: the or-chain lesson is rewritten as "A
  Capability-Gated Second Channel Drifts From the First". Index rows are in `decisions.md`
  and `lessons/overview.md`.

---
## 2026-10-17 — #655 merged with the profile split: PR #659, v0.10.21 (`work:claude-bg-quiescence` D11–D14)

- [D-claude-native-hold](decisions/completion-drain.md#d-claude-native-hold):
  status now merged (`25e3ac345`, v0.10.21); decision 2 records the final shape
  (`build_drain_plan()` selects `NativeTaskHoldProfile` on `completion_evidence()`,
  else main's `_ResidentCompletionProfile`; both compose `RearmWindow`; native tasks
  are `blocked` with `native_task` blockers, no `WorkAssessment` field). Rejected
  rows added: the round-2 `native_hold` flag, a base class with abstract hooks, and
  a Claude Code mod for child completion (D14); residual notes the TUI-primary mod
  idea and the `work:claude-mod-inject` probe.
- [Drain plans](architecture/drain-plans.md#claude-native-holds) and
  [completion drain coordination](architecture/completion-drain-coordination.md):
  dropped "built, not merged" / "pending #655" and the `native_hold` flag; describe
  the one-time profile selection and `RearmWindow`.
- `codebase/vocabulary.md`: **Native hold** renamed **Native task hold** and
  redefined as the profile. `lessons/spawn-lane-operations.md`: "Until #655
  merges" now points at v0.10.21. Index rows in `index.md`, `decisions.md`,
  `decisions/overview.md` updated.

---
## 2026-10-17 — Claude children hold for their own background work: #655 (`work:claude-bg-quiescence`)

- New domain page [decisions/completion-drain.md](decisions/completion-drain.md)
  with [D-claude-native-hold](decisions/completion-drain.md#d-claude-native-hold)
  (built on `fix/claude-bg-quiescence`, not merged): hold on Claude Code's
  `background_tasks_changed` snapshot through the shared resident profile; native
  holds not sticky; explicit `done` always wins; three shared Codex/OpenCode fixes;
  rejected alternatives incl. the implementer's deadline-beats-done. Indexed in
  `decisions.md`, `decisions/overview.md`, `index.md`.
- [Drain plans](architecture/drain-plans.md#claude-native-holds): new mechanism
  subsection (signal facts: `local_agent`/`local_bash`/`parent_task_id`/
  `owned_by_subagent`, full-tree emptying, raw `claude -p` waiting forever;
  subagent `assistant` frames with `parent_tool_use_id`; stdin nudges and
  `control.sock` inject). [Completion drain coordination](architecture/completion-drain-coordination.md)
  composition and invariant 9 note the evidence-only resident path.
- **Superseded** `lessons/spawn-lane-operations.md` "A Claude-Routed Lane Can End
  Its Turn Before Its Gate Finishes" (a prompting workaround) with a short
  history plus the lesson: a prompting workaround for a system defect is a
  deferral; the project's own spawn guidance steered agents into the bug.
- `codebase/vocabulary.md`: **Native hold**. Corrections register: two `chat:c8112`
  occurrences (reuse the resident rearm system; read the diff, not the reports).
- Colocated docs on the branch (`lib/streaming/AGENTS.md`, `.context/CONTEXT.md`)
  already match; no gap.

---
## 2026-10-17 — Identity-v2 PR 2 merged: launch lock and continue sweep, #647 (`work:v010-followups` D65–D75)

- [D-spawn-identity-v2](decisions/spawn-identity.md#pr-2-as-built-the-launch-lock-and-the-sweep):
  new "PR 2 as built" subsection (exact-liveness lock, `--bg` fd hand-off, sweep
  rules, continue resolving an old ID to `chat_record(chat).spawn_id` and refusing,
  never killing, a live owner; successor pointer published before hand-off; pruned
  successor ignored; residuals: concurrent continue → PR 7, cancel in hand-off →
  PR 4, `setsid` descendants, macOS). PR table and status lines updated.
- **Stale content fixed** (#633 merged in v0.10.13 but still called draft/open):
  D-start-gate-guardian status, `decisions.md` rows, `index.md`;
  [process scope](architecture/process-scope.md#parent-death-linkage) parent-death
  section rewritten to the gate as shipped, PDEATHSIG/watchdog rows and
  `parent_death_linked` example removed, new [continue sweep](architecture/process-scope.md#continue-sweep)
  cleanup path; `managed-primary-lifecycle.md` (#614 watchdog never shipped),
  `reconciliation.md`, `per-operation-cost.md`, `codebase/harness-adapters.md`,
  vocabularies. Deleted the **Parent watchdog** vocabulary term (subject gone);
  added **Continue sweep** and **Launch lock**.
- [Process containment lessons](lessons/process-containment.md#continue-must-lock-the-chats-current-owner-not-the-named-spawn):
  lock the chat's current owner; a lock fd survives the hand-off.
- [Test determinism](codebase/test-determinism.md#wait-for-the-runner-before-deleting-a-spawn-directory):
  wait for the runner before `rmtree` (ENOTEMPTY flake).
- [Release sequencing](lessons/release-sequencing.md): CHANGELOG rebuild is
  unconditional after crossing a release commit, anchored on `## [Unreleased]`,
  then the placement check (extends D66).

---
## 2026-10-17 — Black-box primaries cancel on SIGTERM/SIGHUP: #623, PR #658 (`work:relay-signal-exit`)

- [Primary terminal ownership](architecture/primary-terminal-ownership.md):
  black-box TERM/HUP cancel; the handler-context rule widened past locks we
  write (`LaunchGate`, `Event.set()`, PTY `cancel_wait()` are not handler-safe,
  so receivers latch and a main-thread poll acts); new subsection on
  `termios.error` not being an `OSError`. Known gaps: the #623 entry now says
  "on `main` until #658 merges" with the live matrix; added the finalize-window
  seam (identity PR 4) and #657 (native black-box has no KILL, not CLI-reachable).
- Stale content fixed on the same page: the status block said native terminal
  was #614, open; it shipped in #646 (`bc2374b5c`, v0.10.15) and #614 closed
  unmerged. `_TuiExitOrdering`/`RLock` never reached `main`; the cancel-by-order
  section now describes the shipped bool latch. The job-control diagram's
  "parent watchdog" is flagged as a divergence, not yet rewritten.
- [D-blackbox-signal-latch](decisions/primary-terminal-ownership.md#d-blackbox-signal-latch):
  new record (latch-only handler, gate-only stop, shared `cancelled`/130,
  repeat escalation flipped on by review F3). D-cancel-by-order status updated.
  `decisions.md` row added; the D-native-terminal row's #614 status corrected.
- [Process-containment lessons](lessons/process-containment.md#moving-a-blocking-wait-onto-a-thread-moves-its-cleanup-too):
  review F1, the cleanup a threaded wait loses.
- [Troubleshooting](operations/troubleshooting.md#primary-crashed-with-no-exit-or-stop-recorded)
  and the `launch-system.md` signals line updated.
- Colocated: `src/meridian/lib/launch/process/{AGENTS.md,.context/CONTEXT.md}` on
  branch `fix/relay-signal-exit` (`7858f3ddd`).

---
## 2026-10-16 — Cold index kept out of deadline-bound queries: #656 (#644) (`work:index-cold-deadline`)

- [Portable history](architecture/state-system/portable-history.md#a-caller-with-a-deadline-builds-the-index-before-starting-its-clock):
  new subsection with the deadline rule. A caller with a deadline classifies and
  initializes an absent index before it starts its clock, because `query()` keeps
  an absolute caller deadline across the cold build. The section names the two
  callers that follow it (corpus search, descendant evidence), the
  spawn-wait exception, the rejected alternatives, and when the index is cold
  (first read, deletion, schema bump: v11 identity-v2 PR 4, v12 #418 B2).
- [Per-operation cost](architecture/per-operation-cost.md#3-history-index-reads-pay-for-more-than-their-answer-594-600-592)
  and [D-history-index-readers-never-wait-on-session-writers](decisions/history-storage.md#d-history-index-readers-never-wait-on-session-writers):
  pointers to the rule. #592 stays open.
- [Test determinism](codebase/test-determinism.md#cold-index-builds-inside-a-test-deadline-a-flake-class):
  all five instances are recorded (#577/#624, #630/#637, #638/#645, #551/#649,
  #643/#650); #630 was previously listed as open. The shared cause is fixed at
  the source. The 9 `warm_history_index()` call sites stay, to keep the build out
  of the tests' own wall-clock bounds. The helper path is corrected to
  `tests/support/history.py`.
- [Contention diagnosis](lessons/contention-diagnosis.md#before-and-after-come-from-one-corpus-under-one-load):
  new lesson. A before/after pair must come from one corpus under one load; #656's
  PR body mixed a 3.6 s run with a more heavily loaded 8.5 s run.
- [Cleanup program](codebase/cleanup-program.md): the stale "#630 still open"
  line is updated.

## 2026-10-16 — CHANGELOG placement check: #653 (#652) (`work:changelog-misfile-check`)

- [Release sequencing](lessons/release-sequencing.md#landing-a-stacked-chain-2026-10-14):
  the CHANGELOG misfile is now caught by `scripts/check-changelog-placement.py`
  in the pre-push hook and the CI `pr-changelog-note` job (on the merge ref). It
  replaces the manual `git diff` check. Escape hatches:
  `MERIDIAN_CHANGELOG_ALLOW_RELEASED=1` and the `changelog:edit-released` label.
  The manual rebuild stays the fix.
- [Probe runtime placement](lessons/probe-runtime-placement.md#a-throwaway-worktree-shares-the-main-repos-config):
  new section on `git config` in a worktree writing the main repo's shared
  config (C3). Smokes use a `git clone` or `git -c user.name=… commit`.

## 2026-10-16 — Memory program shipped: #634 (MEM-1) and #651 (#418 B1) (`work:memory-audit`)

- New decision [D-session-read-seam](decisions/state.md#d-session-read-seam): the
  B1 seam contract and invariants (one pass per multi-chat read; global log order,
  never a timestamp sort; B2's stable per-stream merge; one append path; the
  prefilter only nominates; `SessionBindings.scope`; poison isolation), the two
  deviations (accepted generation moves to B2, M20; eager-import walk plus tuple,
  M14), the equivalence oracle and the rejected alternatives. Indexed in
  [decisions.md](decisions.md).
- [D-session-per-chat-journals](decisions/state.md#d-session-per-chat-journals):
  status is now B1 shipped, B2 not built. Owner answered Q5 (B4 automatic
  retirement) yes; Q1–4 and Q6–11 are open; B2 follows identity-v2 PR 4.
- [Per-operation cost](architecture/per-operation-cost.md): #634 and #651 in the
  fixed table; §1 says the byte-scan slope remains until B2; §2 notes #612's
  streamed teardown; §5 records the causes, the before/after and what is left (the
  ~230 MiB close residual for B3, cold-index runs); a pass-count measurement rule.
- [Session state](architecture/state-system/session-state.md): the keyed seam
  replaces the v0.10.10 table of full-log readers; scope and poison rules;
  highest-ordinal index generation.
- Stale code references fixed: `_append_session_event` no longer exists
  ([durability and locking](architecture/state-system/durability-and-locking.md),
  [history storage](decisions/history-storage.md)). Status lines updated in
  [future work](open-questions/future-work.md) and
  [Rust rewrite](decisions/rust-rewrite.md).
- Lessons: count every pass per operation
  ([contention diagnosis](lessons/contention-diagnosis.md#count-every-pass-per-operation-not-one-kind-of-fold));
  corpora with increasing clocks hide ordering changes
  ([cost cuts](lessons/cost-cuts-and-safety-checks.md));
  CHANGELOG misfiling on rebase ([release sequencing](lessons/release-sequencing.md));
  model-capacity deaths aren't covered by declared backups
  ([spawn lanes](lessons/spawn-lane-operations.md)).

---
## 2026-10-16 — Identity-v2 approved; PR 1 (start gate + guardian, #633) (`work:v010-followups`)

- New [decisions/spawn-identity.md](decisions/spawn-identity.md):
  D-spawn-identity-v2 (one ID per spawn, launches internal, derived status, gate +
  guardian + sweep, 7-PR plan, #616 rejected) and D-start-gate-guardian. Supersedes
  D-spawn-run-identity (status line now links here) and D-launcher-watchdog's
  mechanism (marked in place).
- New [decisions/rust-rewrite.md](decisions/rust-rewrite.md):
  D-update-crash-waits-for-rust (#628) with carried requirements; sequencing
  recommendation, pending the owner.
- New [lessons/process-containment.md](lessons/process-containment.md). Spawn
  identity lessons added to [state design lessons](lessons/state-design-lessons.md);
  its reaper lessons now say they describe the mechanism being retired.
- [Test determinism](codebase/test-determinism.md): two flake classes, cold index
  builds inside a deadline (#577/#624, #630) and real-time deadlines in fake-clock
  tests (#632).
- **Correction:** [process scope](architecture/process-scope.md) described #614's
  unmerged per-launcher watchdog as the current mechanism. It now describes
  `main` (Linux PDEATHSIG, macOS backend-only polling watchdog, no link for stdio
  children or primaries) and the decided replacement. The anchor is renamed to
  `#parent-death-linkage` and inbound links are updated (managed-primary
  lifecycle, harness adapters, launch system, vocabulary, primary-terminal
  decisions). The #614/#633 collision is flagged for human review.
- Decided-replacement notes added to reconciliation, spawn lifecycle, spawn
  state, the `mark_finalizing` decision, the vocabularies (Launch, Start gate /
  guardian, Heartbeat, Reaper, Parent watchdog), PROC-004 and the per-operation
  cost "In flight" section (the stale #616 fix-pass text was removed).
- Corrections register: a third occurrence for "grew the code instead of
  reusing structure", and one new candidate.

---
## 2026-10-16 — Status sweep: #621 and #631 merged, #616 closed (`work:resume-after-crash`, `work:memory-audit`)

- #621 (launcher output off live TUIs) marked shipped in v0.10.11 (`23eb2d6d2`)
  in both primary-terminal-ownership pages, design principles and the indexes.
  #614 now makes the `tui_terminal_window()` change during its rebase. The
  Claude/Pi SIGTERM gap is tagged #623.
- Lane B marked shipped as #631 (`8ecd44989`) in history-storage,
  session-state, portable-history, native-session-binding, the lesson and
  troubleshooting. The #616-based branch was deleted unmerged. Index schema is
  10 on `main` (8 and 9 burned by unreleased #616 builds); identity-v2 PR 4 plans
  v11 and #418 B2 v12 (state overview, decisions/state.md). The deferral owner
  label is `spawn pN`; #616's `run pN.K` and `ChatBusy` text removed.
- D-spawn-run-identity marked superseded by identity-v2 (#616 closed unmerged);
  the record stays for the requirement and rejected alternatives.
- Troubleshooting: the `OOMPolicy` open question is resolved by a
  `tmux-spawn-.scope.d` drop-in (R18), verified on all 24 live pane scopes.
- [Per-operation cost](architecture/per-operation-cost.md): new section on
  resident memory from full scans (M1, M6). B1 is in progress as MEM-2, MEM-1 in
  progress; neither is merged.

---
## 2026-10-15 — Resume after crash: TUI-quiet launcher and abandoned generations (`work:resume-after-crash`)

- [Primary terminal ownership](architecture/primary-terminal-ownership.md): new
  section on the fd-level `launcher.log` window and its one pointer (PR #621, not
  merged), the #614 integration requirement, and a known gap: Claude/Pi launchers
  have no SIGTERM/SIGHUP handler on `main` or #614. The status block now says #614
  is an open PR. The stale #590 `ESC[J` gap was deleted: #590 is closed and #616
  doesn't carry it.
- New decision [D-tui-quiet-launcher](decisions/primary-terminal-ownership.md#d-tui-quiet-launcher--launcher-output-never-reaches-a-live-tui)
  with rejected alternatives; D-exit-text-append links to it.
- [Session state](architecture/state-system/session-state.md): new section on the
  fold rule that closes abandoned generations (branch
  `fix/abandoned-session-generation`, not merged) and why it is sound. Index file
  names now say `history-v<N>` with the schema per branch, here and in the
  [state overview](architecture/state-system/overview.md); `v6` was stale (main is
  7).
- [Portable history](architecture/state-system/portable-history.md): new
  subsection on live-owner capture deferral and retry on owner stop, plus the
  `main` behavior.
- New decision [D-abandoned-generation-closure](decisions/history-storage.md#d-abandoned-generation-closure);
  decision index rows for both decisions. Stale rows fixed: D-native-terminal (#614
  open), D-exit-text-append (#590 closed), D-spawn-run-identity (#616 open).
- Lessons: [repair the state before relaxing the check](lessons/state-design-lessons.md#repair-the-state-before-relaxing-the-check);
  [a scratch home does not isolate the harness binary](lessons/probe-runtime-placement.md#a-scratch-home-does-not-isolate-the-harness-binary)
  (Codex self-update). Troubleshooting: [primary crashed with no exit or stop
  recorded](operations/troubleshooting.md#primary-crashed-with-no-exit-or-stop-recorded)
  (systemd `OOMPolicy=stop` takes down the pane scope).
- [Managed primary lifecycle](architecture/managed-primary-lifecycle.md): marks
  the launcher watchdog as #614-only. Corrections register: one new candidate.

---
## 2026-10-15 — Corrections register

- New [corrections.md](corrections.md): the human's corrections of agents,
  grouped into ten classes and ten candidates, each with what enforces it.
  Seeded from 139 primary sessions, 2026-09-15 to 2026-10-15 (`spawn:p8116`,
  `spawn:p8117`). Nine of the ten classes are enforced only by a written rule.
  Indexed under Root Pages.

---
## 2026-10-15 — Constant-time program, phase 1: #588/#599/#605/#608 shipped; #418 design B accepted (`work:v010-followups`)

- New [per-operation cost](architecture/per-operation-cost.md): the owner's bar
  (per-spawn operations don't grow with history); what #588 (v0.10.4), #599
  (v0.10.8) and #605 (v0.10.10) fixed; the known violators by root cause with
  issue numbers; the count-based measurement harness and acceptance gate. Added
  design principle 16, which links to it.
- New decision [D-session-per-chat-journals](decisions/state.md#d-session-per-chat-journals)
  (#418 design B), marked "accepted design, pending the owner's approval; not built",
  with rejected alternatives: the offset sidecar (A), ingest-on-observe, ignoring
  older-build writes, waiting for quiescence, literal `state.json`. Added a decision
  index row.
- [Session state](architecture/state-system/session-state.md): the "remaining
  O(n) log" section is rewritten as current truth. Sessions are one global log, not
  per-chat. The empty `chats/` directory is a dead primitive. Lists the measured
  per-launch readers and the #418 plan. #591 links are repointed to #418 here, in
  history-storage, future-work, the state-system overview and contention-diagnosis.
- Status corrections: #588 is shipped (v0.10.4), not "landing". Fixed in the
  decision index, history-storage, codebase-cleanup (#546 was closed by #599),
  portable-history, durability-and-locking and spawn-wait. D-spawn-run-identity now
  records the combined #587+#590 branch (D26) and its review fix pass.
- [Spawn wait](concepts/spawn-wait-barrier.md): SQL-scoped discovery plus an
  authoritative re-read (#599); the descendant mechanism is corrected.
- [Spawn finalization](architecture/spawn-finalization.md#terminal-arbitration-terminalarbitrator):
  the arbitration section is rewritten for `main`. Added the inactivity trigger.
  Removed the 0.5 s completion grace and `watchdog_noop` (both gone). Documented
  the event-woken watchdogs (#605).
- Lessons:
  - [contention diagnosis](lessons/contention-diagnosis.md) is broadened to
    "contention and cost diagnosis": count rather than time on a shared box, warm the
    index, probe end to end;
  - new [cost cuts that weakened safety checks](lessons/cost-cuts-and-safety-checks.md):
    #605's arbitration order, and the combined-branch B1–B3 regressions;
  - [test determinism](codebase/test-determinism.md): wait on what the API reports,
    not on its persistence file (#606/#608).
- [Future work](open-questions/future-work.md): the #591/#592/#594 entry is replaced
  by a per-operation cost program list. The untracked archiver fold is recorded.
  Deleted the stale "spawn-list scaling, an index is not justified" entry: the
  history index exists, and the current listing cost is under #600 in
  [spawn state](architecture/state-system/spawn-state.md).

---
## 2026-10-15 — Primary terminal ownership and exit text (`work:native-terminal-primaries`, `work:relay-exit-text`)

- New [architecture/primary-terminal-ownership.md](architecture/primary-terminal-ownership.md):
  terminal mode per harness, shared foreground job, the SIGINT window and
  signal-handler reentrancy, cancellation by order, the never-write/never-cover
  rule with the `ESC[?1049l` root cause, known gaps.
- New [decisions/primary-terminal-ownership.md](decisions/primary-terminal-ownership.md):
  D-exit-text-append (PR #607, v0.10.9), D-native-terminal,
  D-shared-foreground-job, D-launcher-watchdog, D-cancel-by-order, with rejected
  alternatives.
- [architecture/process-scope.md](architecture/process-scope.md): new
  "Parent-death: one watchdog per launcher" section; native TUI scope is a
  planned PID-tree scope; `parent_death_linked` redefined.
- [architecture/managed-primary-lifecycle.md](architecture/managed-primary-lifecycle.md):
  roles table and "Launcher Death" no longer claim the backend and TUI outlive
  a crashed launcher on POSIX.
- Deleted stale claims: "Claude needs the PTY to observe" / "PTY for session-ID
  detection" (lessons/harness-integration, principles §10, codebase/vocabulary);
  Linux PDEATHSIG as the backend link (codebase/harness-adapters,
  launch-system). The harness-integration PTY section was rewritten as a lesson
  about a relay that observed nothing; the thread-safety lesson gained the
  signal-handler `RLock` finding.
- New [lessons/probe-runtime-placement.md](lessons/probe-runtime-placement.md)
  (the 24 GB autosync incident, #538, meridian-dev-workflow#7). The scratch
  section moved out of dogfooding-pr-builds, and codebase/work-items now points
  to it.

---
## 2026-10-15 — Idle compaction work shipped: v0.10.3, v0.10.5, v0.10.6 (`work:idle-compaction-floor`, `work:idle-close-no-compact`, `work:harness-compaction-cache`)

- [D-idle-compaction-growth-floor](decisions/idle-compaction-growth-floor.md) now
  describes merged behaviour (PR #589). Corrected the OpenCode claim: only
  OpenCode 1.x compacts cold. 2.x `openai` is priced (c = 0.104, need ≈60k/44k).
  Recorded Pi "never" until haowjy/meridian-cli#593, the rejected sensor-level
  cost seam, and the user's patch-release choice despite the key rename.
- New decision [D-idle-compact-over-drafts](decisions/idle-compaction-actuator.md#d-idle-compact-over-drafts--compaction-wins-over-unsent-text)
  (PR #583): the draft guard is deleted on every harness. It records Codex's
  screen-gated Ctrl+C composer clear and why it is gated, plus the
  unreadable-screen retry and notice. Deleted the draft guard, the draft
  unknown-fact row and the "pane-based draft guard must remain" paragraph from
  [Idle notifications](decisions/idle-notifications.md), and the draft re-checks
  from [Idle adapters](architecture/idle-adapters.md).
- [D-opencode-idle-v2](decisions/idle-opencode-v2.md#d-opencode-idle-v2--opencode-2x-gets-its-own-sensor-chosen-by-the-resolved-transport)
  gains the stream-lifetime decision (the primary stream reconnects for the
  primary's lifetime, and compaction is confirmed from durable data).
  D-idle-reopen-return is marked released in v0.10.3.
- [Prompt-cache retention](research/prompt-cache-retention.md): new sections on
  which compaction requests read the cache, and on Pi cache warming versus idle.
- Deleted `research/idle-compaction-break-even.md`. It was the superseded
  first-cut model and described the removed 40k floor as current. Its one
  unique measurement (the 39,386-token primary prefix) moved into the decision
  record.
- Guard list renumbered to match `guards.py` (stale-anchor and not-scheduled
  listed separately).
- Split `decisions/idle-notifications.md` (about 900 lines) by concern, keeping
  every decision ID and heading: [idle-notify-channels](decisions/idle-notify-channels.md)
  (leaf packages, notice context, SMTP password, spawns may notify),
  [idle-opencode-v2](decisions/idle-opencode-v2.md), and
  [idle-compaction-actuator](decisions/idle-compaction-actuator.md). The
  per-harness hosting table moved to
  [Idle adapters](architecture/idle-adapters.md#hosts), which owns
  the mechanism; the decision pages keep the rationale and rejected options.

## 2026-10-15 — History-index readers stop waiting on session writers (`work:v010-followups`, PR #588 landing, #586)

- Added [D-history-index-readers-never-wait-on-session-writers](decisions/history-storage.md#d-history-index-readers-never-wait-on-session-writers)
  and its [decision index](decisions.md) row. It covers the measured holders, the O(1)
  `historical-` prefix guard (with the rejected sidecar, prefilter and drop-the-guard
  options), prefix rules at append time only, the lock-free sessions projection with
  lock-only acknowledgement, touched-chat catch-up, one bounded attempt (the 15 s retry
  budget is rejected), an owned sticky cold build for every caller, and no-arg wait
  discovery capped at `QUERY_TIMEOUT`.
- Superseded [D-b7-via-546](decisions/codebase-cleanup.md#d-b7-via-546-the-spawn-wait-contention-failure-is-fixed-by-pr-546)
  in place. Its root cause was incomplete and #546's retry fix is rejected; the
  fail-closed reasoning stays current. The index, overview and revisit trigger
  were updated to match.
- New lessons page [contention diagnosis](lessons/contention-diagnosis.md): measure the
  holder before widening a timeout; check a cause ranking against the protocol;
  pending-source labels can name a bystander. Linked from
  [measurement discipline](lessons/verification-and-review-discipline.md#measurement-discipline).
- Current truth, marked "landing in #588":
  - [portable history](architecture/state-system/portable-history.md): the
    source-lock exception for sessions, and the one-attempt read budget;
  - [session state](architecture/state-system/session-state.md): the O(1) guard,
    plus a new section on `sessions.jsonl` as the remaining global O(n) log (#591,
    with the per-spawn `state.json` precedent);
  - [durability and locking](architecture/state-system/durability-and-locking.md):
    the append guard;
  - [spawn wait](concepts/spawn-wait-barrier.md): the discovery cap and the
    discovery-pending checkpoint;
  - [future work](open-questions/future-work.md): #591, #592, #594.

---
## 2026-10-14 — Spawn/run identity decided; phases 1–2 built locally (`work:primary-spawn-identity`)

- Added [D-spawn-run-identity](decisions/spawn-identity.md#d-spawn-run-identity) and its
  [decision index](decisions.md) row: user decision, run-row rationale, D-guard narrow `RunId`
  guard, retention and counter specifics, rejected alternatives, accepted provenance cost, and
  PR-B (phases 3–4) remaining scope. Cross-linked from the rejected launch-owned stretches in
  [idle notifications](decisions/idle-notifications.md).
- Architecture, vocabulary and native-session-binding pages are deliberately unchanged: PR-A is
  unmerged and changes no launch behavior; those pages move with PR-B phase 4.

---
## 2026-10-14 — Unknown-key warnings from Pydantic's traversal (`work:v010-followups`, PR #581, v0.10.1)

- Added [D-unknown-keys-from-pydantic](decisions/codebase-cleanup.md#d-unknown-keys-from-pydantic-pydantic-does-the-traversal-for-unknown-key-warnings)
  and its [decision index](decisions.md) row: #580 cause, rejected alternatives, accepted costs.
- Updated current truth: [config precedence](concepts/config-precedence.md) and the
  [cleanup program](codebase/cleanup-program.md) conventions table no longer describe warning
  policy as `ClassVar`s (`config_ignored_keys` deleted); "How the Stack Landed" records the v0.10.0
  regression.
- Lessons: [loads-silently oracle and the #580 audit method](lessons/verification-and-review-discipline.md#a-generalized-diagnostic-path-needs-a-loads-silently-oracle)
  (real-config sweep from scratch copies; `uv run --isolated --with` old-build differential; 0.9.8
  loader at `config.settings`); [keep the lead's verdict out of review briefs](lessons/review-convergence-gate.md#a-review-brief-must-not-carry-the-leads-verdict).
  Moved the stray "Delete the obsolete work" section above Related on that page.

---
## 2026-10-14 — Two-tier identity and the cleanup stack shipped (`work:two-tier-identity`, PRs #570/#571/#574/#575, mars-agents #173)

- Status flips: [decision index](decisions.md), [D-two-tier-identity](decisions/state.md#d-two-tier-identity),
  [D97](decisions/package-management.md#d97-user-scope-packages-live-in-an-ordinary-mars-root-2026-10),
  [project identity](architecture/state-system/project-identity.md), [Mars user scope](architecture/mars-user-scope.md),
  [cleanup program](codebase/cleanup-program.md), [cleanup decisions](decisions/codebase-cleanup.md),
  [static typing](decisions/static-typing.md). #570 → v0.9.10, #571 → v0.9.11, #574 → v0.9.12, mars-agents
  #173 → 0.16.0 (pinned); #575 merged (`d160a07d9`); its first release-on-main preflight failed on 3 `test_signal_canceller`
  AF_UNIX path-length tests and 1 timeout; #579 (bounded app-lane socket path) fixed it and v0.10.0 was cut (`b8da247f9`).
- Folded I17 (`doctor --global` lists every ad-hoc root, `prunable: yes|no`), I14 (activity = newest of
  every signal), I15 (`mint_committed_identity`), I19 (`mars_fallback: active|unsupported|n/a`), I20 (Mars
  release and pin bump before #575; the probe is the safety net behind the pin), I21 (no automatic
  pruning; daily sweep declined) and the I16 addendum (alias gap is narrow; aliases from the fallback
  root left open). [Health checks](operations/health-checks.md) updated for the listing.
- Lessons: [`.venv/bin/*` hard-link rule](lessons/dogfooding-pr-builds.md#writing-over-venvbin-rewrites-uvs-cache);
  [landing a stacked chain](lessons/release-sequencing.md#landing-a-stacked-chain-2026-10-14) (producer before
  the pin, CHANGELOG re-homing, `merge-tree` exit code); [how the stack landed](codebase/cleanup-program.md#how-the-stack-landed)
  (#576 fast-gate gap, #577 Pi flakes). Ad-hoc ids vs AF_UNIX `sun_path` noted on project identity.
- Removed: `scaffold=` mode on `ensure_project_identity`, "edits under `context/<id>/` do not count",
  `mars_fallback: supported|unsupported`, the stale `environment.md` create-on-first-write claim, the
  "#575 merges" revisit trigger, "B7 fixed by #546" (#546 is still open).

---
## 2026-10-14 — Idle reopen is a user return; D1 replaced (`work:idle-close-no-compact`, PR #578, unreleased)

- [Idle notifications](decisions/idle-notifications.md#d-idle-reopen-return--a-meridian-reopen-is-a-user-return):
  **D-idle-stretch-owner rewritten as D-idle-reopen-return** (D6). The
  launcher closes the resumed session's open stretch before exec; D1
  (`owner = MERIDIAN_SPAWN_ID`, `owner-gone`) is now a rejected alternative.
  Removed D1-only claims: guard 4 `owner-gone`, the persisted owner field,
  two-launch mutual cancel, plain-shell arm/fire kill, upgrade losing a
  stretch. Absolute-time invariant, Revisit-when and Sources updated.
- [Idle adapters](architecture/idle-adapters.md#close-and-reopen): close/reopen
  table rewritten (Claude/Pi re-adopt on start but find the row closed;
  Codex/OpenCode never re-adopt); owner-propagation fragility replaced by
  reopen-close placement and harness-key match.
- Ledger row and index description renamed. Break-even research unchanged.

## 2026-10-14 — Launch-owned idle stretches (`work:idle-close-no-compact`, PR #578, unreleased)

- [Idle notifications](decisions/idle-notifications.md#d-idle-stretch-owner--a-stretch-belongs-to-the-launch-that-armed-it):
  new **D-idle-stretch-owner** — a stretch belongs to the launch
  (`MERIDIAN_SPAWN_ID`) that armed it; a reopened launch's fire is `owner-gone`
  and closes it, its arm opens a fresh stretch. The absolute-time invariant now
  says reload recovery holds within one launch only; guard 4 names
  `owner-gone`; state paragraph notes `owner`.
- [Idle adapters](architecture/idle-adapters.md#close-and-reopen): new
  close/reopen table per harness; owner propagation added as a fail-silent
  fragility.
- New research page `research/idle-compaction-break-even.md`
  (analysis, not decided; deleted 2026-10-15), linked from the idle decision as an open question
  and from [prompt-cache retention](research/prompt-cache-retention.md).

## 2026-10-14 — Two-tier project identity and user-scope packages (`work:two-tier-identity`, PR #575)

- [State decisions](decisions/state.md#d-two-tier-identity): replaced the
  "project identity reopened" record with the settled **D-two-tier-identity**:
  invariants, why July's create-on-first-write had to give, settled specifics,
  retention rationale (30 days, `min(N, 7)`, one-hour floor, user-config only),
  rejected alternatives, and what design revision 2 deleted from revision 1
  (absorption, `pending`, `superseded`, the id-keyed lock). Decision 3 keeps its
  record with the create-on-first-write clause marked superseded.
- [Package-management decisions](decisions/package-management.md#d97-user-scope-packages-live-in-an-ordinary-mars-root-2026-10):
  new **D97**: user Mars root `~/.meridian/mars/`, `resolve_mars_roots`,
  `meridian mars --user`, the mutation gate, and the mars-agents
  `--fallback-root` contract and `--help` probe.
- New mechanism pages: [project identity](architecture/state-system/project-identity.md)
  and [Mars user scope](architecture/mars-user-scope.md).
- Removed claims that the first write creates `meridian.toml`, that
  `.meridian/id` is read, that `meridian migrate` exists, or that an
  "established project" predicate exists: [state model](concepts/state-model.md),
  [overview](overview.md), [state-system overview](architecture/state-system/overview.md),
  [roots and work items](architecture/state-system/roots-and-work-items.md)
  (its "Identity Compatibility Migration" section deleted),
  [startup pipeline](architecture/startup-pipeline.md),
  [startup/health decisions](decisions/startup-health-sandbox.md),
  design principle 13 (no VCS dependency for core functionality),
  [state lessons](lessons/state-design-lessons.md),
  [context resolution](concepts/context-resolution.md),
  [future work](open-questions/future-work.md), the
  [cleanup program](codebase/cleanup-program.md) and
  [cleanup decisions](decisions/codebase-cleanup.md).
- Doctor, init and catalog pages: [health checks](operations/health-checks.md)
  (no-state gate, ad-hoc scan, `stale_adhoc_project_dirs`),
  [init](operations/init-command.md) and [bootstrap](operations/bootstrap-command.md)
  (atomic scaffold + id, no carry-over),
  [agent profiles](concepts/model-resolution/agent-profiles.md) and
  [package management](concepts/package-management/overview.md) (content roots).

---
## 2026-10-14 — OpenCode 2.x idle shipped in v0.9.9 (`work:idle-ttl-detection`, PR #573)

- Status flips only: [Idle notifications](decisions/idle-notifications.md),
  [Idle TTL detection](decisions/idle-ttl-detection.md),
  [Prompt-cache retention](research/prompt-cache-retention.md),
  [Idle adapters](architecture/idle-adapters.md) and the
  [decisions index](decisions.md) now say released in v0.9.9 (merge
  `04127891c`) instead of draft or unreleased. Builds up to 0.9.8 keep the V1
  sensor and the 300 s OpenCode default.

---
## 2026-10-14 — OpenAI idle TTL confirmed and measured (`work:idle-ttl-detection`)

- [Idle TTL Detection](decisions/idle-ttl-detection.md): cleared the D5/D7
  review flags (user confirmed 1800 s; still `estimated`) and added the
  provider-not-model TTL limitation.
- [Prompt-Cache Retention](research/prompt-cache-retention.md#measured-idle-window-2026-10-14):
  recorded the `spawn:p7824` ChatGPT-backend probe (Pi window observed,
  OpenCode confounded). Pointed the 1800 s "Revisit when" in
  [Idle Notifications](decisions/idle-notifications.md#revisit-when) at it and
  listed the limitation under [Idle Adapters](architecture/idle-adapters.md#known-fragility).

---
## 2026-10-14 — Renamed harness connection-seam decisions

- Renamed [connection-seams.md](decisions/connection-seams.md) from the
  removed chat-backend name, repaired inbound links, and added the catalog
  entry.

## 2026-10-14 — Cleanup phase 2 captured; chat-backend residue trimmed

- [Cleanup program](codebase/cleanup-program.md): phase 2 marked done with its
  outcome, conventions table, removals and open items;
  [cleanup decisions](decisions/codebase-cleanup.md): `auto_init_cwd` resolved
  (deleted) and new records for state#1 acceptance, the G2 double-validation
  verdict, telemetry batch durability and the JSONL inventory.
- [State decisions](decisions/state.md): new "One JSONL commit rule" record
  (options A/B/C, why A) and "Project identity reopened" (evidence, the user's
  settled sub-decisions, nothing implemented); Decision 3 gains a status line;
  the store-specific-seams record notes `json_doc`.
- Current-truth fixes: [config precedence](concepts/config-precedence.md) no
  longer describes `settings.py` / `BaseSettings` / `hidden_env_specs` and places
  agent overlays in `mars.toml`; [durability and locking](architecture/state-system/durability-and-locking.md)
  gains the JSON sidecar pattern and the JSONL rule; telemetry persistence and
  reader pages, vocabulary, design principles and invariants describe the
  newline commit rule; [startup pipeline](architecture/startup-pipeline.md) and
  [startup decisions](decisions/startup-health-sandbox.md) drop the stale
  `auto_init_cwd` flag and enforced-predicate claims.
- Chat-backend trim (user: "delete them all"): removed the dead `lib/chat/*`
  entries from [future work](open-questions/future-work.md) (section renamed
  "Managed Primary", #498 kept); [connection-seam decisions](decisions/connection-seams.md)
  cut to D15, D16, D23, D32 (D20 was already superseded by the `EventSemantics`
  descriptor in state decisions); removed the chat gap and superseded chat
  decision from the sandbox pages.

---
## 2026-10-14 — Idle adapter reference split

- New [Idle Adapter Architecture](architecture/idle-adapters.md) page owns the
  current idle locations, per-harness coverage, adapter contracts and known
  fragility, moved verbatim from [Idle Notifications](decisions/idle-notifications.md).
- The decision page retains the rationale, rejected alternatives and decision
  records; indexes, domain overviews and cross-references now point readers to
  the appropriate layer.

---
## 2026-10-14 — Idle phase 2 captured (`work:idle-ttl-detection`, PR #573)

- [Idle notifications](decisions/idle-notifications.md): new
  D-opencode-idle-v2 section (V2 sensor, resolved-version dispatch, user return
  on inbox enqueue/delivery, Codex no-change), a per-harness coverage table, V2
  adapter contract; removed the stale OpenCode `summarize`/300 s claims and the
  flat 60 s push.
- [Idle TTL detection](decisions/idle-ttl-detection.md): new OpenCode provider
  TTL section (D7, flagged with D5).
- [Harness integration lessons](lessons/harness-integration.md): new "A dialect
  switch must move every consumer of the stream".
- Marked PR #559 as merged and released in v0.9.5 across the decision index,
  native-session-identity, native-session-binding and claude-native-sessions.

---
## 2026-10-14 — Archived chat app-server pages removed

- Removed on the user's instruction; the chat app server and dev frontend
  (`ChatCommandHandler`, `FrontendLauncher`, `dev_frontend`, `meridian chat --dev`)
  no longer exist in `src/`. Deleted `decisions/dev-frontend.md`,
  `research/vite-portless-funnel.md`, `research/t3-code-reference.md`,
  `lessons/chat-normalization-repair.md`, the `dev_frontend.*` telemetry rows
  (event catalog and overview), and the WebSocket fanout teardown item in
  future-work. Index, domain overviews and inbound links updated.
- Kept `decisions/chat-backend.md`: it records live seams (`ServerRequestHandler`/
  `InteractiveHandler`, `terminal_outcome`/`clears_signal`, spawn `max_depth`,
  TUI passthrough). Only its link to the deleted T3 page was removed.
- Moved the Tailscale Funnel prerequisites into
  [tailscale-serve-semantics.md](research/tailscale-serve-semantics.md);
  `meridian artifact serve --funnel` still depends on them.

---
## 2026-10-14 — meridian-web removed

- Removed meridian-web from the KB on the user's instruction (its REST/WebSocket backend is no longer shipped): ecosystem map section, repo-map diagram edge, index summary.

---
## 2026-10-14 — Cleanup program phase 1 captured (`work:pyright-strict`)

- New [Cleanup program](codebase/cleanup-program.md) page: the six structural
  problems, the five-phase plan with status, phase-1 bugs and deletions, the
  conventions phase 1 made explicit, and what it left open (#569).
- New [Codebase cleanup](decisions/codebase-cleanup.md) decision page:
  D-cleanup-scope, D-b7-via-546, D-spawn-archive-deleted, D-auto-init-inert.
- Launch pages now name the prepare/bind pipeline as the composition seam; the
  production `build_launch_context()` facade is gone (launch-system, invariants
  I-1/I-2/I-6/I-11, launch and model-resolution decisions, vocabularies,
  system overview, composition pipeline, future-work). Closed the
  "subprocess creates the row before composition" gap. Pointed the invariant
  references at the former invariants page (the `.meridian/invariants/` file
  does not exist).
- Removed descriptions of deleted machinery: `requires_app_server`,
  `ExtensionSurface.HTTP` and `project_uuid` (extension system), lazy dispatch
  (startup pipeline and decision), archived-spawns store (state pages),
  `HarnessContract` submodels (harness abstraction), `ops/manifest.py`.
  Flagged the inert `auto_init_cwd` and the meridian-web backend question.
- [Codebase guide](codebase/guide.md): package list, descriptor step for new
  commands, state description. Static typing: resolved the fast-gate flag.

---
## 2026-10-13 — Static typing decisions recorded (`work:pyright-strict`)

- New [Static typing](decisions/static-typing.md) decision page: strict pyright
  with warnings fatal and why April's seven demotions were reversed, the
  per-rule suppression policy, boundary narrowing over casts, public
  cross-module names, explicit nested handler registration, rejected
  alternatives. Flags the fast gate still running pyright without `--warnings`.
- Corrected the type-check command in [codebase guide](codebase/guide.md).
- Indexed in the decision catalog, decisions overview and KB index.

---
## 2026-10-14 — Idle TTL detection shipped; running-session upgrade lesson (`work:idle-ttl-detection`)

- [Idle TTL Detection](decisions/idle-ttl-detection.md): added a v0.9.8 status
  line with the installed-build verification, the D5 cache-hit evidence (flag
  kept), and the consequence that sessions running across the upgrade lose idle.
- New lesson [Bundled mods outlive the CLI they call](lessons/harness-integration.md#bundled-mods-outlive-the-cli-they-call);
  linked from [D-idle-adapter-hosting](decisions/idle-notifications.md#d-idle-adapter-hosting--where-each-adapter-runs).
- Decision index row now says shipped in v0.9.8 instead of "not yet merged".

---
## 2026-10-13 — Idle TTL decision split (`work:idle-ttl-detection`)

- Split the self-contained [D-idle-ttl-detection](decisions/idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd) record from the aggregate idle-notifications page into its own decision page.
- Updated the decision catalog, KB index, research cross-reference, and this log; the adapter contracts remain with adapter hosting because they describe the same boundary.

---
## 2026-10-13 — Idle TTL detection fix recorded (`work:idle-ttl-detection`)

- Synced to branch `fix/idle-ttl-detection` (`556a35795`, not yet merged). Added
  [D-idle-ttl-detection](decisions/idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd):
  Claude TTL detection reads the chat's launch-recorded `native_store` through
  `MERIDIAN_CHAT_ID`, never a cwd; `--cwd` was removed from `meridian idle arm`.
  It also adds the Pi provider table (`openai-codex` 1800 s, estimated;
  `xai`/`deepseek` unknown).
- Superseded the D-pi-cache-retention claim that `openai-codex` stays push-only,
  and dropped its structural-debt row.
- [Prompt-Cache Retention](research/prompt-cache-retention.md): added the Pi
  provider findings and a ChatGPT-backend section. That section corrects the
  claim that GPT-5.6's public-API 30 min applies to ChatGPT-auth Codex. Qualified
  the same claim in [Spawn Wait Barrier](concepts/spawn-wait-barrier.md) and
  [Launch Session Initiation](decisions/launch-session-initiation.md); noted the
  recorded-store input on [Harness Adapters](codebase/harness-adapters.md#idle-bundle-ports).

---
## 2026-10-12 — Probe-safety and branch-forcing lessons recorded (`work:runtime-efficiency`)

- Added two lessons to [Verification and Review Discipline](lessons/verification-and-review-discipline.md):
  force the branch under test (a near-budget fixture whose optional detail still fit
  proved nothing) and assert the non-admitted branch stays owed; audit inherited
  probe cleanup, including a legacy native driver that signalled saved PID/PGID
  despite earlier claims. Production behavior and architecture pages unchanged; no
  other touched node contained a conflicting claim.

---
## 2026-10-12 — Build-provenance lesson recorded (`work:runtime-efficiency`)

- Added to [Verification and Review Discipline](lessons/verification-and-review-discipline.md)
  that packaging does not compile and package-hash parity is not source freshness;
  freezes must compile all bundles from clean source and parity-check before a
  unique manifest. No page in the KB made a positive whole-package qualification
  claim for the stale revisions, so no other node changed.

---
## 2026-10-12 — Owned launch and mandatory notice error recorded (`work:runtime-efficiency`)

- Synced to source `accb1ce5` (draft PR #563; no qualification of these bytes recorded):
  launch separates the owned process from creation observation so foreground abort
  and drain do not await creation metadata; pre-spawn cancellation launches nothing;
  positive launch/handoff still follows publication. Corrected the earlier claim
  that the candidate already covered initial-creation cancellation and clarified that
  only per-chunk records rewrites were removed, not log appends or the combined-log
  stat. Notices keep error presence in the mandatory prefix; a frame that cannot
  fit leaves the item owed. Updated [Pi Runtime](decisions/pi-runtime.md),
  [Coordination](architecture/pi-runtime/coordination.md), the review lesson and
  future work.

---
## 2026-10-12 — Managed-Bash thermo follow-up reconciled (`work:runtime-efficiency`)

- Reconciled against PR #563 source `1cf72c73` (draft, unreleased, no final
  qualification recorded here): publication-owned failure ordering with void
  claims, published snapshot versus live execution, single-flight recovery,
  authoritative log availability and shared task outcome, wait cancellation at
  acknowledgement staging, ESRCH authority retirement, ping lifetime guards,
  truthful declined detach, and metadata-only `/ps` with one selected preview.
- Updated [Pi Runtime Coordination](architecture/pi-runtime/coordination.md),
  [Pi Runtime](decisions/pi-runtime.md), [vocabulary](architecture/pi-runtime/vocab.md),
  [lifecycle](architecture/pi-lifecycle.md) and the ordering lesson in
  [Verification and Review Discipline](lessons/verification-and-review-discipline.md);
  replaced the earlier caller-handler ping description as stale. Routed foreground
  attachment ownership and post-shell polling qualification to
  [future work](open-questions/future-work.md#pi-result-delivery-limits).

---
## 2026-10-12 — Managed-Bash efficiency candidate captured as unreleased (`work:runtime-efficiency`)

- Recorded the source candidate's queue-owned coordination transitions, lifecycle-only
  `log_bytes` snapshots, physical process-group retirement, explicit consumption
  boundary, rejected periodic flush/timer alternatives, and honest crash/error
  linearization limits in [Pi Runtime](decisions/pi-runtime.md).
- Synced to `a6dcc2ca`: cold recovery appends `ownership_lost` to an existing durable
  `execution_error` instead of replacing it.
- Synced to `878d0323`/`2491983d` (v0.9.5 base merged, no schema change): ping-claim failure
  ordering fix recorded in Pi Runtime, with an ordering-test lesson in
  [Verification and Review Discipline](lessons/verification-and-review-discipline.md).
- Kept the shipped v0.9.4 delivery decision and the separate polling, CLI-startup,
  and log-follow follow-ups unchanged. The candidate is approved for its PR only;
  no merge, install, deploy, or production cleanup is claimed.

---
## 2026-10-12 — Pi result delivery reconciled after v0.9.4 (`work:runtime-efficiency`)

- PR #560 merged (`0af8f259`) and shipped in v0.9.4 (`4281e21e`). Folded the
  pending tool-boundary and bounded-aggregate sections into the settled decision in
  `decisions/pi-runtime.md`, removed unmerged/unreleased wording in
  `architecture/pi-runtime/coordination.md`, `architecture/pi-lifecycle.md`,
  `lessons/harness-integration.md` and `open-questions/future-work.md`. 0.9.3 stays
  named only as the idle-only predecessor. Resource, authority and queue-loss limits
  are kept; broader storage, polling, logging and startup cost is issue #561.

---
## 2026-10-11 — Pi tool-boundary result delivery noted as pending (`work:pi-result-delivery`)

- Marked idle-only watcher publication as the installed (0.9.3) contract in
  `architecture/pi-runtime/coordination.md`, `decisions/pi-runtime.md` and
  `lessons/harness-integration.md`, and recorded PR #560's tool-boundary steer as
  pending, unmerged intent.
- Replacement pass: the earlier cancellation-budget/preemption description is
  removed as non-current. The pages now describe the finite idle-only collector,
  direct report reads, hint-never-proves-absence rule and accepted 30.5 s idle
  tradeoff; cross-cutting lessons went to the verification and review-convergence
  pages (stale bundles, delete before cancel). Queue-loss and all-history-scan
  limits are tracked in `open-questions/future-work.md`, not as reproduced bugs.
  Fold into the decision after merge and release.

---
## 2026-10-11 — Pi bounded aggregate delivery captured, pending release (`work:pi-result-delivery`)

- Recorded the 32-member / 64 KiB sequential-read bound, deferred-remainder
  behavior and visible-outcome framing as pending PR #560 source in
  `decisions/pi-runtime.md` and `architecture/pi-runtime/coordination.md`; added
  the per-item-vs-aggregate lesson to `lessons/verification-and-review-discipline.md`
  and routed the cross-cutting efficiency follow-up to issue #561 via
  `open-questions/future-work.md`. Pending merge/release wording kept.
- Follow-up (source 6a509676): fair optional-text allocation replaced the greedy
  packet fill (a review-blocking turn amplification), the queue-loss limit now
  notes it can pause unrelated automatic publication, and the low-priority backlog
  note was reworded. Final native/model/full gates are not claimed here.

---
## 2026-10-11 — Claude native exit verification captured (`work:exit-identity-reporting`)

- Reconciled the native-session identity decision and architecture pages with the
  launch-correlated Claude classic-hook receipt: fresh nonce, actual child PID,
  bounded atomic per-spawn record, immutable entry check, final-reason allowlist,
  and exact final transcript/header validation before exit-chat allocation.
- Marked the reporting-only PR #559 limit (“only Pi verifies exits”; Claude
  `/clear` remains untracked) as superseded while retaining its rationale against
  trampoline/history guesses. Missing or stale evidence, hard kills, and old runs
  remain unresolved/unverified; the production-wheel primary/spawn/fork probe was
  still open at capture.
- Updated native-only-history pruning guidance and the KB catalog/index. The
  separate boundary plugin remains distinct from optional interactive idle.

---
## 2026-10-11 — Final Claude native qualification captured (`work:exit-identity-reporting`)

- Closed the built-wheel evidence gap: real primary `/clear` → `/exit` selected a
  verified new exit chat and exact continuation resumed it; a real cheap Haiku
  spawn matched receipt PID, worker PID and transcript; and the 0.9.3 fork →
  `/clear` → `/exit` path kept the fork entry distinct, verified the post-clear
  exit, and preserved the source transcript hash.
- Captured the source-semantics correction from `b74dc0b3`: native fork startup
  reports `SessionStart.source=fork`, so it is an initial-identity source beside
  `startup` and `resume`. The no-turn/no-transcript case remains unresolved.
- Recorded the alias-safe adapter fix (`3630a1f0b`): every final transcript is
  validated before returning exit evidence, even when the core allocator can
  reuse an existing chat alias without its existence callback. The broad full
  gate was still running; no result is inferred.

---
## 2026-10-08 — Idle notifications designed and built (`work:idle-cache-notify`)

- **New decision page:** [Idle Notifications](decisions/idle-notifications.md).
  Records the user-settled design (U1–U6) for idle push, cache-expiry warning and
  idle compaction in primaries, the runtime-probe corrections, implementation
  seams, and rejected alternatives. Built for all four harnesses.
- **New research page:** [Prompt-Cache Retention by Harness](research/prompt-cache-retention.md).
- **Corrected stale facts:**
  - The harness identity env var is `_MERIDIAN_HARNESS` (since PR #388), not
    `MERIDIAN_HARNESS`. Fixed in launch-system, spawn-wait-barrier,
    config-precedence, configuration-guide, aliases-and-routing and future-work;
    D32 and D57 got a rename note.
  - At design capture, the Pi role var was `_MERIDIAN_PI_SESSION_ROLE`. The
    implementation replaced it with cross-harness `MERIDIAN_SESSION_ROLE` and
    added the primary-only Pi idle extension gate.
  - `autocompact` is a token count, not a percentage. Fixed in the
    config-precedence and configuration-guide examples; the projection bug is
    GitHub #548.
- **Added:**
  - the `MERIDIAN_SECRET_*` stripping boundary
    ([launch-system](architecture/launch-system.md#child-env-boundaries));
  - the catalog-driven config-key declaration rules
    ([config-precedence](concepts/config-precedence.md#declaring-config-keys)).
- **Flagged:** the 3000-second spawn-wait yield default rests on a premise the
  cache research contradicts. Flags are in spawn-wait-barrier and
  launch-session-initiation.

---
## 2026-09-26 — PR #534 shipped as v0.7.0

PR #534 merged to `main` as `3dae2434`; CI released v0.7.0 (`cd063928`). #519, #520,
#526 and #531 are closed as superseded. Status lines on the native-session-identity,
native-only-history, legacy-import, history-storage and harness-session pages now say
shipped instead of unmerged or draft. Checked against the final docs pass
(`d077bd08`, `16316419`, `72741b3a`): Pi primaries load Meridian's managed extensions
with `-e` (fixed in `pi-lifecycle`, `launch-system`, `harness-adapters`); `session
repair --native` covers Claude, Codex, OpenCode and Pi; 0.6.7 runs finish only if they
survive the reinstall.

---
## 2026-09-26 — PR #534 old-Pi recovery, archive digest, upgrade guide

Reconciled with `feat/native-session-identity` @ `50f95c1a` (diff `853aab3a..50f95c1a`).
- **Settled** the open 0.6.7 Pi decision in
  [legacy-native-import](decisions/legacy-native-import.md#old-pi-chats-that-067-never-bound-content-proven-recovery-manual-repair):
  content-proof rule, primaries manual, 155 of 770 bound, `session repair` contract,
  old ID-only repair deleted. Mechanism sections "Legacy Pi recovery" and "Session
  repair" in [architecture](architecture/legacy-native-import.md); bind sources
  updated in binding and identity pages; doctor output in health-checks.
- **New decision** [D-history-portable-digest-stored-json](decisions/history-storage.md#d-history-portable-digest-stored-json);
  portable-history updated. The schema-namespace decision's rollback note now says
  downgrade is unsupported, matching the real 0.6.7 rollback test.
- **Retention:** native-only-history states that harness retention (Claude
  `cleanupPeriodDays`) bounds history unless archived, and links the upgrade guide.
- **Future work:** the Pi entry is removed; #537 (restore `session_sha256`) added.
- **Lessons:** two entries in [native-session-identity](lessons/native-session-identity.md).

---
## 2026-09-26 — PR #534 round-3 probe: index schema namespace, snapshot reads, prompt delivery

Reconciled with `feat/native-session-identity` @ `e194ceea` (diff `242d7eaa..e194ceea`).
- **New decision** [D-history-index-schema-namespace](decisions/history-storage.md#d-history-index-schema-namespace):
  index files, queue, locks and latch named by schema; no in-place migration (0.6.7
  runners wedged); rollback note. Row added to [decisions.md](decisions.md);
  D-history-index-initialization no longer says "outdated".
- **[portable-history](architecture/state-system/portable-history.md):** new sections
  "Index files are named by schema" and "Retained snapshots are read only when a ref
  selects them"; [native-transcript-reads](architecture/native-transcript-reads.md)
  resolver diagram and index-use bullet updated;
  [native-only-history](decisions/native-only-history.md) records the import/restore regression fix.
- **Paths:** `history.sqlite3` → `history-v6.sqlite3` in the state-system overview tree,
  session-state and native-only-history.
- **[launch-system](architecture/launch-system.md#starting-prompt-delivery):** starting
  prompt delivery per harness and the `--add-dir` regression.
- **Open decision:** 0.6.7 Pi chats with no recorded ID, in
  legacy-native-import and future-work (settled the same day; see the entry above).
- **[spawn-output-contract](concepts/spawn-output-contract.md):** `chat_id`,
  `continue_chat_id` and `run_boundary` are in sparse JSON; `harness_session_id` stays out.
- **Lessons:** upgrade probing (`uvx --isolated`, verify old is old, old-state start,
  overlap, no edits under running lanes) in [dogfooding-pr-builds](lessons/dogfooding-pr-builds.md);
  three entries in [native-session-identity](lessons/native-session-identity.md).

---
## 2026-09-26 — PR 3 settled: runner history no longer written

Reconciled with `feat/stop-runner-history` @ `c1fa08e4` (review PASS WITH FIXES, fix
lanes E–G merged).
- **Rewritten:** [attempt-facts-and-delivery](architecture/attempt-facts-and-delivery.md):
  the emit path is hooks → fan-out → `note_event_delivered` → terminal
  (`_run_event_hooks`, no `EmitOutcome`); "What is still written" became "The runner
  stream is retired" (`RETIRED_RUNNER_STREAM_FILENAMES`, blind mode is a read trap only).
- **[decisions/native-only-history](decisions/native-only-history.md):** status landed;
  new "PR 3: the stream is deleted" section with the dogfood-row migration decision
  (read-time rewrite and per-command startup rejected) and the prune-rule decisions
  and measurements; upgrade and leftover-file consequences.
- **New sections:** runner-history prune mechanism in
  [operations/session-archive-pruning](operations/session-archive-pruning.md#runner-history-prune);
  dogfood-row migration in [operations/health-checks](operations/health-checks.md#dogfood-row-migration)
  (doctor flow, `repaired`, warning codes, background repairs); a quarantined-row
  entry in [operations/troubleshooting](operations/troubleshooting.md); the authority
  re-arm in [D-history-index-initialization](decisions/history-storage.md#d-history-index-initialization);
  "only Pi reports an exit identity" in
  [decisions/native-session-identity](decisions/native-session-identity.md#accepted-limits).
- **Structure follow-up:** moved the runner-history prune operator contract out of
  `codebase/session-operations.md` into `operations/session-archive-pruning.md`; that
  page covers explicit maintenance rather than transcript reading. Repaired inbound
  anchors in attempt-facts-and-delivery, state-system overview and health checks.
  Trimmed duplicated migration and prune mechanics from the decision page while
  preserving its rationale, rejected alternatives and measurements.
- **Stale "still written / until PR 3" text removed** from native-transcript-reads,
  state-system overview and portable-history, launch-system (retry rotation list),
  completion-drain-coordination invariant 8, native-session-binding (dogfood rows),
  lessons/harness-integration, codebase/vocabulary, decisions/chat-backend,
  decisions/state (#376 note), `decisions.md` and `index.md` rows.
- **Lessons:** a test helper that reads runner bytes trips the blind trap; a repair
  must clear the failure it latched ([native-session-identity](lessons/native-session-identity.md));
  a lane commit can carry another step's staged deletions — squash-merge
  ([spawn-lane-operations](lessons/spawn-lane-operations.md)).

---
## 2026-09-25 — Structure pass after PR 2 reconciliation

- **Split:** [architecture/native-transcript-reads.md](architecture/native-transcript-reads.md)'s
  "Run facts and delivery" section (the emit path, per-harness attempt folds, report
  precedence, the Pi lifecycle sidecar, other removed stream readers, and what PR 3
  still has to delete) into a new page,
  [architecture/attempt-facts-and-delivery.md](architecture/attempt-facts-and-delivery.md).
  It reads as a separate concern from the resolver/reader and search projection that
  remain in the original page: one is about finding and reading a chat's transcript,
  the other is about computing and delivering a run's own facts from live events.
  Repaired the three inbound `#run-facts-and-delivery` anchors
  ([native-session-binding](architecture/native-session-binding.md),
  [launch-system](architecture/launch-system.md),
  [connection-seam decisions](decisions/connection-seams.md)) to point at the new page's
  `#attempt-folds` or `#the-emit-path` anchors, and added the new page to
  [index.md](index.md), [state-system/overview.md](architecture/state-system/overview.md)
  and [lessons/native-session-identity.md](lessons/native-session-identity.md).
- **Trimmed duplication:** [decisions/native-only-history.md](decisions/native-only-history.md)
  restated measurement tables and mechanism detail (the search projection's schema
  and nomination/verification steps, the query-path timing breakdown, the emit path,
  and the bounded stream-reader replacements) that now lives in the architecture
  pages above. Replaced those passages with links; kept the rationale, rejected
  alternatives, and incident narratives (the NF1 regression, the `sitecustomize`
  argv-gate miss) that are decision-specific.
- **Checked, no change needed:** [lessons/overview.md](lessons/overview.md) and
  [index.md](index.md) already cataloged
  [lessons/native-session-identity.md](lessons/native-session-identity.md) and
  [lessons/spawn-lane-operations.md](lessons/spawn-lane-operations.md).
  [state-system/session-state.md](architecture/state-system/session-state.md) and
  [state-system/durability-and-locking.md](architecture/state-system/durability-and-locking.md)
  already cross-reference correctly; their anchors into
  `native-transcript-reads.md` were unaffected by the split.

---
## 2026-09-25 — PR 2 (native reads) settled; KB reconciled with what landed

PR 2 is draft PR #526 (`feat/native-reads` @ `3ae3fce8`). Reviews, fix lanes A–D, the
recheck and the before/after measurement on runtime copies are done. Pages that
described it at `1124ef38` as partly "designed" now describe the landed code.

- **Rewritten:**
  [architecture/native-transcript-reads.md](architecture/native-transcript-reads.md).
  It now covers:
  - `SessionLogTarget(source, view_label)` and the one `TranscriptSource.native`
    constructor;
  - the `NativeRead` reader;
  - capture reading the exact generation's key;
  - witness and snapshot modules in `harness/`, and `SearchProjection`;
  - the S1 `complete` rule;
  - `session index` status and rebuild;
  - `EmitOutcome`, `AttemptFold` factories, the NF1 usage rule and report precedence;
  - the Pi `event_sinks` sidecar;
  - the lazy blind-mode hook;
  - V2 measurements.
- **Updated:**
  - [decisions/native-only-history.md](decisions/native-only-history.md): status;
    bindings from `by_native_key` with the index alternative rejected; the S1
    `complete` rule; V2 numbers; the < 1 s miss accepted and attributed to CLI import
    cost (#527); R3 landed results; NF1; blind-mode hook with rejected argv gate; PR 3
    scope; `--include-archives` (search removed, browse filter kept).
  - Status rows in [decisions.md](decisions.md),
    [native-session-identity](decisions/native-session-identity.md),
    [native-session-binding](architecture/native-session-binding.md) and
    [session-operations](codebase/session-operations.md).
  - [decisions/connection-seams.md](decisions/connection-seams.md): `EventObserverRegistry`
    was deleted (D15 partly superseded).
  - [decisions/history-storage.md](decisions/history-storage.md): PR #494 merged
    (2026-09-18, v0.5.0); it is no longer a draft.
  - [state-system/session-state.md](architecture/state-system/session-state.md),
    [state-system/overview.md](architecture/state-system/overview.md) and
    [durability-and-locking](architecture/state-system/durability-and-locking.md):
    removed `sessions-index.sqlite3`, `session_journal.py`,
    `sessions-append-state.json` and `session_aggregate.py`, which no code references;
    documented the two `history-index/` projections.
  - [codebase/test-determinism.md](codebase/test-determinism.md):
    `wait_for_history_phase` → `wait_for_phase` over observed live events.
  - [operations/troubleshooting.md](operations/troubleshooting.md),
    [operations/health-checks.md](operations/health-checks.md),
    [lessons/state-design-lessons.md](lessons/state-design-lessons.md),
    [lessons/harness-integration.md](lessons/harness-integration.md) and
    [architecture/launch-system.md](architecture/launch-system.md): no user or agent
    guidance to read runner `history.jsonl`; the liveness artifact set is current.
- **New lessons:** an argv-gated `sitecustomize` misses `python -m` children; a
  diagnostic flag must not discard authoritative cost (in
  [lessons/native-session-identity.md](lessons/native-session-identity.md)). New page
  [lessons/spawn-lane-operations.md](lessons/spawn-lane-operations.md): declared
  backups do not cover a mid-run 401, and a Claude-routed lane can end its turn before
  a background gate finishes.

---
## 2026-09-25 — Native session identity decision split

Moved the one-time legacy-key import rationale and policy from
[native session identity](decisions/native-session-identity.md) into its own
[decision page](decisions/legacy-native-import.md). The broader identity decision
now links to it; the mechanics remain in
[legacy native import architecture](architecture/legacy-native-import.md). Updated
the architecture and native-only-history inbound links, decision overview, and
catalog. The existing preview-rebuild inconsistency in history-storage remains
flagged for content-owner review; no truth was changed.

---
## 2026-09-25 — PR 1 restructure settled; native-only history (PR 2) captured

PR 1 (#520) finished its foundation restructure (P0–P5, head `2eddcd68`). PR 2's
settled decisions are captured while it is being built on `feat/native-reads`.

- **New pages:**
  - [decisions/native-only-history.md](decisions/native-only-history.md): why history
    is native-only; old runner history option C (drop); the search projection
    choice and its rejected alternatives; R2b's accepted trade-offs; the metadata
    index calling the shared fold (R3); run facts; retention decisions.
  - [architecture/native-transcript-reads.md](architecture/native-transcript-reads.md):
    the one resolver, pN labels, the search projection, the emit path and the Pi
    phase sidecar, each marked merged or designed.
  - [lessons/dogfooding-pr-builds.md](lessons/dogfooding-pr-builds.md): PR builds on
    real roots, nested-env pinning, `uv tool install` relinking, scratch placement,
    and masked push exit codes.
- **Rewritten:**
  [architecture/native-session-binding.md](architecture/native-session-binding.md) now
  describes the restructured seams. The deleted `NativeIdentityPlan`,
  `verify_native_identity`, `finalize_run_boundary` and entry/exit spawn columns are
  gone from the page.
- **Updated:**
  - [decisions/native-session-identity.md](decisions/native-session-identity.md):
    status; the binding and drift rules; the run-boundary table; import results with
    Pi interactive-root candidates; restructure rejected alternatives; phases for
    PR 2 and PR 3.
  - [architecture/legacy-native-import.md](architecture/legacy-native-import.md):
    in-place OpenCode read, 15-minute deferral backoff, Pi root candidates.
  - [architecture/pi-native-sessions.md](architecture/pi-native-sessions.md):
    reopen-lineage readback.
  - [architecture/claude-native-sessions.md](architecture/claude-native-sessions.md):
    `observe_after_exit`.
  - [lessons/native-session-identity.md](lessons/native-session-identity.md):
    `mode=ro` recalibrated; three new lessons.
  - [lessons/harness-integration.md](lessons/harness-integration.md): harness store
    env vars cleared in conftest.
  - the former invariants page I-4 and [architecture/launch-system.md](architecture/launch-system.md):
    `conclude_native_run`.
- **Stale runner-history claims removed:**
  - [decisions/history-storage.md](decisions/history-storage.md) marks the parts that
    are superseded (content authority, the FTS rejection, the runner-stream preview
    checkpoint, eager preview warming).
  - [architecture/state-system/portable-history.md](architecture/state-system/portable-history.md)
    is scoped to retention and archive, with native-only capture.
  - [codebase/session-operations.md](codebase/session-operations.md) gets the one
    resolver; the nonexistent segment file names are removed.
  - [codebase/session-log-rendering.md](codebase/session-log-rendering.md),
    [codebase/vocabulary.md](codebase/vocabulary.md),
    [codebase/harness-adapters.md](codebase/harness-adapters.md),
    [architecture/state-system/overview.md](architecture/state-system/overview.md) and
    [architecture/state-system/session-state.md](architecture/state-system/session-state.md)
    are updated to match.
- **Indexes:** [decisions.md](decisions.md) and [index.md](index.md) are updated.

Provenance: `work:native-harness-session-identity`, `chat:c6945`; investigators
`spawn:p7133` (PR 1 code), `spawn:p7134` (KB audit), `spawn:p7135` (PR 2 branch) and
`spawn:p7136` (session mining).

---
## 2026-09-25 — Native session identity: one-time legacy import (PR 1 slice)

Captured the user decision "Auto-import once" and the slice at `8e8fe485`
(`slice/pr1-legacy-import`, which merges into #520).
[decisions/native-session-identity.md](decisions/native-session-identity.md) gains a
new section on importing pre-key chats once: its rules, real-state counts, and the
scope of Pi's own spawn dirs, which is recorded as an open decision and not as a
harness limit. The section also adds two rejected alternatives and a phase-1b row. It
narrows "incomplete keys are never repaired" so the rule excludes only the one-time
import.
[architecture/native-session-binding.md](architecture/native-session-binding.md)
describes the shared lock-scoped `session_bindings()` path. The one-time migration
details move to [architecture/legacy-native-import.md](architecture/legacy-native-import.md),
which covers the trigger, sequence, crash recovery, candidate stores, the OpenCode
copy snapshot with its fingerprint, deferral, report mode, and cost.
[architecture/state-system/session-state.md](architecture/state-system/session-state.md)
names the marker. A new lesson, "A Once-Only Marker Turns Transient Failures Into
Permanent Ones", goes in
[lessons/native-session-identity.md](lessons/native-session-identity.md). The rows
in decisions.md and index.md are updated to match.

Reconciled at `fc234735` (documentation only after `8e8fe485`): the Pi scope
paragraph no longer implies that all 32 top-level Pi root files would bind. A
read-only comparison matched 7 files to 9 recorded chats.

---
## 2026-09-25 — Native session identity: whole-change review, fix pass, real-Pi one turn (phase 3 capture)

Corrected phase-2 text that made the Claude `/tui fullscreen` trampoline successor
the run's exit key: the whole-change review showed the correlation accepts an
unrelated concurrent chat, the `exit_key` plumbing was deleted, and Claude exit stays
`unresolved` with the successor as a diagnostic only. Updated
[decisions/native-session-identity.md](decisions/native-session-identity.md)
(header-validated locators, primary-runner initial check, two typed failures through
one route, complete-key tracked reads, Claude exit limit, Q3 qualification, phases
mapped to the planned E/F PRs), [architecture/native-session-binding.md](architecture/native-session-binding.md)
(new Runner order section with the teardown join before the boundary read; typed
refusal sources; tracked reads; testing and preflight),
[architecture/claude-native-sessions.md](architecture/claude-native-sessions.md)
(first-line `sessionId` validation, atomic symlink publication, diagnostic-only
successor), and [architecture/pi-native-sessions.md](architecture/pi-native-sessions.md)
(typed `unbound`, agent-dir credential route, one-turn create, read-after-teardown,
v2 cost). Fixed matching claims in harness-adapters, launch-harness-compatibility,
session-operations, decisions.md, and index.md. Added lessons: "Terminal Status Is
Not Process Exit" (harness-integration), "Lane Reviews Do Not Review the
Composition" (review-convergence-gate), and "Mirror CI prerequisites in the local
gate" (verification-and-review-discipline); extended "Green Suites Did Not Find the
Seam Defects". Split the three native-session lessons (Green Suites…, Terminal
Status…, When a Fix Pass Is Not Converging…) out of harness-integration into
[lessons/native-session-identity.md](lessons/native-session-identity.md).

Provenance: `work:native-harness-session-identity` (`review/integrated-pr1.md`,
`evidence/integrated-fix-report.md`, `evidence/lane-q3-report.md`,
`evidence/lane-q2-report.md`, `evidence/lane-d-fix2-report.md`); source
`fix/native-session-wrapper` @ `95db4d03`.

---
## 2026-09-25 — Native session identity: source key, exit identity, real-Pi facts (phase 2 capture)

Extended [decisions/native-session-identity.md](decisions/native-session-identity.md)
with the one-recorded-source-key decision (and the two P1s that forced it),
qualified-event-only identity for all harnesses, typed `ambiguous_native_file`, and
observed exit identity (Pi quit, Claude trampoline successor as exit key); phases
updated. Rewrote [architecture/native-session-binding.md](architecture/native-session-binding.md)
for the source key, runner order, run boundary, typed refusals, and per-harness
store→child mapping. Added exit observation and runtime-verified Pi 0.87.1 facts
(lazy persistence, ctx invalidation on replacement, cost) to
[architecture/pi-native-sessions.md](architecture/pi-native-sessions.md).
Rewrote [architecture/claude-native-sessions.md](architecture/claude-native-sessions.md)
as Claude native sessions: config-dir overlays were removed from the code in May, so
the overlay, materialization, and `--continue`-after-isolation content was deleted.
Removed the matching overlay-pruning content from
[operations/health-checks.md](operations/health-checks.md). Corrected Claude
transcript resolution (no trust-ordered root chain) in session-operations, and the
trampoline status in launch-harness-compatibility and harness-adapters. Added two
process lessons to [lessons/harness-integration.md](lessons/harness-integration.md).

Provenance: `work:native-harness-session-identity`.

---
## 2026-09-24 — Native session identity structure and continuation model selection

Clarified page ownership across the native identity decision, cross-harness
binding architecture, and Pi-specific session behavior; added binding links from
the harness concept and state-system overview. Split continuation model-selection
and startup-selection material from [session initiation](concepts/session-initiation.md)
into [continuation model selection](concepts/continuation-model-selection.md).

---
## 2026-09-24 — Native session identity: exact-target entry (phase 1 capture)

Rewrote [decisions/native-session-identity.md](decisions/native-session-identity.md)
around the settled exact-target entry rule and recorded why the observed-entry /
Pi RPC-primary plan was dropped. Added
[architecture/native-session-binding.md](architecture/native-session-binding.md)
for the cross-harness seams. Rewrote
[architecture/pi-native-sessions.md](architecture/pi-native-sessions.md) from
discovery-era behavior to exact identity plus verified Pi 0.87.1 facts, keeping
the readback-topology section. Marked
[decisions/native-source-argument-admission.md](decisions/native-source-argument-admission.md)
superseded. Narrowed recovery in session-reference-resolution, made Claude
trampoline handling diagnostic-only across claude-session-isolation,
launch-harness-compatibility and harness-adapters, and corrected identity
claims in session-initiation, launch-system, session-operations,
session-state, invariants, lessons and vocabulary.

Provenance: `work:native-harness-session-identity`.

---
## 2026-09-23 — Native session identity decision

Added [decisions/native-session-identity.md](decisions/native-session-identity.md)
as the canonical record for the approved, unimplemented cross-harness contract.
Added it to the decision catalogs and vocabulary, and linked the earlier
history-storage and session-reference-resolution records plus portable-history
architecture to clarify their relationship to the new decision. No source or
runtime behavior is claimed as changed.

Provenance: `work:native-harness-session-identity`.

---
## 2026-09-23 — Pi native session identity and readback

Added [architecture/pi-native-sessions.md](architecture/pi-native-sessions.md)
to record how fresh Pi primaries acquire a native session identity and how Pi
journals are read back — the two mechanisms behind a confirmed cross-conversation
leak and display mixing. The page documents current behavior only: primary
identity is discovered from a shared session directory by newest-mtime among
same-`cwd` candidates (collision hazard; persisted ID is not proof of a correct
association), and readback renders physical JSONL order rather than Pi's active
root-to-leaf branch. No correction is prescribed; the source work item remains
open. Cross-linked from `harness-adapters`, `session-operations`,
`session-log-rendering`, `pi-lifecycle`, and `claude-session-isolation`.

Provenance: `work:investigate-pi-model-selection-for-luna`; `spawn:p6615`.

## 2026-09-22 — Canonical self force-takeover reconciliation

Corrected stale feature-branch status for declared-package self-sync and
pending-canonical recovery: both shipped in Mars 0.13.2 through PR #164.
Recorded D96, which keeps default sync protective while making explicit
`mars sync --force` the authority to overwrite an eligible selected canonical
self path and publish installed ownership. Reconciled the decision and
mechanism pages after implementation and contract verification on feature
branch `f7a18fd`: unsafe links and wrong-shaped paths remain protected, and
preexisting adoptions remain outside pending-canonical recovery. The branch is
not merged or released, so every page retains the Mars 0.14.1 runtime
divergence.

---
## 2026-09-16 — Target-constrained fallback shipped

Mars 0.14.0 / Meridian 0.4.6-rc.1. Updated model-policies overlay vs profile-backup
boundary, launch-bundle schema 4 / report 2, routing target permission (deleted
linked-harness fallback), continue recorded selection, and session `model_selection`
events. Marked the fallback decision as shipped.

---
## 2026-09-15 — Split history-storage decisions from state decisions

Moved the self-contained history-authority, index, retention, restore, and native-capture
records from `decisions/state.md` to `decisions/history-storage.md`. Updated the
decision catalog, chronological links, and portable-history cross-reference; retained
the original record anchors. The runtime-state page now links to the extracted page.

Validation: `meridian kg check kb`; `meridian mermaid check kb`.

---


## 2026-09-14 — Post-stop capture scope correction

The user clarified that existing capture should be repaired rather than a new
snapshot system invented. Updated the current state decision, architecture and
index to withdraw universal exact-stop reconstruction and select post-stop native
observation with exact record association. p6109 approved the direction with a medium
qualification-result finding; the primary corrected it and p6110 closed it at design
level. Updated architecture, decision and index summaries to remove pending-review
wording. No production implementation, runtime or PR-readiness approval is claimed.
The earlier account exhaustion remains historical evidence, not a blocker on these
completed reviews. Source: `work:next-minor-planning/reviews/post-stop-scope-review.md`.


## 2026-09-14 — History #499/#500 investigation capture

Captured settled repair intent from `work:next-minor-planning` without changing
production or active design artifacts. The portable-history architecture and state
decision trail now record the separate 15-second index-initialization gate, same-lock
recheck, durable genuine-failure latch, manual retry, and non-sticky contention/
cancellation contract for #500.

For #499, the KB now separates the preexisting Pi native-message grammar defect from
PR #494's partial-stream capture/selection regression and corrects the stale OpenCode
source-precedence description. It records the preferred
separate atomic snapshot, raw OpenCode preservation, bounded shared validation, and
the immutable original descriptor required across rearchive. At that review,
exact provider completion qualification remained open; reviewer p6104 accepted F2–F5
at design-contract level only. The post-stop scope correction above supersedes the
exact-stop requirement; neither review establishes implementation or runtime proof.

PR #494 remains draft, all repair TODOs remain there, and #498 is excluded. Earlier
suite/CI and pinned approvals do not close the newly discovered issues.

Validation: `meridian kg check .`; `meridian mermaid check .`.

---

## 2026-08-26 — Session-browse implementation reconciliation (work:session-browse)

Reconciled the pre-implementation capture against the approved implementation
at `cfcd0e1a`. The re-entry decision now records the shared durable-id authority
used by both listing and Enter-time resolution. The Codex live-fork material is
current truth: the earlier corruption remains as rationale, while the documented
contract now describes bounded streaming, complete-record publication, and
commit-aware compensation. Removed all live claims that the Codex fix was still
pending. Narrowed the prompt_toolkit decision from an agent-inferred global TUI
mandate to the session-browse picker scope the human actually approved.

No `DIVERGENCE/` log existed for this work item; the final review and runtime
probe reported full requirements alignment.

Validation: `meridian kg check .`; `meridian mermaid check .`.

---

## 2026-08-07 — Session-browse design knowledge capture (session-browse design, chat:c5810)

### Trigger

The session-browse design phase settled (four design rounds, a reversed TUI
decision backed by three evidence lanes, and requirements sign-off). Implementation
not started. Capturing durable decisions before implementation begins.

### What changed

**New content:**
- `decisions/tui-framework.md`: D-tui-prompt-toolkit decision. prompt_toolkit
  full-screen Application chosen for the session-browse picker. Evidence synthesis
  from POC bake-off (3 variants, measured), picker source study (6 tools, lifecycle
  LOC quantified), and web stability research. Rejected: hand-rolled rich
  Live+termios, textual, fzf subprocess, curses.

**Updated content:**
- `concepts/session-initiation.md`: new "Session Re-entry Model" section documenting
  `Resume | Fork | Blocked(reason)`, advisory vs authoritative resolution, fork-on-live
  semantics, per-harness fork safety table. Updated "Bare Flag Inference" section:
  bare `--continue` now canonicalizes to `session browse` (supersedes "intentionally
  excluded" note). Updated argv normalization for `canonicalize_argv`.
- `decisions/launch-session-initiation.md`: D-bare-continue-browse decision (argv
  rewrite, supersedes D-argv-normalization-sentinel exclusion note). D-session-reentry
  decision (ops-owned re-entry model, fork-on-live rationale). Supersession note
  added to D-argv-normalization-sentinel bare-ref semantics.
- `codebase/session-operations.md`: new "Session Browse" section documenting the
  command surface, entry forms, presentation gating, and ops-layer seams.
- `lessons/harness-integration.md`: new "Codex Live-Fork Rollout Corruption" lesson
  documenting probe-proven partial JSONL cloning on live sources, required fix shape,
  and the complete-records-only snapshot discipline.

**Navigation updates:**
- `decisions.md`: three new chronological entries (D-tui-prompt-toolkit,
  D-bare-continue-browse, D-session-reentry). TUI framework added as a domain row.
  Waiting-and-sessions domain row updated.
- `index.md`: `tui-framework.md` added to decisions section. Session-initiation
  description updated with re-entry model.
- `decisions/overview.md`: TUI framework domain page added.
- `concepts/overview.md`: session-initiation description updated.

**Colocated (meridian-cli, not this repo):**
- `src/meridian/lib/harness/.context/CONTEXT.md`: new "Codex: Live-Fork Rollout
  Corruption" section documenting the known bug and required fix, placed where the
  next Codex-touching agent will see it.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-29 — Engine version constraints knowledge capture (mars-agents PR #158, meridian-cli PR #485)

### Trigger

The `requires-mars` / `requires-meridian` feature settled: implemented, reviewed through four rounds, runtime-probed, and shipped as ready-for-review PRs. Mars-agents PR #158 (release:minor) adds the resolver walk-down, engine exclusion map, CLI escape hatches, and `engine_fallbacks` report. Meridian-cli PR #485 (release:patch) exports `MERIDIAN_VERSION` to the mars subprocess.

### What changed

**New content:**
- `decisions/package-management.md` D93: hard-constraint-with-fallback engine requirements. Source study synthesis (pip/Cargo/Bundler/npm/Go), rejected alternatives table, parsing rules, exclusion provenance, diagnostics-derive-from-final-graph lesson, migration strategy.
- `concepts/package-management/resolution-algorithm.md`: new "Engine Version Constraints" section with walk-down loop diagram, exclusion set mechanics, engine check placement, version parsing rules, and new invariant I-6 (engine exclusions are monotone).
- `concepts/package-management/vocabulary.md`: three new terms (engine exclusions, engine fallback, engine requirement).
- `concepts/package-management/overview.md`: `requires-mars` and `requires-meridian` added to `mars.toml` example.
- `concepts/package-management/sync-model.md`: `--ignore-requires-mars` and `--ignore-requires-meridian` added to sync modes table; `engine_fallbacks` noted in SyncReport description.

**Navigation updates:**
- `decisions.md`: D93 entry added to both the chronological table and the Package Management domain row.
- `decisions/overview.md`: D93 added to the package-management domain page listing.

### Colocated (mars-agents, not this repo)

- `src/resolve/AGENTS.md`: already updated on the feature branch. Verified accurate against final code: restart algorithm convergence argument includes engine exclusion monotone, engine check placement, walk-down semantics.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-25 — Release-sequencing failure capture and ecosystem staleness repair

### Trigger

mars-agents 0.12.0 shipped a breaking hook-schema change while the
meridian-base fragment migration sat unpushed since 24 Jul. Every published
meridian-base was rejected by the new mars. The failure exposed a structural
gap: cross-repo release sequencing has no enforcement.

### What changed

**New content:**
- `lessons/release-sequencing.md`: cross-repo release sequencing discipline.
  The structural gap (independent repos, independent release processes,
  nothing pairing them), the concrete instance (mars 0.12.0 vs unpushed
  meridian-base 0.10.0), and three discipline points (verify consumer
  migration is published, search at the data's location, runtime probe for
  cross-repo bumps).

**Extended content:**
- `lessons/verification-and-review-discipline.md`: new "Verify the search
  scope contains the fact" rule under Verification Validity. A negative
  search result is evidence only when the target can contain the fact.
- `lessons/verification-campaign-history.md`: new "Static gate fully green
  on a shipped ecosystem break" section documenting the second confirming
  instance of the runtime-probe principle (cross-repo dependency break vs
  the prior within-repo schema-change instance).
- `operations/troubleshooting.md`: new "Pre-Init Failed" section
  documenting the two known causes of `pre_init_failed` (stale task_dir
  from removed worktree; Claude blocked by deny_headless_harnesses) and
  the diagnostic that `--task-dir` does not override the stale work-item
  value (meridian-cli#482).

**Stale content fixed:**
- `ecosystem/prompt-packages/meridian-base.md`: version 0.8.9 updated to
  0.10.0. Hooks section rewritten from D90 per-target event schema to D91
  native fragment schema (claude.json/codex.json fragments, `${MARS_HOOK_DIR}`
  substitution, matcher fields). mars-agents version requirement updated
  from "post-v0.10.6, PR #133" to "0.12.0+ (D91 fragment schema)".

**Navigation updates:**
- `lessons/overview.md`: added release-sequencing entry; updated
  verification-and-review-discipline description.
- `index.md`: added release-sequencing entry; updated
  verification-and-review-discipline description.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-25 — KB structural consolidation and staleness repair

- Replaced the root overview with the current CLI/MCP, Mars `.mars/`, harness
  bundle, and `meridian.toml` identity model.
- Converted `decisions.md` to an index and split launch decisions into
  composition, process ownership, session initiation, and compatibility pages.
- Made state architecture canonical for on-disk mechanics; recast the state
  concept and lesson pages as mental model and explicit history.
- Split reusable verification rules and the convergence gate from the retained
  campaign chronology.
- Updated Mars ownership/retention, target output, cache, routing, harness, and
  startup pages against current source; deleted the duplicate observability
  page after moving its durable/transient boundary to telemetry architecture.
- Concentrated state and source-simplification measurement provenance on their
  canonical pages.

## 2026-07-25 — Convergence-gate phase capture (mars-agents PR #147, consolidated)

### Trigger

The review-gate phase of `mars-native-hook-events` settled: 3 review
rounds on the consolidated PR #147 (retargeted from `chore/delete-dead-code`
to `main`, absorbing PRs #138, #146, #139/#140, perf optimizations, and
dead-code deletion). Blocking findings 3 -> 1 -> 0. Lock v3, ownership
lifecycle, data-loss fixes, no-op sync optimization, and dead-code deletion
all ship as one minor breaking release.

### What changed

**Stale content fixed:**
- `concepts/package-management/sync-model.md`: lock version updated from v2
  to v3 throughout. v2 lock TOML example replaced with v3 schema showing
  per-output lifecycle state. Dead `Merge` action removed from Plan->Apply
  table (three-way merge was deleted as dead code). Invariant I-6 updated
  from "v2 lock is always written" to v3. v2 promotion description updated
  for disk-consulting classification. `reconcile/fs_ops.rs` reference
  removed (module was cleaned up in dead-code deletion).
- `concepts/package-management/vocabulary.md`: "Mars lock" entry updated
  from "Schema v2" to v3 with lifecycle state description and v2 promotion
  semantics.

**Updated content:**
- `lessons/verification-and-review-discipline.md`: three new sections added
  from the convergence-gate phase:
  - Symlink-following hash bug class: 4 instances across 3 review rounds
    (mars-agents#152), the structural rule that ownership-deciding
    comparisons must use symlink_metadata-based validation.
  - Convergence-gate process: frozen read-only worktrees, fix lanes from
    findings, severity-must-fall trajectory, cross-model review for
    correlated blind spots, one-editor-per-checkout rule.
  - CI trigger gap: `pull_request` default types exclude `edited`;
    retargeting a PR base does not queue CI; close/reopen workaround.
- `lessons/overview.md`: lesson page description updated.
- `index.md`: lesson page description updated.

**Colocated (mars-agents, not this repo):**
- `src/surface_ownership/.context/CONTEXT.md`: deletion-vs-replacement
  authority distinction made explicit (the contract gap that caused 8 review
  rounds to miss half the ownership model).

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-24 — Hook-fragment review lessons and two-surface ownership capture (mars-agents PR #138)

### Trigger

Eleven review rounds on the mars-agents hook-fragment implementation (PR #138)
settled with a SHIP verdict. The review produced five generalizable engineering
lessons and a significant architectural addition (the two-surface ownership
model and retention seam) that the prior D91 capture did not cover.

### What changed

**New content:**
- `lessons/verification-and-review-discipline.md`: five cross-cutting patterns
  grounded in specific evidence -- capability token consumption, wrong-gate
  regression tests, vacuous verification, diff-vs-run, convergence as a
  redesign signal. These are general engineering knowledge that outlives
  mars-agents.
- `architecture/mars-compiler.md`: new "Two-Surface Ownership Model" section
  documenting path ownership (`OutputRecord`) and config-entry ownership
  (`ConfigEntryRecord` + `emitted_json`), plus the retention seam
  (`surface_ownership/retention.rs`) that makes wrong states unrepresentable.

**Stale content fixed:**
- `architecture/mars-compiler.md`: module map updated -- removed deleted
  `stale.rs` from `config_entries/`, added `surface_ownership/` directory
  module with `mod.rs` and `retention.rs`.
- `open-questions/mars-feature-gaps.md` section 3 (Hook Distribution):
  updated from D90-only to cover D91 (native fragments). Cursor hook writing
  (mars-agents#131) marked resolved. Priority list updated.
- `decisions/package-management.md` D91: added the two-surface ownership
  contract and retention seam as a key implementation paragraph.
- `decisions.md`: D91 foundational entry expanded with two-surface ownership
  model and retention seam reference.

**Navigation updates:**
- `lessons/overview.md`: added new lesson page.
- `index.md`: added new lesson page to lessons section.

### Colocated gaps reported (not edited)

Mars-agents `src/surface_ownership/` lacks a `.context/CONTEXT.md`. The module
doc in `mod.rs` covers both surfaces and the retention contract, but the
retention seam's design rationale (why `RemovalPlan::build` partitions before
mutation, why `WritePermit` binds rather than carries, why file-output removals
are deliberately outside the outcome model) is not captured in colocated docs.
Proposed content reported in the KB-lead's final message.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-24 — KB-autosync gap chain knowledge capture (reaper-scope-regression phase 2)

### Trigger

The `reaper-scope-regression` work item's second phase settled: root-caused
the stranded-commit gap in git-autosync, fixed the mars-agents hook event
vocabulary bug (PR #129, v0.10.6), shipped the `context-autosync` hook in
meridian-base (v0.8.9), and completed a full audit of mars hook event
mappings revealing 6 of 12 wrong.

### What changed

**Stale content fixed:**
- `open-questions/mars-feature-gaps.md` section 3 (Hook Distribution):
  rewrote from "partially resolved" to document the full vocabulary audit
  findings (Claude fix shipped, Codex/OpenCode/Cursor bugs live), the
  stale-artifact residue problem, and the settled native-passthrough
  direction (future work item `mars-native-hook-events`).

**New content:**
- `ecosystem/prompt-packages/meridian-base.md`: version updated to 0.8.9;
  new Hooks section documenting all three Mars-compiled hooks
  (`deny-generic-agent`, `deny-interactive-prompts`, `context-autosync`)
  with the session-end gap rationale and mars-agents version requirement.
- `concepts/hooks-and-plugins.md`: new "Session-End Coverage Gap" section
  documenting the builtin-hook coverage gap and the `context-autosync`
  bridge between Mars-compiled harness hooks and Meridian's builtin hook
  system.
- `codebase/test-determinism.md`: new "Tight Subprocess Timeouts" flake
  class documenting the `test_spawn_prompt_input.py` instance (timeout=3
  on `subprocess.run`, lost to CI load in #469).

### Already current (verified, no change needed)

- `architecture/mars-compiler.md`: describes the compiler pipeline and
  hooks module architecture without specific event mappings — nothing
  contradicts the fix.
- mars-agents `docs/config/mcp-and-hooks.md`: Claude `session.end` row
  already updated to `SessionEnd (exact)` in PR #129. Remaining rows
  (Codex, OpenCode, Cursor) accurately describe current (buggy) mars
  behavior — the KB captures the audit findings, not a docs correction
  for behavior that still exists.
- `decisions/package-management.md`: hook-related decisions (D36, D37)
  cover collision resolution and Windows quoting, not event vocabulary —
  no stale content.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-24 — Triage-designs-impl knowledge capture (PRs #466/#467, #113/#171 pending)

### Trigger

Four triaged design issues went design → implementation → adversarial review →
fix pass → PRs: #449 (Codex backend death), #171 (Claude transcript
resolution), #334 (params.json bind refresh), #113 (sparse JSON projections).
All four settled with PRs open; merge gate awaiting human approval.

### What changed

**Stale content fixed:**
- `concepts/spawn-output-contract.md`: "spawn show is full inspection, all
  fields" replaced with curated sparse JSON projection via `to_cli_wire()`.
  Progressive disclosure table updated. Multi-spawn behavior updated for
  per-spawn projection. New "Sparse JSON Projections" section covers
  `spawn show` (tri-state `--full`, `harness_session_id` drop), `spawn wait`,
  `work show`, `hooks run`, and contract tests.
- `open-questions/process-scope.md`: PROC-009 updated to mark diagnostics and
  liveness-projection pieces as shipped (PR #467). Remaining scope narrowed to
  the identity redesign.
- `open-questions/future-work.md`: deleted resolved "Codex Death Diagnostics
  (#449)" entry (shipped in PR #467).

**New content:**
- `codebase/harness-adapters.md`: new "Connection Death Diagnostics" section
  documenting the single idempotent terminal owner pattern, post-connect
  enrichment parity, and cause-typing stability.
- `codebase/session-operations.md`: "Transcript Source Resolution" expanded with
  trust-ordered root chain (recorded → canonical → ambient), `config_root_hint`
  seam, rejected alternatives (persist path, canonicalize at creation).
- `architecture/process-scope.md`: new "Scope Liveness Projection" section
  documenting `scope_liveness()`, `is_pgid_reachable`, and the three-signal
  model wired through all reaper consumers.
- `architecture/state-system.md`: `params.json` annotated as two-phase in the
  directory layout.
- `decisions.md`: four new foundational entries (sparse JSON projections, single
  terminal owner, trust-ordered transcript resolution, two-phase params.json).
- `lessons/harness-integration.md`: new "Optional Defaults on Widened Seams Mask
  Missed Callers" lesson from the #171 review-gate finding.
- `lessons/overview.md`: lesson index updated.

### Already current (verified, no change needed)

- `concepts/harness-abstraction.md`: connection death shape section already
  covers cross-harness normalization from the PRs #446/#447 audit; #449 adds
  Codex enrichment detail in `codebase/harness-adapters.md` without changing the
  cross-cutting invariant.
- `concepts/spawn-lifecycle.md`: no stale content from this sprint.
- `decisions/session-reference-resolution.md`: recovery provenance chain is
  orthogonal to config-root resolution; no update needed.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-23 — Aggressive unit-test prune knowledge capture (PRs #462/#463)

### Trigger

The delete/re-tier and functional-core shrink slices settled with both PRs open,
reviewed, and gated.

### What changed

- Added `decisions/testing.md` for behavior-owned test tiers, the coherent
  security-suite exception, evidence-gated deletion, consolidate-not-copy
  re-tiering, rejected alternatives, and CI-safe fake-executable observation.
- Replaced stale `codebase/guide.md` guidance that assigned signal behavior to
  unit tests with the current platform-tier boundary.
- Linked the decision from the decision overview, root catalog, chronological
  decision index, design principles, and the earlier source-simplification
  lesson.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-22 — Bugfix sprint knowledge capture (PRs #459/#460)

### Trigger

Bugfix sprint settled: PR #459 (bugfix-ci-unblock) and PR #460
(spawn-lifecycle-fixes) address #442 (deny-headless after background
submission), #448 (Codex failed turn normalized as success), #450 (stale
continue task_dir), #456 (inherited env vars in tests), #457 (--from UUID
resolution), plus test pruning. PRs open; merge gate awaiting human approval.

### What changed

**Stale content fixed:**
- `concepts/harness-abstraction.md`: Codex terminal status table rewritten
  from unconditional `turn/completed` → succeeded to the payload-resolver
  turn-status ladder (completed/failed/interrupted/unknown). "Leanest terminal
  outcome signal" claim replaced with the actual ladder. Codex doctrine
  paragraph updated to reflect richer classification while preserving the
  "succeeded = turn-completion, not work-correctness" principle.

**New content:**
- `concepts/harness-abstraction.md`: new "TerminalEventOutcome succeeded
  invariant" section documenting the construction-time check that makes
  succeeded-with-error unrepresentable.
- former invariants page: new "Terminal Event Outcome Invariant" section.
- `decisions/launch.md`: new D-headless-deny-at-prepare decision recording
  the policy placement principle (policy decisions in preparation, bind purely
  materializes) and the structural rationale.
- `decisions.md`: new foundational entry and updated Launch domain row.
- `lessons/harness-integration.md`: new "Test Env Isolation" lesson
  documenting the `_MERIDIAN_*` private env var inheritance bug and fix.
- `lessons/overview.md`: lesson index updated.
- `architecture/launch-system.md`: prepare_launch_surface description updated
  to mention policy gates.
- `open-questions/process-scope.md`: new PROC-009 for root_pid identity
  conflation (#449).
- `open-questions/future-work.md`: new deferred entries for #458 (wait
  fail-fast) and #449 medium-effort diagnostics piece.

### Already current (verified, no change needed)

- `architecture/launch-system.md`: prepare/bind split description already
  covers the two-phase architecture; only the prepare description needed a
  one-line policy-gate mention.
- `codebase/harness-adapters.md`: no Codex-specific terminal detail that
  needed updating (terminal semantics live in concepts/harness-abstraction.md).
- `concepts/spawn-lifecycle.md`: no stale content from this sprint.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-17 — Harness-hardening audit knowledge reconciliation (PRs #446/#447, v0.3.36/v0.3.37)

### Trigger

Harness-hardening audit completed: PR #446 (harden Claude/Codex/OpenCode death
and inject contracts) and PR #447 (retire dead Pi notification-lifecycle machinery,
closes #440), both merged to main.

### What changed

**Stale content fixed:**
- `architecture/pi-lifecycle.md`: child cleanup rewritten — `PiCompletionCleanup`
  now uses injected descendant cancellation via persisted spawn rows only, not
  lifecycle PID/PGID telemetry. Spawn phases: notification wait phase removed.
  Drain correctness constraints: "rowless subspawns" removed from blocker
  categories; `notification timeout` removed from precedence examples.
  "Notification Timeout" section replaced with "Follow-Up Marker" describing the
  retained `last-notification.json` disk-state contract.
- `architecture/completion-drain-coordination.md`: `PiPrivateWorkLedger` inventory
  updated — removed rowless subspawns, PID/PGID cleanup handles; now tracks only
  managed bash, follow-up marker, and read failures.
- `decisions.md`: completion-drain-convergence foundational entry updated (ledger
  scope narrowed). New foundational entry for the harness-hardening audit decision.
- `concepts/harness-abstraction.md`: removed stale "redesign successor" framing
  from `meridian-spawn-watch` description.
- `open-questions/future-work.md`: deleted resolved #440 entry (Pi notification
  telemetry — retired, not fixed).

**New content:**
- `concepts/harness-abstraction.md`: new "Cross-Harness Death and Inject Contracts"
  section documenting the uniform death normalization invariant and pointing to the
  colocated per-adapter contracts.
- `decisions.md`: new foundational entry capturing the four-defect-class audit
  results, per-harness verdicts, and retirement decisions.

### Already current (verified, no change needed)

- `codebase/harness-adapters.md`: Pi-specific notes still accurate (disk-backed
  quiescence description references notification markers, which are retained).
- `lessons/harness-integration.md`: extension injection lesson references
  `PiDiskWatcher` and `PiQuiescenceTracker` (both retained); module paths
  accurate.
- `architecture/pi-runtime/vocab.md`: vocabulary entries remain accurate (bash
  records, spawn-origins, last-notification.json, env-var correlation all
  retained).
- `codebase/test-determinism.md`: Pi fidelity helpers section references
  `pi_process_exit_event` and `write_pi_bash_record` (both retained).

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-17 — cbc-followup knowledge reconciliation (PR #444, closes #427)

### Trigger

PR #444 (cbc-followup, 21 commits over main): lock GC via validated-exclusive
sweep (#427), torn-tail-proof JSONL appends, session cleanup ordering invariant,
bounded first-writer wait for projection reads, spawn_aggregate extraction.
Follow-up to PR #422 (concurrency by construction, v0.3.34).

### What changed

**Stale content fixed:**
- `architecture/state-system.md`: "never unlinked" claims amended with GC-seam
  exception; `launch-boundary` and `gc.lock` added to lock layout; JSONL append
  section rewritten for torn-tail repair; `delete_published_spawn` attributed to
  `spawn_aggregate.py`; `mutate_scope_projection` marked private.
- `concepts/state-model.md`: three "never unlinked" claims updated with GC-seam
  exception.
- former invariants page: "Stable Lock Inodes" invariant rewritten for GC
  seam and forbidden pattern.
- `decisions/state.md`: "Never-unlink" decision amended as "Stable lock inodes
  with validated-exclusive GC seam" covering PR #422 + #444; striping rejection
  and forbidden pattern documented.
- `decisions.md` (index): "never-unlinked" phrasing updated; #444 decisions
  added to the State domain row.
- `open-questions/future-work.md`: "Lock-Inode Accumulation (#427)" entry
  deleted (resolved).
- `architecture/spawn-finalization.md`: `delete_published_spawn` moved from
  `repository.py` row to new `spawn_aggregate.py` row in the module map.
- `lessons/thermo-nuclear-audit.md`: "never-unlinked" annotation updated with
  GC seam provenance.

**New content:**
- `decisions/state.md`: three new decision entries — torn-tail-proof JSONL
  appends, spawn_aggregate extraction, generic mutation framework rejected.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-17 — Probe-fix cycle knowledge reconciliation (post-PR #375/v0.3.34)

### Trigger

Follow-up probe-fix cycle on branch `task/pi-probe-fixes`: full real-Pi
re-run exposed issues #432-#435 (fixed), filed #438/#439/#440 (deferred).
Terminal publication architecture changed (publication barrier), Pi exit
classification became category-complete, Pi RPC inject semantics tightened,
fakes gained process-exit fidelity helpers.

### What changed

**Stale content fixed:**
- `architecture/completion-drain-coordination.md`: diagram and "Publication
  precedes cleanup" section replaced with "Publication is a manager-owned
  barrier" describing `_publish_terminal`, `resolve_terminal_outcome`, and
  per-spawn cleanup keying. Invariant 7 updated. Provenance updated.
- `architecture/drain-plans.md`: terminal publication paragraph rewritten
  for the manager-owned barrier (removed reference to drain loop owning
  publication).
- `open-questions/future-work.md`: "Publish-at-Deadline Redesign (#431)"
  rewritten as "Store-Level Publication Ordering (#431)" with resolved items
  marked. Three new deferred entries: typed terminal provenance (#438),
  cleanup telemetry categories (#439), Pi notification telemetry (#440).

**New content:**
- `decisions.md`: new foundational entry "Terminal publication barrier" for
  the `_publish_terminal` / `resolve_terminal_outcome` / per-spawn cleanup
  architecture.
- `architecture/pi-lifecycle.md`: two new drain-correctness safeguards:
  category-complete exit classification via `classify_outstanding_work()`
  and canonical subprocess-exit precedence via `PI_SUBPROCESS_EXIT_ERROR_PREFIX`.
- `codebase/test-determinism.md`: new "Pi Process-Exit Fidelity Helpers"
  section documenting `pi_process_exit_event()` and `write_pi_bash_record()`.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-17 — Post-drain-streaming-cleanup knowledge reconciliation (PR #375)

### Trigger

PR #375 (drain-streaming-cleanup, 21 commits) completed the post-drain-convergence
streaming cleanup: migration scaffolding deletion, drain composition-root
inversion, Pi drain test consolidation, resident rearm budget, Pi timeout
policy documentation, timeout carrier unification, shared reconciliation
decisions extraction, and windows-gate CI demotion.

### What changed

**Stale content fixed:**
- `decisions/model-resolution.md` D72: updated `ExecutionBudget` reference to
  `execution_policy.timeout` (ExecutionBudget was deleted in PR #375; timeout
  is now a single carrier in minutes).
- `concepts/model-resolution/model-policies.md`: same ExecutionBudget reference
  corrected.
- `open-questions/future-work.md`: agent overlay timeout item updated for
  new carrier name.
- `decisions/launch.md` D-win-ci: marked as superseded by POSIX-first; windows-gate
  CI demoted to non-blocking.
- `decisions.md` domain index Launch row: noted windows-gate as superseded.

**New content:**
- `decisions.md`: new foundational entry "Timeout and completion policy" covering
  single timeout carrier, opt-in ceilings, spawn-scoped rearm budget, and
  rejected alternatives.
- `architecture/drain-plans.md`: new sections "Rearm budget" (signal-gated,
  unlimited default, spawn-scoped, exhaustion behavior, Mars schema limitation)
  and "Timeout carrier" (single `execution_policy.timeout`, dual-carrier deletion
  rationale).
- `architecture/completion-drain-coordination.md`: provenance updated.
- `open-questions/future-work.md`: new "Streaming & Completion" section with
  two deferred items: #430 (resident nudge serialization + test seeding) and
  #431 (publish-at-deadline redesign, prefer_drain_outcome conflation,
  streaming_runner extraction).
- `codebase/test-determinism.md`: new section "PiDrainScenario Builder" and
  "Bounded History-Phase Polling" pattern.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-17 — POSIX-first supersedes Windows-first principle

The 2026-07-17 platform decision (POSIX-first: Linux/macOS supported; native
Windows never worked and is not planned) superseded the prior "Windows Is
First-Class" design principle. Doc-only repo sweep landed on branch
`task/docs-posix-first`; this entry reconciles the durable KB so a cold agent
no longer reads Windows as a supported product target.

- former design-principles page §12 rewritten from "Windows Is
  First-Class" to "POSIX-First Platform Stance" (current truth; history now
  lives in the decision record, not the principle list).
- `principles/overview.md`: principle descriptor updated (Windows-first →
  POSIX-first platform stance).
- `overview.md`: "Windows is first-class" key property replaced with the
  POSIX-first stance.
- `codebase/guide.md`: platform package descriptor and the "Platform
  coverage required" guidance reframed from "consider Windows semantics" to
  POSIX-first (branches legacy/untested, not to be expanded).
- `decisions.md`: new foundational entry "POSIX-first supersedes
  Windows-first" recording rationale and the supersession.

Architecture/concept/lesson pages describing existing Windows code branches
(locking, process-scope, signals, path resolution) were left intact — they
document concrete reality the POSIX-first decision explicitly permits to
remain (legacy, untested, best-effort), not forward-looking support claims.

---

## 2026-07-16 — Thermo-nuclear audit knowledge capture

### Trigger

The thermo-nuclear audit (issue #389) completed. PR #390 (qi-docs-hygiene)
flagged three KB pages as stale against the current three-adapter,
`__status.json`-era architecture: `architecture/launch-system.md`,
`architecture/state-system.md`, `concepts/extension-system.md`. The audit
also produced durable decisions (rejected alternatives, peer-benchmark
verdicts) worth preserving.

### What changed

**Stale page fixes:**
- `architecture/launch-system.md`: "Four Driving Adapters" replaced with
  "Three Driving Adapters" (REST app path removed — `lib/app/` was
  archived). Mermaid diagram updated. Section 3 (REST App Path) removed;
  streaming-serve renumbered to section 3. Harness list corrected to
  include Cursor in the CWD policy table.
- `architecture/state-system.md`: split state layout corrected. Work items
  now live under the context work root (`<context.work>/<slug>/__status.json`),
  not `.meridian/work-items/`. Added staging directory to spawn layout.
  Work Item Store section updated for directory-based `__status.json`
  pattern.
- `concepts/extension-system.md`: HTTP app server marked as archived.
  `requires_app_server` commands return `app_server_archived`. Surface
  allocation table updated to show only active CLI/MCP surfaces. Core model
  diagram updated. RemoteExtensionInvoker section removed.
- `architecture/system-overview.md`: reduced from "three surfaces + one
  in-process adapter" to "two external surfaces". Mermaid diagrams updated
  to remove `lib/app/`, add Cursor and Pi to harness list, change "four"
  to "three" driving adapters.
- `architecture/overview.md`: app-server.md entry updated to note archived
  status.
- `index.md`, `decisions.md`: stale "four driving adapters" and "JSONL
  event stores" references updated.

**New lesson page:**
- `lessons/thermo-nuclear-audit.md`: two-source adversarial verification
  method, six rejected alternatives with specific evidence (SpawnOwnership,
  exactly-once reaper gating, wholesale TypeAdapter, shared read_json_object,
  ruff TID251, strict raw-event rejection), three defended peer-benchmark
  strengths, one dropped steal.

### Validation

`meridian kg check .` and `meridian mermaid check .` run before committing.

---

## 2026-07-16 — Drain-plan boundary extraction

### Trigger

The completion-drain convergence capture expanded the drain-plan and resident
completion material within the terminal-finalization page.

### What changed

- `architecture/drain-plans.md`: new owner for streaming `DrainPlan`
  composition and resident completion behavior.
- `architecture/spawn-finalization.md`: retained terminal-write authority and
  store-level finalization; links to the extracted drain-plan page.
- `architecture/overview.md` and `index.md`: indexed the new architecture page.

### Validation

`meridian kg check .` and `meridian mermaid check .` pass.

## 2026-07-15 — Completion drain convergence design capture

### Trigger

The `work:drain-convergence` design phase settled the shared Pi/resident
completion architecture and completed a Linux/POSIX crash/concurrency probe for
child-row publication.

### What changed

- `architecture/completion-drain-coordination.md`: new cross-cutting target
  architecture covering composition, evidence authority, Pi-private work,
  cleanup and persistence invariants, current divergences, and reversible phases.
- `architecture/atomic-child-row-publication.md`: new publication protocol and
  bounded evidence, including the unsafe sibling-stage layout and untested gates.
- `architecture/pi-lifecycle.md`: explicitly scoped as current checkout behavior,
  linked to the target, and updated for shipped unresolved-candidate tombstones.
- `decisions.md`, `architecture/overview.md`, and `index.md`: indexed the settled
  decision and new pages.

### Validation

Run `meridian kg check .` and `meridian mermaid check .` before committing.

## 2026-06-21 — Rootless commands, established project, test determinism capture

### Trigger

meridian-cli PRs #338 (rootless commands + startup model refactor) and #339 (test de-flake infrastructure). Cross-cutting capture from the session that designed and shipped these changes.

### What changed

- `architecture/startup-pipeline.md`: added `READ_ROOTLESS` to invocation class table; new section "Rootless Commands and Established Project" documenting `resolve_cli_project_root()`, `cwd_has_project_id()`, `exit_no_established_project()`, and the SystemExit-through-except-Exception footgun fix.
- `concepts/state-model.md`: new "Established Project" section with the established-project predicate, `cwd_has_project_id()`, and link to #341 migration direction (`.meridian/id` → `meridian.toml`).
- `concepts/config-precedence.md`: fixed stale project-root discovery description (removed ancestor walk-up steps 3-5 from #335; now correctly describes explicit / env / literal-CWD).
- `decisions/startup-health-sandbox.md`: new sections "Rootless commands" and "Established Project Detection" with decision rationale for #338.
- `codebase/test-determinism.md`: **new page** capturing the test-determinism infrastructure (`AsyncDeterminism` + `FakeClock`, `process_race.py`, advance-past-boundary principle, behavioral vs safety-bound timeout distinction, windows-gate flake pattern).

### Validation

`meridian kg check kb/` → pending (need kg in a project context).

---

## 2026-06-20 — Knowledge-layers KB guidance refresh

### Trigger

meridian-base commit `6fa9cc6` updated the `knowledge-layers` skill with four corrected conventions:
KB structure guidance now says read the local KB `AGENTS.md`; current-truth guidance
now distinguishes concept/architecture/operations pages from decision records;
vocabulary guidance no longer hardcodes a filename; ingest guidance now says update
log only for structural/major changes when the KB guide asks for it.

### What changed

- `research/llm-wiki-pattern.md`: replaced stale `kb-writer` agent references with
  `kb-lead`; reworked "When to Archive" → "When to Remove" section with the
  current-truth/page-type distinction; updated "When to Revise" to explain the
  decision-record exception; cross-referenced `AGENTS.md` for the full KB
  maintenance guide.

### Validation

`meridian kg check .` → 0 errors. `meridian mermaid check research/llm-wiki-pattern.md` → 1 block valid.

---

## 2026-06-20 — Mars `models prompting` canonical policy resolution (commit `1859a16`)

### Trigger
Post-ship KB capture for mars-agents commit `1859a16`: `mars models prompting` agent resolution now uses canonical launch policy path (`resolve_policy` with `PolicyInput`) instead of shallow overlay/profile/default token triage. Output reflects final `policy.routing` after model-policies, harness routing, and model clearing. Adds `--refresh-models` / `--no-refresh-models`; direct model-alias refs keep catalog-backed name resolution.

### What changed
- `decisions/model-resolution.md` — D86 expanded with canonical-policy rationale, shallow-resolver rejection, and `1859a16` correction note
- `concepts/model-resolution/aliases-and-routing.md` — retrieval section updated for launch-policy path, refresh flags, and `policy.routing` output semantics
- `concepts/model-resolution/vocabulary.md` — agent-first resolution term updated
- `architecture/mars-model-refresh.md` — `models prompting` listed as refresh consumer

### Validation
Run `meridian kg check kb` and `meridian mermaid check kb/concepts/model-resolution/` before committing.

---

## 2026-06-13 — Spawn lifecycle / reliability KB capture (PR #328)

### Trigger

PR #328 (`feature/spawn-capability-injection`) shipped six concurrent spawn-lifecycle
improvements: capability-gated spawn injection, reserve-before-prep crash recovery,
codex terminal semantics clarification, resident-done contract, sandbox bind-mount
context-root fix, and meridian-base v0.7.20 profile removal.

### What changed

- `decisions/launch.md`: added D-spawn-capability-gate and D-reserve-before-prep decisions.
- `concepts/spawn-lifecycle.md`: added Spawn Capability Gating section, Reserve-Before-Prep
  Crash Window section, clarified Resident Turn Boundaries (done = residency control, not
  status source), updated invariants.
- `concepts/harness-abstraction.md`: added Terminal Status Semantics section — per-harness
  terminal_outcome table, Codex `succeeded` = turn-completion doctrine.
- `architecture/sandbox-projection.md`: added Context-Root Existence Filtering section —
  codex bwrap missing-path abort, debug-only logging, git clone root exemption.
- `ecosystem/prompt-packages/meridian-base.md`: noted v0.7.20 removal of default
  orchestrator and subagent profiles.
- `decisions.md`, `index.md`: updated cross-references.

### Related inline docs

- `src/meridian/lib/launch/spawn_guidance.py` — capability gate + harness-templated contracts
- `src/meridian/lib/launch/context.py` — `_collect_context_projection_roots` existence filter
- `src/meridian/lib/harness/semantics.py` — `terminal_outcome()` per-harness classification
- `src/meridian/lib/streaming/resident_drain.py` — ResidentDrainCoordinator
- `src/meridian/lib/streaming/completion_nudge.py` — 270s completion nudge
- `src/meridian/lib/state/spawn_signals.py` — crash-only done/rearm signal files

---

## 2026-06-12 — OpenCode parent-session scope capture

### Trigger

Commit `2621f68c` generalized Codex main-thread protection into a shared primary event
scope and applied it to OpenCode's global event stream. Child OpenCode task sessions
remain visible in session logs but no longer complete, fail, clear signals for, or
supply reports for the parent spawn.

### What changed

- `concepts/harness-abstraction.md`: added the Primary Event Scope concept under the
  connection model.
- `codebase/vocabulary.md`: added the Primary event scope term.
- `index.md`, `codebase/overview.md`: refreshed the harness-adapters summary.

### Already-current items verified

- `codebase/harness-adapters.md` already documents `HarnessConnection.primary_event_scope`,
  Codex `threadId` vs OpenCode parent `sessionID`, the child-event persistence rule,
  and the Codex/OpenCode unscoped fallback difference.

### Related inline docs

The meridian-cli worktree capture updated `src/meridian/lib/harness/`,
`src/meridian/lib/streaming/`, and `docs/harness-integration.md` to replace the old
Codex-only wording with the shared parent-scope contract.

---

## 2026-06-06 — Post-#314/#317/#318 knowledge currency refresh

### Trigger

Batch of merged PRs (#314 prompt.py decomposition, #316 mars-agents 0.8.1/v3-only
bundle schema, #317 Codex thread-aware drain, #318 ambient-cwd task_cwd resolution)
left inline docs and KB pages stale. kb-lead reconciliation pass.

### What changed

**Inline docs (meridian worktree, `refactor/promptpy-decomposition`):**
- `launch/.context/CONTEXT.md`: resolution priority list rewritten to match
  `cwd.py` docstring exactly (steps 1, 2, 3, 3.5, 4); removed "or authority root
  fallback" per-step qualifiers that confused with-priority fallback vs
  separate priority level.

**KB — decisions:**
- `decisions/launch.md`: D-model-invocable + D-model-invocable-vs-user-invocable
  rewritten for current truth — Mars owns inventory rendering (bundle
  `prompt_surface.inventory_prompt`), Meridian embeds verbatim, Python-side
  `build_agent_inventory_prompt()` was deleted in #314.
- `decisions/launch.md`: **new** D-mars-owns-inventory — bundle-only inventory
  contract, no Python fallback.
- `decisions/launch.md`: **new** D-headless-claude-deny — headless Claude denied
  by default with 2026-06-15 driver, startup warning on override.
- `decisions/launch.md`: **new** D-agent-copy-key — `settings.meridian.agent_copy`
  key rename, auto-scaffold on init.
- `decisions/launch.md`: D-control-root-task-cwd-split env var updated
  (`MERIDIAN_TASK_CWD` → `MERIDIAN_TASK_DIR`) and system prompt block heading
  (`# Task Working Directory` → `# Source-edit directory`).
- `decisions.md`: summary entries updated for deleted `build_agent_inventory_prompt()`
  and new PR #314 decisions.

**KB — concepts/codebase/architecture:**
- `codebase/vocabulary.md`: "Delegation preference guidance" entry — removed
  reference to deleted `with_agent_inventory_guidance()`; now describes Mars-rendered
  bundle inventory.
- `codebase/harness-adapters.md`: removed `with_agent_inventory_guidance()` reference;
  replaced with Mars-rendered bundle paragraph.
- `concepts/harness-abstraction.md`: removed `with_agent_inventory_guidance()` reference;
  replaced with Mars-rendered bundle paragraph.
- `architecture/launch-system.md`: task_cwd resolution table rewritten — removed
  `--no-worktree`/`--worktree` flag rows, added ambient-cwd priority (step 3.5);
  updated `MERIDIAN_TASK_CWD` → `MERIDIAN_TASK_DIR`; updated `# Task Working Directory`
  → `# Source-edit directory`.
- `architecture/spawn-finalization.md`: `MERIDIAN_TASK_CWD` → `MERIDIAN_TASK_DIR`;
  system prompt heading updated.

### Already-current items verified
- Task CWD instruction injection paragraph in `launch/.context/CONTEXT.md` already
  described #318 contract (MERIDIAN_TASK_DIR, shell cwd is project root, cd or
  absolute paths).
- Codex thread-aware drain already documented in both `harness/AGENTS.md` and
  `harness/.context/CONTEXT.md`.
- v4 `launch_actions` removal already described in `architecture/mars-launch-bundle.md`.

---

## 2026-05-30 — Pi quiescence architecture refresh (PR #297)

### Trigger

PR #297 tightened Pi disk-backed quiescence and split the streaming implementation.
Session review found KB pages still naming retired lifecycle event-file surfaces and stale
Pi launch examples.

### What changed

- `architecture/pi-lifecycle.md`: current-only Pi extension split, disk-backed
  quiescence rule, PR #297 drain correctness constraints, removed historical event-file
  appendix from the live page
- `codebase/harness-adapters.md`, `architecture/launch-system.md`: Pi launch examples
  updated to `managed-bash` + `meridian-spawn-watch`
- `lessons/harness-integration.md`: Pi extension lesson now names disk-state authority,
  `PiDiskWatcher`, and `PiQuiescenceTracker` correctly
- `lessons/pi-rpc-quiescence-impl.md`: narrowed to still-current Windows/test-design
  lessons
- `architecture/pi-runtime/vocab.md`: current env vars and spawn-watch ownership

### Validation

Run `meridian kg check kb` and `meridian mermaid check kb/architecture/` before committing.

---

## 2026-05-23 — Mars model refresh and routing parity (PR #72)

### Trigger

Post-ship KB capture for mars-agents PR #72: `ModelsRefreshControl`,
`--refresh-models` / `--no-refresh-models`, `ensure_fresh` on launch-bundle,
probe refresh matrix, default `harness_order`, routing parity across models CLI
and launch-bundle, Cursor build-time `harness_model` effort resolution.

### What changed

- **New:** `architecture/mars-model-refresh.md` — focused catalog + probe refresh page
- `architecture/mars-routing.md` — Routing Parity (PR #72) section; cross-link refresh page
- `architecture/mars-launch-bundle.md` — `candidate_slugs` diagnostic; catalog refresh before build
- `architecture/cursor-harness.md` — build-time effort resolution; legacy Meridian projector labeled
- `concepts/package-management/sync-model.md` — symmetric `--refresh-models` / `--no-refresh-models` table
- `concepts/package-management/vocabulary.md` — `ModelsRefreshControl` term
- `index.md`, `architecture/overview.md` — navigation for launch-bundle, model-refresh pages
- `open-questions/mars-feature-gaps.md` — gap #7 Meridian refresh-models passthrough stub

### Validation

Run `meridian kg check kb` and `meridian mermaid check kb/architecture/` before committing.

---

## 2026-05-22 — Launch-bundle schema v2 follow-up (routing/scaffold detail pass)

### Trigger

Targeted follow-up after the first schema-v2 update: source audit showed a few
stale or ambiguous claims remained in launch-bundle/routing docs (old scaffold
placeholder names, `model_token` wording at harness boundary, and over-specific
enum-label examples in Mars routing diagnostics).

### What changed

- `architecture/mars-launch-bundle.md`:
  - Split "full Mars v2 schema" vs "Meridian-consumed subset" explicitly
  - Added top-level `agent_body` row and clarified `agent`/`agent_body` optionality
  - Corrected routing semantics:
    - `model_token` documented as selected routing/policy token
    - `harness_model` documented as the harness-boundary model ID when present
  - Listed current Mars routing diagnostic fields as Mars-owned and ignored by Meridian
  - Replaced stale scaffold placeholder table with v2 `scaffold_slots` keys:
    `completion_contract`, `context_prompt`, `user_prompt_file`, `context_files`,
    `prior_session_context`, `spawn_metadata` (all `###SLOT###`)
  - Tightened compatibility text to exact schema v2 + mars >= 0.5.0
- `architecture/mars-routing.md`:
  - Kept SelectionKind/MatchEvidence and RouteDecisionReport conceptual, but removed
    stale hard-coded enum-label examples
  - Clarified diagnostic labels are Mars-internal serialized strings, not Meridian contract

### Validation

Run `meridian kg check kb` and `meridian mermaid check kb/architecture/` before committing.

## 2026-05-22 — Bundle schema v2 / mars 0.5.0 KB update

### Trigger

`bundle_adapter.py` confirmed `_SUPPORTED_BUNDLE_SCHEMA_VERSION = 2` and `_MARS_BUNDLE_MIN_VERSION = "0.5.0"`, resolving the human-review flag left by the 2026-05-22 model-resolution KB pass. The KB still reflected schema v1 and mars 0.4.8rc3.

### What changed

- `architecture/mars-launch-bundle.md`:
  - Updated Bundle Structure header from "version 1" to "version 2, mars >= 0.5.0"
  - Updated `version` row to say currently `2`
  - Updated `routing` row to include `harness_model`
  - Added `routing` object subsection documenting the four Meridian-consumed fields (`model`, `model_token`, `harness`, `harness_model`) and noting that Mars-internal diagnostic fields (`selection_kind`, `match_evidence`, `route_trace`) are ignored by Meridian
  - Updated Compatibility section to reflect v2 as the required schema with exact-match validation
- `architecture/mars-routing.md`:
  - Added "Mars-internal DTO" clarification to the `RouteDecisionReport` section
  - Made explicit that Mars-internal consumers are `build::policy::*` (all Mars modules), not Meridian
  - Added note that Meridian reads only `model`, `model_token`, `harness`, `harness_model` from the bundle routing object
- `log.md` (this file):
  - Removed the stale human-review FLAG from the 2026-05-22 model-resolution pass
  - Removed the stale "Human-review note" from the structural follow-up entry above it

### Validation

Run `meridian kg check kb` and `meridian mermaid check kb/architecture/` before committing.

---

## 2026-05-22 — Model-resolution KB structural follow-up

### Trigger

Post-cleanup review found a few live stale references still lingering in the
decision and related pages:
`resolve_harness_routing()`, `resolve_policies()` as the routing-thread anchor,
and the old candidate-chain helper names in D75. A related link label in
`config-precedence.md` still read like the compiler was current authority.

### What changed

- `decisions/model-resolution.md`: rewrote D52/D53/D55 to point at the Mars
  bundle path / `resolve_launch_policy()` instead of `resolve_harness_routing()`
  or `resolve_policies()` as the routing anchor; replaced D75 helper-list
  references with current module-level references
- `concepts/model-resolution/model-policies.md`: changed "primary compilation"
  wording to "primary launch resolution" and retargeted the related-page link to
  the Mars bundle routing path
- `concepts/model-resolution/agent-profiles.md`: tightened the related-page label
  to "full bundle-based resolution pipeline"
- `concepts/config-precedence.md`: retitled the D73 related link text so it no
  longer reads as current compiler authority

### Validation

- `meridian kg check kb` → 0 errors, 6 warnings (all existing flag blocks)
- `meridian mermaid check /home/jimyao/.meridian/git/haowjy-meridian-cli-kb/kb/concepts/model-resolution` → 1 Mermaid block valid

---

## 2026-05-22 — Model-resolution KB: remove deprecated Meridian compiler as current runtime authority

### Trigger

Code audit (p2178/p2179) found that several model-resolution KB pages still presented
`compile_launch_params()` / `compiler.py` as the canonical runtime authority, and
referenced a non-existent `resolve_harness_routing()` function. Current code routes
PRIMARY/SPAWN_PREPARE through `_resolve_policy_from_bundle()` → `bundle_adapter.request_and_resolve()`.
The compiler module is marked deprecated in its own docstring.

Also flagged: D73/D74 decisions described the Meridian-owned compiler as authoritative;
`resolve_harness_routing()` was cited as a live function; the 7-source harness cascade
was presented as the current runtime path. None of these match current code.

Human-review discrepancy (at this audit point on 2026-05-22): task prompt mentioned
bundle schema v2 / mars 0.5.0, but inspected code then had
`_SUPPORTED_BUNDLE_SCHEMA_VERSION = 1` and `_MARS_BUNDLE_MIN_VERSION = "0.4.8rc3"`.
No KB changes for those values in this pass — `architecture/mars-launch-bundle.md`
(which then reflected v1) was left intact pending follow-up confirmation.

### What changed

- `concepts/model-resolution/overview.md`: replaced `compile_launch_params()` pipeline
  diagram and two-pass compiler narrative with bundle-based pipeline diagram and
  current entry points; added legacy note for deprecated `compiler.py`
- `concepts/model-resolution/aliases-and-routing.md`: removed 7-source
  `resolve_harness_routing()` cascade (function doesn't exist in current code) and
  compiler-era harness-availability fallback section; added current Mars bundle
  routing section; kept AliasEntry/RunnablePath/ModelSelectionContext material
- `concepts/model-resolution/model-policies.md`: replaced compiler-centric
  implementation paragraph (removed `resolve_harness_routing()` cascade reference
  and `compiler.py` canonical claim); updated to bundle-based policy evaluation
- `concepts/model-resolution/vocabulary.md`: updated `Resolved policies` definition
  to reference `resolve_launch_policy()` / Mars bundle path
- `concepts/model-resolution/agent-profiles.md`: narrowed `resolve_model()` sentence
  to catalog-lookup context; updated Harness Availability Fallback section to remove
  non-existent function references (`_try_harness_availability_fallback()` etc.) and
  fixed broken `#harness-availability-fallback` anchor
- `concepts/config-precedence.md`: removed `_resolve_model_policy_overrides()` reference
  (function doesn't exist); replaced with bundle execution policy resolver description
- `decisions/model-resolution.md`: marked D73 and D74 as superseded by Mars
  launch-bundle routing, preserved text for historical context

### Validation

- `meridian kg check kb` → 0 errors, 5 warnings (all pre-existing flag blocks)
- `meridian mermaid check kb/concepts/model-resolution/` → 1 block valid

---

## 2026-05-19 — Spawn finalization cross-reference health pass

### Trigger

Structural health check after the spawn-exit-schema-split capture pass exposed one broken KB reference in the new spawn-finalization material.

### What changed

- `architecture/spawn-finalization.md`: removed a broken work-item link target and left the work-item reference as plain text so the page no longer points at a non-existent KB path.

### Validation

- `meridian kg check kb` after the edit still reports `0 errors`.

---

## 2026-05-16 — Session initiation semantics and PR #216 knowledge capture

### Trigger

Work item audit-dev-workflow-scope shipped unified session-initiation semantics (--fork/--fork-fresh split, bare flag inference, primary --from, argv normalization sentinel, four-layer content composition model). No KB pages existed for the four-mode initiation model or the identity-lock and argv-normalization design decisions.

### What changed

**New page:**
- `concepts/session-initiation.md` — Authoritative page for the four-mode session initiation model (--continue, --fork, --fork-fresh, --from). Covers: mode semantics and use cases, decision tree, four-layer content composition model (why Layer 3 is user-turn not system prompt), identity lock (fork vs fork-fresh), bare flag inference, argv normalization sentinel pattern, mutual exclusion matrix, surface coverage table, and implementation seam (`resolve_task_context_inputs`).

**Updated pages:**
- `decisions/launch.md` — Added Session Initiation section (2026-05 PR #216) with four new decisions: D-fork-identity-lock, D-prior-context-user-turn, D-argv-normalization-sentinel, D-from-fork-mutual-exclusion.
- `concepts/composition-pipeline.md` — Added session-initiation.md to Related Pages.
- `concepts/overview.md` — Added session-initiation entry.
- `index.md` — Added session-initiation.md to concepts section.
- `decisions.md` — Added chronological entries and updated Launch row in domain index.

### Validation

Ran `meridian kg check kb` and `meridian mermaid check kb` — see below.


---

## 2026-05-16 — Candidate-aware harness routing (PR #219)

### Trigger

Post-ship KB capture for PR #219 "Candidate-aware model harness routing" (merged 2026-05-16).

### What changed

**New decision records in `decisions/model-resolution.md`:**
- D76: Harness-specific model IDs via `RunnablePath` — `AliasEntry.harness_candidates` and `runnable_paths`; `harness_model_id` threading to bind time.
- D77: Explicit harness = force — `validate_harness_compatibility()` now checks only adapter availability, not model compatibility, for explicit harness selections.
- D78: Invalid override key skip — model-policy rules with all-invalid override keys silently skipped at profile load time instead of kept as phantom no-ops.

**Concept updates:**
- `concepts/model-resolution/aliases-and-routing.md`: added "Harness-Specific Model IDs" section covering `RunnablePath`, `AliasEntry.runnable_paths`, and `harness_model_id_for()`; updated `AliasEntry` and `ModelSelectionContext` code blocks with new fields.
- `concepts/model-resolution/overview.md`: added note in "What Gets Resolved" that `harness_model_id` may differ from `canonical_model_id` at the harness command boundary, with link to aliases-and-routing.

---

## 2026-05-16 — Bootstrap command, release CI fix, PR #184 carrier consolidation

### Trigger

Post-ship KB capture for PR #215 (bootstrap command), PR #213 (release CI semver fix), and PR #184 structural simplification waves.

### What changed

**New page:**
- `operations/bootstrap-command.md` — `meridian bootstrap --add` contract: init steps + guided session launch, difference from `meridian init --add`, rootless startup class rationale, idempotency.

**Navigation updates:**
- `operations/overview.md`: added bootstrap-command.md entry.
- `index.md`: added bootstrap-command.md to operations section.

**Decision update — `decisions/launch.md`:**
- Added Structural Simplification (2026-05, PR #184) section covering Wave 1 (four indirections removed), Wave 2 (session carrier consolidation: `SessionRequest` + `PrimarySessionMetadata` typed carriers replace `**kwargs`), and Wave 3 (LaunchSpec flattened to `ResolvedLaunchSpec`, compat bridges removed).

**Release CI note:**
- Release CI (`release-on-merge.yml`) now computes next version from highest stable `v*` semver tag, not just latest commit tag. Prevents stale lower-version tags from shadowing higher versions. (PR #213, 2026-05-14)

---

## 2026-05-16 — control_root/task_cwd split (PR #210)

### Trigger

Post-ship KB capture for PR #210 "Preserve control root and task cwd across launch" (merged 2026-05-13). The split of `execution_cwd` into `control_root` + `task_cwd` touched spawn/session record schemas and continue/fork authority logic.

### What changed

- `architecture/spawn-finalization.md`: updated `PreparedExecutionHandoff` schema to show `control_root`, `task_cwd`, and `execution_cwd` (legacy alias) as three separate fields with explanatory note.
- `architecture/claude-session-isolation.md`: fixed stale `source_execution_cwd` reference to `source_control_root` in the `--continue` flow diagram.
- `architecture/launch-system.md`: added `## control_root / task_cwd Split` section documenting the two fields on `LaunchContext`, `bind_launch_context()` behavior, env/prompt injection, and continue/fork authority.
- `decisions/launch.md`: added `D-control-root-task-cwd-split` decision record with rationale, alternatives rejected, and behavior description.

### Validation

- `meridian kg check kb/`: run after edits.
- `meridian mermaid check kb/`: run after edits.

---

## 2026-05-13 — model-invocable feature knowledge capture (PR #208)

### Trigger

Post-ship KB capture for the model-invocable agent visibility + native-skill prompt
suppression feature.

### What changed

**New decision records in `decisions/launch.md`:**
- D-model-invocable: filter seam choice (inventory prompt boundary vs catalog scan)
- D-model-invocable-vs-user-invocable: model-facing vs CLI-user surfaces are separate
- D-skill-doc-suppression: supplemental_documents gate on supports_native_skills

**Concept updates:**
- `concepts/composition-pipeline.md`: added "Skill Delivery: Two Channels, One Correct"
  section documenting the supplemental_documents vs --append-system-prompt split, and
  "Agent Inventory Prompt" section documenting the model-invocable filter.
- `concepts/model-resolution/agent-profiles.md`: added `model-invocable` frontmatter
  field documentation with Mars contract gap note.

**Lessons:**
- `lessons/arch-refactor-pitfalls.md`: added `prepare_prompt_payload()` or-chain
  pitfall — silently drops appended_system_prompt when projected_content has system_prompt.

**Open questions:**
- `open-questions/mars-feature-gaps.md`: added gap #6 — Mars does not formally specify
  model-invocable preservation for compiled agent profiles (haowjy/mars-agents#40).

**Index updates:**
- `decisions.md`: updated Launch domain row and added two foundational summaries.

---

## 2026-05-09 — Process-scope cleanup structural pass

### Trigger

Structural health check after process-scope cleanup KB capture. The burst added
process-scope architecture, launch-decision, and future-work material; the
deferred-work details were duplicated between architecture and the general
future-work page.

### What changed

**Structure:**
- Added `open-questions/process-scope.md` as the focused home for PROC-004 and
  PROC-007 follow-up details.
- Replaced the detailed PROC entries in `open-questions/future-work.md` with a
  short pointer, keeping the general future-work page as an index by domain.
- Kept `architecture/process-scope.md` focused on current mechanism and
  architectural summary, with a link to the open-question details.

**Navigation and cross-references:**
- Added the new process-scope open-questions page to `index.md` and
  `open-questions/overview.md`.
- Updated launch-decision and future-work cross-references to point at the new
  focused page.

### Validation

- Link and Mermaid validation rerun after edits; see the maintainer run report
  for command results.

---

## 2026-05-08 — Architecture-refactor KB structural pass

### Trigger

Post-write structural health check after PR #184 architecture-refactor lessons
were captured in the KB. The new lesson page and managed-primary cleanup
refinement needed navigation, bidirectional links, and stale-neighbor checks.

### What changed

**Navigation and cross-references:**
- Added `lessons/arch-refactor-pitfalls.md` to `lessons/overview.md`.
- Linked the PR #184 lesson page from the managed-primary lifecycle,
  launch-decision, health-check, and observability pages where the pitfalls are
  operationally relevant.

**Content health:**
- Updated stale neighboring summaries in `concepts/spawn-lifecycle.md`,
  `decisions.md`, `operations/troubleshooting.md`, and
  `operations/health-checks.md` so they match the refined invariant: Skip
  decisions send no signals; finalize-as-failed decisions may clean up tracked
  managed-primary runtime children when metadata safely identifies them.

### Validation

- `meridian kg check kb/`: 0 errors, existing FLAG warnings only.
- `meridian mermaid check kb/`: all Mermaid blocks valid.

---

## 2026-05-06 — Claude auth persistence and managed-primary reaper safety

### Trigger

Post-implementation KB capture for `primary-lifecycle-and-claude-auth-fixes`.
The implementation changed Claude overlay cleanup semantics and the managed
primary passive reaper boundary; those invariants needed durable documentation
outside the work directory.

### What changed

**Architecture:**
- Added `architecture/managed-primary-lifecycle.md` covering Codex/OpenCode
  launcher/backend/TUI roles, why launcher death is abnormal evidence, passive
  reconciliation limits, explicit `spawn cancel` cleanup, and missing/corrupt
  `primary_meta.json` candidate handling.
- Updated `architecture/claude-session-isolation.md` with auth/config state
  persistence from Claude overlays, known files (`.claude.json`,
  `.credentials.json`), concurrency policy, and best-effort failure behavior.

**Concepts and operations:**
- Updated `concepts/spawn-lifecycle.md` with `orphan_primary` semantics and the
  managed-primary passive-reaper invariant.
- Updated `operations/troubleshooting.md` with diagnosis and cleanup guidance
  for `orphan_primary`.
- Updated navigation in `index.md`, `architecture/overview.md`, and decisions
  pages.

### Validation

- KB link and Mermaid validation rerun after edits; see the run report for
  command results.

---

## 2026-05-06 — Chat normalization repair capture

### Trigger

Post-implementation KB capture for the `codex-dev-tailscale-no-response` work item. The existing chat normalization docs were stale against the repaired harness mappings and did not preserve the live-smoke operational lessons.

### What changed

**Architecture refresh:**
- Updated `architecture/chat/normalization.md` to reflect the repaired normalized chat contract:
  - assistant and reasoning text stay in `content.delta`
  - tool lifecycle is canonical `item.started` / `item.updated` / `item.completed`
  - exactly one `turn.completed` per actual turn
- Rewrote the per-harness mapping summaries for Claude, Codex, and OpenCode to match the current raw event shapes, including Claude aggregated `tool_use` / `tool_result`, Codex `item/agentMessage/delta` plus terminal `agentMessage` fallback, OpenCode part-role tracking and `message.updated` snapshot repair.
- Added the shared completion-dedupe rule to the normalization page and expanded source/test references.

**Pipeline contract refresh:**
- Updated `architecture/chat/event-pipeline.md` so the `content` family no longer claims command or file-change output stream kinds.
- Clarified that tool output stays on `item.*` payloads to avoid frontend double-rendering.

**Lessons capture:**
- Added `lessons/chat-normalization-repair.md` covering the root failure mode, compatibility-adapter framing, completion-dedupe lessons, replay obligations, and live-smoke cautions.
- Updated `lessons/overview.md` and `index.md` to include the new lessons page.

### Validation

- KB links and structure validated after edits; see the work-session run report for command results.

## 2026-05-06 — Codex approval routing and projected roots maintenance

### Trigger

Post-work KB maintenance after `codex-workspace-approval-routing`. The kb-writer update added current knowledge for managed-primary Codex approval routing, Codex remote TUI root mirroring, and `projected_roots`; the KB needed a structural and staleness pass.

### What changed

**Content health:**
- Verified `codebase/harness-adapters.md` already describes the implemented handler-based managed-primary approval flow and Codex remote TUI root mirroring.
- Updated `concepts/workspace-projection.md` and `architecture/workspace/overview.md` so `projected_roots` is documented as the current launch-spec seam, not a pending refactor.
- Updated `architecture/launch-system.md` to clarify that workspace roots flow through `run_params.projected_roots` / `ResolvedLaunchSpec.projected_roots`; `extra_args` remains user passthrough only.
- Updated OpenCode projection descriptions from parent-env suppression to additive `OPENCODE_CONFIG_CONTENT` merge behavior.
- Marked D-primary-approval defect bullets and D48 suppression language as implemented/past-tense while preserving the decision rationale.

**Structure:**
- No page split, merge, or rename needed. The topic already has focused concept, architecture, codebase, and decision pages with cross-links.

### Validation

- Link and Mermaid validation rerun after edits; see the maintainer run report for command results.

---

## 2026-05-03 — v2 full restructure

### Motivation

The v1 KB accumulated three structural problems over time:

1. **Decision entanglement** — architecture pages (especially `architecture/chat-backend.md`) carried inline decision rationale (D1–D29) rather than pure mechanism descriptions. Decisions blended with how-it-works content made both harder to read.
2. **Overloaded pages** — `concepts/config-and-context.md`, `concepts/model-and-agent-resolution.md`, `concepts/package-management.md`, and `architecture/chat-backend.md` each covered 3–6 unrelated topics. Splitting was deferred too long.
3. **Missing ecosystem coverage** — `meridian-web`, prompt packages (`meridian-base`, `meridian-dev-workflow`, `meridian-prompter`), and the mars-agents compiler had no KB pages. Agents were navigating these repos without reference.

### What changed

**Structure — domain reorganization:**
- `architecture/chat-backend.md` + two companion pages → `architecture/chat/` subtree (9 pages)
- `architecture/telemetry-system.md` → `architecture/telemetry/` subtree (4 pages)
- `architecture/workspace-system.md` → `architecture/workspace/` subtree (4 pages)
- `concepts/model-and-agent-resolution.md` → `concepts/model-resolution/` subtree (4 pages)
- `concepts/package-management.md` → `concepts/package-management/` subtree (5 pages)
- `concepts/config-and-context.md` → three focused pages: `concepts/config-precedence.md`, `concepts/context-resolution.md`, `concepts/workspace-projection.md`

**Decisions — extracted and relocated:**
- Inline decision rationale extracted from architecture pages into domain pages under `decisions/`
- New domain decision pages: `state-and-launch.md`, `chat-backend.md`, `model-resolution.md`, `package-management.md`, `telemetry.md`, `workspace.md`
- Existing `decisions/dev-frontend.md` retained and updated
- Root `decisions.md` converted to thin chronological index; each entry links to domain page anchor
- Architecture pages now contain one-sentence links to decisions rather than inline rationale

**Ecosystem — new domain:**
- Added `ecosystem/` domain (14 pages) covering repos that had zero KB coverage
- `ecosystem/meridian-web/`: overview, extension system, chat extension, connection protocol
- `ecosystem/prompt-packages/`: overview, meridian-base, meridian-dev-workflow, meridian-prompter, prompt-principles
- `ecosystem/jupyter-workbench/`: migrated from root `jupyter-workbench/` into ecosystem subtree

**Preserved:**
- All `codebase/`, `operations/`, `principles/`, `research/`, `lessons/`, `open-questions/` pages carried forward intact (minor link fixes only)
- All decision IDs (D-1 through D-70+) preserved as heading anchors in domain pages so existing cross-references remain valid

### Page count

v1: 55 pages across 11 domains  
v2: ~97 pages across 12 domains (+`ecosystem/`)

The increase comes from splitting 6 overloaded pages into 26 focused pages and adding 14 new ecosystem pages.

### Trigger

KB audit identified 11 pages with structural issues (decision entanglement, topic overload). Ecosystem gap analysis identified 4 repos with zero KB coverage. Rewrite scoped to fix the identified gaps; well-structured pages were not rewritten.

---

## 2026-05-05 — Post launch/startup structural health pass

### Trigger

Two implementation tracks shipped in close succession: launch resolution layer consolidation and CLI lazy-import startup optimization. The KB needed a structural check for stale entries, merge conflicts, oversized docs, and cross-reference integrity.

### What changed

**Structure — decision-page split:**
- Split oversized `decisions/state-and-launch.md` into focused pages:
  - `decisions/state.md`
  - `decisions/launch.md`
  - `decisions/startup-health-sandbox.md`
- Kept `decisions/state-and-launch.md` as a compatibility map so older links still land on a routing page.
- Updated `decisions.md`, `decisions/overview.md`, and `index.md` navigation to list the new focused pages.

**Cross-references:**
- Retargeted startup, sandbox, health-check, config-precedence, model-resolution, and package-management links to the new focused decision pages where the target concept now lives.
- Preserved compatibility links for broad legacy references to the combined state/launch page.

**Content health:**
- Removed stale `doctor-cache.json` from the root overview runtime tree after confirming the source tree only retained a pycache artifact.
- Flagged a telemetry bootstrap contradiction: startup docs describe process-seam-only telemetry installation, but current source still has operation-layer `setup_telemetry()` call sites in `src/meridian/lib/ops/spawn/api.py` and no `TelemetryBootstrap` class was found.

### Validation

- Merge-conflict scan: no conflict markers found.
- Link and Mermaid validation rerun after edits; see the maintainer run report for command results.
