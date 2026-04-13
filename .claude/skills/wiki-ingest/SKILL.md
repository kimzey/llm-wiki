---
name: wiki-ingest
description: Ingest a new source into the research wiki. Reads the source, extracts key concepts, creates/updates all relevant wiki pages, and updates index + log. Wiki content is written in Thai.
origin: user
---

# Wiki Ingest Workflow

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps (follow in order — do not skip)

### 1. Read schema

Read `CLAUDE.md` — check frontmatter format and page templates.

### 2. Read source

- Single file input → Read that file
- Folder input → Glob `[folder]/**/*.md` → process each file sequentially through steps 3–7
- If markdown contains inline images → Read text first, then Read images separately afterwards

### 3. Create Source Summary Page

Create `wiki/sources/[slug].md` using the template in CLAUDE.md.

- slug = source filename or title converted to lowercase-hyphens
- Include full frontmatter (title, type, source_file, url, published, tags, related, created, updated)
- **Required**: include a back-link to the raw source file at the top of the content body
  - Local file: `**Full source**: [[../../raw/path/to/file.md|Original file]]`
  - External URL: `**Full source**: [URL](url)`

### 4. Update Concept Pages

For each concept found in the source:

```
Glob wiki/concepts/ → check if page exists
  exists     → Read it → merge new info, flag contradictions
  not exists → create new page using template in CLAUDE.md
```

Every concept page must have:
- Back-link to source: `[[wiki/sources/[slug]]]`
- If new info contradicts existing → add section "ข้อถกเถียง / มุมมองที่ต่างกัน" — never delete existing content

### 5. Update Book Page (if source is a book or chapter)

Read/Create `wiki/books/[slug].md` using the template in CLAUDE.md.
Add new chapter summary under the "สรุปแต่ละบท" section.

### 6. Update index.md

Read `index.md` → add new rows to all relevant tables.

- "หน้า" column = `[[wiki/sources/slug|title]]` — always links to wiki page, never to raw
- "Source file" column = `[[wiki/sources/slug]]` — same wiki page link

### 7. Append log.md

```
## [YYYY-MM-DD] ingest | [source title]
- Created: [list]
- Updated: [list]
- Concepts: [list]
```

### 8. Report results

```
Ingest complete.
Created: N pages
Updated: N pages
All touched files: [list]
```

## Prohibited

- Never delete existing content from concept pages — always merge
- Never modify files in `raw/` — read only
- Never create a page without frontmatter

## Required

- Every Source Summary Page must have a back-link to the raw source
  - Use a blockquote or first heading after frontmatter
  - Format: `**Full source**: [[../../raw/path/to/file.md|Original file]]` or `[title](url)`
