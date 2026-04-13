---
description: "Ingest source files from raw/inbox/ into the wiki. Usage: /ingest or /ingest raw/inbox/filename.md"
---

Ingest the following source into the wiki: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Follow the **wiki-ingest** skill exactly:

1. Read CLAUDE.md for schema and templates
2. Determine input source:
   - If $ARGUMENTS is empty → use `raw/inbox/` as default folder
   - If $ARGUMENTS points to a file → use that file
   - If $ARGUMENTS points to a folder → use that folder
3. **Categorize and move files** (new workflow):
   - For each .md file in the input location:
     - **Read entire file** to understand content (not just frontmatter)
     - **Determine main category**:
       * Has `url:` OR `type: clip` → `raw/clips/`
       * Has `author:` OR `type: book` → `raw/books/`
       * Has `type: note` OR no frontmatter → `raw/notes/`
       * Image files → `raw/assets/`
     - **Determine subfolder** by analyzing content:
       * Examples: hr-policy.md → `raw/notes/hr/`, python-tut.md → `raw/clips/tech/programming/`
       * Use AI to analyze title, content, topics
     - **Create full path**: `mkdir -p [main_folder]/[subfolder]/`
     - **Move file**: `mv [source] [main_folder]/[subfolder]/[filename]`
     - If unclear → Ask user: "ไฟล์ [filename] น่าจะอยู่ที่ [suggested_path] ถูกไหม?"
   - Track all moves for reporting
4. Read the source file(s) in full — if processing a folder, process each file sequentially
5. **Determine wiki category** for the source summary page:
   - First, check if raw file is in `raw/notes/{category}/` → use that category
   - If not, analyze the raw file location and content to determine appropriate category
   - **Flexible**: Create new categories as needed (e.g., `database/`, `security/`, `frontend/`)
   - Use lowercase, hyphenated names for consistency
6. Create wiki/sources/[category]/[slug].md for each source
   - Create the category directory first if it doesn't exist: `mkdir -p wiki/sources/{category}/`
   - Use **obsidian-markdown** syntax: wikilinks `[[...]]`, callouts `> [!note]`, proper frontmatter property types
   - Note the `[[../../../raw/...]]` link (extra `../` due to subdirectory)
6. For each concept found:
   - Glob wiki/concepts/ + Grep wiki/concepts/ to check for duplicates
   - Update existing page OR create new one using obsidian-markdown syntax
7. If the source is a book/chapter: update wiki/books/[slug].md
8. Update index.md (add all new rows with categorized paths — links to wiki pages only, never raw/)
9. Append to log.md with the format: `## [YYYY-MM-DD] ingest | [title]`
10. Verify: Write tool guarantees file exists — skip unless Obsidian is open and you want to confirm vault visibility
11. Report which files were categorized, moved, created, and updated

If $ARGUMENTS is empty, default to processing all files in `raw/inbox/`
