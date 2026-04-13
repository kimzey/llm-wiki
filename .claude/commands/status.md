---
description: "Show a dashboard of wiki stats — page counts, recent activity, raw files not yet ingested, and suggested next actions."
---

Show the current status of the wiki at /Users/kimzey/Desktop/local-valut/

Follow the **wiki-status** skill:

1. Count files in each wiki/ subdirectory (concepts, books, sources, synthesis)
2. Count raw source files in each raw/ subdirectory (clips, books, notes) — exclude .gitkeep
3. Check raw/ for any files that haven't been ingested yet (no matching wiki/sources/ page)
4. Read the last 5 entries from log.md
5. Check when lint was last run (search log.md for "lint" entries)

Output a clean dashboard showing:
- Page counts table
- Raw source counts
- Files in raw/ not yet ingested (if any)
- Last 5 log entries
- Suggested next actions (e.g., "ingest 2 pending clips", "run /lint — last ran 8 days ago")
