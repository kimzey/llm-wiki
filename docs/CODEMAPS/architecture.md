<!-- Generated: 2026-04-14 | Files scanned: 301 | Token estimate: ~700 -->

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
│  .claude/skills/    ← reusable skill impls (11)     │
│  docs/CODEMAPS/     ← this file + 4 others          │
└──────────────────────┬──────────────────────────────┘
                       │ LLM reads schema → operates on ↓
┌──────────────────────▼──────────────────────────────┐
│  Layer 2 — Wiki (LLM-owned, human-read)             │
│  wiki/concepts/  (63)  ← principle/theory pages     │
│  wiki/sources/   (98)  ← per-source summary pages   │
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
│  raw/notes/  (140)  ← manual notes by topic folder  │
│  raw/assets/        ← locally downloaded images     │
└─────────────────────────────────────────────────────┘
```

## Data Flow — Ingest Operation

```
human drops file → raw/inbox/
    ↓
/ingest
    ↓
auto-categorize: clips/ or notes/ or books/ or assets/
    ↓
move to destination: raw/<type>/<category>/<file>
    ↓
LLM reads raw file(s)
    ↓  [obsidian-cli: search for duplicates if Obsidian open]
create wiki/sources/<category>/<slug>.md
    ↓
identify concepts → create/merge wiki/concepts/<slug>.md
    ↓
if book chapter → update wiki/books/<slug>.md
    ↓
update index.md (new rows in all tables)
    ↓
append to log.md
    ↓
verify with obsidian-cli (if open)
```

## Link Hierarchy (strict — never skip layers)

```
index.md
    ↓ Obsidian [[links]] only
wiki/sources/<category>/  wiki/concepts/  wiki/books/  wiki/synthesis/  wiki/canvas/  wiki/bases/
    ↓ "Full source" link (sources only)
raw/clips/  raw/notes/  raw/books/
```

**Rule**: Only `wiki/sources/` pages may link to `raw/`. index.md never links to raw directly.

## Source Categories (9 total)

| Category | Files | Purpose |
|----------|-------|---------|
| `ai-context/` | 8 | AI coding context, .claude structure, agents, commands, skills |
| `frameworks/` | 14 | Arona, Haystack, OpenRAG, LangFlow |
| `infrastructure/` | 2 | Network fundamentals, home server admin |
| `javascript/` | 1 | Bun runtime (NEW) |
| `langchain/` | 16 | Step-by-step tutorials, deep dives, LangGraph, LangSmith |
| `observability/` | 6 | Distributed tracing, OTel, context propagation |
| `policy/` | 1 | HR policy |
| `rag/` | 30 | RAG theory, LlamaIndex, chunking, evaluation, vector DBs |
| `sellsuki/` | 20 | Design tokens, components, RAG guides, agent plans |

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
