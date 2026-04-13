---
description: "Update an existing wiki page with new information. Usage: /update [page path] [what to add/change]"
---

Update wiki page: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Parse $ARGUMENTS as: [page-path] [description of what to add or change]
Example: `/update wiki/concepts/cognitive-load.md เพิ่มตัวอย่างจาก CLT theory`

Steps:
1. Read CLAUDE.md — check conventions
2. Read the target page in full
3. Make the requested changes:
   - **เพิ่มข้อมูล**: merge into appropriate section, don't duplicate
   - **แก้ไขข้อมูล**: if contradicts existing content, keep both under "ข้อถกเถียง / มุมมองที่ต่างกัน"
   - **เพิ่ม cross-link**: add [[link]] in relevant section
4. Update frontmatter `updated:` to today's date
5. Update index.md if tags or summary changed
6. Append to log.md: `## [YYYY-MM-DD] update | [page name] — [brief description]`

If the page path isn't found, Glob wiki/**/*.md to find the closest match and confirm.
