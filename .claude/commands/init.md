---
description: "Initialize a new LLM Wiki vault in the current directory. Usage: /init [project name]"
---

Initialize a new LLM Wiki vault in the current working directory.

Project name: $ARGUMENTS (use "Personal Wiki" if empty)

Follow these steps exactly — do not skip any step:

---

## Step 1 — Create directory structure

Create all directories (use `mkdir -p` or Write tool):

```
raw/clips/
raw/books/
raw/notes/
raw/assets/
wiki/concepts/
wiki/books/
wiki/sources/
wiki/sources/ai-context/
wiki/sources/frameworks/
wiki/sources/infrastructure/
wiki/sources/langchain/
wiki/sources/observability/
wiki/sources/policy/
wiki/sources/rag/
wiki/sources/sellsuki/
wiki/synthesis/
wiki/canvas/
wiki/bases/
docs/CODEMAPS/
.reports/
.claude/commands/
.claude/skills/wiki-ingest/
.claude/skills/wiki-query/
.claude/skills/wiki-lint/
.claude/skills/wiki-status/
.claude/skills/wiki-research/
.claude/skills/defuddle/
.claude/skills/json-canvas/
.claude/skills/obsidian-bases/
.claude/skills/obsidian-cli/
.claude/skills/obsidian-markdown/
```

---

## Step 2 — Create CLAUDE.md

Write `CLAUDE.md` with the full LLM Wiki schema below.
Replace `[PROJECT NAME]` with the value from $ARGUMENTS.

