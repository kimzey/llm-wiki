<!-- Generated: 2026-04-14 | Files scanned: 301 | Token estimate: ~800 -->

# Schema — LLM Wiki

## Page Types & Locations

| Type | Directory | Extension | When to create |
|------|-----------|-----------|----------------|
| `source` | `wiki/sources/` | `.md` | One per raw source file |
| `concept` | `wiki/concepts/` | `.md` | One per idea/principle/framework across sources |
| `book` | `wiki/books/` | `.md` | One per book (aggregates chapter sources) |
| `synthesis` | `wiki/synthesis/` | `.md` | Analyses, comparisons, answers to big questions |
| `canvas` | `wiki/canvas/` | `.canvas` | Visual knowledge maps (JSON Canvas format) |
| `base` | `wiki/bases/` | `.base` | Obsidian Bases database views over frontmatter |

## Frontmatter (all wiki .md pages)

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
> **Full source**: [[../../../raw/path/to/file.md|Original file]]
  (note: extra ../ for categorized sources)
  (or external URL: **Full source**: [Title](https://...))

## สรุป          — what does this article cover?
## ประเด็นสำคัญ  — bullet points of key arguments
## ข้อมูลน่าสนใจ — notable data/evidence
## สิ่งขัดแย้ง   — contradictions with existing wiki
## Concepts      — [[wiki/concepts/slug|Name]] links
```

## Concept Page Structure

```markdown
## สรุปสั้น       — 1–3 sentence definition
## อธิบาย         — deeper explanation
## ประเด็นสำคัญ   — bullet points
## ตัวอย่าง       — examples / case studies
## ความสัมพันธ์   — [[wiki/concepts/slug|Name]] links
## แหล่งที่มา     — [[wiki/sources/slug|Title]] links
## จาก [Source]  — section added when updating from new source
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

## Canvas (`wiki/canvas/`)
| Page | Topic | Date |

## Bases (`wiki/bases/`)
| Page | Description | Date |
```

**Critical rule**: index.md "Source file" column → `[[wiki/sources/slug]]` (NOT raw/ path)
**Critical rule**: index.md never contains `[[raw/...]]` or `[...](raw/...)` links

## Language Rules

| Content | Language |
|---------|---------|
| Wiki body text (สรุป, อธิบาย, etc.) | Thai primary |
| Technical terms, proper nouns, titles | English |
| Frontmatter keys | English |
| Code blocks, data | English only |
| CLAUDE.md, .claude/ config files | English only |

## Naming Conventions

- File names: `lowercase-with-hyphens.md` (or `.canvas`, `.base`)
- Dates: `YYYY-MM-DD` always
- Obsidian links: `[[wiki/concepts/slug|Display Name]]`
- Never modify raw/ files
- Never overwrite wiki page without reading first — always merge
- Contradictions → add **"Conflicting Views"** section, never silently overwrite

## Obsidian-Markdown Requirements (use `obsidian-markdown` skill)

- Wikilinks: `[[wiki/concepts/slug|Title]]` — not plain markdown links
- Callouts: `> [!note]`, `> [!warning]`, `> [!tip]` — not blockquotes
- Frontmatter types: `tags` must be YAML list `[tag1, tag2]`, dates as `YYYY-MM-DD`
- Canvas files: JSON format per json-canvas spec — use `json-canvas` skill
- Base files: YAML format per obsidian-bases spec — use `obsidian-bases` skill
