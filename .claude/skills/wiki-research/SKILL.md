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

### 3. Read web pages (use defuddle)

For each URL found in step 2, read its content using **defuddle** (not WebFetch):

```bash
defuddle parse <url> --md
```

- defuddle removes navigation, ads, and clutter → cleaner markdown, fewer tokens
- **Exception**: URLs ending in `.md` → use WebFetch directly (already markdown)
- **Exception**: URLs behind login/paywall → skip or note as inaccessible

### 4. Summarize findings

Respond in Thai:
- What was found (2–5 points)
- Whether it contradicts anything in the current wiki
- URLs worth ingesting as raw sources

### 5. Ask user

"ต้องการให้อัปเดต wiki ด้วยข้อมูลนี้เลยไหม?"
- Yes → update relevant concept pages + append log.md
- No → done

If user wants to ingest a URL as a full source → suggest `/clip [url]` to save it to raw/clips/ first

### 6. Append log.md (if updated)

```
## [YYYY-MM-DD] research | [topic]
- Sources searched: [URLs]
- Pages updated: [list]
```

## Notes

- Never hallucinate URLs — use WebSearch for real results
- If nothing better than existing content is found → say so clearly
- Images on web pages → describe as text only, do not download
- defuddle requires the `defuddle` CLI to be installed (`npm install -g defuddle`)
  - If not installed → fallback to WebFetch
