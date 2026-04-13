---
description: "Create a new concept page. Usage: /new-concept [concept name]"
---

Create a new concept page for: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Steps:
1. Read CLAUDE.md to get the concept page template
2. Check if a concept page for this topic already exists (Glob wiki/concepts/)
   - If yes: show the existing page and ask if the user wants to update it instead
   - If no: proceed to create
3. Create `wiki/concepts/[slug].md` using the template from CLAUDE.md
   - slug = concept name in lowercase-hyphens (transliterate Thai if needed)
   - Fill frontmatter: title (Thai preferred), type: concept, tags, created: today's date
   - Leave sections as empty stubs
4. Update index.md — add row to Concepts table: `[[wiki/concepts/slug|title]]`
5. Append to log.md: `## [YYYY-MM-DD] update | created concept: [name]`
6. Report: "Created [[wiki/concepts/[slug]]]"

If $ARGUMENTS is empty, ask: "What is the concept name?"
