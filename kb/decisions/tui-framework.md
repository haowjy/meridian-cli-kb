# Decisions: TUI Framework

Decisions governing interactive full-screen terminal UI in Meridian.

## D-tui-prompt-toolkit: prompt_toolkit for full-screen TUI (2026-08, session-browse design)

**Decision:** Meridian's interactive TUI surfaces use prompt_toolkit's
full-screen `Application` model (`prompt_toolkit>=3.0,<4`). The framework
owns raw mode, input decoding, resize, and terminal restore. Dependency
cost: pygments and wcwidth, both already transitive in the tree (rich pulls
pygments; wcwidth is independent and tiny).

**Scope:** This decision governs any future Meridian TUI feature, not only
session browse. New interactive full-screen surfaces should start from
prompt_toolkit rather than re-litigating framework choice.

**Evidence (three converging lanes, after a prior round-3 choice of
hand-rolled rich Live + termios was reversed on user direction):**

1. **POC bake-off** built the same picker three ways. prompt_toolkit:
   176 LOC, Enter-to-exec echo 55 ms, `stty -a` byte-identical before and
   after across exec, Ctrl-C, Esc, and `q` exits, Ctrl-Z safely ignored by
   default. Hand-rolled rich Live + termios: 203 LOC, Ctrl-Z left the shell
   not sane (`fg` did not repaint, canonical mode leaked into the resumed
   picker). The 27 LOC surplus was entirely raw-mode restore, escape
   parsing, ESC timing, resize polling, and signal handling. Textual:
   176 LOC, Enter-to-exec 256 ms, required explicit worker shutdown
   coordination, hit a teardown race needing a guard.

2. **Picker source study** (ranger, visidata, pgcli, frogmouth, posting,
   pyfzf) quantified the lifecycle code burden. Hand-rolled terminal
   lifecycle: ranger ~740 LOC with 6+ SIGTSTP race fixes over its history;
   visidata ~1642 LOC with resize and escape bugs. Framework-based
   (pgcli/frogmouth/posting): zero terminal lifecycle LOC. prompt_toolkit
   is the only framework with proven subprocess terminal handoff in the wild
   (pgcli `enable_suspend=True`).

3. **Web research** assessed stability and maturity. prompt_toolkit 3.0.x:
   stable API for years, patch-release cadence, tiny dependency set,
   adopted by pgcli/mycli/ptpython. Textual: major-version churn (1.x
   through 8.x since late 2024, one release yanked as breaking),
   `rich>=14.2` lower-bound pin imposed on consumers.

**SIGTSTP policy:** Ctrl-Z is deliberately ignored (prompt_toolkit's
default). The POC proved that hand-rolled SIGTSTP handling broke the
terminal; "nothing happens" removes the failure at zero code. Binding
`suspend_to_background` later is one line if users ask.

**Headless testing:** `create_pipe_input` + `DummyOutput` drive the real
Application without a terminal, replacing most PTY-only test requirements.

**Rejected alternatives:**

| Alternative | Why rejected |
|---|---|
| Hand-rolled rich Live + termios | More code than the framework version, and the surplus is lifecycle/signal code that generates bugs over time. Ctrl-Z broke the terminal in the POC. The source study showed this is an ongoing cost, not a one-time one. |
| Textual | Major-version API churn (1.x through 8.x), `rich>=14.2` pin on our tree, ~200 ms extra teardown latency on the exec path, worker/teardown race in the POC. |
| fzf subprocess (pyfzf pattern) | No programmatic control over preview, footer, or verb behavior. External binary dependency. |
| Curses (hand-rolled) | Same lifecycle burden as rich/termios, without rich's rendering. |

**Revisit when:** prompt_toolkit 3.x reaches end of life, or a new
framework emerges with proven subprocess handoff, headless testing, and
stability comparable to pgcli's dependency on prompt_toolkit. Do not
revisit for aesthetic preferences — the decision is load-bearing on
terminal safety and lifecycle code avoidance.

`chat:c5810` `work:session-browse`

## Related

- [../concepts/session-initiation.md](../concepts/session-initiation.md) — session modes that browse activates
- [../codebase/session-operations.md](../codebase/session-operations.md) — session browse command surface
