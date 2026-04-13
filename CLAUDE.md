# LLM Wiki — Schema & Operating Instructions

This is the master schema for this personal research knowledge base.
The LLM (Claude Code) owns and maintains the `wiki/` directory entirely.
The human curates sources, asks questions, and directs the analysis.

---

## Directory Layout

```
local-valut/
├── CLAUDE.md          ← this file (schema + instructions)
├── index.md           ← content catalog (LLM updates on every ingest)
├── log.md             ← append-only operation log
├── raw/               ← source documents — NEVER modify these
│   ├── inbox/         ← drop new files here (auto-categorized on /ingest)
│   ├── clips/         ← web articles clipped via Obsidian Web Clipper or /clip
│   ├── books/         ← book files, PDFs, or chapter notes
│   ├── notes/         ← manual notes typed by the user
│   └── assets/        ← locally downloaded images
└── wiki/              ← LLM-generated wiki (read by human, written by LLM)
    ├── concepts/      ← concept pages (principles, theories, ideas)
    ├── books/         ← book summary pages
    ├── sources/       ← per-source summary pages
    ├── synthesis/     ← analyses, comparisons, cross-cutting essays
    ├── canvas/        ← visual knowledge maps (.canvas files)
    └── bases/         ← Obsidian Bases database views (.base files)
```

**Important workflow change**: When adding new source files, place them in `raw/inbox/`. Running `/ingest` will automatically categorize and move them to the appropriate folder (clips/books/notes/assets) before processing.

---

## Language Rules

- **Wiki content**: Thai as the primary language. Use English for technical terms,
  proper nouns, titles, and citations where Thai translation would be awkward.
- **Headings in wiki pages**: Thai preferred, English acceptable for technical headings.
- **Code/data**: English only.
- **Frontmatter fields**: English keys, Thai/English values.
- **This file (CLAUDE.md)**: English only — instructions for the LLM.
- **`.claude/` config files**: English only — commands, skills, README.

---

## Page Frontmatter

Every wiki page must start with YAML frontmatter:

```yaml
---
title: "Page title"
type: concept | book | source | synthesis
tags: [tag1, tag2]
sources: [filename1.md, filename2.md]   # raw sources this page draws from
related: [wiki/concepts/foo.md]          # cross-links to other wiki pages
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

---

## Page Types & Templates

### 1. Concept Page (`wiki/concepts/`)
For ideas, theories, frameworks, terms that appear across multiple sources.

```markdown
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
(1–3 sentences — what is this concept?)

## อธิบาย
(deeper explanation)

## ประเด็นสำคัญ
- bullet points

## ตัวอย่าง / กรณีศึกษา

## ความสัมพันธ์กับ concept อื่น
(link to related pages)

## แหล่งที่มา
(cite source names used)
```

### 2. Book Summary Page (`wiki/books/`)
For books — one page per book.

```markdown
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
(what the book is about — 1 paragraph)

## ประเด็นหลัก
(bullet points of key arguments/ideas)

## สรุปแต่ละบท
### Chapter 1 — Chapter title
...

## Concepts ที่เกี่ยวข้อง
(link to wiki/concepts/)

## คำคมที่น่าจดจำ
> quote (p.XX)

## ความคิดเห็น / วิจารณ์
```

### 3. Source Summary Page (`wiki/sources/`)
For web articles, papers, or any clipped/raw source. One page per source file.

```markdown
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

> **Full source**: [[../../raw/clips/filename.md|Original file]]
>
> Or for external URL: **Full source**: [Article title](url)

## สรุป
(brief — what does this article cover?)

## ประเด็นสำคัญ
- 

## ข้อมูล / หลักฐาน ที่น่าสนใจ

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

