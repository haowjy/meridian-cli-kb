# Legacy native session import

## Legacy import

A once-only import binds legacy chats only when recorded facts identify an exact native source. Transient I/O failures defer the import instead of marking candidates absent: a once-only marker would otherwise make the error permanent. One swallowed SQLite error made 201 of 251 sessions look absent. See [legacy import decisions](../decisions/legacy-native-import.md) and the source checkout's `src/meridian/lib/ops/.context/CONTEXT.md`.

## Late binding after the marker

Older builds may record an ID after the import; doctor and primary background repair revisit the marker's eligible misses with exact validation.

## Legacy Pi recovery

Uniqueness is not proof of identity. Automatic recovery requires matching retained content as well as a unique candidate; shared-root primary sessions require manual repair. See [the recovery decision](../decisions/legacy-native-import.md#old-pi-chats-that-067-never-bound-content-proven-recovery-manual-repair).

## Session repair

Manual repair validates the chosen native source, then rechecks identity and ownership under the session lock. A force option permits a cwd or time mismatch but cannot override an existing binding or ownership conflict.

