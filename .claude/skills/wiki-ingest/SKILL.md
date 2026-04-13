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
- Use **obsidian-markdown** syntax throughout:
  - Wikilinks: `[[wiki/concepts/slug|Title]]` for all cross-links
  - Callouts: `> [!note]`, `> [!warning]`, `> [!tip]` to highlight key points
  - Frontmatter property types must be valid (text, list, date — not freeform)

### 4. Check for Duplicate Concepts (obsidian-cli if available)

Before creating/updating concept pages, check for existing coverage:

```
If Obsidian is open:
  obsidian search query="[concept name]" limit=5 vault="local-valut"
  → use results to decide create vs. update

If Obsidian is not open:
  Glob wiki/concepts/ → check filenames
  Grep wiki/concepts/ for concept name → check existing content
```

### 5. Update Concept Pages

For each concept found in the source:

```
check if page exists (via search result or Glob)
  exists     → Read it → merge new info, flag contradictions
  not exists → create new page using template in CLAUDE.md
```

Every concept page must have:
- Back-link to source: `[[wiki/sources/[slug]]]`
- Use **obsidian-markdown** syntax (wikilinks, callouts for important notes)
- If new info contradicts existing → add section "ข้อถกเถียง / มุมมองที่ต่างกัน" — never delete existing content

### 6. Update Book Page (if source is a book or chapter)

Read/Create `wiki/books/[slug].md` using the template in CLAUDE.md.
Add new chapter summary under the "สรุปแต่ละบท" section.

### 7. Update index.md

Read `index.md` → add new rows to all relevant tables.

- "หน้า" column = `[[wiki/sources/slug|title]]` — always links to wiki page, never to raw
- "Source file" column = `[[wiki/sources/slug]]` — same wiki page link

### 8. Append log.md

```
## [YYYY-MM-DD] ingest | [source title]
- Created: [list]
- Updated: [list]
- Concepts: [list]
```

### 9. Verify (obsidian-cli if available)

```
If Obsidian is open:
  obsidian read file="[slug]" vault="local-valut"
  → confirm Obsidian sees the new page correctly

If Obsidian is not open:
  skip — Write tool guarantees file exists at absolute path
```

### 10. Report results

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
- Use **obsidian-markdown** syntax for all wiki pages — wikilinks, callouts, proper frontmatter
- obsidian-cli steps are **optional** — always fallback to Glob/Grep/Read if Obsidian is not open
