# Claude native sessions

Claude sessions share a store for a config root. Selecting the newest file or a similar directory slug can select another concurrent conversation; Meridian uses the recorded store and exact ID. See [native session identity](native-session-binding.md).

## Reading transcripts

Tracked reads use the recorded store and ID, independent of the caller's working directory. A missing store is unbound rather than a reason to search ambient roots.

## TUI trampoline

A `/tui fullscreen` successor inferred from prompt history is diagnostic only. A nearby unrelated chat can satisfy the same prompt check, so it cannot prove ownership or replace the entry binding. Resuming Claude in print mode to probe it was rejected because it performs model work and is not deterministic.

## Launch-owned SessionStart journal and current chat

Claude's launch-owned observations distinguish the entry session from later TUI switches. A switch updates the current chat without rebinding the entry; a later continuation selects that current chat and checks its exact transcript. An observed session may not have a readable transcript yet. A disabled hook, full journal or killed launcher can leave a switch unobserved; the last accepted observation remains authoritative. See the source checkout's `src/meridian/lib/launch/native_run.py` and [native session binding](native-session-binding.md).

The journal reader accepts only the launch nonce and child PID, preventing a nested Claude from publishing into its parent's run. It advances its cursor after each durable observation: advancing the entire batch first could lose a switch, while replaying already published observations could transiently rewind the current chat.
