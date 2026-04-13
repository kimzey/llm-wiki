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

### 2. Determine input source

- If no argument provided → use `raw/inbox/` as default
- If argument is a file → use that file
- If argument is a folder → use that folder
- Glob all `.md` files in the input location

### 3. Categorize and move files (NEW STEP)

For each file found:

**3.1 Read the entire file** to understand content

**3.2 Determine main category** based on frontmatter and content:

| Condition | Main folder |
|-----------|-------------|
| Has `url:` field OR `type: clip` | `raw/clips/` |
| Has `author:` OR `type: book` OR book summary | `raw/books/` |
| Has `type: note` OR no frontmatter | `raw/notes/` |
| Image file (png, jpg, jpeg, gif, webp) | `raw/assets/` |

**3.3 Determine subfolder** by analyzing content:

Analyze the file's title, content, and topics to determine appropriate subfolder. Examples:

**For `raw/notes/`:**
- HR-related content → `raw/notes/hr/`
- Meeting notes → `raw/notes/meetings/`
- Ideas/brainstorming → `raw/notes/ideas/`
- Project plans → `raw/notes/projects/`
- Policy documents → `raw/notes/policy/`
- Personal journal → `raw/notes/journal/`
- Research notes → `raw/notes/research/`

**For `raw/clips/`:**
- Tech/programming → `raw/clips/tech/`
- Business/finance → `raw/clips/business/`
- Science/research → `raw/clips/science/`
- Design → `raw/clips/design/`
- News/current events → `raw/clips/news/`
- Tutorials → `raw/clips/tutorials/`

**For `raw/books/`:**
- Business → `raw/books/business/`
- Self-improvement → `raw/books/self-improvement/`
- Fiction → `raw/books/fiction/`
- Technical → `raw/books/technical/`

**3.4 Create folder structure and move file:**

```bash
# Create full path
mkdir -p [main_folder]/[subfolder]

# Move file
mv [source_path] [main_folder]/[subfolder]/[filename]
```

Example:
```bash
# HR policy document
mkdir -p raw/notes/hr
mv raw/inbox/hr-policy.md raw/notes/hr/hr-policy.md

# Python tutorial clip
mkdir -p raw/clips/tech/programming
mv raw/inbox/python-tutorial.md raw/clips/tech/programming/python-tutorial.md
```

**3.5 If categorization is unclear**, ask user with suggested path:
```
ไฟล์ [filename] ไม่สามารถระบุหมวดหมู่ได้อัตโนมัติ
AI วิเคราะห์ว่าน่าจะเป็น: [suggested_path]
ถูกไหม? หรือต้องการ path อื่น?
```

**3.6 Log the move:**
- Keep track of: `filename: inbox → main_folder/subfolder/`

### 4. Read source

- Single file input → Read that file
- Folder input → Process each moved file sequentially
- If markdown contains inline images → Read text first, then Read images separately afterwards

### 5. Create Source Summary Page

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

### 6. Check for Duplicate Concepts

Before creating/updating concept pages, check for existing coverage:

```
Glob wiki/concepts/**/*.md → scan filenames
Grep pattern="[concept name]" path="wiki/concepts/" → scan body text
→ use results to decide create vs. update
```

Use `obsidian search` only if you need Obsidian-specific metadata (tags, backlinks) that Grep can't provide.

### 7. Update Concept Pages

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

### 8. Update Book Page (if source is a book or chapter)

Read/Create `wiki/books/[slug].md` using the template in CLAUDE.md.
Add new chapter summary under the "สรุปแต่ละบท" section.

### 9. Update index.md

Read `index.md` → add new rows to all relevant tables.

- "หน้า" column = `[[wiki/sources/slug|title]]` — always links to wiki page, never to raw
- "Source file" column = `[[wiki/sources/slug]]` — same wiki page link

### 10. Append log.md

```
## [YYYY-MM-DD] ingest | [source title]
- Created: [list]
- Updated: [list]
- Concepts: [list]
```

### 11. Verify

Write tool guarantees file exists at absolute path — no extra verification needed.
Optionally run `obsidian read file="[slug]"` if Obsidian is open and you want to confirm the vault sees it.

### 12. Report results

```
Ingest complete.

Categorized and moved: N files
- file1.md: inbox → clips
- file2.md: inbox → notes
- ...

Created: N wiki pages
- wiki/sources/file1.md
- wiki/concepts/concept1.md
- ...

Updated: N wiki pages
- wiki/concepts/existing.md
- wiki/books/book1.md
- ...

Index and log updated.
```

## Prohibited

- Never delete existing content from concept pages — always merge
- Never modify files in `raw/` EXCEPT for moving from `raw/inbox/` to target categories
- Never create a page without frontmatter
- Never move files out of `raw/` (only move within `raw/` from inbox to categories)

## Required

- Every Source Summary Page must have a back-link to the raw source
  - Use a blockquote or first heading after frontmatter
  - Format: `**Full source**: [[../../raw/path/to/file.md|Original file]]` or `[title](url)`
- Use **obsidian-markdown** syntax for all wiki pages — wikilinks, callouts, proper frontmatter
- Primary: Glob + Grep + Read — always works, no app dependency
- obsidian-cli: use only for backlinks, tag queries, or Dataview metadata
- **NEW**: Always categorize and move files from `raw/inbox/` before processing
