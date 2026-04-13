---
description: "Run a health check on the entire wiki. Finds orphans, broken links, missing concepts, contradictions, and suggests next sources."
---

Run a full health check on the wiki at /Users/kimzey/Desktop/local-valut/

Follow the **wiki-lint** skill:

1. Read index.md and glob wiki/**/*.md — compare to find index gaps
2. Check for:
   - **Orphan pages** — files with no inbound [[links]]
   - **Broken links** — [[links]] that point to non-existent files
   - **Missing concept pages** — concepts mentioned in 2+ pages but no dedicated page
   - **Contradictions** — conflicting claims between pages on the same concept
   - **Stale content** — pages not updated in 60+ days with newer source material available
   - **Empty sections** — headings with no content body
   - **Data gaps** — concepts with fewer than 2 sources, suggest what to find next

3. Output a markdown health report with checkboxes for each issue
4. Ask which issues to fix immediately
5. Append to log.md: `## [YYYY-MM-DD] lint | [summary]`
