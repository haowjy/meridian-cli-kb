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

| Class | What the owner wants, and why | Enforced by | Seen |
|---|---|---|---|
| Claimed a fact, cause, fix or readiness without checking the live system, or asked the owner what the agent could have checked | Read the source, the diff, the state or the running system first, and label the rest a hypothesis. Canaries must drive the path the real agent uses. Reports from implementers and reviewers are not the diff, and a PR body's wording decides whether an issue closes. Unchecked claims sent work after causes that weren't there and shipped changes nobody asked for | `doc:` base `explore-and-engage` (say what each claim rests on); recurred after it was loaded | 18 times in 14 sessions |
| Fixed only the instance the owner named, without the session or the system around it | Re-read the session and the surrounding design, audit the whole class, and settle every entry point that shares the behavior. A correction names one instance; patching it alone leaves the same problem one command over | `doc:` base `intent-modeling` (a correction names one instance of a class); recurred after it was loaded | 9 times in 7 sessions |
| Argued or designed for a reading of the request the owner did not mean, or took a question as approval | Restate the requested behavior, separate independent mechanisms, and answer a design question before starting. Treat a question or an example as a question or an example, not as the requirement. Each misreading cost a design round built on the wrong problem | `doc:` base `intent-modeling` and `/knowledge-layers` ("maybe X?" is not a choice); recurred after both were loaded | 10 times in 8 sessions |
| Grew the code instead of reusing structure and deleting what it replaced | Before proposing a new mechanism, field or table, find the one that already exists and extend or replace it; justify a large diff. Two copies of one concept drift, and the second copy was the source of real bugs | `doc:` dev-workflow `/code` (delete what the change replaces), meridian-cli `AGENTS.md` (a ten-file change means the abstraction is wrong). Neither loads while a lead designs, where most occurrences happened | 6 times in 6 sessions |
| Labelled a non-breaking change `release:minor` | `release:minor` or `major` only when a change breaks behavior, config or on-disk state; additions and fixes are `release:patch`, even when docs described the bug. A minor release collects on a release branch, so a mislabel holds back a change that could ship now | `doc:` meridian-cli `docs/releasing.md` (label by breakage, not size); recurred after it was written | 3 times in 3 sessions |
| Reported or deferred a found in-scope bug instead of fixing it | Fix a verified defect inside the task, including the same rule elsewhere in the code, then report; file an issue only for what is out of scope. A deferred in-scope fix costs a second review round for work the agent already understood | `none` | 3 times in 3 sessions |
| Kept machinery, or added hardening, for a constraint or threat the project doesn't have | Check an inherited rule against the project's own rules before paying for it. State the threat model before an adversarial review, and send findings outside it back unfixed. Unchecked hardening more than doubled one enforcer before it was cut to a fifth of that size | `none` | 3 times in 3 sessions |
| Committed prompt edits, or gave them to a coding agent | Prompt-package edits are writing work: give them to a writing agent (such as `prompt-dev`) with the prompt-writing skills, and leave them uncommitted so the owner reviews every prompt diff | `doc:` `prompts/AGENTS.md` (leave changes uncommitted); nothing where a lead picks the agent for prompt work | 2 times in 2 sessions |
| Put a choice or a queued fix to the owner without what it changes and why | For each option, say what it changes and why it is recommended; for a fix, say what the command does afterwards. The owner can't rank what they can't evaluate | `none` (`/grill` asks for a recommended answer, not what each option changes) | 2 times in 2 sessions |
| Worked on a different target than the one the owner named: session, harness, tool | Use the exact target named. New code work gets its own worktree and PR, prompt work stays on the package's `main` | `doc:` meridian-cli `AGENTS.md` and `prompts/AGENTS.md` for branches; `none` for naming the target | 3 times in 2 sessions |

## Candidates

Seen once. Each moves into the table on its second occurrence.

- **Accepted a symptom that clears on its own as fixed.** The fix removes the
  symptom; "the TUI redraws over it" or "it goes away on retry" is not a fix.
- **Rested a design on limits that change too fast to track.** No build
  decision rests on model or provider limits, which change every few weeks;
  one number replaces a per-model table.
- **Generalized a default from the owner's own packages to every adopter.**
  Weigh who else adopts a default before calling it safe: adopters can want
  the opposite for reasons the owner's packages don't show.
- **Proposed deleting a seam kept for planned work.** Check whether unused
  code is a seam for something on the roadmap before calling it dead.
- **Started work another session already owned.** Check who owns a task
  before launching it, so two sessions don't race on one change.
- **Didn't bump a dependent package.** Check sibling-repo pins, such as the
  `mars-agents` version, before calling a change done.
- **Presented a malformed command.** Check that a command handed to the owner
  is complete and runs as written.
- **Re-derived a settled design instead of building on it.** Reopen a settled
  design only with new evidence.
- **Shaped a general artifact around developers.** Write a general artifact
  for everyone who reads it, not only developers.

Design corrections on specific features are choices, not agent-behavior
classes; they belong in the [topic pages](index.md).
