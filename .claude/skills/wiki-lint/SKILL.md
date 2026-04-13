---
name: wiki-lint
description: Health check for the research wiki. Finds orphan pages, missing cross-links, contradictions, concept gaps, stale content, and suggests new sources to find.
origin: user
---

# Wiki Lint Workflow

Vault root: `/Users/kimzey/Desktop/local-valut/`

## Steps

### 1. Survey the entire wiki
```
Read index.md
Glob wiki/**/*.md → list all actual pages
Compare → find pages missing from index
```

### 2. Check 7 points

#### A. Orphan Pages — pages with no inbound links
Read all pages → grep for `[[filename]]` → which pages are never mentioned?

#### B. Broken Links — links pointing to non-existent files
Grep pattern `\[\[wiki/.*\]\]` in all pages → verify target files exist

#### C. Missing Concept Pages — concepts mentioned but without their own page
Grep for terms in source/synthesis pages that look like concepts → check if `wiki/concepts/` page exists

#### D. Contradictions — conflicting information
Read concept pages with multiple sources → look for language indicating conflict: e.g. "ตรงข้าม", "แย้ง", "ต่างจาก"

#### E. Stale Content — potentially outdated information
Check frontmatter `updated:` → pages not updated in 60+ days that have newer sources available

#### F. Empty Sections — sections with no content body
Read concept pages → find sections with only a heading and no content

#### G. Data Gaps — topics needing more sources
Analyze concept pages → any concept with fewer than 2 sources → suggest what to ingest next

### 3. Output health report

```markdown
# Wiki Health Report — [YYYY-MM-DD]

## Summary
- Total pages: N
- Issues found: N

## A. Orphan Pages
- [ ] wiki/concepts/foo.md — no inbound links

## B. Broken Links
- [ ] wiki/sources/bar.md → [[wiki/concepts/missing]] file not found

## C. Missing Concept Pages
- [ ] "X" — mentioned in 3 pages but no dedicated page

## D. Contradictions
- [ ] wiki/concepts/baz.md — source A and B give conflicting info

## E. Stale Content
- [ ] wiki/concepts/qux.md — not updated in 90 days

## F. Empty Sections
- [ ] wiki/books/foo.md — section "Reviews" still empty

## G. Suggested Sources to Ingest
- "X topic" — only 1 source, recommend finding 2–3 more
```

### 4. Append log.md
```
## [YYYY-MM-DD] lint | found N issues
- Orphans: N, Broken links: N, Missing concepts: N
```

### 5. Ask user
"Which issues would you like to fix now?"
