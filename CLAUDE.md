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
│   ├── clips/         ← web articles clipped via Obsidian Web Clipper
│   ├── books/         ← book files, PDFs, or chapter notes
│   ├── notes/         ← manual notes typed by the user
│   └── assets/        ← locally downloaded images
└── wiki/              ← LLM-generated wiki (read by human, written by LLM)
    ├── concepts/      ← concept pages (หลักการ, ทฤษฎี, ไอเดีย)
    ├── books/         ← book summary pages (สรุปหนังสือ)
    ├── sources/       ← per-source summary pages
    └── synthesis/     ← analyses, comparisons, cross-cutting essays
```

---

## Language Rules

- **Wiki content**: Thai as the primary language. Use English for technical terms,
  proper nouns, titles, and citations where Thai translation would be awkward.
- **Headings**: Thai preferred.
- **Code/data**: English only.
- **Frontmatter fields**: English keys, Thai/English values.
- **This file (CLAUDE.md)**: English (instructions for the LLM).

---

## Page Frontmatter

Every wiki page must start with YAML frontmatter:

```yaml
---
title: "ชื่อหน้า"
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
title: "ชื่อ Concept"
type: concept
tags: []
sources: []
related: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

## สรุปสั้น
(1-3 ประโยค — ตอบว่า concept นี้คืออะไร)

## อธิบาย
(อธิบายลึกขึ้น)

## ประเด็นสำคัญ
- bullet points

## ตัวอย่าง / กรณีศึกษา

## ความสัมพันธ์กับ concept อื่น
(link ไปหน้าที่เกี่ยวข้อง)

## แหล่งที่มา
(cite ชื่อ source ที่ใช้)
```

### 2. Book Summary Page (`wiki/books/`)
For books — one page per book.

```markdown
---
title: "ชื่อหนังสือ"
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
(หนังสือเกี่ยวกับอะไร — 1 ย่อหน้า)

## ประเด็นหลัก
(bullet points ของ key arguments/ideas)

## สรุปแต่ละบท
### บทที่ 1 — ชื่อบท
...

## Concepts ที่เกี่ยวข้อง
(link ไปที่ wiki/concepts/)

## คำคมที่น่าจดจำ
> quote (p.XX)

## ความคิดเห็น / วิจารณ์
```

### 3. Source Summary Page (`wiki/sources/`)
For web articles, papers, or any clipped/raw source. One page per source file.

```markdown
---
title: "ชื่อบทความ"
type: source
source_file: raw/clips/filename.md
url: ""
published: YYYY-MM-DD
tags: []
related: []
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

> **อ่านเต็ม**: [[../../raw/clips/filename.md|ไฟล์ต้นฉบับ]]
>
> หรือถ้าเป็น external URL: **อ่านเต็ม**: [URL](url)

## สรุป
(สั้นๆ — บทความนี้พูดถึงอะไร)

## ประเด็นสำคัญ
- 

## ข้อมูล / หลักฐาน ที่น่าสนใจ

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

## Concepts ที่เกี่ยวข้อง
(link ไปที่ wiki/concepts/)
```

### 4. Synthesis Page (`wiki/synthesis/`)
For analyses, comparisons, or answers to big questions — filed as permanent wiki pages.

```markdown
---
title: "ชื่อการวิเคราะห์"
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

## Workflows

### A. Ingest (เพิ่ม source ใหม่)

When the user says "ingest [filename]" or drops a file into `raw/`:

1. Read the source file in full.
2. Briefly discuss key takeaways with the user (2-4 bullet points, ask if focus area needed).
3. Create a **source summary page** in `wiki/sources/`.
4. Identify concepts mentioned — for each:
   - If a concept page exists → update it with new info, note if contradicts previous claims.
   - If no concept page exists → create one.
5. If source is a book chapter → update or create the **book page** in `wiki/books/`.
6. Update `index.md` — add the new source and any new/updated pages.
7. Append an entry to `log.md`.

A single ingest may touch 5–15 wiki pages. That is expected and correct.

### B. Query (ถามคำถาม)

When the user asks a question:

1. Read `index.md` to find relevant pages.
2. Read those pages in full.
3. Synthesize an answer with citations (link to wiki pages and raw sources).
4. Ask: "ต้องการบันทึกคำตอบนี้เป็นหน้า synthesis ไหม?" — if yes, file it.

### C. Lint (ตรวจสุขภาพ wiki)

When the user says "lint" or "ตรวจ wiki":

Check for:
- [ ] หน้าที่ไม่มี inbound links (orphans)
- [ ] ข้อมูลขัดแย้งกันระหว่างหน้า
- [ ] Concepts ที่ถูก mention แต่ไม่มีหน้าของตัวเอง
- [ ] Source pages ที่ยังไม่ได้ link ไป concept ไหนเลย
- [ ] index.md ที่ไม่ครบ
- [ ] หัวข้อที่ควรค้นหาเพิ่มเติม

Output a health report as a markdown checklist.

---

## index.md Format

```markdown
# Index

## Sources (`wiki/sources/`)
| หน้า | สรุปสั้น | Source file | วันที่ |
|------|----------|-------------|--------|

## Concepts (`wiki/concepts/`)
| หน้า | สรุปสั้น | Tags |
|------|----------|------|

## Books (`wiki/books/`)
| หน้า | ผู้แต่ง | Tags |
|------|---------|------|

## Synthesis (`wiki/synthesis/`)
| หน้า | สรุปสั้น | วันที่ |
|------|----------|--------|
```

---

## log.md Format

Each entry uses this prefix so it's grep-able:

```
## [YYYY-MM-DD] ingest | ชื่อ source
## [YYYY-MM-DD] query | ชื่อคำถาม
## [YYYY-MM-DD] lint | สรุปผล
## [YYYY-MM-DD] update | ชื่อหน้าที่อัปเดต
```

---

## Cross-linking Rules

- Always use relative paths: `[[wiki/concepts/foo]]` (Obsidian wiki links) or `[foo](../concepts/foo.md)`.
- Prefer Obsidian `[[...]]` syntax for inter-wiki links so graph view works.
- External URLs go in frontmatter `url:` field and in source citation sections only.

---

## Conventions

- File names: lowercase, hyphens, no spaces. Thai transliterated or English. e.g. `cognitive-load.md`, `kahneman-thinking-fast-slow.md`
- Dates: `YYYY-MM-DD` always.
- Never delete raw source files.
- Never overwrite a wiki page without reading it first — always merge, not replace.
- If a source contradicts an existing claim, note both views in the concept page under a "ข้อถกเถียง / มุมมองที่ต่างกัน" section. Do not silently overwrite.
