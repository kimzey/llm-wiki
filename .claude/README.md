# .claude — Configuration Guide

This folder is Claude Code's "brain" for this vault.
It contains commands (slash commands), skills (automatic behaviors), and settings.

For Thai documentation, see [README-th.md](README-th.md).

---

## File Structure

```
.claude/
├── README.md              ← this file (English)
├── README-th.md           ← Thai version
├── plan.md                ← implementation plan for skill updates
├── settings.json          ← project settings
├── commands/              ← slash commands (/ingest, /query, etc.)
│   ├── ingest.md
│   ├── query.md
│   ├── lint.md
│   ├── status.md
│   ├── research.md
│   ├── clip.md            ← NEW: clip URL → raw/clips/
│   ├── canvas.md          ← NEW: create visual knowledge map
│   ├── base.md            ← NEW: create Obsidian Bases view
│   ├── new-concept.md
│   ├── new-book.md
│   ├── synthesis.md
│   ├── note.md
│   └── update.md
└── skills/                ← behaviors that activate automatically by context
    ├── wiki-ingest/       ← ingest workflow (uses obsidian-markdown + obsidian-cli)
    ├── wiki-query/        ← query workflow (uses obsidian-cli search)
    ├── wiki-lint/         ← lint workflow (uses obsidian-cli backlinks)
    ├── wiki-status/       ← status dashboard (uses obsidian-cli tags)
    ├── wiki-research/     ← research workflow (uses defuddle)
    ├── defuddle/          ← extract clean markdown from URLs
    ├── json-canvas/       ← create/edit .canvas files
    ├── obsidian-bases/    ← create/edit .base database views
    ├── obsidian-cli/      ← interact with Obsidian via CLI
    └── obsidian-markdown/ ← write proper Obsidian Flavored Markdown
```

---

## What are Commands?

Commands are **immediately invokable** using a `/` prefix.
Claude Code loads the matching file from `commands/` and executes the prompt inside it.

```
Type in Claude Code:  /ingest raw/clips/article.md
Claude will:          load commands/ingest.md → follow all steps
```

---

## What are Skills?

Skills are **sets of behaviors Claude activates automatically** when context matches.
No slash required — Claude reads SKILL.md and knows what to do.

e.g. saying "ingest this file" → skill `wiki-ingest` activates automatically.

---

## All Commands

### `/ingest [path]`
Add a new source into the wiki end-to-end.
Accepts a single file or a folder path.
Uses **obsidian-markdown** syntax for all created pages.
Optionally uses **obsidian-cli** to verify files if Obsidian is open.

```
/ingest raw/clips/article.md
/ingest raw/notes/otel/
```

---

### `/query [question]`
Ask a research question against the accumulated wiki.
Uses **obsidian-cli** search for full-text matching if Obsidian is open.

```
/query what is cognitive load theory
/query compare System 1 vs System 2
```

---

### `/research [topic]`
Search the web to fill knowledge gaps in the wiki.
Uses **defuddle** to read web pages cleanly instead of WebFetch.

```
/research distributed tracing
/research              ← runs /lint first to find gaps, then proposes topics
```

---

### `/clip [url]`
Clip a web page to `raw/clips/` using defuddle, then optionally ingest it.

```
/clip https://example.com/article
```

---

### `/lint`
Run a full health check on the wiki.
Finds orphans, broken links, missing concepts, contradictions, stale content, and data gaps.
Optionally uses **obsidian-cli** backlinks for more accurate orphan detection.

> Recommended: run every 1–2 weeks.

---

### `/status`
Show a dashboard of current wiki state: page counts, raw files pending ingestion, recent activity, suggested next actions.
Optionally shows tag stats via **obsidian-cli** if Obsidian is open.

> Start every new session with `/status`.

---

### `/canvas [topic]`
Create a visual knowledge map (`.canvas` file) for a topic, using wiki pages as nodes.

```
/canvas RAG pipeline
/canvas machine learning fundamentals
```

---

### `/base [name]`
Create an Obsidian Bases database view (`.base` file) for wiki content.

```
/base sources     ← table of all sources
/base concepts    ← card gallery of all concepts
/base books       ← book library table
/base all         ← full wiki dashboard
```

---

### `/new-concept [name]`
Create a blank concept page without ingesting a source.

```
/new-concept cognitive load
/new-concept spaced repetition
```

---

### `/new-book [title]`
Create a book summary page.

```
/new-book Thinking Fast and Slow
/new-book Atomic Habits
```

---

### `/synthesis [topic or question]`
Deep analysis across multiple wiki sources, saved as a permanent synthesis page.

```
/synthesis compare learning theories
/synthesis relationship between dopamine and motivation
```

Differs from `/query`: `/synthesis` always creates a permanent page, not just a chat reply.

---

### `/note [text]`
Save a quick note to `raw/notes/` for later ingestion.

```
/note this paper on attention seems to contradict what I learned before
```

---

### `/update [page-path] [what to change]`
Update an existing wiki page without ingesting a new source.
Uses **obsidian-markdown** syntax for edits.

```
/update wiki/concepts/cognitive-load.md add real-world examples
/update wiki/books/atomic-habits.md revise chapter 3 summary
```

---

## All Skills

Skills run **automatically** by context — no slash needed.

| Skill              | Activates when                            | What it does                              |
|--------------------|-------------------------------------------|-------------------------------------------|
| `wiki-ingest`      | mention ingest / drop a file path         | full ingest workflow + obsidian-markdown  |
| `wiki-query`       | ask any research question                 | search wiki → answer → offer synthesis    |
| `wiki-research`    | mention research gap / web search needed  | defuddle URLs → summarize → update wiki   |
| `wiki-lint`        | mention lint / health check               | check 7 points → checklist report         |
| `wiki-status`      | mention status / what's in the wiki       | show current dashboard + tag stats        |
| `defuddle`         | user provides a URL to read               | extract clean markdown from web page      |
| `obsidian-markdown`| creating/editing any .md wiki page        | ensure correct Obsidian syntax            |
| `obsidian-cli`     | search vault / verify files / backlinks   | interact with running Obsidian instance   |
| `json-canvas`      | working with .canvas files                | create/edit visual canvases               |
| `obsidian-bases`   | working with .base files                  | create/edit database views                |

---

## Recommended Workflows

### Start a new session
```
/status
```

### Clip a web article and ingest it
```
/clip https://example.com/article
→ Claude clips to raw/clips/ and asks if you want to ingest
```

### Every time you clip via Obsidian Web Clipper
```
1. Obsidian Web Clipper → save to raw/clips/
2. /ingest raw/clips/filename.md
```

### When you want to know something
```
/query [question]
```
If wiki has the answer → immediate response.
If not → Claude suggests what source to ingest.

### Fill knowledge gaps
```
/research [topic]
```

### Wiki health check (every 1–2 weeks)
```
/lint
```

### Deep analysis
```
/synthesis [topic]
```

### Visualize a knowledge area
```
/canvas [topic]
```

### Browse wiki as database
```
/base sources     or     /base all
```

---

## Technical Notes

- **Commands** are loaded from `commands/*.md` each time you type `/command-name`
- **Skills** are loaded automatically from `skills/*/SKILL.md` by context matching
- **settings.json** sets `effortLevel: high` — Claude applies maximum effort to every operation
- **obsidian-cli** steps are always optional with fallback — they require Obsidian to be open
- Edit any command/skill at any time — open the `.md` file and change the prompt
- Add new commands by creating a new `.md` file in `commands/`
