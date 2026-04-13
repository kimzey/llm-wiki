---
description: "Ingest a source file into the wiki. Usage: /ingest raw/clips/filename.md"
---

Ingest the following source into the wiki: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Follow the **wiki-ingest** skill exactly:

1. Read CLAUDE.md for schema and templates
2. Read the source file in full
3. Present 3–5 key takeaways
4. Create wiki/sources/[slug].md
5. For each concept found: check if a concept page exists → update or create
6. If the source is a book/chapter: update wiki/books/[slug].md
7. Update index.md (add all new rows)
8. Append to log.md with the format: `## [YYYY-MM-DD] ingest | [title]`
9. Report which files were created and updated

If $ARGUMENTS is empty, ask: "ไฟล์ไหนที่ต้องการ ingest? (ระบุ path เช่น raw/clips/article.md)"
