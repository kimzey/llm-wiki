<!-- Generated: 2026-04-13 | Files scanned: 157 | Token estimate: ~650 -->

# Operations — LLM Wiki

## Available Commands (`.claude/commands/`)

| Command | Trigger | Core Action |
|---|---|---|
| `ingest.md` | `/ingest <path>` | Read raw source → create source page → update/create concept pages → update index + log |
| `query.md` | `/query <question>` | Read index → find relevant pages → synthesize answer → optionally file as synthesis page |
| `research.md` | `/research <topic>` | Check index → WebSearch gaps → summarize findings → optionally update wiki + log |
| `lint.md` | `/lint` | Health check: orphans, contradictions, missing concept pages, stale index rows |
| `status.md` | `/status` | Dashboard: wiki page counts, recent log entries, unprocessed raw folders |
| `synthesis.md` | `/synthesis <title>` | Create wiki/synthesis/<slug>.md from current analysis |
| `update.md` | `/update <page>` | Update an existing wiki page with new info (read first, then merge — never replace) |
| `new-book.md` | `/new-book <title>` | Create wiki/books/<slug>.md skeleton |
| `new-concept.md` | `/new-concept <name>` | Create wiki/concepts/<slug>.md skeleton |
| `note.md` | `/note <text>` | Save quick note to raw/notes/ and optionally ingest immediately |

## Available Skills (`.claude/skills/`)

| Skill | Purpose |
|---|---|
| `wiki-ingest/SKILL.md` | Full ingest workflow implementation |
| `wiki-research/SKILL.md` | Web research + wiki gap-filling |
| `wiki-query/SKILL.md` | Query answer synthesis with citations |
| `wiki-lint/SKILL.md` | Wiki health check workflow |
| `wiki-status/SKILL.md` | Status dashboard generation |

## Workflow: Ingest (most common)

```
/ingest raw/notes/<folder>/            # folder → glob all .md
/ingest raw/clips/article.md           # single file

1. Read source in full (images: text first, then images if needed)
2. Create wiki/sources/<slug>.md
3. For each concept found:
   - wiki/concepts/<slug>.md exists? → merge new info, note contradictions
   - doesn't exist? → create new concept page
4. Book chapter? → update wiki/books/<slug>.md
5. Append rows to index.md tables
6. Append entry to log.md: ## [YYYY-MM-DD] ingest | <title>
```

## Workflow: Query

```
/query <question>

1. Read index.md → identify relevant wiki pages
2. Read those pages in full
3. Synthesize answer with [[wiki page]] citations
4. Offer: "Save as synthesis page?" → if yes → /synthesis <title>
```

## Workflow: Lint

```
/lint

Check for:
- Orphan wiki pages (no inbound links from index or other wiki pages)
- Concepts mentioned in 2+ sources but no dedicated concept page
- Source pages not linked to any concept
- Conflicting claims between pages → flag for "Conflicting Views" section
- index.md rows that are stale or missing
- Topics in raw/ not yet ingested
```

## Log Format (grep-able)

```
## [YYYY-MM-DD] ingest   | <source title>
## [YYYY-MM-DD] query    | <question summary>
## [YYYY-MM-DD] lint     | <summary of findings>
## [YYYY-MM-DD] research | <topic>
## [YYYY-MM-DD] update   | <page name>
## [YYYY-MM-DD] setup    | <description>
```

Quick inspection: `grep "^## \[" log.md | tail -10`
