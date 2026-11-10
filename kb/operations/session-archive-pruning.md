# Session archive pruning

## Runner-History Prune

Runner-history pruning is explicit and defaults to a dry run. It removes redundant retired runner streams only when native history is available and safe to retain; it protects live scopes and rechecks identity before deletion. See the [history guide](https://github.com/haowjy/meridian-cli/blob/main/docs/history.md) and [prune rationale](../architecture/native-transcript-reads.md).