```markdown
# [PROJECT NAME] — Schema & Operating Instructions

This is the master schema for this personal research knowledge base.
The LLM (Claude Code) owns and maintains the `wiki/` directory entirely.
The human curates sources, asks questions, and directs the analysis.

---

## Directory Layout

\```
vault/
├── CLAUDE.md          ← this file (schema + instructions)
├── index.md           ← content catalog (LLM updates on every ingest)
├── log.md             ← append-only operation log
├── raw/               ← source documents — NEVER modify these
│   ├── clips/         ← web articles clipped via Obsidian Web Clipper
│   ├── books/         ← book files, PDFs, or chapter notes
│   ├── notes/         ← manual notes typed by the user
│   └── assets/        ← locally downloaded images
└── wiki/              ← LLM-generated wiki (read by human, written by LLM)
    ├── concepts/      ← concept pages (principles, theories, ideas)
    ├── books/         ← book summary pages
    ├── sources/       ← per-source summary pages (categorized by topic)
    │   ├── ai-context/
    │   ├── frameworks/
    │   ├── infrastructure/
    │   ├── langchain/
    │   ├── observability/
    │   ├── policy/
    │   ├── rag/
    │   └── sellsuki/
    └── synthesis/     ← analyses, comparisons, cross-cutting essays
\```

---

## Language Rules

- **Wiki content**: Thai as the primary language. Use English for technical terms,
  proper nouns, titles, and citations where Thai translation would be awkward.
- **Headings in wiki pages**: Thai preferred, English acceptable for technical headings.
- **Code/data**: English only.
- **Frontmatter fields**: English keys, Thai/English values.
- **This file (CLAUDE.md)**: English only — instructions for the LLM.
- **`.claude/` config files**: English only.

---

## Page Frontmatter

Every wiki page must start with YAML frontmatter:

\```yaml
---
title: "Page title"
type: concept | book | source | synthesis
tags: [tag1, tag2]
sources: [filename1.md, filename2.md]
related: [wiki/concepts/foo.md]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
\```

---

## Page Types & Templates

### 1. Concept Page (`wiki/concepts/`)

\```markdown
---
title: "Concept name"
type: concept
tags: []
sources: []
related: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

## สรุปสั้น
(1–3 sentences)

## อธิบาย
(deeper explanation)

## ประเด็นสำคัญ
- bullet points

## ตัวอย่าง / กรณีศึกษา

## ความสัมพันธ์กับ concept อื่น

## แหล่งที่มา
\```

### 2. Book Summary Page (`wiki/books/`)

\```markdown
---
title: "Book title"
type: book
author: ""
year:
tags: []
sources: []
related: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

## ภาพรวม

## ประเด็นหลัก

## สรุปแต่ละบท
### Chapter 1 — Title
...

## Concepts ที่เกี่ยวข้อง

## คำคมที่น่าจดจำ
> quote (p.XX)

## ความคิดเห็น / วิจารณ์
\```

### 3. Source Summary Page (`wiki/sources/`)

\```markdown
---
title: "Article title"
type: source
source_file: raw/clips/filename.md
url: ""
published: YYYY-MM-DD
tags: []
related: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

> **Full source**: [[../../../raw/clips/filename.md|Original file]]

## สรุป

## ประเด็นสำคัญ
-

## ข้อมูล / หลักฐาน ที่น่าสนใจ

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

## Concepts ที่เกี่ยวข้อง
\```

**Note**: Source pages are organized into category subdirectories (ai-context/, frameworks/, infrastructure/, langchain/, observability/, policy/, rag/, sellsuki/). The Full source link uses `../../../raw/` because files are one level deeper.

### 4. Synthesis Page (`wiki/synthesis/`)

\```markdown
---
title: "Analysis title"
type: synthesis
tags: []
sources: []
related: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

## คำถาม / โจทย์

## สรุปคำตอบ

## การวิเคราะห์

## ข้อสรุป

## หัวข้อที่ควรศึกษาต่อ
\```

---

## Workflows

### A. Ingest (add a new source)

1. Categorize and move files from `raw/inbox/` to appropriate folders (clips/books/notes/assets).
2. Read the source file in full. If folder → Glob all .md files, process sequentially.
3. Determine category → Create `wiki/sources/{category}/<slug>.md`.
4. For each concept: update existing page OR create new one. Note contradictions.
5. If book chapter → update `wiki/books/<slug>.md`.
6. Update `index.md`.
7. Append to `log.md`.

### B. Query

1. Read `index.md` → find relevant pages.
2. Read those pages in full.
3. Synthesize answer with citations.
4. Offer: "Save as synthesis page?"

### C. Research

1. Check `index.md` for existing coverage.
2. WebSearch to fill gaps.
3. Summarize findings (2–5 points), note conflicts.
4. Offer: "Update wiki?"
5. Append to `log.md`.

### D. Lint

Check for:
- [ ] Orphan pages (no inbound links)
- [ ] Conflicting information between pages
- [ ] Concepts in 2+ pages but no dedicated page
- [ ] Source pages not linked to any concept
- [ ] Stale or missing index.md entries
- [ ] Topics worth researching further

---

## index.md Format

\```markdown
# Index

## Sources (`wiki/sources/`)
| Page | Summary | Source file | Date |
|------|---------|-------------|------|
| [[wiki/sources/category/slug\|Title]] | One-line summary | [[wiki/sources/category/slug]] | YYYY-MM-DD |

## Concepts (`wiki/concepts/`)
| Page | Summary | Tags |
|------|---------|------|
| [[wiki/concepts/slug\|Title]] | One-line summary | tag1, tag2 |

## Books (`wiki/books/`)
| Page | Author | Tags |
|------|--------|------|
| [[wiki/books/slug\|Title]] | Author name | tag1, tag2 |

## Synthesis (`wiki/synthesis/`)
| Page | Summary | Date |
|------|---------|------|
| [[wiki/synthesis/slug\|Title]] | One-line summary | YYYY-MM-DD |
\```

**Rules:**
- "Page" column = Obsidian link to a `wiki/` page only — never `raw/`
- "Source file" column = `[[wiki/sources/category/slug]]` (includes category)
- Navigation to raw: index → wiki/sources/category/slug → "Full source" → raw file

---

## log.md Format

\```
## [YYYY-MM-DD] ingest   | source title
## [YYYY-MM-DD] query    | question summary
## [YYYY-MM-DD] lint     | summary of findings
## [YYYY-MM-DD] update   | page name updated
## [YYYY-MM-DD] research | topic
## [YYYY-MM-DD] setup    | description
\```

---

## Link Hierarchy Rules

```
index.md
    ↓ links to
