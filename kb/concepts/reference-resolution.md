# File references

Relative `-f` references resolve from the selected task directory, with no per-file fallback: a fallback would make the chosen file depend on what happens to exist on the machine. `kb:` references resolve inside the project authority's configured KB and must stay within its root. The two anchors stay separate, so working in another checkout or worktree doesn't redirect KB lookup ([launch system](../architecture/launch-system.md#authority-root-and-task-directory)). See `src/meridian/lib/launch/reference.py` and `src/meridian/lib/launch/cwd.py`.
