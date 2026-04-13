<!-- Generated: 2026-04-13 | Files scanned: 157 | Token estimate: ~700 -->

# Schema — LLM Wiki

## Page Types & Locations

| Type | Directory | When to create |
|---|---|---|
| `source` | `wiki/sources/` | One per raw source file |
| `concept` | `wiki/concepts/` | One per idea/principle/framework that appears across sources |
| `book` | `wiki/books/` | One per book (aggregate of chapter sources) |
| `synthesis` | `wiki/synthesis/` | Analyses, comparisons, answers to big questions |

## Frontmatter (all wiki pages)

```yaml
---
title: "Page title"
type: concept | book | source | synthesis
tags: [tag1, tag2]
sources: [filename1.md, filename2.md]   # raw source files this page draws from
related: [wiki/concepts/foo.md]          # cross-links to other wiki pages
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

Source pages also have: `source_file`, `url`, `published`
Book pages also have: `author`, `year`

## Source Page Structure

```markdown
> **Full source**: [[../../raw/notes/folder/file.md|Original file]]
  (or external URL)

## สรุป          — what does this article cover?
## ประเด็นสำคัญ  — bullet points of key arguments
## ข้อมูลน่าสนใจ — notable data/evidence
## สิ่งขัดแย้ง   — contradictions with existing wiki
## Concepts      — links to wiki/concepts/
```

## Concept Page Structure

```markdown
## สรุปสั้น       — 1–3 sentence definition
## อธิบาย         — deeper explanation
## ประเด็นสำคัญ   — bullet points
## ตัวอย่าง       — examples / case studies
## ความสัมพันธ์   — links to related concepts
## แหล่งที่มา     — cite source pages used
```

## index.md Format

```markdown
## Sources (`wiki/sources/`)
| Page | Summary | Source file | Date |
| [[wiki/sources/slug\|Title]] | one-line | [[wiki/sources/slug]] | YYYY-MM-DD |

## Concepts (`wiki/concepts/`)
| Page | Summary | Tags |
| [[wiki/concepts/slug\|Title]] | one-line | tag1, tag2 |

## Books (`wiki/books/`)
| Page | Author | Tags |

## Synthesis (`wiki/synthesis/`)
| Page | Summary | Date |
```

**Critical rule**: index.md "Source file" column → `[[wiki/sources/slug]]` (NOT raw/ path)

## Language Rules

| Content | Language |
|---|---|
| Wiki body text (สรุป, อธิบาย, etc.) | Thai primary |
| Technical terms, proper nouns, titles | English |
| Frontmatter keys | English |
| Code blocks, data | English only |
| CLAUDE.md, .claude/ config files | English only |

## Naming Conventions

- File names: `lowercase-with-hyphens.md`
- Dates: `YYYY-MM-DD` always
- Obsidian links: `[[wiki/concepts/slug|Display Name]]`
- Never modify raw/ files
- Never overwrite wiki page without reading first — always merge
- Contradictions → add **"Conflicting Views"** section, never silently overwrite