wiki/sources/*.md   wiki/concepts/*.md   wiki/books/*.md   wiki/synthesis/*.md
    ↓ links to (sources only)
raw/clips/*.md   raw/notes/*.md   raw/books/*.md
```

- Only `wiki/sources/` pages may link to `raw/`
- index.md must never contain `[[raw/...]]` links
- All inter-wiki links use Obsidian `[[...]]` syntax

---

## Conventions

- File names: lowercase, hyphens, no spaces
- Dates: `YYYY-MM-DD` always
- Never delete raw source files
- Never overwrite a wiki page without reading it first — always merge
- Contradictions → add **"Conflicting Views"** section, never silently overwrite
- Synthesis-worthy query answers → file as `wiki/synthesis/` pages
```

---

## Step 3 — Create index.md

```markdown
# Index

วันที่อัปเดตล่าสุด: [TODAY DATE]

---

## Sources (`wiki/sources/`)

| Page | Summary | Source file | Date |
|------|---------|-------------|------|
| _(empty)_ | | | |

---

## Concepts (`wiki/concepts/`)

| Page | Summary | Tags |
|------|---------|------|
| _(empty)_ | | |

---

## Books (`wiki/books/`)

| Page | Author | Tags |
|------|--------|------|
| _(empty)_ | | |

---

## Synthesis (`wiki/synthesis/`)

| Page | Summary | Date |
|------|---------|------|
| _(empty)_ | | |
```

---

## Step 4 — Create log.md

```markdown
# Log

บันทึกการดำเนินการทั้งหมดใน wiki นี้ — เรียงตามเวลา ไม่ลบ ไม่แก้ไข

---

## [TODAY DATE] setup | initialized [PROJECT NAME]

- Created directory structure (raw/, wiki/, .claude/)
- Created CLAUDE.md schema
- Created index.md and log.md
- Ready for first ingest
```

---

## Step 5 — Create .gitignore

```
.DS_Store
*.local.json
.obsidian/workspace*
.obsidian/cache
```

---

## Step 6 — Create .claude/settings.json

```json
{
  "effortLevel": "high"
}
```

---

## Step 7 — Create .claude/commands/ stubs

Copy or recreate these command files (each follows the same format as this file):
- `ingest.md` — `/ingest [path]`
- `query.md` — `/query [question]`
- `research.md` — `/research [topic]`
- `lint.md` — `/lint`
- `status.md` — `/status`
- `synthesis.md` — `/synthesis [topic]`
- `new-concept.md` — `/new-concept [name]`
- `new-book.md` — `/new-book [title]`
- `note.md` — `/note [text]`
- `update.md` — `/update [page] [what]`
- `clip.md` — `/clip [url]`
- `canvas.md` — `/canvas [topic]`
- `base.md` — `/base [name]`
- `init.md` — `/init [name]` (this file)

---

## Step 8 — Create .claude/skills/ stubs

Create a `SKILL.md` file in each skill folder explaining when and how it activates:
- `wiki-ingest/SKILL.md`
- `wiki-query/SKILL.md`
- `wiki-lint/SKILL.md`
- `wiki-status/SKILL.md`
- `wiki-research/SKILL.md`

---

## Step 9 — Initialize git (if not already a repo)

Run: `git init && git add . && git commit -m "init: initialize LLM Wiki vault"`

Only if the directory is not already a git repository.

---

## Step 10 — Report

After completing all steps, show a summary:

```
✓ [PROJECT NAME] initialized

Directory structure:
  raw/clips/   raw/books/   raw/notes/   raw/assets/
  wiki/concepts/   wiki/books/   wiki/sources/ (with category subdirs)   wiki/synthesis/
  .claude/commands/   .claude/skills/

Files created:
  CLAUDE.md   index.md   log.md   .gitignore
  .claude/settings.json
  .claude/commands/ (10 commands)
  .claude/skills/ (5 skills)

Next steps:
  1. Drop source files into raw/notes/ or raw/clips/
  2. Run /ingest [path] to start building the wiki
  3. Run /status to see current state at any time
```
