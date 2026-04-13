# Log

บันทึกการดำเนินการทั้งหมดใน wiki นี้ — เรียงตามเวลา ไม่ลบ ไม่แก้ไข

---

## [2026-04-13] ingest | raw/notes/Hetstack — Haystack Deep Dive (11 files)

- สร้าง source pages: haystack-phase1-overview, haystack-phase2-indexing, haystack-phase3-retrieval, haystack-phase4-advanced, haystack-phase5-production, haystack-phase6-observability (6 ไฟล์ — grouped by phase)
- สร้าง concept pages: haystack-framework, rag-evaluation (2 ไฟล์)
- อัปเดต concept pages: hybrid-search-bm25-vector (เพิ่ม Haystack RRF implementation details)
- ครอบคลุมเนื้อหา: Haystack 2.x architecture (Component-Pipeline-DocumentStore), Indexing/Query Pipeline, BM25+Semantic+Hybrid Search (RRF), Cross-encoder Ranker, Custom Components, Agent Systems, Conversational RAG, Evaluation (RAGAS/Faithfulness), Hayhooks API, Langfuse/OTel observability, Production stack
- อัปเดต index.md (6 sources, 2 concepts ใหม่)

## [2026-04-13] ingest | raw/notes/tool-rag — RAG Tools & Frameworks (5 files)

- สร้าง source pages: rag-decision-guide, langchain-llamaindex-deep-dive, langchain-rag-guide, llamaindex-deep-dive, llamaindex-full-guide (5 ไฟล์)
- สร้าง concept pages: langchain-framework, llamaindex-framework, rag-chunking-strategies (3 ไฟล์)
- อัปเดต concept pages: rag-retrieval-augmented-generation (เพิ่ษ LangChain/LlamaIndex approaches, 4 chunking strategies), semantic-caching (เพิ่ม LangChain/LlamaIndex implementations)
- ครอบคลุมเนื้อหา: LangChain vs LlamaIndex deep comparison, RAG decision matrix (DIY vs Framework), Chunking strategies (Fixed/Semantic/Hierarchical/Sentence Window), LangChain Ecosystem (LangChain/LangGraph/LangSmith/LangServe/Hub), LlamaIndex Ecosystem (LlamaHub/LlamaParse/LlamaTrace), LCEL syntax, IngestionPipeline + Cache, QueryEngine, Metadata Filters, ParadeDB + Dragonfly architecture, Sellsuki use case
- อัปเดต index.md (5 sources, 3 concepts ใหม่)

## [2026-04-13] ingest | raw/notes/central-component — Sellsuki Components Library (4 files)

- สร้าง source pages: sellsuki-design-tokens-service, sellsuki-design-tokens-flow, sellsuki-design-tokens-reference, sellsuki-components-docs (4 ไฟล์)
- สร้าง concept pages: design-tokens, web-components-lit (2 ไฟล์)
- ครอบคลุมเนื้อหา: Design Tokens 2-tier architecture, CSS Variables vs TypeScript Types, Sellsuki spacing system reference (21 levels), Web Components + Lit Framework, 3 Core Systems (Theme/I18n/Toast), Component catalog 30+
- อัปเดต index.md (4 sources, 2 concepts ใหม่)

## [2026-04-13] ingest | raw/notes/arona — Arona RAG System (29 files)

- สร้าง source pages: arona-overview, arona-rag-techniques, arona-vs-langchain, arona-tool-call-agent, arona-stateless-rag, arona-sellsuki-analysis, arona-deepdive, arona-learn-series (8 ไฟล์)
- สร้าง concept pages: rag-retrieval-augmented-generation, hybrid-search-bm25-vector, semantic-caching, stateless-rag-design (4 ไฟล์)
- ครอบคลุมเนื้อหา: RAG architecture, Hybrid Search (BM25+Vector), Semantic Caching, Stateless Design, Tool Calling, Agent Loop, Arona vs LangChain comparison, Sellsuki use case
- อัปเดต index.md (8 sources, 4 concepts ใหม่)

## [2026-04-13] ingest | raw/notes/ai-context — AI-Assisted Development with Claude Code (8 files)

- สร้าง source pages: ai-context-phase1 ถึง ai-context-phase8 (8 ไฟล์)
- สร้าง concept pages: claude-code-ai-cli, ai-agents-system, commands-and-skills, mcp-model-context-protocol, ai-hooks-system, ai-dispatch-system, deterministic-ai-generation, clean-architecture-go
- อัปเดต index.md (8 sources, 8 concepts ใหม่)

## [2026-04-13] ingest | raw/notes/otel — OpenTelemetry & Distributed Tracing (6 files)

- สร้าง source pages: distributed-tracing-context-propagation, otel-trace-propagation-migration-guide, opentelemetry-deep-dive, opentelemetry-deep-dive-part2, fiber-context-otel-bug, settextmappropagator-explained
- สร้าง concept pages: distributed-tracing, context-propagation, opentelemetry, w3c-trace-context, otel-baggage, otel-go-instrumentation
- อัปเดต index.md (6 sources, 6 concepts)

## [2026-04-13] setup | สร้าง wiki ครั้งแรก

- สร้างโครงสร้าง directory ทั้งหมด
- สร้าง CLAUDE.md (schema)
- สร้าง index.md และ log.md
- โดเมน: research & notes (Thai primary)
- Page types: concepts, books, sources, synthesis
- Sources: Obsidian Web Clipper + manual notes
