---
description: "Query the wiki to answer a research question. Usage: /query [your question]"
---

Answer this research question using the wiki: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Follow the **wiki-query** skill:
1. Read index.md to identify relevant pages
2. If Obsidian is open: `obsidian search query="[key terms]" vault="local-valut"` to find additional matches in body text
   If Obsidian is not open: use index.md + Grep as fallback
3. Read those pages (prioritize: concepts → synthesis → books → sources)
4. Follow related links if they lead to more relevant content
5. Synthesize a Thai-primary answer with citations using [[wiki/page]] links
6. If the wiki has no information: say so and ask if a source should be ingested
7. After answering, ask: "ต้องการบันทึกคำตอบนี้เป็นหน้า synthesis ไหม?"
   If yes → create wiki/synthesis/[slug].md using **obsidian-markdown** syntax (callouts, wikilinks, proper frontmatter)
8. Append to log.md: `## [YYYY-MM-DD] query | [question summary]`

If $ARGUMENTS is empty, ask what the user wants to know.
