# Spawn Output Contract

A completed spawn's report is the primary result; accounting is available on demand. The default transcript pointer lets a reader inspect the source conversation when a report is incomplete. `--metadata` adds accounting without hiding the report; `--verbose` serves diagnostics instead. See `docs/commands.md` and `src/meridian/lib/ops/spawn/.context/CONTEXT.md` for output formats.

JSON defaults are curated per command to keep routine coordination small. Machine consumers must treat those outputs as contracts even when the human text remains concise.
