---
description: "Search the web to fill knowledge gaps in the wiki. Usage: /research <topic or question>"
---

Research the following topic to fill gaps in the wiki: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Follow the **wiki-research** skill exactly:

1. Read index.md to understand what already exists on this topic
2. Identify what's missing or shallow in the current wiki
3. Use WebSearch to find authoritative sources on the gap
4. Read each URL using **defuddle** (not WebFetch):
   ```
   defuddle parse <url> --md
   ```
   - Exception: URLs ending in `.md` → use WebFetch directly
   - Exception: defuddle not installed → fallback to WebFetch
5. Summarize findings in Thai and note conflicts with existing wiki content
6. Ask: "ต้องการให้อัปเดต wiki ด้วยข้อมูลนี้เลยไหม?"
   - Yes → update relevant concept pages, append log.md
   - Want to save as full source → suggest `/clip [url]`

If $ARGUMENTS is empty, run /lint first to find data gaps, then propose research topics.
