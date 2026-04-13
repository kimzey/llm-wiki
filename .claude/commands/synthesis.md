---
description: "Create a synthesis/analysis page from wiki knowledge. Usage: /synthesis [topic or question]"
---

Create a synthesis page for: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Steps:
1. Read index.md to find all pages relevant to this topic
2. Read those pages (concepts, books, sources)
3. Synthesize a deep analysis drawing from multiple wiki pages
4. Create `wiki/synthesis/[slug].md` using the synthesis template from CLAUDE.md
   - Use **obsidian-markdown** syntax: wikilinks `[[...]]`, callouts `> [!note]` for key findings, proper frontmatter
   - Include the question/topic, full analysis, conclusions, and suggested further study
   - Cite all sources using [[wiki/page]] links
5. Update index.md — add row to Synthesis table: `[[wiki/synthesis/slug|title]]`
6. Append to log.md: `## [YYYY-MM-DD] synthesis | [topic]`
7. Report: "Created [[wiki/synthesis/[slug]]]"

If $ARGUMENTS is empty, ask: "What topic would you like to synthesize?"
