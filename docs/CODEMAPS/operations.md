<!-- Generated: 2026-04-14 | Files scanned: 271 | Token estimate: ~700 -->

# Operations — LLM Wiki

## Available Commands (`.claude/commands/` — 14 files)

| Command | Trigger | Core Action |
|---------|---------|-------------|
| `ingest.md` | `/ingest <path>` | Read raw → source page → concept pages → index + log |
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

## Available Skills (`.claude/skills/` — 10 folders)

| Skill | Purpose | Used By |
|-------|---------|---------|
| `wiki-ingest` | Full ingest workflow | /ingest |
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
/ingest raw/notes/<folder>/            # folder → glob all .md
/ingest raw/clips/article.md           # single file

1. Read source in full (images: text first, then read if needed)
2. [obsidian-cli] search vault for duplicate concepts (if Obsidian open)
3. Create wiki/sources/<slug>.md  (obsidian-markdown syntax)
4. For each concept found:
   - wiki/concepts/<slug>.md exists? → merge new info, note contradictions
   - doesn't exist? → create new concept page (obsidian-markdown syntax)
5. Book chapter? → update wiki/books/<slug>.md
6. Append rows to index.md tables
7. Append entry to log.md: ## [YYYY-MM-DD] ingest | <title>
8. [obsidian-cli] verify file visible in Obsidian (if open)
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
