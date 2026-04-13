<!-- Generated: 2026-04-13 | Files scanned: 157 | Token estimate: ~600 -->

# Architecture — LLM Wiki

## System Type
Personal knowledge base (LLM Wiki pattern) — markdown-first, Obsidian-compatible, git-versioned.
No runtime processes. No database. No API. Pure file system.

## Three-Layer Architecture

```
┌─────────────────────────────────────────────────────┐
│  Layer 3 — Schema / Config                          │
│  CLAUDE.md          ← master schema + workflows     │
│  .claude/commands/  ← slash command triggers        │
│  .claude/skills/    ← reusable skill implementations│
└──────────────────────┬──────────────────────────────┘
                       │ LLM reads schema → operates on ↓
┌──────────────────────▼──────────────────────────────┐
│  Layer 2 — Wiki (LLM-owned, human-read)             │
│  wiki/concepts/     ← principle/theory pages        │
│  wiki/sources/      ← per-source summary pages      │
│  wiki/books/        ← book summary pages            │
│  wiki/synthesis/    ← analyses, comparisons         │
│  index.md           ← content catalog (LLM updates) │
│  log.md             ← append-only operation log     │
└──────────────────────┬──────────────────────────────┘
                       │ wiki pages link down to ↓
┌──────────────────────▼──────────────────────────────┐
│  Layer 1 — Raw Sources (immutable, human-curated)   │
│  raw/clips/         ← Obsidian Web Clipper articles │
│  raw/books/         ← PDFs / book chapter notes     │
│  raw/notes/         ← manual notes by topic folder  │
│  raw/assets/        ← locally downloaded images     │
└─────────────────────────────────────────────────────┘
```

## Data Flow — Ingest Operation

```
human drops file → raw/notes/<topic>/
    ↓
/ingest <path>
    ↓
LLM reads raw file(s)
    ↓
creates wiki/sources/<slug>.md
    ↓
identifies concepts → creates/updates wiki/concepts/<slug>.md
    ↓
if book chapter → creates/updates wiki/books/<slug>.md
    ↓
updates index.md (new rows in relevant tables)
    ↓
appends to log.md
```

## Link Hierarchy (strict — never skip layers)

```
index.md
    ↓ Obsidian [[links]] only
wiki/sources/  wiki/concepts/  wiki/books/  wiki/synthesis/
    ↓ "Full source" link (sources only)
raw/clips/  raw/notes/  raw/books/
```

**Rule**: Only `wiki/sources/` pages may link to `raw/`. index.md never links to raw directly.

## Tooling Stack

- **Editor**: Obsidian (graph view, Dataview, Marp, Web Clipper)
- **Agent**: Claude Code (CLI) — owns wiki/, never touches raw/
- **VCS**: git (entire vault is a repo)
- **Search**: index.md at current scale; qmd (optional) for larger scale
