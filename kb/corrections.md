# Corrections Register

Each time the owner corrects an agent working on Meridian, the correction is
recorded here under the mistake behind it. A class that recurs gets fixed
where the next agent can't repeat it: architecture first, then types, a lint,
a test, a tool permission, and a written rule last (`/correct`). The format is
defined in `/knowledge-layers` `resources/corrections/`.

A class leaves this page once its lesson sits where the agent makes the
decision and the mistake has stopped recurring. Most rows below already have a
written rule and recurred after it landed: the rule was loaded and the mistake
happened anyway, so a stronger fix is still owed.

## Classes

Each occurrence names the session (`chat:`) or spawn (`spawn:`), the date, and
what the owner corrected. The IDs and dates show whether a mistake recurred
after the rule meant to prevent it landed, and keep a later miner from counting
the same correction twice.

| Class | Correct behavior | Enforced by | Occurrences |
|---|---|---|---|
| Claimed a fact, cause, fix or readiness without checking the live system, or asked the owner what the agent could have checked | Read the source, the diff, the state or the running system first, and label the rest a hypothesis. Canaries must drive the path the real agent uses. Reports from implementers and reviewers are not the diff, and a PR body's wording decides whether an issue closes. Unchecked claims sent work after causes that weren't there and shipped changes nobody asked for | `doc:` base `explore-and-engage` (say what each claim rests on); recurred after it was loaded | `chat:c6518` 2026-09-23: said the code worked; it didn't run<br>`chat:c7233` 2026-10-02: reported delayed notices fixed; they still happened<br>`chat:c7433` 2026-10-11: claimed a fix would work with no evidence that it would<br>`chat:c7485` 2026-10-12: presented a change as ready while it was still unverified<br>`chat:c7524` 2026-10-14: designed against other tools without researching them<br>`chat:c7796` 2026-10-15: asserted a cause without checking whether the messages had been received<br>`chat:c8112` 2026-10-17: took the implementer's and reviewers' reports as the diff's scope; the diff held four unrequested Codex and OpenCode changes<br>`chat:c8112` 2026-10-17: reported #655 finished; its PR body said `Issue #655`, not `Fixes #655`, so the issue stayed open<br>`chat:c8397` 2026-10-19: concluded Codex and OpenCode never wake, from canaries that drove paths the agent doesn't use<br>`chat:c8445` 2026-10-19: judged Codex wake-up from a shell canary, a path the agent doesn't use<br>`chat:c8445` 2026-10-19: blamed the owner's observed Codex auto-continue on goal mode without testing it<br>`chat:c8468` 2026-10-20: recommended completion changes before any runtime probe<br>`chat:c8468` 2026-10-20: rested the wait window on a signed-in Codex cache penalty that no longer exists<br>`chat:c8514` 2026-10-21: was about to document Claude as having no sandbox; it takes one through `--settings`<br>`chat:c8514` 2026-10-21: documented OpenCode and Pi tool rules as unsupported without checking the harnesses; both enforce them natively<br>`chat:c8820` 2026-10-23: blamed a double report on reads not consuming; the cause was no-arg wait skipping children that had already finished<br>`chat:c9589` 2026-10-27: argued from Codex and OpenCode plugins lacking agents; OpenCode's gap was in Mars's plugin build<br>`chat:c10170` 2026-11-06: kept asking which warnings the owner meant instead of scanning the local install and config |
| Fixed only the instance the owner named, without the session or the system around it | Re-read the session and the surrounding design, audit the whole class, and settle every entry point that shares the behavior. A correction names one instance; patching it alone leaves the same problem one command over | `doc:` base `intent-modeling` (a correction names one instance of a class); recurred after it was loaded | `chat:c7233` 2026-10-02: reviewed only the changes, not the full system<br>`chat:c7341` 2026-10-09: answered without re-reading the session<br>`chat:c7736` 2026-10-15: designed against storage the project had already moved to one file per chat<br>`chat:c7796` 2026-10-15: handled the changes at the surface only<br>`chat:c8397` 2026-10-18: patched the dead `work.start` event alone instead of reviewing every event name<br>`chat:c8397` 2026-10-19: answered for the Plugin API's versioning when the owner meant the whole project's<br>`chat:c8765` 2026-10-23: proposed a sandbox change from Claude and Codex probes alone; the owner required every harness researched and a Mars, meridian-cli and profile split<br>`chat:c9182` 2026-10-26: moved the stale work-ID problem into `meridian work current` instead of removing it<br>`chat:c9182` 2026-10-26: settled `work done` alone; reopening an item and starting an archived one move it too |
| Argued or designed for a reading of the request the owner did not mean, or took a question as approval | Restate the requested behavior, separate independent mechanisms, and answer a design question before starting. Treat a question or an example as a question or an example, not as the requirement. Each misreading cost a design round built on the wrong problem | `doc:` base `intent-modeling` and `/knowledge-layers` ("maybe X?" is not a choice); recurred after both were loaded | `chat:c8112` 2026-10-17: argued against a mod deciding completion; the owner meant moving injection into a mod<br>`chat:c8397` 2026-10-18: took a question about how a filter would work as approval and began pre-dev without answering it<br>`chat:c8468` 2026-10-20: kept framing sticky rearm and a 55-minute backstop when the owner meant a child handshake and an independent caller checkpoint, and assumed a fixed window where the harness's idle time decides it<br>`chat:c8788` 2026-10-24: read `mars.toml` as one project-default value and framed the network setting as highest-surface-wins; the owner meant merging across configuration surfaces<br>`chat:c9019` 2026-10-24: took one package's hand-built plugin as the requirements for a generic plugin export<br>`chat:c9019` 2026-10-24: built one plugin per graph package; the owner meant one plugin of the full resolved graph<br>`chat:c9182` 2026-10-27: gave per-chat links to primaries only over two rounds; the owner had assumed every execution owns its links<br>`chat:c10236` 2026-11-06: took the owner's answer to "what should the limit be" as a request for a work limit and designed it through to a hard stop |
| Grew the code instead of reusing structure and deleting what it replaced | Before proposing a new mechanism, field or table, find the one that already exists and extend or replace it; justify a large diff. Two copies of one concept drift, and the second copy was the source of real bugs | `doc:` dev-workflow `/code` (delete what the change replaces), meridian-cli `AGENTS.md` (a ten-file change means the abstraction is wrong). Neither loads while a lead designs, where most occurrences happened | `chat:c6525` 2026-09-23: shipped 729 additions and 92 deletions without a reason for the size<br>`chat:c7524` 2026-10-14: added a lot of new code and deleted none<br>`chat:c7736` 2026-10-16: proposed a new watchdog beside the existing `parent_watchdog.py`<br>`chat:c8112` 2026-10-17: planned a one-shot Claude-only nudge beside the existing nudge and rearm system<br>`chat:c8514` 2026-10-21: framed a harness tool-name map as new work; Mars already had one, and Meridian's second copy caused the name bugs<br>`chat:c9186` 2026-10-25: proposed a new `audience` frontmatter field; the profile's `mode` field already covered it |
| Labelled a non-breaking change `release:minor` | `release:minor` or `major` only when a change breaks behavior, config or on-disk state; additions and fixes are `release:patch`, even when docs described the bug. A minor release collects on a release branch, so a mislabel holds back a change that could ship now | `doc:` meridian-cli `docs/releasing.md` (label by breakage, not size); recurred after it was written | `chat:c9589` 2026-10-27: labelled #796 and #254 minor; #254 only widened the user config<br>`chat:c9677` 2026-10-28: kept offering minor for #801's bug fix because the docs had described the bug<br>`chat:c10119` 2026-11-05: wrote a new journal row at v2 and called the change minor for that; a v1 row would have broken nothing |
| Reported or deferred a found in-scope bug instead of fixing it | Fix a verified defect inside the task, including the same rule elsewhere in the code, then report; file an issue only for what is out of scope. A deferred in-scope fix costs a second review round for work the agent already understood | `none` | `chat:c6945` 2026-09-25: reported a bug a reviewer found instead of fixing it<br>`chat:c9677` 2026-10-28: fixed chat-ID resolution for `spawn message` only and offered `show`, `wait` and `cancel` as a follow-up issue<br>`chat:c9758` 2026-10-28: recommended deferring the rest of the cancel rewrite and the repair pass; both could be done in the task and shipped in v0.18.0 |
| Kept machinery, or added hardening, for a constraint or threat the project doesn't have | Check an inherited rule against the project's own rules before paying for it. State the threat model before an adversarial review, and send findings outside it back unfixed. Unchecked hardening more than doubled one enforcer before it was cut to a fifth of that size | `none` | `chat:c7736` 2026-10-16: kept a second spawn identity to satisfy an old no-migration rule the project didn't require<br>`chat:c9566` 2026-10-27: recommended amending an invariant the owner then deleted as not mattering<br>`chat:c10236` 2026-11-06: let a security reviewer set the Codex hook's threat model; the enforcer grew from 637 to 1,138 lines before the owner cut it to 219 |
| Committed prompt edits, or gave them to a coding agent | Prompt-package edits are writing work: give them to a writing agent (such as `prompt-dev`) with the prompt-writing skills, and leave them uncommitted so the owner reviews every prompt diff | `doc:` `prompts/AGENTS.md` (leave changes uncommitted); nothing where a lead picks the agent for prompt work | `chat:c7245` 2026-10-03: the owner had to say prompt changes stay uncommitted until they review them<br>`chat:c9589` 2026-10-27: ran prompt lanes on `coder`, which committed; both branches were reset and the work handed to `prompt-dev` |
| Put a choice or a queued fix to the owner without what it changes and why | For each option, say what it changes and why it is recommended; for a fix, say what the command does afterwards. The owner can't rank what they can't evaluate | `none` (`/grill` asks for a recommended answer, not what each option changes) | `chat:c8514` 2026-10-21: left four of twelve questions unanswerable for want of reasons, and listed a Mars change without saying why it was needed<br>`chat:c10170` 2026-11-06: ranked #712 as next work without saying what the command would do after the fix |
| Worked on a different target than the one the owner named: session, harness, tool | Use the exact target named. New code work gets its own worktree and PR, prompt work stays on the package's `main` | `doc:` meridian-cli `AGENTS.md` and `prompts/AGENTS.md` for branches; `none` for naming the target | `chat:c7233` 2026-10-02: worked on a different session than the one the owner named<br>`chat:c7485` 2026-10-12: ran `opencode run` where the owner meant `meridian spawn`<br>`chat:c7485` 2026-10-12: worked on `main` instead of a new worktree and PR |

