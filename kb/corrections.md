# Corrections Register

Each time the human corrects an agent working on Meridian, the correction is
recorded here under the mistake behind it. A class that recurs gets fixed
where the next agent can't repeat it: architecture first, then types, a lint,
a test, a tool permission, and a written rule last (`/correct`). The format is
defined in `/knowledge-layers` `resources/corrections/` (`register.md`,
`code.md`).

Almost every class below is enforced only by a written rule the agent was
already given. That is the pattern this page exists to break: the rule was
present and the mistake happened anyway.

Seeded on 2026-10-15 from 139 primary sessions, 2026-09-15 to 2026-10-15
(`spawn:p8116`, `spawn:p8117`). Six quotes were spot-checked against
transcripts; `chat:c6525` has no searchable transcript.

## Classes

| Class | Correct behavior | Enforced by | Occurrences |
|---|---|---|---|
| Claimed a fact, cause, fix, or readiness without checking the live system | Check the source, state, or running system before claiming; label the rest as a hypothesis | `doc:` base `explore-and-engage` (say what each claim rests on; loaded by every interactive lead since 2026-10-15) | `chat:c6518` 2026-09-23 "ur code just doesn't work"<br>`chat:c7233` 2026-10-02 "it looks like the delayed notices are still happening"<br>`chat:c7433` 2026-10-11 "how do u know this will fix it"<br>`chat:c7485` 2026-10-12 "why is it unverified"<br>`chat:c7524` 2026-10-14 "did u do research against them?"<br>`chat:c7796` 2026-10-15 "is it because you didn't recieve them?"<br>`chat:c8112` 2026-10-17 "what did u change about codex and opencode?" (the lead had taken the implementer's and reviewers' reports as the diff's scope; reading the diff found four unrequested shared-behavior changes, all then gated to Claude only)<br>`chat:c8112` 2026-10-17 "gh says 655 is still open" (the lead reported #655 "fully wrapped up"; its PR body said `Issue #655`, not `Fixes #655`, so the merge never closed it)<br>`chat:c8397` 2026-10-19 "why just pi? what sbout codex and opencode?" (the lead had concluded Codex and OpenCode never wake from canaries that drove other paths: OpenCode's non-resuming `POST /api/shell`, a bare Codex app-server with no TUI; OpenCode's agent tools turned out to wake it)<br>`chat:c8445` 2026-10-19 "if u used "shell" to test it, YOU wouldn't wake it" (same class: the Codex verdict rested on a canary of a path the agent doesn't use)<br>`chat:c8445` 2026-10-19 "nope, its not the goal..." (the lead told the owner their observed Codex auto-continue was "likely" goal mode, a cause the canary had not tested against what the owner saw)<br>`chat:c8468` 2026-10-20, entry 12: "do we have evidence...? (use deepseek to test)" (completion recommendations preceded a runtime probe)<br>`chat:c8468` 2026-10-20, entry 29: "codex aparently doesn't have a cache penalty on when logged in now" (the wait-window rationale assumed an expiring signed-in cache)<br>`chat:c8514` 2026-10-21 "we shoudln't approximate tools from sandbox... right? like claude? it doens't have a sandbo?x" (the lead was about to document Claude as having no sandbox; Claude takes one through `--settings`, and Meridian simply wasn't passing it)<br>`chat:c8514` 2026-10-21 "the tool rules i think can be represented in the harnesses... right?" (the design was documenting OpenCode and Pi tool rules as unsupported without checking what each harness could carry; both enforce them natively now)<br>`chat:c8820` 2026-10-23 "then how and why did we get a double report?" (the lead had blamed the repeat on reads not consuming and built read-consumption for it; the cause in code was no-arg wait discovery skipping children that had already finished) |
| Fixed only the instance the human named, without reconstructing the session or the system around it | Re-read the session and the surrounding architecture; audit the whole class, not the one instance | `doc:` base `intent-modeling` (a correction names one instance of a class; loaded by every interactive lead since 2026-10-15) | `chat:c7233` 2026-10-02 "review the full system and not just the changes"<br>`chat:c7341` 2026-10-09 "can u read the session again"<br>`chat:c7736` 2026-10-15 "i thought we moved to a single file per chat"<br>`chat:c7796` 2026-10-15 "only tackling these changes at a surface level"<br>`chat:c8397` 2026-10-18 "or maybe there is better wording for all the events... any events u think need clean up?" (for #687's dead `work.start`, the lead proposed only making `work switch` also fire `work.started`; the owner had just rejected that collapse with "then shouldn't there be 2 events", and the audit of the whole set produced [D-hook-event-names](decisions/hook-event-contract.md#d-hook-event-names--past-tense-moments-named-for-what-just-happened))<br>`chat:c8397` 2026-10-19 "for any versioning / semver for our project" (asked how Plugin API versions were compared, the lead kept answering for the Plugin API; the owner meant the project's whole versioning, which became [D-compat-policy](decisions/compatibility-versioning.md#d-compat-policy--one-minor-deprecation-new-builds-read-old-data-old-builds-refuse-new-data))<br>`chat:c8765` 2026-10-23 "and can u make sure to fully research opencode codex and pi properly, and then lets design what needs to be a mars-agent change, what needs to be meridian-cli change..." (after Claude and Codex probes, the lead proposed redefining `workspace-write` as network-on plus profile stopgaps, and asked three times to start; the owner first asked "should we add to the mars spec...?" and "is this a mars-agents change?", then required every harness researched and a Mars / meridian-cli / profile split, which became [sandbox platform access](decisions/sandbox-platform-access.md)) |
| Wrote tests by default: too many, or in place of probing the real system | A few tests on durable behavior; settle runtime questions by probing the real system | `doc:` dev-workflow `/testing`, `/probe`; meridian-cli `tests/AGENTS.md` | `chat:c6326` 2026-09-19 "minimize the number of unit tests"<br>`chat:c6452` 2026-09-22 "make sure we didn't overtest"<br>`chat:c7796` 2026-10-15 "probing and using actual system" |
| Worked somewhere other than the target the human named: session, harness, branch | Use the exact target named; new work gets its own worktree and PR | `doc:` meridian-cli `AGENTS.md` (worktrees), dev-workflow `/pre-dev` | `chat:c7233` 2026-10-02 "i meant a2"<br>`chat:c7485` 2026-10-12 "u would be using meridian spawn... not opencode run"<br>`chat:c7485` 2026-10-12 "don't be on main... make a new PR + worktree" |
| Merged agent roles, or dropped a model-routing reason, while editing prompts | Keep roles separate when behavior, ownership, or default model differs; check the history before collapsing one | `doc:` prompter `agent-artifacts` (read an agent's history before merging, deleting, or narrowing it) | `chat:c7245` 2026-10-03 "its only big prototypes that could be useful with a (weaker) frontend dev"<br>`chat:c7796` 2026-10-15 "parallel lanes is not its only job"<br>`chat:c7796` 2026-10-15 "frontend-coder actually i think is still needed" |
| Grew the code instead of reusing structure and deleting what it replaced | Reuse what exists, delete what the change replaces, and justify a large diff | `doc:` meridian-cli `AGENTS.md` ("10-file edits = wrong abstraction"), dev-workflow `/code` | `chat:c6525` 2026-09-23 "729 additions & 92 deletions... why so much added?"<br>`chat:c7524` 2026-10-14 "why is there so much new code, but none deleted?"<br>`chat:c7736` 2026-10-16 "i thought we already had a watchdog?" (the identity-v2 design proposed a new watchdog without accounting for `parent_watchdog.py`; the answer generalized and replaced it)<br>`chat:c8112` 2026-10-17 "having the same pi/codex rearm system would be better" (the lead's first plan for #655 was a one-shot Claude-only nudge beside the resident nudge and rearm system)<br>`chat:c8514` 2026-10-21 "don't we have a map of tools...?" (the lead framed Mars owning harness tool names as new work; Mars already had the table, and Meridian's second table was the source of the name bugs)<br>`chat:c8820` 2026-10-23 "tbh... shouldn't spawn wait just be enough...?" (to stop a repeated delivery, the lead was turning `spawn show`, `spawn report show` and `session log` into consumption writers; header-only delivery made the repeat one short ping and the writers were dropped) |
| Overrode the model or routing default | Use the profile's default model and its fallbacks; override only on the human's request or a stated requirement | `doc:` harness spawn guidance; meridian-cli `AGENTS.md` (cheap smoke models) | `chat:c6452` 2026-09-22 "never use astra as the explorer... its way too expensive"<br>`chat:c6945` 2026-09-25 "plz be using deepseek (not pro) as the primary code model" |
| Designed around a Meridian-owned history instead of the native transcript | Read native transcripts; a Meridian id binds to one native session; indexes stay rebuildable projections | `architecture:` transcript providers reject runner `history.jsonl` (meridian-cli `lib/harness/.context/session-transcripts.md`) | `chat:c6518` 2026-09-23 "we ALWAYS are reading from the transcript"<br>`chat:c6945` 2026-09-25 "stop relying on history.jsonl for anything" |
| Edited prompts from memory without reading the required references | Read the referenced resources in full before writing | `architecture:` the prompt-text principles load in the `/prompt-principles` body; `doc:` prompter `prompt-dev` for the level resources | `chat:c7796` 2026-10-15 "why didn't u load these resources?"<br>`chat:c7796` 2026-10-15 "don't our prompt principles already prefer positive frameing" |
| Left stacks, browsers, worktrees, or scratch running after the work ended | Name what you create with your spawn id, stop and delete it before reporting, sweep a failed spawn's leftovers, keep raw captures out of work dirs | `doc:` dev-workflow `/probe`, `/post-dev`, `@prober` description; base `/work-artifacts`; `architecture:` context repos sync an allowlist (notes, images, work state) | dev-workflow 0.13.12 2026-10-01 "clean up everything the branch claimed"<br>`chat:c7796` 2026-10-15 "we have to make sure they clean up after themselves" (global OOM kill at 19:03; dev stacks held 12.8 GB, `/tmp` 38 GB) |
| Made an explicit-only model an automatic fallback | Keep a model out of an agent's fallbacks until the human decides it belongs among that agent's backups; the decision is per agent | `doc:` prompter `agent-artifacts`. `no-fallback` set in each agent's profile | `chat:c7796` 2026-10-15 "options but not a fallback?"<br>`chat:c7796` 2026-10-15 "i don't think grok should be a fallback yet" |
| Argued or designed for a reading of the request the human did not mean | Restate the requested behavior and distinguish independent mechanisms before evaluating alternatives | `doc:` base `intent-modeling` | `chat:c8112` 2026-10-17 "I mean move inject into a mod etc. and then we don't need the input stream anymore" (the lead argued against moving the completion decision into a mod instead)<br>`chat:c8468` 2026-10-20, entry 22: "completion window for what, for the parent one...?" and "the outside \"wait\" will always still end at 50 minutes. and this is recursive" (the lead kept framing sticky rearm and a 55-minute backstop instead of the child handshake and independent caller checkpoint); entry 24: "it depends on whatever the harnesses's idle time is set to"<br>`chat:c8788` 2026-10-24 "btw, when i say \"mars.toml\" etc, i mean we shoudl be merging stuff across the different surfaces of configuration", then "welll... idk if it hsould be wins... should it be merged...?" (the lead had read "mars.toml" as one project-default value and kept framing the network as highest-surface-wins) |

`/correct` ran on 2026-10-15 for the top three classes and the three prompt-authoring classes: each now has its rule where the agent decides, or loaded instead of opened. Tests-instead-of-probing is unchanged; its rules in `/testing` and `/probe` already sit at the point of choice.

The cleanup and explicit-only-model classes each come from one session;
treat them as weaker evidence than classes seen across sessions.

## Candidates

Seen once. Each moves into the table on its second occurrence.

- **Reported completion before waiting for a spawn.** Wait for delegated work
  and read its result first. `chat:c7153` 2026-09-27 "its done, why didnt u
  wait urself?" Enforced by `doc:` harness spawn guidance.
- **Reported a found bug instead of fixing it.** Fix a verified in-scope
  defect, then report. `chat:c6945` 2026-09-25 "if it found a bug, shouldn't
  u make the change to that?"
- **Committed prompt edits the human wanted to review first.** Leave prompt
  edits uncommitted until the human says to commit. `chat:c7245` 2026-10-03
  "dont push or commit anything, I want to personally review any prompt
  changes". This was that session's instruction; later sessions asked for
  commits.
- **Put a design spec in the PR.** Keep active design in the work item.
  `chat:c6525` 2026-09-23 "it should be in its work item?"
- **Didn't bump a dependent package.** Check sibling-repo pins before calling
  a change done. `chat:c6209` 2026-09-16 "why didn't u bump mars-agents?"
- **Wrote speculative guards into a prompt.** Add defensive code only for a
  failure that can happen. `chat:c6188` 2026-09-16 "a lot of the time they are
  useless code". Enforced by `doc:` dev-workflow `/solo-dev`.
- **Presented a malformed command.** Check a command you hand the human is
  complete. `chat:c7405` 2026-10-11 "why did it separate the --from ?"
- **Re-derived a settled design instead of progressing.** `chat:c6452`
  2026-09-22.
- **Shaped a general artifact around developers.** `chat:c7258` 2026-10-08
  "it shouldn't be dev focused".
- **Accepted a symptom that clears on its own as fixed.** The fix removes the
  symptom; "the TUI redraws over it" or "it goes away on retry" is not a fix.
  `chat:c7524` 2026-10-15 "it still garbles the screen when claude hasn't
  redrawn --- we shouldn't garble the text at all".
- **Kept machinery to honour a constraint the project doesn't have.** Check an
  inherited design rule against the project's own rules before paying for it.
  `chat:c7736` 2026-10-16 "do we even need a runID?" (#616's second identity
  existed to satisfy an old "no data migration" rule that AGENTS.md's
  no-backward-compatibility rule never required; the owner's questions led to
  [D-spawn-identity-v2](decisions/spawn-identity.md#d-spawn-identity-v2)).
- **Landed KB edits through a branch instead of the autosynced checkout.**
  Edit the KB in `$MERIDIAN_CONTEXT_KB_DIR`, including pages about unmerged
  code. `chat:c8174` 2026-10-18 "knowledge base is supposed to be auto synced
  as well", then "no... pages describing code that hasn't merged yet should
  also directly edit it usually imo" (the lead had just written a KB rule
  sending unreleased-code pages through a PR). Both in one session. Enforced
  by `doc:` KB `AGENTS.md` ("Landing KB changes").
- **Took a design question as approval.** Answer the question before
  starting. `chat:c8397` 2026-10-18 "add them, and then I think can u add
  filter...? how would u add filter?" (the lead replied "The full scope is
  approved" and began pre-dev without explaining the filter's design; it
  became `when.kind`).
- **Put a decision or change to the owner without the reason needed to judge
  it.** Say what each option changes and why it is recommended. `chat:c8514`
  2026-10-21 "ask again and explain better" (four of a twelve-question round
  were left pending because the owner couldn't evaluate them), then "Treat X:*
  as X *. - why?" (listed as a Mars change without its reason: OpenCode reads
  the colon literally). Both in one session.
- **Generalized a default from the owner's own packages to every adopter.**
  Weigh who else adopts the default before calling it safe. `chat:c8788`
  2026-10-24 "\"no profile in the packages wants its shell offline\" - not
  true, there's tons of reasons to have it offline" (default `network: allow`
  rested on the packaged profiles; adopters who expect a sandboxed agent
  offline were not weighed).
- **Opened a PR for a docs-only change.** `chat:c8514` 2026-10-21 "docs don't
  need a PR" (draft #713 asked for merge approval on a docs fix).
- **Advanced a structural change without the strict maintainability
  review.** `chat:c8514` 2026-10-21 "did we have /thermo-nuclear-review
  review?"
- **Didn't record repeated corrections.** `chat:c7796` 2026-10-15 "aren't
  repeated mistakes supposed to be recorded into the kb...?" Enforced by
  `doc:` base `@kb-lead` and this register.

Design corrections on specific features (harness resolution order, OpenCode
2.x over 1.x, model catalog layers, raw-flag precedence, warn instead of
block) are decisions, not agent-behavior classes; they belong in the
[decision index](decisions.md).
