# Cross-Repo Release Sequencing

Independent producer and consumer releases can each pass their own gates while their combination fails. For a breaking change, verify the consumer migration is published and usable, not merely committed locally, and confirm the actual dependency version installed by the consumer. Run the new binary against a representative downstream project before calling a change a pin bump. Search for consumers where their declarations actually live: a negative search in a summary or parent manifest can miss them.

When landing a stacked chain, release the producer before updating and gating the consumer pin. Open PRs also need to meet any release-process changes made while they were open. `git merge-tree --write-tree` prints a tree ID even on conflict; check its exit status.

See [release process](../decisions/release-process.md), [package management](../decisions/package-management.md), and [residue cleanup](residue-cleanup-discipline.md).
