# Mars Compiler Cleanup

Tests isolating the Mars cache should set `MARS_CACHE_DIR`; platform cache discovery does not necessarily honor XDG variables. See `src/source/AGENTS.md` in mars-agents. Compiler structure and diagnostic routing belong in `src/compiler/AGENTS.md`.
