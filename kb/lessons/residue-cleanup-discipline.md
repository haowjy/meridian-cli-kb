# Residue Cleanup Discipline

When managed output changes shape, removal must recognize what prior versions actually wrote. Verify migrations against real prior-version artifacts; fixtures modeled on current output can miss legacy formats. Run legacy removal before writing replacements, and restrict it to artifacts that need it so it cannot remove new or user-owned entries.

Use the same ownership model for new artifact types. Lock-recorded emissions allow structural removal against what was actually written; see [package management decisions](../concepts/package-management.md).
