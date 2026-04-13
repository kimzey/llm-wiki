---
description: "Update an existing wiki page with new information. Usage: /update [page path] [what to add/change]"
---

Update wiki page: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Parse $ARGUMENTS as: [page-path] [description of what to add or change]
Example: `/update wiki/concepts/cognitive-load.md add real-world examples`

Steps:
1. Read CLAUDE.md — check conventions
2. Read the target page in full
3. Make the requested changes using **obsidian-markdown** syntax:
   - **Add info**: merge into appropriate section, don't duplicate
   - **Correct info**: if it contradicts existing content, keep both under a "ข้อถกเถียง / มุมมองที่ต่างกัน" section
   - **Add cross-link**: add `[[wiki/concepts/slug|Title]]` wikilinks in the relevant section
   - **Highlight**: use callouts `> [!note]` / `> [!warning]` for important additions
4. Update frontmatter `updated:` to today's date
5. Update index.md if tags or summary changed
6. Append to log.md: `## [YYYY-MM-DD] update | [page name] — [brief description]`

If the page path isn't found, Glob wiki/**/*.md to find the closest match and confirm.
