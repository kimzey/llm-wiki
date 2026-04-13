---
name: wiki-query
description: Answer research questions by searching the wiki. Reads index, drills into relevant pages, synthesizes Thai-primary answers with citations, and optionally files the answer as a synthesis page.
origin: user
---

# Wiki Query Workflow

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps

### 1. Find relevant pages

Run both in parallel:

1. Read `index.md` — scan titles and summaries for relevant rows
2. Grep the vault for key terms — finds body text not in index titles:

```
Grep pattern="[key terms]" path="wiki/" output_mode="files_with_matches"
```

Use `obsidian search` only when you need Obsidian-specific features (backlinks, tag queries, Dataview metadata)

### 2. Read relevant pages

Read every page identified in step 1, then follow related links if they lead to more relevant content.
Priority order: concepts > synthesis > books > sources > raw

### 3. Answer the question

- Answer in Thai as the primary language
- Cite every claim with `[[wiki/concepts/foo]]` or `(source: wiki/sources/bar.md)`
- If the wiki has no information → say clearly: "ยังไม่มีข้อมูลใน wiki — ต้องการ ingest source เพิ่มไหม?"

### 4. Output format by question type

| Question type   | Format                 |
| --------------- | ---------------------- |
| Explain concept | prose + bullet points  |
| Compare         | markdown table         |
| Book summary    | structured summary     |
| Deep analysis   | essay + synthesis page |
| Find pattern    | list + examples        |

### 5. Ask to save

For any non-trivial answer, ask:
"ต้องการบันทึกคำตอบนี้เป็นหน้า synthesis ไหม?"
If yes → create `wiki/synthesis/[slug].md` using **obsidian-markdown** syntax:

- Use callouts `> [!note]` / `> [!warning]` to highlight key findings
- Use wikilinks `[[...]]` for all cross-references
- Fill all frontmatter fields correctly
  Then update index.md + append log.md

### 6. Append log.md

```
## [YYYY-MM-DD] query | [brief question]
- Pages consulted: [list]
- Synthesis created: yes/no
```

## Notes

- Good answers compound over time — the more things filed as synthesis, the smarter the wiki gets
- Never hallucinate — if not in wiki, say so
- Primary search: Read `index.md` + Grep — always works, no app dependency
- Use `obsidian search` only for backlinks, tag queries, or Dataview metadata
