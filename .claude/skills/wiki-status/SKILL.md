---
name: wiki-status
description: Show a quick dashboard of the wiki state — page counts, recent activity, raw files pending ingestion, and suggested next actions.
origin: user
---

# Wiki Status Dashboard

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps

### 1. Count all pages by type
```
Glob wiki/concepts/**/*.md → count
Glob wiki/books/**/*.md    → count
Glob wiki/sources/**/*.md  → count
Glob wiki/synthesis/**/*.md → count
Glob raw/**/*.md           → count (exclude .gitkeep)
```

### 2. Read log.md
Show the 5 most recent entries.

### 3. Read index.md
Check completeness — are all wiki pages listed?

### 4. Check raw/ for un-ingested files
For each file in `raw/`, check whether a matching `wiki/sources/{category}/` page exists (note: sources are now in subdirectories by category).
List any raw files that have not been ingested yet.

### 5. Tag stats
```bash
obsidian tags sort=count counts vault="local-valut"
→ show top 10 tags in the vault
```

Fallback (if Obsidian is not open):
```
Grep pattern="^tags:" path="wiki/" output_mode="content" → parse manually
```

Note: obsidian tags handles both frontmatter tags and inline #tags correctly in one command.

### 6. Output Dashboard

```markdown
# Wiki Status — [YYYY-MM-DD HH:MM]

## Page counts
| Type       | Count |
|------------|-------|
| Concepts   | N     |
| Books      | N     |
| Sources    | N     |
| Synthesis  | N     |
| **Total**  | **N** |

## Raw Sources
| Type   | Count |
|--------|-------|
| Clips  | N     |
| Books  | N     |
| Notes  | N     |

## Top Tags (if available)
[from obsidian-cli tags]

## Recent Activity (5 entries)
[from log.md]

## Pending Ingestion
[raw/ files with no matching wiki/sources/ page]

## Suggested Next Actions
- [ ] If raw/ has un-ingested files → list them
- [ ] If wiki is empty → suggest starting with /ingest
- [ ] If lint hasn't run in > 7 days → suggest /lint
```
