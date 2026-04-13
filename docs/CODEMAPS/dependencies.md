<!-- Generated: 2026-04-13 | Files scanned: 157 | Token estimate: ~400 -->

# Dependencies — LLM Wiki

## External Tools

| Tool | Role | Required? |
|---|---|---|
| **Claude Code** (CLI) | Primary agent — reads, writes, maintains wiki | Yes |
| **Obsidian** | Human-facing reader/editor — graph view, Dataview, Marp | Strongly recommended |
| **git** | Version control for entire vault | Yes |
| **Obsidian Web Clipper** | Browser extension → converts articles to markdown | Optional |

## Obsidian Plugins (recommended)

| Plugin | Purpose |
|---|---|
| **Dataview** | Query frontmatter → dynamic tables/lists over wiki pages |
| **Marp** | Render wiki content as slide decks |
| **Graph View** (core) | Visualize link network — spot hubs and orphans |

## Optional CLI Tools

| Tool | Purpose | Status |
|---|---|---|
| **qmd** | Local markdown search (BM25 + vector + LLM rerank) — MCP or CLI | Not installed |
| Custom grep scripts | `grep "^## \[" log.md \| tail -10` — log inspection | Built-in |

## Agent Configuration Files

| File | Purpose |
|---|---|
| `CLAUDE.md` | Master schema: directory layout, page templates, workflow rules, link hierarchy |
| `.claude/settings.json` | Claude Code project settings |
| `.claude/settings.local.json` | Local overrides (not committed) |
| `.claude/README.md` | English guide to .claude/ structure |
| `.claude/README-th.md` | Thai version |

## Source Format Support

| Format | Ingestion path |
|---|---|
| `.md` (Obsidian Web Clipper) | `raw/clips/` |
| `.md` (manual notes) | `raw/notes/<topic>/` |
| `.pdf` / book chapter `.md` | `raw/books/` |
| Images (`.png`, `.jpg`) | `raw/assets/` — LLM reads text first, then images separately |

## Web Search (Research workflow)

- **WebSearch tool** (Claude Code built-in) — used by `/research` and `wiki-research` skill
- No external search API key required
- Results optionally filed as new wiki/sources/ pages
