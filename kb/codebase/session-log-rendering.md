# Session Log: Read Pipeline Design

Design decisions for the `meridian session log` rendering pipeline, introduced in PR #281. Covers ToolCall normalization, clean markdown output format, the `--raw`/`--no-truncate` flag design, and content pipeline ordering.

## Source Files

| Module | Role |
|---|---|
| `src/meridian/lib/harness/transcript.py` | Transcript parsing, `ToolCall` normalization, `TranscriptMessage` types |
| `src/meridian/lib/ops/session_log_render.py` | Output rendering: clean vs raw format, tool collapsing, content cleaning, truncation |

---

## Decision 1: Structured ToolCall as Single Source of Truth

### What

A `ToolCall` NamedTuple lives in `harness/transcript.py` and represents a harness-agnostic tool invocation:

```python
class ToolCall(NamedTuple):
    name: str   # canonical lowercase: bash, read, write, edit, grep, stdin, ...
    body: str   # meaningful payload: command string, file path, pattern, etc.
```

`TranscriptMessage` carries `tool_call: ToolCall | None` and `is_tool_result: bool`. Normalization happens once at parse time in `_normalize_tool()`.

### Normalization table

| Raw harness name(s) | Canonical `name` | Notes |
|---|---|---|
| `bash` | `bash` | Direct |
| `exec_command`, `shell`, `terminal`, `run_command` | `bash` | Codex variants; extracts `cmd` from JSON body |
| `write_stdin` | `stdin` | Body always empty — interaction marker only |
| `read`, `write`, `edit`, `grep` | lowercase as-is | Standard file/search tools |
| anything else | lowercased name or `"tool"` | Fallback |

### Why

The render layer (`session_log_render.py`) works from `ToolCall.name` without knowing which harness produced the event. Tool collapsing looks identical across Claude, Codex, OpenCode, and Cursor sessions because normalization happened earlier at parse time — not scattered across rendering code.

Without the typed field, the renderer would parse content strings to detect tools, producing fragile harness-specific logic in the display layer. The `ToolCall` type makes the tool identity explicit and testable.

The typed field flows up through `AbsoluteTranscriptMessage` → `SessionLogEntryMessage` to the render layer.

---

## Decision 2: Clean Markdown Output as Default

### What

`meridian session log` defaults to **clean mode**: markdown-style output suitable for both human reading and agent consumption.

**Clean mode format:**
- Entry separator: `---`
- Entry header: `**Role** [N]` (e.g., `**Assistant** [3]`)
- Tool invocations collapsed to one-liners (see tool collapsing below)
- Navigation footer with `Previous:` / `Next:` commands
- Hint appended when tool output was collapsed: `Use --no-truncate to expand tool outputs`

**Raw mode** (`--raw` / verbosity > 0) emits debug-oriented headers: `--- N [segment S · messages M-M] [role] ---`. It preserves raw content without harness XML stripping.

### Why

Session log entries were previously opaque walls of text containing harness XML wrappers (`<bash-stdout>`, `<command-name>`, `<system-reminder>`, etc.). Clean mode strips these wrappers via `clean_content()` and collapses tool round-trips.

The key insight for readability: tool invocations and their results are typically structural noise when reading conversation history. A collapsed one-liner (`  $ git status`, `  Read src/foo.py`) conveys what happened without dumping the full output. Agents navigating their own conversation history get readable flow rather than raw event dumps.

**Tool collapsing rules in clean+truncate mode:**
- `bash` tool → `  $ <command>`; if the paired `tool_result` shows a non-zero exit code, appends `  (failed: exit N)` or `  (failed)`
- `read`, `write`, `edit`, `grep` → `  Read <path>`, `  Write <path>`, etc.
- `stdin` → `  (stdin)`
- other tools → `  <name>: <detail>` or `  <name>`
- `tool_result` without a paired tool call → `  (tool output): <first line>`

In `--no-truncate` mode, tools expand inline with indented output rather than collapsing.

---

## Decision 3: Orthogonal `--raw` × `--no-truncate` Flags

### What

Two independent concerns, two flags that compose freely:

| Flags | Output format | Content | Tool handling |
|---|---|---|---|
| *(none)* | Clean markdown | Preview (80 lines / 8000 chars) | Collapsed one-liners |
| `--no-truncate` | Clean markdown | Full content | Expanded inline |
| `--raw` | Debug headers | Preview | Raw content |
| `--raw --no-truncate` | Debug headers | Full content | Raw content |

### Why

"Clean vs raw" and "truncated vs full" are independent concerns. An agent that wants readable format but full content (e.g., to read a long assistant response) uses `--no-truncate` without `--raw`. An agent debugging harness events uses `--raw`. The flags compose without surprising interactions.

The alternative — a single flag like `--full` that meant both raw format and full content — would force agents to receive debug headers when they only needed more content.

**Note:** Tool collapsing is a clean-mode feature. In raw mode, tool messages appear as their raw content strings regardless of truncation setting.

---

## Decision 4: Content Pipeline Order — Clean → Truncate → Render

### What

The processing pipeline for entry content:

```
TranscriptMessage (typed, with ToolCall fields)
  ↓
clean_content(text)           # strip harness XML wrappers, normalize whitespace
  ↓
_truncate_preview(content)    # limit to 80 lines / 8000 chars (clean+truncate mode)
  ↓
render_entry(entry)           # format headers, collapse/expand tools
  ↓
render_session_log(...)       # assemble output with navigation and hints
```

`clean_content()` handles: `<bash-stdout>/<bash-stderr>` blocks, `<bash-input>` → `$ cmd`, `<command-name/args/message>` → rendered command, `<system-reminder>`, `<local-command-caveat>`, `<system_notification>` → `[notification: ...]`, ANSI escape stripping, and triple-newline normalization.

### Why this order matters

**Structured data flows through the entire pipeline.** The full `TranscriptMessage` objects with typed `ToolCall` fields are available at render time. This enables:

1. **`clean_content` before truncation** — XML wrappers are stripped before the character/line limit is applied, so the 8000-char preview window covers useful content rather than harness XML.
2. **Tool collapsing uses typed fields** — `_render_collapsed_tools()` reads `message.tool_call.name` directly, not by parsing content strings. The normalization result from parse time is still intact.
3. **Truncation operates on cleaned content** — the preview truncation marker (`...[truncated: ...]`) appears in human-readable context, not in the middle of an XML tag.

**The anti-pattern avoided:** Truncating in the command layer (`session_log.py`) before handing off to the render layer would force the renderer to work with already-truncated strings. Typed `ToolCall` fields would be lost, tool collapsing would fall back to string parsing, and `clean_content` would operate on already-truncated raw XML.

---

## Related Pages

- [session-operations.md](session-operations.md) — User-facing `session log` command: navigation flags, segment model, common patterns
- [harness-adapters.md](harness-adapters.md) — Per-harness transcript format differences; how provider-specific events map to the `TranscriptMessage` type
- [../concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — Policy/mechanism split that motivates harness-agnostic normalization
- [../concepts/spawn-output-contract.md](../concepts/spawn-output-contract.md) — Progressive disclosure: spawn report → session log → no-truncate
