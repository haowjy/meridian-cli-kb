# Session browse

`meridian session browse` is a full-screen picker for re-entering past sessions; the interface is documented in meridian-cli `docs/history.md`.

## Re-entry is decided at activation

The operations layer decides whether a row resumes, forks or is blocked, and rechecks binding and liveness when the user activates it: liveness can change between display and activation, so the listed action is advisory. A live session forks instead of attaching twice, and a chat with no recorded native binding is blocked rather than guessed ([native session binding](native-session-binding.md)). The chat lifetime lock remains the real guard against concurrent attachment. Starting from a row inherits its agent, last-used model and work; running sessions qualify, because this reads recorded context rather than attaching.

## Rows for recognition, transcripts on demand

Rows show what helps judge the cost of re-entry: idle time and a cache-expiry warning. Live and stopped sessions sort together. Staleness is computed from existing idle and stop facts rather than a journal write on every turn; a running session with no idle evidence shows an unknown deadline, not a guessed warning.

Highlighting a row doesn't read its transcript. The transcript opens full-screen on request: a permanently split preview clipped most transcripts at ordinary terminal heights, and a separate preview cache could go stale or show the wrong session. Rendering normalizes tool calls first so the display needs no harness-specific parsing, and cleans content before truncating so the preview limit applies to useful text. Readable versus raw, and truncated versus full, are independent choices.

Search filters before the row limit, so a match beyond the first page doesn't look absent; one query defines matching for both the count and the rows. The lineage view shows primary-session `--fork` and `--from` ancestry.

## Framework: prompt_toolkit, in Python

prompt_toolkit owns the terminal lifecycle and subprocess handoff. In a prototype it restored terminal state on every exit path and handed off in 55 ms; hand-rolled rich and termios left the terminal broken after Ctrl-Z, and Textual took 256 ms and needed worker-shutdown coordination. The picker ignores Ctrl-Z because suspending it isn't safe. This choice is specific to the picker; another surface should weigh its own handoff and testing needs.

The picker stayed in Python because profiling showed startup and state reads, not drawing, dominate latency; a Rust frontend would duplicate the state model or pay for the same Python reads. First paint measured 0.48–0.60 s, above the 200 ms target.
