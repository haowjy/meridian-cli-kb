# Native Session Identity

A chat is bound to one native key: harness, native store and native session ID. The store matters because the same ID in another store can name a different conversation. Continue and fork require the exact recorded key and validate the native header; they refuse an incomplete, missing, ambiguous or contradictory binding rather than guessing from timestamps, filenames, prose, primary metadata or ambient roots. A wrong guess can disclose another conversation and permanently bind it to the wrong chat. Native transcripts, not runner history or search indexes, are content authority. See [native session binding](../architecture/native-session-binding.md) and [native-only history](native-only-history.md).

## Entry evidence is a harness-guaranteed exact target

Before execution Meridian assigns or obtains an exact target, or verifies the recorded source. Harness guarantees about opening that target provide entry evidence; a contradictory first owned signal fails the attempt. A later switch does not change the entry binding. An observed-entry-only rule was rejected because Pi's interactive TUI has no pre-input RPC gate, leaving it untrackable despite a harness guarantee about the assigned target.

Identity may come only from the assigned target or qualified owned events. Choosing a newest same-directory journal, parsing model prose, or recursively finding an ID-shaped value in a payload can bind another process's conversation.

## One recorded source key

The source chat's recorded native store and ID travel together through resume and fork; each adapter translates that key into its harness's mechanism. Separate harness-specific source fields drifted, causing one fork to use an ambient store and another to find a same-ID decoy.

## One binding rule, one drift rule

Binding fills missing fields but does not overwrite a different value. A first owned signal that contradicts a pre-exec assignment, or a fork that reuses its source ID, fails; subsequent observations are diagnostic or identify a separate current chat. `relocate` is the explicit path-changing exception. Exact legacy import and repair do not permit arbitrary rebinding.

## Exit identity is observed or unresolved

The entry chat stays fixed even if a TUI switches conversations. Pi can prove an exit through an owned final quit record; absent valid evidence its exit is unresolved. Claude records the last owned live observation instead of claiming a verified exit. A spawn reference may follow the accepted current chat, while a chat reference keeps naming its own native conversation.

## Current chat replaces final-exit proof for Claude

Claude's launch-correlated `SessionStart` observations update the current chat. Hard kill, a disabled hook or a full journal can leave switches unobserved; no timestamp or directory scan fills the gap. A same-project successor found in shared history is not evidence it belongs to this process. A newly observed chat has fresh history and no copied lease or fork lineage; an existing chat retains its own settings. Native `/resume` into a chat held by another run is recorded and warned about, not prevented.

## Paths move only through `relocate`

An explicit `session relocate` changes recorded paths without changing the chat's harness or native ID; continue never silently rebinds. A Claude missing-file search is restricted to the recorded ID and config root, validates headers, and refuses multiple distinct copies. A found file does not become a new identity. Relocation must know both ends before suggesting a runnable command; worktree paths cannot safely be inferred from a project root.

Relocate updates the harness-specific directory information where needed without moving or truncating transcripts. It cannot repair a wrong ID. The miss search is Claude-only ([#863](https://github.com/haowjy/meridian-cli/issues/863)); path-derived project identity does not survive a project move ([#864](https://github.com/haowjy/meridian-cli/issues/864)); a killed relocation can orphan its OpenCode server ([#867](https://github.com/haowjy/meridian-cli/issues/867)).

## Accepted limits

A verified file can be replaced between preflight and harness open; detection can fail the attempt after input has already reached the model. UUID collision checks are not atomic reservations against external writers. An unsaved Pi session may have no transcript and cannot yet be continued. The [legacy import](legacy-native-import.md) is bounded and uses exact evidence rather than repairing identity on every read.
