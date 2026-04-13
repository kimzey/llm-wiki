---
description: "Create a new book summary page. Usage: /new-book [book title]"
---

Create a new book summary page for: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Steps:
1. Read CLAUDE.md to get the book page template
2. Check if a book page for this title already exists (Glob wiki/books/)
   - If yes: show the existing page and ask if the user wants to update it instead
3. Create `wiki/books/[slug].md` using the template from CLAUDE.md
   - slug = book title in lowercase-hyphens
   - Fill frontmatter: title, type: book, author, year, created: today's date
   - Use **obsidian-markdown** syntax: wikilinks `[[...]]` for concept cross-links, callouts for key quotes
4. If there's a raw source file for this book in raw/books/, ask if they want to ingest it now
5. Update index.md — add row to Books table: `[[wiki/books/slug|title]]`
6. Append to log.md: `## [YYYY-MM-DD] update | created book: [title]`

If $ARGUMENTS is empty, ask: "What is the book title?"
