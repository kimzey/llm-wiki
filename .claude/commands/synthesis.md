---
description: "Create a synthesis/analysis page from wiki knowledge. Usage: /synthesis [topic or question]"
---

Create a synthesis page for: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Steps:
1. Read index.md to find all pages relevant to this topic
2. Read those pages (concepts, books, sources)
3. Ask the user: "ต้องการให้วิเคราะห์ในแง่ไหน? (เปรียบเทียบ / สรุปรวม / วิจารณ์ / หา pattern / อื่นๆ)"
4. Synthesize a deep analysis drawing from multiple wiki pages
5. Create `wiki/synthesis/[slug].md` using the synthesis template from CLAUDE.md
   - Include the question/topic, full analysis, conclusions, and "หัวข้อที่ควรศึกษาต่อ"
   - Cite all sources using [[wiki/page]] links
6. Update index.md — add row to Synthesis table
7. Append to log.md: `## [YYYY-MM-DD] synthesis | [topic]`
8. Report: "สร้างหน้า [[wiki/synthesis/[slug]]] แล้ว"

If $ARGUMENTS is empty, ask: "ต้องการวิเคราะห์หัวข้ออะไร?"
