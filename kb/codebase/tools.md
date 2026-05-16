# Codebase: Tools

Three utility libraries for working with knowledge bases and markdown documentation. All are internal to Meridian; the `meridian kg`, `meridian mermaid`, and `meridian markdown` commands expose them to users.

## KG Analysis (`lib/kg/`)

Knowledge graph analysis for documentation. Builds a document graph from markdown files, detects broken links, identifies missing backlinks, and finds connected clusters.

**Purpose:** `meridian kg check <path>` catches broken cross-references before they drift undetected. Used as a validation gate on KB changes.

**Key operations:**
- `meridian kg <path>` — quick stats: file count, link count, broken link count
- `meridian kg graph <path>` — link topology with `--depth N` (default 3), `--external` for external URLs, `--exclude PATTERN` (repeatable)
- `meridian kg check <path>` — broken-link gate; exits 1 if any broken links found

**Ignore patterns:** `.kgignore` at the scan root (gitignore-style via `pathspec`, shared via `lib/ignores.py`).

**Output:** `--format json` for machine-readable; text tree by default. Tree markers: `(already shown)` = previously rendered; `(N links hidden)` = truncated by depth; `(not found)` = unresolvable local target.

**When to use:** before committing KB changes, when exploring unfamiliar documentation structure, after structural refactoring.

## Mermaid Validation (`lib/mermaid/`)

Validates Mermaid diagram content in markdown files, `.mmd` files, and `.mermaid` files.

**Purpose:** catches diagram syntax errors before they render as broken diagrams. Run after editing any diagram.

**Two parsers:**
- **Python heuristic parser** (default) — fast, offline, catches common failures: unknown diagram types, unclosed directives, mismatched block boundaries
- **JS strict parser** (optional) — invokes Node.js subprocess; precise but requires Node.js available

**Entry point:** `meridian mermaid check <path>`

- Exit 0: clean
- Exit 1: validation errors
- Exit 2: target path not found

**Ignore patterns:** `.mermaidignore` at the scan root.

**When to use:** after editing diagrams; in review of docs containing Mermaid; CI gate for documentation.

## Markdown Extraction (`lib/markdown/`)

Shared markdown parsing layer (wraps `markdown-it-py`). Extracts structured content from markdown documents: headings, fenced code blocks, links, wikilinks.

**Purpose:** provides a common extraction layer used by both `lib/kg/` (for link analysis) and `lib/mermaid/` (for fenced Mermaid block discovery). Not exposed as a user-facing command — internal shared library.

**What it extracts:**
- Headings (level + text)
- Fenced code blocks (language + content)
- Markdown links (text + target)
- Wikilinks (`[[page]]` style)

## Shared Ignore Patterns (`lib/ignores.py`)

Both `lib/kg/` and `lib/mermaid/` use `lib/ignores.py` for gitignore-style pattern matching via `pathspec`. Load patterns from `.kgignore` or `.mermaidignore` at the scan root. Persistent exclusions belong in these files rather than as repeated `--exclude` flags.

## Validation Flow (Minimal)

```bash
meridian kg check path/to/kb/     # 1. broken link check
meridian mermaid check path/to/kb/ # 2. diagram validation
# Fix issues, re-run until exit codes clean
```

Run before every KB commit. See [../architecture/system-overview.md](../architecture/system-overview.md) for where these tools fit in the overall codebase.
