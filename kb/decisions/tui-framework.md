# Session-Browse TUI Framework

## Prompt_toolkit for full-screen TUI

Session browse uses prompt_toolkit for terminal lifecycle and subprocess handoff. In a picker prototype, it restored terminal state across exit paths and handed off in 55 ms; hand-rolled rich and termios left the terminal broken on Ctrl-Z, and Textual took 256 ms and needed worker shutdown coordination. The picker ignores Ctrl-Z to avoid unsafe suspension.

This choice is specific to session browse. Reconsider it for another surface against that surface's terminal handoff and testing needs.
