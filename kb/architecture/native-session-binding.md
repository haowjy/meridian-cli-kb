# Native session binding

A chat is bound to one native key: harness, native store and native session ID. The store is part of the key because the same ID in another store can name a different conversation. Continue and fork require the exact recorded key and validate the native header; they refuse an incomplete, missing, ambiguous or contradictory binding instead of guessing from timestamps, filenames, prose, primary metadata or ambient roots. A wrong guess can disclose another conversation and bind it to the wrong chat permanently. Contract: the source checkout's `src/meridian/lib/harness/AGENTS.md`. Harness specifics: [Claude](claude-native-sessions.md), [Pi](pi-native-sessions.md).

## Entry evidence is an exact target

Before execution Meridian assigns or obtains an exact target, or verifies the recorded source for resume and fork. The harness's guarantee that it opens that target is the entry evidence; a contradictory first owned signal fails the attempt. Requiring an observed entry instead would leave Pi's interactive TUI untrackable: it has no RPC gate before input, though Pi guarantees it opens the assigned target.

Identity comes only from the assigned target or qualified owned events. Picking the newest journal in the same directory, parsing model prose, or searching a payload for an ID-shaped value can bind another process's conversation. Discovery and primary metadata can't establish identity either: a detected file may belong to another session, and a spawn mirror can't supply an identity the chat never bound. `--from` may omit an unavailable best-effort pointer; continue and fork refuse.

Tracked launches refuse passthrough flags that would override Meridian-managed native identity, operation, store or persistence: the harness would then open or persist a session other than the one the binding records.

## One source key, one binding rule

The source chat's store and ID travel together through resume and fork, and each adapter translates that key into its harness's mechanism. Separate harness-specific source fields drifted: one fork used an ambient store, another found a same-ID decoy.

Binding fills missing fields and never overwrites a different value. A first owned signal that contradicts a pre-exec assignment fails, as does a fork that reuses its source ID; later observations are diagnostic or identify a separate current chat. Legacy import and repair need exact evidence too ([legacy native import](legacy-native-import.md)).

## Entry is fixed; exit is observed or unresolved

The entry chat stays fixed even when a TUI switches conversations. Pi proves its exit with an owned final quit record; without valid evidence the exit is unresolved. Claude records the last owned live observation as the current chat instead of claiming a verified exit. A spawn reference may follow the accepted current chat; a chat reference keeps naming its own conversation ([session references](../concepts/session-references.md)).

A newly observed chat has fresh history and no copied lease or fork lineage; an existing chat keeps its own settings. Native `/resume` into a chat another run holds is recorded and warned about, not prevented.

The child must exit and teardown must be joined before the run concludes, because Pi writes its exit record while shutting down: real Pi showed the runner reading it about 4 s before Pi wrote `quit`. The fix orders the read after teardown, with no polling or timeout ([lesson](../lessons/native-session-identity.md#terminal-status-is-not-process-exit)).

## Settings belong to the conversation

Final binding persists policy on the captured entry generation before native execution. The primary's final snapshot doesn't exist yet when its session scope opens, so recording preview policy there, or waiting until after the run, would give live observations the wrong settings. Observed aliases inherit the selected conversation's policy and work. Older generations without a snapshot read only their recorded spawn link; there is no newest-owner fallback.

A `pN` continuation or fork resolves the selected chat's current generation exactly as `cN` does, settings, work and source history included; it never pairs that chat's native identity with the original run's policy.

## Paths move only through `relocate`

`session relocate` changes recorded paths without changing the harness or native ID, and updates harness-specific directory information without moving or truncating transcripts; continue never silently rebinds. It can't repair a wrong ID, and it must know both ends before suggesting a runnable command, because a worktree path can't be inferred from a project root. A Claude missing-file search is limited to the recorded ID and config root, validates headers, and refuses several distinct copies; a found file doesn't become a new identity.

## Accepted limits

- A verified file can be replaced between preflight and harness open; detection fails the attempt, possibly after input already reached the model.
- UUID collision checks are not atomic reservations against external writers.
- An unsaved Pi session may have no transcript and can't be continued yet.
- The missing-file search is Claude-only ([#863](https://github.com/haowjy/meridian-cli/issues/863)); path-derived project identity doesn't survive a project move, and that is not planned to change ([#864](https://github.com/haowjy/meridian-cli/issues/864)); a killed relocation can orphan its OpenCode server ([#867](https://github.com/haowjy/meridian-cli/issues/867)).
