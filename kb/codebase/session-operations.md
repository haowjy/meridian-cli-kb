# Session operations

## Session Browse

Browse resumes a stopped session and forks a running one, avoiding simultaneous attachment to the same native session. It blocks re-entry for an unbound chat instead of guessing its identity. See [session browse decisions](../decisions/session-browse.md) and the source checkout's `src/meridian/cli/.context/CONTEXT.md`.

## Transcript Source Resolution

Tracked history reads only the chat's bound native source. Substituting another transcript or runner output could expose unrelated content or mislead the reader. See [native session identity](../decisions/native-session-identity.md) and [native transcript reads](../architecture/native-transcript-reads.md).

A transcript path is resolved from the recorded native store and ID rather than persisted as a fixed path, because a transcript may be re-seeded into another store. See [Claude native sessions](../architecture/claude-native-sessions.md#reading-transcripts).

Session log navigation and output are described in the CLI help and [rendering rationale](session-log-rendering.md).

