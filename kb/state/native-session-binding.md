# Native session binding

A chat is bound to one native key: harness, native store and native session ID. The store is part of the key because the same ID in another store can name a different conversation. Continue and fork require the exact recorded key and validate the native header; they refuse an incomplete, missing, ambiguous or contradictory binding instead of guessing from timestamps, filenames, prose, primary metadata or ambient roots. A wrong guess can disclose another conversation and bind it to the wrong chat permanently. Contract: the source checkout's `src/meridian/lib/harness/AGENTS.md`. Harness specifics: [Claude](../harnesses/claude.md), [Pi](../harnesses/pi.md).

## Entry evidence is an exact target

Before execution Meridian assigns or obtains an exact target, or verifies the recorded source for resume and fork. The harness's guarantee that it opens that target is the entry evidence; a contradictory first owned signal fails the attempt. Requiring an observed entry instead would leave Pi's interactive TUI untrackable: it has no RPC gate before input, though Pi guarantees it opens the assigned target.

Identity comes only from the assigned target or from owned events: signals correlated to this launch, such as Claude's launch-nonce and child-PID journal ([Claude](../harnesses/claude.md#launch-owned-sessionstart-journal-and-current-chat)). Picking the newest journal in the same directory, parsing model prose, or searching a payload for an ID-shaped value can bind another process's conversation. Discovery and primary metadata can't establish identity either: a detected file may belong to another session, and a spawn mirror can't supply an identity the chat never bound. `--from` may omit an unavailable best-effort pointer; continue and fork refuse.

Tracked launches refuse passthrough flags that would override Meridian-managed native identity, operation, store or persistence: the harness would then open or persist a session other than the one the binding records.

## One source key, one binding rule

The source chat's store and ID travel together through resume and fork, and each adapter translates that key into its harness's mechanism. Separate harness-specific source fields drifted: one fork used an ambient store, another found a same-ID decoy.

Binding fills missing fields and never overwrites a different value. A first owned signal that contradicts a pre-exec assignment fails, as does a fork that reuses its source ID; later observations are diagnostic or identify a separate current chat. Legacy import and repair need exact evidence too ([below](#chats-from-before-binding)).

## Entry is fixed; exit is observed or unresolved

The entry chat stays fixed even when a TUI switches conversations. Pi proves its exit with an owned final quit record; without valid evidence the exit is unresolved. Claude records the last owned live observation as the current chat instead of claiming a verified exit. A spawn reference may follow the accepted current chat; a chat reference keeps naming its own conversation ([session references](../launch/session-references.md)).

A newly observed chat has fresh history and no copied lease or fork lineage; an existing chat keeps its own settings. Native `/resume` into a chat another run holds is recorded and warned about, not prevented.

The child must exit and teardown must be joined before the run concludes, because Pi writes its exit record while shutting down: real Pi showed the runner reading it about 4 s before Pi wrote `quit`. The fix orders the read after teardown, with no polling or timeout: polling for the file can't show teardown finished.

## Settings belong to the conversation

Final binding persists policy on the captured entry generation before native execution. The primary's final snapshot doesn't exist yet when its session scope opens, so recording preview policy there, or waiting until after the run, would give live observations the wrong settings. A chat observed after an in-TUI switch inherits the selected conversation's policy and work. Older generations without a snapshot read only their recorded spawn link; there is no newest-owner fallback.

A `pN` continuation or fork resolves the selected chat's current generation exactly as `cN` does, settings, work and source history included; it never pairs that chat's native identity with the original run's policy.

## Paths move only through `relocate`

`session relocate` changes recorded paths without changing the harness or native ID, and updates harness-specific directory information without moving or truncating transcripts; continue never silently rebinds. It can't repair a wrong ID, and it must know both ends before suggesting a runnable command, because a worktree path can't be inferred from a project root. A Claude missing-file search is limited to the recorded ID and config root, validates headers, and refuses several distinct copies; a found file doesn't become a new identity.

## Chats from before binding

A once-only import binds chats written before native binding existed, and only when recorded facts and a matching native store give an unambiguous key (source checkout `src/meridian/lib/state/.context/CONTEXT.md`, one-time native identity import).

- **Failures defer the import.** A transient I/O failure or damaged source defers the import instead of marking candidates absent: the once-only marker would make the error permanent. One swallowed SQLite error once made 201 of 251 sessions look absent. Older builds may record an ID after the import ran, so doctor and primary background repair revisit the marker's eligible misses with the same exact validation.
- **Old Pi chats need content proof.** Old Pi spawns sometimes recorded no native ID. Automatic recovery needs a unique candidate in the spawn's own directory and matching retained prompt or report content. Uniqueness alone is not proof: a unique candidate with matching cwd and time once selected the wrong session, and bindings are immutable, so recovery is judged by its wrong bindings as well as its yield. A project-local spawn ID can't prove ownership of a user-global Pi directory, and primaries used a shared directory, so they are never bound automatically.
- **Manual repair.** `session repair` is the user-verified path for ambiguous cases. It validates the chosen native source, then rechecks identity and ownership under the session lock. Its force option tolerates a cwd or time mismatch but can't override an existing binding or an ownership conflict.

Identity code needs integrated checks beyond component tests: decoy stores, identity-shaped prose, mismatched file contents, concurrent unrelated sessions and real host lifecycle transitions. Green suites missed these seam defects, and a fake can reproduce event order while missing the host's context invalidation.

## Accepted limits

- A verified file can be replaced between preflight and harness open; detection fails the attempt, possibly after input already reached the model.
- UUID collision checks are not atomic reservations against external writers.
- An unsaved Pi session may have no transcript and can't be continued yet.
- The missing-file search is Claude-only ([#863](https://github.com/haowjy/meridian-cli/issues/863)); path-derived project identity doesn't survive a project move, and that is not planned to change ([#864](https://github.com/haowjy/meridian-cli/issues/864)); a killed relocation can orphan its OpenCode server ([#867](https://github.com/haowjy/meridian-cli/issues/867)).
