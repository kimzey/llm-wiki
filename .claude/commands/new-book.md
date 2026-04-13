---
description: "Create a new book summary page. Usage: /new-book [book title]"
---

Create a new book summary page for: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Steps:
1. Read CLAUDE.md to get the book page template
2. Check if a book page for this title already exists (Glob wiki/books/)
   - If yes: show the existing page and ask if the user wants to update it instead
3. Ask for: author name, publication year, and any initial notes or chapter summary to add
4. Create `wiki/books/[slug].md` using the template from CLAUDE.md
   - slug = book title in lowercase-hyphens
   - Fill frontmatter: title, type: book, author, year, created: today's date
   - Pre-fill any info the user provided
5. If there's a raw source file for this book in raw/books/, ask if they want to ingest it now
6. Update index.md — add row to the Books table
7. Append to log.md: `## [YYYY-MM-DD] update | สร้าง book: [title]`

If $ARGUMENTS is empty, ask: "ชื่อหนังสืออะไร?"
