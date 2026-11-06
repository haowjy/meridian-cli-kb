# Idle TTL Detection

## D-idle-ttl-detection — Detection reads launch records, never a cwd

Resolve the transcript from the chat's launch-recorded native store. An agent can change cwd while the transcript remains in its launch store; guessing from current cwd can silently miss it or select the wrong file. A missing or invalid binding yields an unknown TTL. See [native session binding](../architecture/native-session-binding.md).

## OpenCode provider TTL

Provider-derived TTLs are estimates, not guarantees. OpenCode's provider comes from the session observation, and unsupported providers have unknown TTL. Signed-in Codex and Work retention is not established by the old 1800-second estimate; [#707](https://github.com/haowjy/meridian-cli/issues/707) tracks the idle warning. See [prompt-cache retention](../research/prompt-cache-retention.md) for evidence and caveats.
