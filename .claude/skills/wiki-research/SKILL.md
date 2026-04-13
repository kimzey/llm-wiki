---
name: wiki-research
description: Search the web to fill knowledge gaps identified in the wiki. Reads index, finds shallow or missing topics, searches web, summarizes findings in Thai, and optionally updates wiki pages.
origin: user
---

# Wiki Research Workflow

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps

### 1. Identify gap

- If a topic is provided → search for that topic
- If no topic → Read `index.md` → find concepts with fewer than 2 sources, or concept pages with empty sections

### 2. Search

Use WebSearch to find:
- Articles or papers that best explain the concept
- Information that contradicts or updates existing wiki content
- Examples and case studies still missing from the wiki

### 3. Summarize findings

Respond in Thai:
- What was found (2–5 points)
- Whether it contradicts anything in the current wiki
- URLs worth ingesting

### 4. Ask user

"ต้องการให้อัปเดต wiki ด้วยข้อมูลนี้เลยไหม?"
- Yes → update relevant concept pages + append log.md
- No → done

### 5. Append log.md (if updated)

```
## [YYYY-MM-DD] query | research: [topic]
- Sources searched: [URLs]
- Pages updated: [list]
```

## Notes

- Never hallucinate URLs — use WebSearch for real results
- If nothing better than existing content is found → say so clearly
- Images on web pages → describe as text only, do not download
