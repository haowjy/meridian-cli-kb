# Idle TTL Detection

## D-idle-ttl-detection — Detection reads launch records, never a cwd

The idle service passes the chat's launch-recorded native store to the harness
TTL detector. Claude resolves the live session ID inside that store and reads
the latest assistant cache-creation usage. Looking up the current working
directory would miss the transcript after the agent changes directories: Claude
files it under the launch directory. Missing or invalid identity and unreadable
transcripts yield an unknown TTL rather than a guessed location. See
[native session binding](../architecture/native-session-binding.md).

Pi chooses a TTL from the observed provider and `PI_CACHE_RETENTION`. Anthropic
is known at 300 seconds for short retention and 3600 for long. OpenAI long
retention and OpenAI Codex use an estimated 1800 seconds; unsupported providers
and OpenAI short retention are unknown. The OpenAI Codex estimate matches the
Codex harness estimate, but is not a backend guarantee. The
[retention research](../research/prompt-cache-retention.md#chatgpt-backend-codex-and-pis-openai-codex)
describes the evidence and limits.

Provider-based detection does not distinguish models, so an older model may
receive an estimate that is too long. A configured `ttl_seconds` takes
precedence over detection.

## OpenCode provider TTL

OpenCode uses the observed provider and estimates 1800 seconds for OpenAI;
other providers have unknown TTL. Signed-in Codex and Work retention is not
established by that estimate; [#707](https://github.com/haowjy/meridian-cli/issues/707)
tracks the idle warning. See [prompt-cache retention](../research/prompt-cache-retention.md)
for evidence and caveats.
