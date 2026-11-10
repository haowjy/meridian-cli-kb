# Pi native sessions

Pi can create a new identity when a requested session file is missing or empty, so Meridian validates the exact recorded source before resume or fork. Pi also discovers existing IDs by scanning headers rather than filenames; collision checks must use the same evidence. Create persistence can be lazy, so an assigned but unwritten session is not yet resumable. See [native session identity](native-session-binding.md) and the source checkout's `src/meridian/lib/harness/pi_identity.py`.

## Exit observation

The session-boundary extension supplies launch-correlated observations; the launcher reads its final record after process teardown, because Pi's shutdown hook can run after the RPC turn ends. A missing or invalid record leaves exit unresolved rather than inventing identity. Pi invalidates an extension context on session replacement; the known stale-context error clears exit proof without poisoning identity. A Pi upgrade that changes the error text needs requalification. See the source checkout's `src/meridian/pi_runtime/README.md`.

Even after preflight, external replacement of the source can make Pi open a different session before the mismatch is detected. Detection fails the attempt but cannot retract input already sent.

## Journal topology and readback

Pi reopens the root-to-leaf ancestry ending at the last physical journal entry. Meridian uses that view for tracked sessions, omitting abandoned branches and marking incomplete graphs partial; an explicit file read may show physical order. See [native transcript reads](native-transcript-reads.md#search-projection).
