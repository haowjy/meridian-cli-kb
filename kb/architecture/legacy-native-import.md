# Legacy native session import

A once-only import binds chats written before native binding existed, and only when recorded facts and a matching native store give an unambiguous key. Contract: the source checkout's `src/meridian/lib/state/.context/CONTEXT.md` (one-time native identity import).

## Failures defer the import

A transient I/O failure or damaged source defers the import instead of marking candidates absent: the once-only marker would make the error permanent. One swallowed SQLite error once made 201 of 251 sessions look absent. Older builds may record an ID after the import ran, so doctor and primary background repair revisit the marker's eligible misses with the same exact validation.

## Old Pi chats need content proof

Old Pi spawns sometimes recorded no native ID. Automatic recovery needs a unique candidate in the spawn's own directory and matching retained prompt or report content. Uniqueness alone is not proof: a unique candidate with matching cwd and time once selected the wrong session, and bindings are immutable. A project-local spawn ID can't prove ownership of a user-global Pi directory, and primaries used a shared directory, so they are never bound automatically.

## Manual repair

`session repair` is the user-verified path for ambiguous cases. It validates the chosen native source, then rechecks identity and ownership under the session lock. Its force option tolerates a cwd or time mismatch but can't override an existing binding or an ownership conflict.
