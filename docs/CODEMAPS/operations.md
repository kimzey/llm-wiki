<!-- Generated: 2026-04-14 | Files scanned: 301 | Token estimate: ~800 -->

# Operations — LLM Wiki

## Available Commands (`.claude/commands/` — 14 files)

| Command | Trigger | Core Action |
|---------|---------|-------------|
| `ingest.md` | `/ingest <path>` | Categorize raw → source page → concept pages → index + log |
| `query.md` | `/query <question>` | index → relevant pages → synthesize → optionally save synthesis |
| `research.md` | `/research <topic>` | WebSearch gaps → summarize → optionally update wiki + log |
| `lint.md` | `/lint` | Health check: orphans, contradictions, missing pages, stale index |
| `status.md` | `/status` | Dashboard: page counts, recent log, unprocessed raw folders |
| `synthesis.md` | `/synthesis <title>` | Create wiki/synthesis/<slug>.md from current analysis |
| `update.md` | `/update <page>` | Update existing wiki page — read first, merge, never replace |
| `new-book.md` | `/new-book <title>` | Create wiki/books/<slug>.md skeleton |
| `new-concept.md` | `/new-concept <name>` | Create wiki/concepts/<slug>.md skeleton |
| `note.md` | `/note <text>` | Save quick note to raw/notes/ + optionally ingest |
| `clip.md` | `/clip <url>` | defuddle URL → raw/clips/<slug>.md → optionally ingest |
| `canvas.md` | `/canvas <title>` | Create wiki/canvas/<slug>.canvas visual map |
| `base.md` | `/base <title>` | Create wiki/bases/<slug>.base database view |
| `init.md` | `/init` | Initialize a new LLM Wiki vault from scratch |

## Available Skills (`.claude/skills/` — 11 folders)

| Skill | Purpose | Used By |
|-------|---------|---------|
| `wiki-ingest` | Full ingest workflow with auto-categorization | /ingest |
| `wiki-research` | Web research + wiki gap-filling | /research |
| `wiki-query` | Query synthesis with citations | /query |
| `wiki-lint` | Wiki health check | /lint |
| `wiki-status` | Status dashboard | /status |
| `obsidian-markdown` | Syntax for all wiki page writes | Every write operation |
| `obsidian-cli` | Vault search/read via Obsidian API | /ingest, /query (when Obsidian open) |
| `defuddle` | Clean markdown from web URLs | /clip, /research |
| `json-canvas` | Create .canvas visual maps | /canvas |
| `obsidian-bases` | Create .base database views | /base |

## Workflow: Ingest (most common)

```
/ingest                           # empty → process raw/inbox/*
/ingest raw/notes/<folder>/        # folder → glob all .md
/ingest raw/clips/article.md       # single file

=== Auto-Categorization Phase ===
For each file in input:
  1. Read full file content
  2. Determine type: clips/ (has url:), books/ (has author:), notes/ (no frontmatter), assets/ (images)
  3. Determine subfolder by content analysis (AI-categorized)
  4. Move: raw/<type>/<category>/<file>

=== Ingestion Phase ===
1. Read source in full (images: text first, then read if needed)
2. Determine wiki category from raw path & content
   - Check raw/notes/<category>/ first
   - Or analyze content for category match
   - Create new categories as needed (flexible)
3. [obsidian-cli] search vault for duplicate concepts (if Obsidian open)
4. Create wiki/sources/<category>/<slug>.md (obsidian-markdown syntax)
5. For each concept found:
   - wiki/concepts/<slug>.md exists? → merge new info, note contradictions
   - doesn't exist? → create new concept page (obsidian-markdown syntax)
6. Book chapter? → update wiki/books/<slug>.md
7. Append rows to index.md tables (all relevant sections)
8. Append entry to log.md: ## [YYYY-MM-DD] ingest | <title>
9. [obsidian-cli] verify file visible in Obsidian (if open)

=== Output ===
Report: categorized, moved, created, updated files
```

## Workflow: Clip + Ingest

```
/clip <url>

1. defuddle parse <url> --md  (fallback: WebFetch)
2. Extract: title, description, domain from metadata
3. Write to raw/clips/<slug>.md with frontmatter
4. Ask: "ต้องการ ingest เข้า wiki เลยไหม?"
→ if yes → run /ingest workflow on that file
```

## Workflow: Research

```
/research <topic>

1. Check index.md for existing coverage
2. WebSearch authoritative sources
3. defuddle each URL for clean markdown
4. Summarize findings (2–5 points), note conflicts
5. Ask: "อัปเดต wiki ด้วยไหม?" → if yes → update concepts + log
```

## Workflow: Canvas / Base

```
/canvas <title>          /base <title>
→ uses json-canvas skill  → uses obsidian-bases skill
→ writes wiki/canvas/     → writes wiki/bases/
→ updates index.md        → updates index.md
```

## Category System (9 active categories)

### Fixed Categories
- `ai-context/` — AI coding, .claude structure, agents
- `frameworks/` — RAG frameworks (Arona, Haystack, OpenRAG)
- `infrastructure/` — DevOps, networking, servers
- `langchain/` — LangChain ecosystem
- `observability/` — OTel, tracing, monitoring
- `rag/` — RAG theory, LlamaIndex, vector DBs
- `sellsuki/` — Sellsuki-specific content

### Flexible Categories (created as needed)
- `javascript/` — JavaScript runtime, TypeScript, Bun (NEW)
- `policy/` — HR policies, organizational docs (NEW)

### Auto-Categorization Rules
- **From raw path**: `raw/notes/<category>/` → `wiki/sources/<category>/`
- **From content**: AI analyzes title, topics, domain
- **Fallback**: Ask user for confirmation if unclear

## Log Format (grep-able)

```
## [YYYY-MM-DD] ingest   | <source title>
## [YYYY-MM-DD] query    | <question summary>
## [YYYY-MM-DD] lint     | <summary of findings>
## [YYYY-MM-DD] research | <topic>
## [YYYY-MM-DD] update   | <page name>
## [YYYY-MM-DD] clip     | <url or title>
## [YYYY-MM-DD] setup    | <description>
```

Quick inspection: `grep "^## \[" log.md | tail -10`

## Checkpoint System

Checkpoints stored in `.claude/checkpoints.log`:
- Format: `YYYY-MM-DD-HH:MM | <name> | <git-sha>`
- Use `/checkpoint create <name>` after major work
- Use `/checkpoint verify <name>` to compare states
- Keeps last 5 checkpoints automatically
