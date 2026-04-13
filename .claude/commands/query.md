---
description: "Query the wiki to answer a research question. Usage: /query [your question]"
---

Answer this research question using the wiki: $ARGUMENTS

Vault: /Users/kimzey/Desktop/local-valut/

Follow the **wiki-query** skill:
1. Read index.md to identify relevant pages
2. Read those pages (prioritize: concepts → synthesis → books → sources)
3. Follow related links if they lead to more relevant content
4. Synthesize a Thai-primary answer with citations using [[wiki/page]] links
5. If the wiki has no information: say so and ask if I should ingest a source
6. After answering, ask: "ต้องการบันทึกคำตอบนี้เป็นหน้า synthesis ไหม?"
7. Append to log.md: `## [YYYY-MM-DD] query | [question summary]`

If $ARGUMENTS is empty, ask what the user wants to know.
