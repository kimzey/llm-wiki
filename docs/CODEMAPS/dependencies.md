<!-- Generated: 2026-04-14 | Files scanned: 271 | Token estimate: ~450 -->

# Dependencies — LLM Wiki

## External Tools

| Tool | Role | Required? |
|------|------|-----------|
| **Claude Code** (CLI) | Primary agent — reads, writes, maintains wiki | Yes |
| **Obsidian** | Human-facing reader: graph view, Bases, Web Clipper | Strongly recommended |
| **git** | Version control for entire vault | Yes |
| **Obsidian Web Clipper** | Browser extension → converts articles to markdown | Optional |
| **defuddle** | CLI tool: extract clean markdown from web URLs (no ads/nav) | Optional (skill has fallback) |

## Obsidian Plugins (recommended)

| Plugin | Purpose |
|--------|---------|
| **Bases** (core) | Database views over frontmatter — `.base` files in wiki/bases/ |
| **Canvas** (core) | Visual knowledge maps — `.canvas` files in wiki/canvas/ |
| **Graph View** (core) | Visualize link network — spot hubs and orphans |
| **Dataview** | Query frontmatter → dynamic tables/lists over wiki pages |

## Skills (`.claude/skills/`)

| Skill | Purpose | Required? |
|-------|---------|-----------|
| `wiki-ingest` | Full ingest workflow (source → concepts → index → log) | Core workflow |
| `wiki-research` | Web research + wiki gap-filling | Used by /research |
| `wiki-query` | Query answer synthesis with citations | Used by /query |
| `wiki-lint` | Wiki health check workflow | Used by /lint |
| `wiki-status` | Status dashboard generation | Used by /status |
| `obsidian-markdown` | Syntax guidelines for ALL wiki page writes | Required for all writes |
| `obsidian-cli` | Search/read/verify vault via Obsidian socket API | Optional (Obsidian must be open) |
| `defuddle` | Extract clean markdown from URLs | Optional (fallback: WebFetch) |
| `json-canvas` | Create `.canvas` visual maps in wiki/canvas/ | Used by /canvas |
| `obsidian-bases` | Create `.base` database views in wiki/bases/ | Used by /base |

## Agent Configuration Files

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Master schema: directory layout, page templates, workflows, link hierarchy |
| `.claude/settings.json` | Claude Code project settings |
| `.claude/settings.local.json` | Local overrides (not committed) |
| `.claude/README.md` | English guide to .claude/ structure |
| `.claude/README-th.md` | Thai version |
| `.claude/plan.md` | Current implementation plan (pending tasks) |

## Source Format Support

| Format | Ingestion path |
|--------|---------------|
| `.md` (Obsidian Web Clipper) | `raw/clips/` |
| `.md` (manual notes) | `raw/notes/<topic>/` |
| `.pdf` / book chapter `.md` | `raw/books/` |
| Images (`.png`, `.jpg`) | `raw/assets/` — LLM reads text first, then images |
| Web URL | defuddle → `raw/clips/[slug].md` via /clip |

## Web / Search Services

| Service | Used By |
|---------|---------|
| WebSearch (Claude built-in) | /research, wiki-research skill |
| WebFetch (Claude built-in) | Fallback when defuddle not installed |
| Obsidian socket API | obsidian-cli skill (requires Obsidian open) |
