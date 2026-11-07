# Review Convergence

Freeze the target and classify recurring findings by their underlying defect. If the same class recurs, examine the boundary or representation rather than adding another exception. Omission cannot mean deletion when it can also mean input was unreadable or unprocessed.

Exercise actual prior-release artifacts through the new command, including unreadable input, partial failure, and retry after repair. A decoder test alone cannot establish write ordering or that the gate is in the right place. See [residue cleanup](residue-cleanup-discipline.md).

## Lane Reviews Do Not Review the Composition

Review the integrated result against the full contract: separately reviewed lanes can leave gaps between their boundaries. A green merge and passing lane reviews do not establish the combined behavior.

## State the threat model before an adversarial review

An adversarial reviewer assumes the strongest attacker it can picture unless the brief says otherwise, and a lead who sends its findings on to a coder has adopted that attacker. Say what the check defends against, and what it doesn't, before asking anyone to attack it. A finding outside that model goes back, unfixed.

The Codex shell hook shows the cost. Its rules guard against accidents, but no threat model was stated, so three review rounds hardened it against an agent attacking its own rule. The enforcer grew from 637 to 1,138 lines and still couldn't stop a REPL or `find -exec`. Stating the real threat cut it to 219 lines of [wildcard matching](../decisions/tool-permissions.md#d-codex-shell-hook).

## A review brief must not carry the lead's verdict

Ask reviewers for evidence, constraints, and tradeoffs without supplying the desired verdict. Otherwise a reviewer can echo the framing even when its evidence supports another result.

A positive hint may narrow work but must not prove absence. Before adding cancellation and preemption around slow work, ask whether the work should run at all. See [Pi runtime decisions](../decisions/pi-runtime.md).
