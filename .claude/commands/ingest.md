---
description: "Ingest a source file into the wiki. Usage: /ingest raw/clips/filename.md"
---

Ingest the following source into the wiki: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Follow the **wiki-ingest** skill exactly:

1. Read CLAUDE.md for schema and templates
2. Read the source file(s) in full — if $ARGUMENTS is a folder, Glob all .md files inside it and process each one sequentially
3. Create wiki/sources/[slug].md for each source
   - Use **obsidian-markdown** syntax: wikilinks `[[...]]`, callouts `> [!note]`, proper frontmatter property types
4. For each concept found:
   - If Obsidian is open: `obsidian search query="[concept]" vault="local-valut"` to check for duplicates
   - If Obsidian is not open: Glob wiki/concepts/ to check filenames
   - Update existing page OR create new one using obsidian-markdown syntax
5. If the source is a book/chapter: update wiki/books/[slug].md
6. Update index.md (add all new rows — links to wiki pages only, never raw/)
7. Append to log.md with the format: `## [YYYY-MM-DD] ingest | [title]`
8. Verify (if Obsidian is open): `obsidian read file="[slug]" vault="local-valut"` to confirm file is visible
9. Report which files were created and updated

If $ARGUMENTS is empty, ask: "Which file or folder would you like to ingest? (e.g. raw/clips/article.md or raw/notes/folder/)"
