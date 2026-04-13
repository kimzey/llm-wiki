<!-- Generated: 2026-04-14 | Files scanned: 271 | Token estimate: ~650 -->

# Architecture — LLM Wiki

## System Type
Personal knowledge base (LLM Wiki pattern) — markdown-first, Obsidian-compatible, git-versioned.
No runtime processes. No database. No API. Pure file system.

## Three-Layer Architecture

```
┌─────────────────────────────────────────────────────┐
│  Layer 3 — Schema / Config                          │
│  CLAUDE.md          ← master schema + workflows     │
│  .claude/commands/  ← slash command triggers (14)   │
│  .claude/skills/    ← reusable skill impls (10)     │
│  docs/CODEMAPS/     ← this file + 4 others          │
└──────────────────────┬──────────────────────────────┘
                       │ LLM reads schema → operates on ↓
┌──────────────────────▼──────────────────────────────┐
│  Layer 2 — Wiki (LLM-owned, human-read)             │
│  wiki/concepts/  (47)  ← principle/theory pages     │
│  wiki/sources/   (94)  ← per-source summary pages   │
│  wiki/books/     (0)   ← book summary pages         │
│  wiki/synthesis/ (1)   ← analyses, comparisons      │
│  wiki/canvas/    (0)   ← visual maps (.canvas)      │
│  wiki/bases/     (0)   ← database views (.base)     │
│  index.md              ← content catalog            │
│  log.md                ← append-only operation log  │
└──────────────────────┬──────────────────────────────┘
                       │ wiki pages link down to ↓
┌──────────────────────▼──────────────────────────────┐
│  Layer 1 — Raw Sources (immutable, human-curated)   │
│  raw/clips/         ← Obsidian Web Clipper articles │
│  raw/books/         ← PDFs / book chapter notes     │
│  raw/notes/  (130)  ← manual notes by topic folder  │
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
    ↓  [obsidian-cli: search for duplicates if Obsidian open]
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
wiki/sources/  wiki/concepts/  wiki/books/  wiki/synthesis/  wiki/canvas/  wiki/bases/
    ↓ "Full source" link (sources only)
raw/clips/  raw/notes/  raw/books/
```

**Rule**: Only `wiki/sources/` pages may link to `raw/`. index.md never links to raw directly.

## Tooling Stack

| Tool | Role |
|------|------|
| Claude Code (CLI) | Agent — owns wiki/, reads raw/, never modifies raw/ |
| Obsidian | Human-facing reader: graph view, Bases, Web Clipper |
| git | Version control for entire vault |
| defuddle (skill) | Extract clean markdown from web URLs |
| obsidian-cli (skill) | Search/read vault via Obsidian socket API |
| obsidian-markdown (skill) | Syntax guidelines for all wiki page writes |
| json-canvas (skill) | Create .canvas visual maps in wiki/canvas/ |
| obsidian-bases (skill) | Create .base database views in wiki/bases/ |
