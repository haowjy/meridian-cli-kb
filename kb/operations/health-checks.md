# Doctor and pruning

Doctor scans the current project by default. Cross-project maintenance is explicit because it can be expensive and can touch paths a nested sandbox can't reach. Prune advice appears only when stale artifacts exist, not when the remaining sessions are live. Commands: meridian-cli `docs/commands.md`.

Stored and derived spawn status differ: a dead runner reads as crashed before its outcome is written. Reconciliation writes the outcome and cleans up processes; pruning must protect active state ([derived status](../architecture/state-system/derived-status.md), [project identity](../architecture/state-system/project-identity.md)).

Doctor also migrates rows written by the early native-history build, and a primary launch runs the same repair in the background ([native transcript reads](../architecture/native-transcript-reads.md#early-rows-migrate-once-not-on-read)).
