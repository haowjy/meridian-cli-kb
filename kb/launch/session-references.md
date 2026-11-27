# Spawn and chat references

A reference's shape decides whether it names a spawn (`pN`), a chat (`cN`), a selector or a native session ID. The grammar and per-command behavior are in meridian-cli `docs/commands.md` and `src/meridian/cli/.context/CONTEXT.md`.

## A missing chat or spawn is an error

A missing tracked ID never falls through to an untracked native session: otherwise a mistyped chat ID could resume a different conversation. Untracked native IDs are accepted only where operating on a session Meridian never saw makes sense. Tracked references need an exact recorded binding ([native session binding](../state/native-session-binding.md)).

Reads may use historical fallbacks that resuming refuses: showing an old transcript is less risky than resuming the wrong session.

## A spawn names itself; a chat names its occupant

A spawn reference names that exact spawn, even when another run later occupies its chat. For spawn actions, a chat reference names the run currently occupying the chat, otherwise its own latest linked session, never a child. Resolving to the newest spawn in the chat's family could make `cancel` act on an unrelated child. Following a native `/resume` is best effort.

`--from` a chat uses that chat's transcript. `--from` a primary's spawn renders that session under its chat ID, with the named spawn's own status and report, so a handoff block reads the same whichever ID the caller passed.

## Chat IDs are the name Meridian speaks

Guidance, handoffs and `--from` blocks name a session by chat ID; spawn IDs are still accepted as input everywhere. A chat ID changes after Claude's `/clear`, and the messaging guidance already says so; the owner judged that warning enough, so Meridian builds nothing around `/clear` (no extra ID fields, no telling agents to pass spawn IDs instead).

## IDs are machine-wide

A spawn or chat ID names one session on the whole machine, so a bare ID works from any project and in `--from`, `spawn show`, `session log` or `meridian message` without `-C`. Issuing an ID creates its claim: a symlink under `~/.meridian/machine/ids/` that points at the runtime root the ID was issued for. A lookup is one `readlink`, and the symlink's exclusive create is the only authority on which numbers are taken. Mechanism and the allocation rules: the source checkout's `src/meridian/lib/state/.context/CONTEXT.md` ("Machine-wide IDs") and `src/meridian/lib/ops/.context/CONTEXT.md`.

A claim is not a registry. A registry of projects was turned down for [context sync](../context-sync.md), [telemetry](../telemetry.md) and [messaging](../spawns/messaging.md), each time because it has to be refreshed and can go stale. The allocator writes a claim once, as the act of issuing the ID, and nothing ever updates it: a pruned project leaves its claims pointing at nothing, which only records the numbers as used. The claim directory is an on-disk format, not a Python detail, so the [Rust rewrite](../codebase/rust-rewrite.md) ports it and has nothing to renumber.

IDs issued before claims existed keep their numbers and still count per project. Hundreds of work documents and file names quote a `pN` or `cN`, and renumbering would break every one. An unclaimed ID below the lowest claim is looked up in the current project, so from another project it needs `-C <project>`, and from it a bare old ID can quietly name a different session. Above the lowest claim, an ID with no claim is no such ID. Project-qualified IDs (`meridian-cli/p41`) were passed over: they lengthen every ID, and a project ID is shared by every clone, so it still doesn't name one session across machines. On another machine, `<machine>/<id>` names a session ([machines](../machines.md)).

A name that works everywhere doesn't make sessions in different projects parent and child. Parent, owner, children, owed results and observation records stay inside one runtime root, and the caller's spawn ID counts as the caller only in its own root. Otherwise an old `p41` in one project, continuing a session in another, would become the parent there of a spawn that project's own `p41` never started. Toward another project a session is a sender, not a parent; `--continue` into another project re-runs itself under `-C`, so the launch gets that project's identity and the caller gets no child.

## Ownership starts with the session

Each session generation links to its spawn when it starts, and a run becomes the chat's owner only then: publishing ownership earlier would make every failed launch path undo it. The chat lifetime lock prevents simultaneous continues, and until the new session starts a chat reference still names the previous owner. An exact continue reserves its row against its target chat, so an early failure doesn't name the caller's conversation.

Linking at start replaced reader-side scans for missing links, which cost about 0.9 s per miss on a project with 2,750 spawns. Older chats that were never linked stay addressable by spawn ID.