## Concepts ที่เกี่ยวข้อง
(link to wiki/concepts/)
```

### 4. Synthesis Page (`wiki/synthesis/`)
For analyses, comparisons, or answers to big questions — filed as permanent wiki pages.

```markdown
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
```

---

## Available Skills & Tools

The following skills are available to Claude for use in all workflows:

| Skill | When to use |
|-------|-------------|
| `obsidian-markdown` | Creating or editing ANY `.md` wiki page — ensures correct wikilinks, callouts, frontmatter |
| `obsidian-cli` | Search vault, verify files, check backlinks — requires Obsidian to be open; always has fallback |
| `defuddle` | Reading a web URL — cleaner than WebFetch, removes nav/ads. Use `defuddle parse <url> --md` |
| `json-canvas` | Creating `.canvas` visual knowledge maps in `wiki/canvas/` |
| `obsidian-bases` | Creating `.base` database views in `wiki/bases/` |

---

## Workflows

### A. Ingest (add a new source)

When the user says "ingest [filename]" or provides a path to `raw/`:

**NEW WORKFLOW**: Files should be placed in `raw/inbox/` first. `/ingest` will:

1. **Categorize and move files** from `raw/inbox/` to appropriate folders:
   - Has `url:` OR `type: clip` → `raw/clips/`
   - Has `author:` OR `type: book` → `raw/books/`
   - Has `type: note` OR no frontmatter → `raw/notes/`
   - Image files → `raw/assets/`
2. Read the source file in full. If a folder is given, Glob all `.md` files and process each sequentially.
3. Create a **source summary page** in `wiki/sources/` using **obsidian-markdown** syntax (wikilinks, callouts, proper frontmatter).
3. Identify concepts mentioned — for each:
   - `obsidian search query="[concept]"` to check for duplicates; if it fails, Glob `wiki/concepts/` as fallback.
   - If a concept page exists → update it with new info, note if it contradicts previous claims.
   - If no concept page exists → create one using **obsidian-markdown** syntax.
4. If source is a book chapter → update or create the **book page** in `wiki/books/`.
5. Update `index.md` — add new rows to all relevant tables.
6. Append an entry to `log.md`.
7. Verify with `obsidian read file="[slug]"`; if it fails, skip — Write tool guarantees file exists.

A single ingest may touch 5–15 wiki pages. That is expected and correct.

### B. Query (answer a research question)

When the user asks a question:

1. Read `index.md` to find relevant pages.
2. `obsidian search query="[terms]"` for full-text search across the vault; if it fails, use index.md + Grep as fallback.
3. Read those pages in full.
4. Synthesize an answer with citations (link to wiki pages).
5. Ask: "Would you like to save this answer as a synthesis page?" — if yes, file it using **obsidian-markdown** syntax.

### C. Research (fill gaps via web search)

When the user says "research [topic]" or runs `/research`:

1. Check `index.md` for existing coverage of the topic.
2. Use WebSearch to find authoritative sources filling the gap.
3. Read each URL using **defuddle**: `defuddle parse <url> --md` (fallback to WebFetch if not installed).
4. Summarize findings (2–5 points) and note conflicts with existing wiki content.
5. Ask: "Would you like to update the wiki with this information?" — if yes, update relevant concept pages.
6. Append an entry to `log.md`.

### D. Clip (save a web page as a raw source)

When the user provides a URL to clip or runs `/clip [url]`:

1. Run `defuddle parse <url> --md` to extract clean content.
2. Also extract `title`, `description`, `domain` metadata.
3. Write to `raw/clips/[slug].md` with frontmatter header.
4. Ask: "ต้องการ ingest เข้า wiki เลยไหม?"

### E. Lint (wiki health check)

When the user says "lint" or runs `/lint`:

Check for:
- [ ] Pages with no inbound links (orphans) — use `obsidian backlinks` if available, else Grep
- [ ] Conflicting information between pages
- [ ] Concepts mentioned in 2+ pages but no dedicated page
- [ ] Source pages not linked to any concept
- [ ] index.md entries that are missing or stale
- [ ] Topics worth researching further

Output a health report as a markdown checklist.

---

## index.md Format

```markdown
# Index

## Sources (`wiki/sources/`)
| Page | Summary | Source file | Date |
|------|---------|-------------|------|
| [[wiki/sources/slug\|Title]] | One-line summary | [[wiki/sources/slug]] | YYYY-MM-DD |

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

