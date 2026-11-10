# Doctor and pruning

Doctor defaults to the current project; cross-project scans are explicit because they can be expensive and affect other projects. See the [command reference](https://github.com/haowjy/meridian-cli/blob/main/docs/commands.md) and [startup and health decisions](../decisions/startup-health-sandbox.md#doctor-and-health-checks).

Stored and derived spawn status differ: a dead runner can appear crashed before its outcome is persisted. Reconciliation writes the outcome and cleans up processes; pruning must protect active state. See [derived status](../architecture/state-system/derived-status.md) and [project retention](../architecture/state-system/project-identity.md#retention).

## Dogfood-row migration

Doctor migrates rows written by the early native-history build; a primary launch also runs the repair in the background. See [native-only history](../architecture/native-transcript-reads.md).