## Candidates

Seen once. Each moves into the table on its second occurrence.

- **Accepted a symptom that clears on its own as fixed.** The fix removes the
  symptom; "the TUI redraws over it" or "it goes away on retry" is not a fix.
  `chat:c7524` 2026-10-15: treated screen garbling that clears when Claude
  redraws as acceptable.
- **Rested a design on limits that change too fast to track.** No build
  decision rests on model or provider limits, which change every few weeks;
  one number replaces a per-model table. `chat:c10236` 2026-11-06: asked for
  per-model and per-provider limits the design had no use for.
- **Generalized a default from the owner's own packages to every adopter.**
  Weigh who else adopts a default before calling it safe: adopters can want
  the opposite for reasons the owner's packages don't show. `chat:c8788`
  2026-10-24: rested a default of `network: allow` on the packaged profiles
  alone.
- **Proposed deleting a seam kept for planned work.** Check whether unused
  code is a seam for something on the roadmap before calling it dead.
  `chat:c9589` 2026-10-27: a dead-code review proposed deleting the extension
  capability check, kept for third-party extensions.
- **Started work another session already owned.** Check who owns a task
  before launching it, so two sessions don't race on one change.
  `chat:c9589` 2026-10-28: launched a Mars version bump another session was
  already doing.
- **Didn't bump a dependent package.** Check sibling-repo pins, such as the
  `mars-agents` version, before calling a change done. `chat:c6209`
  2026-09-16: left the `mars-agents` pin unbumped.
- **Presented a malformed command.** Check that a command handed to the owner
  is complete and runs as written. `chat:c7405` 2026-10-11: handed over a
  command with `--from` split away from its value.
- **Re-derived a settled design instead of building on it.** Reopen a settled
  design only with new evidence. `chat:c6452` 2026-09-22: re-derived a
  design already settled.
- **Shaped a general artifact around developers.** Write a general artifact
  for everyone who reads it, not only developers. `chat:c7258` 2026-10-08:
  wrote a general artifact for developers only.

Design corrections on specific features are choices, not agent-behavior
classes; they belong in the [topic pages](index.md).