## Canvas (`wiki/canvas/`)
| Page | Topic | Date |
|------|-------|------|
| [[wiki/canvas/slug\|Title]] | Visual map of X | YYYY-MM-DD |

## Bases (`wiki/bases/`)
| Page | Description | Date |
|------|-------------|------|
| [[wiki/bases/slug\|Title]] | Database view of X | YYYY-MM-DD |
```

**Rules:**
- Every "Page" column = Obsidian link to a `wiki/` page only
- "Source file" column = `[[wiki/sources/slug]]` link (same wiki page) — to reach raw, open that page and click "Full source"
- **index.md must never contain `[[raw/...]]` or `[...](raw/...)` links**

---

## log.md Format

Each entry uses this prefix so it's grep-able:

```
## [YYYY-MM-DD] ingest | source title
## [YYYY-MM-DD] query | question summary
## [YYYY-MM-DD] lint | summary of findings
## [YYYY-MM-DD] update | page name updated
```

---

## Link Hierarchy Rules

The system has exactly three layers. Links must only go **downward** — never skip a layer, never link sideways to `raw/`.

```
index.md
    ↓ links to
wiki/sources/*.md   wiki/concepts/*.md   wiki/books/*.md   wiki/synthesis/*.md
    ↓ links to
raw/clips/*.md   raw/notes/*.md   raw/books/*.md
```

### Layer 1 — index.md

- The "Page" column in every table must be an Obsidian link to a **wiki page only**.
  - Sources table:   `[[wiki/sources/slug|Title]]`
  - Concepts table:  `[[wiki/concepts/slug|Title]]`
  - Books table:     `[[wiki/books/slug|Title]]`
  - Synthesis table: `[[wiki/synthesis/slug|Title]]`
- The "Source file" column links to the **same `wiki/sources/` page** — `[[wiki/sources/slug]]` — so the reader can click through to find the raw link via "Full source" inside that page.
- **index.md must never contain `[[raw/...]]` or `[...](raw/...)` links.**
- Navigation path to raw: index → `wiki/sources/slug` → "Full source" → raw file.

### Layer 2 — wiki/sources/*.md

- Must contain exactly **one** "Full source" link pointing to the raw source file.
  - Local file: `**Full source**: [[../../raw/path/to/file.md|Original file]]`
  - External URL: `**Full source**: [Article title](https://...)`
- All other links in the body must point to `wiki/concepts/` or `wiki/sources/` — not to `raw/`.

### Layer 2 — wiki/concepts/*.md

- Links in body must point to `wiki/sources/` or `wiki/concepts/` only.
- Must never link directly to `raw/`.

### Layer 2 — wiki/synthesis/*.md, wiki/books/*.md

- Same rule as concepts: links go to other wiki pages only.
- Never link to `raw/` directly.

### Summary rule

> **Only `wiki/sources/` pages may link to `raw/`. Everything else links to wiki pages.**

---

## Cross-linking Syntax

- Prefer Obsidian `[[...]]` syntax for inter-wiki links so graph view works.
- External URLs go in frontmatter `url:` field and in "Full source" citation only.

---

## Image Handling

Markdown files from Obsidian Web Clipper may contain inline images (e.g. `![[raw/assets/img.png]]`).

- Read the markdown text first in full.
- If important diagrams or charts are referenced, Read the image files separately afterwards to gain additional context.
- Do not attempt to read all images in one pass — prioritize text, then images where needed.
- Locally stored images live in `raw/assets/`. Never modify them.

---

## Conventions

- File names: lowercase, hyphens, no spaces. e.g. `cognitive-load.md`, `kahneman-thinking-fast-slow.md`
- Dates: `YYYY-MM-DD` always.
- Never delete raw source files.
- Never overwrite a wiki page without reading it first — always merge, not replace.
- If a source contradicts an existing claim, note both views in the concept page under a **"Conflicting Views"** section. Do not silently overwrite.
- Answers from queries that represent new synthesis (comparisons, analyses, discovered connections) should be filed as `wiki/synthesis/` pages — not left in chat history.
