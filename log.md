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

## [2026-04-13] ingest | raw/notes/liamaindex — LlamaIndex Tutorial Series (4 files)

- สร้าง source pages: llamaindex-phase1-introduction, llamaindex-phase2-ingestion-indexing, llamaindex-phase3-querying-rag, llamaindex-phase4-agents-production (4 ไฟล์)
- อัปเดต concept pages: llamaindex-framework (เพิ่ม Settings object, Query/Chat/Retriever comparison, Response Modes, Postprocessors, Agents, Workflows, Multi-Modal, SQL, Fine-tuning, Production Checklist)
- ครอบคลุมเนื้อหา: LlamaIndex vs LangChain, 5 Building Blocks (Documents/Nodes, Loaders, Indexes, Query Engines, LLMs/Embeddings), Settings object (v0.10+), RAG Hello World, Query Pipeline, LlamaHub 100+ loaders (PDF/Web/Database/Notion/Google), Node Parsers (Sentence/Semantic/Hierarchical/Markdown/SimpleFile), IngestionPipeline + Cache, Index Types (Vector/Summary/Tree/Keyword/KnowledgeGraph), Vector Stores (Chroma/Pinecone/Weaviate/Qdrant), Persist & Load, Retrievers (Vector/BM25/Hybrid/AutoMerging/Recursive), Postprocessors (Similarity/Keyword/CohereRerank/LLMRerank), Response Modes (refine/compact/tree_summarize/accumulate), Query Engines (Retriever/SubQuestion/Router/MultiStep), Chat Engines (Simple/Context/CondensePlusContext/OpenAI), Advanced RAG (HyDE/Sentence Window/RAG Fusion), Evaluation (Faithfulness/Relevancy/Correctness), Observability (Arize Phoenix/Callbacks), Agents (ReAct/OpenAI/Tools), Workflows (event-driven), Multi-Modal (รูปภาพ+ข้อความ), SQL integration, Fine-tuning Embeddings, Caching Strategies, Async & Performance, FastAPI Integration, Production Checklist
- อัปเดต index.md (4 sources ใหม่)

## [2026-04-13] ingest | raw/notes/rag-knowledge — RAG Agent Complete Knowledge (12 files)

- สร้าง source pages: rag-knowledge-overview, rag-complete-knowledge, rag-glossary-data-prep, chunking-deepdive-complete, rag-vs-agent-explained, sellsuki-rag-diy-guide, rag-integration-guide-complete, rag-client-channels-complete (8 ไฟล์)
- สร้าง concept pages: llm-large-language-model, ai-agent, agentic-rag (3 ไฟล์)
- ครอบคลุมเนื้อหา: RAG & Agent knowledge base (12 docs, ~300 หน้า), LLM fundamentals, Prompt Engineering, Embedding, Vector Database, RAG pipeline, Advanced RAG (Hybrid Search, Re-ranking), Agent & Agentic RAG, Multi-Agent, Evaluation metrics, Production concerns, 10 วิธี Chunking, 14 ช่องทาง integration (LINE, Slack, Discord, etc.), MCP สำหรับ Claude Code/Cursor, DIY RAG สำหรับ Sellsuki
- อัปเดต index.md (8 sources, 3 concepts ใหม่)

## [2026-04-13] setup | สร้าง wiki ครั้งแรก

- สร้างโครงสร้าง directory ทั้งหมด
- สร้าง CLAUDE.md (schema)
- สร้าง index.md และ log.md
- โดเมน: research & notes (Thai primary)
- Page types: concepts, books, sources, synthesis
- Sources: Obsidian Web Clipper + manual notes

## [2026-04-13] ingest | raw/notes/LangChain — LangChain Complete Tutorial (17 files)

- สร้าง source pages: step1-langchain-basics, step2-tools-agents, step3-rag-tutorial, step4-langgraph-tutorial, step5-advanced-rag-tutorial, step6-multi-agent-tutorial, step7-evaluation-tutorial, step8-deployment-tutorial (8 ไฟล์ — tutorial steps 1-8)
- สร้าง source pages: langchain-advanced-guide, langchain-deep-dive, langgraph-deep-dive, langsmith-deep-dive, lang-ecosystem-complete, multi-agent-deep-dive, deployment-deep-dive, advanced-rag-deep-dive (8 ไฟล์ — deep-dive articles)
- อัปเดต concept pages: langchain-framework, ai-agent, agentic-rag (เพิ่ม sources จาก tutorial)
- ครอบคลุมเนื้อหา: LangChain พื้นฐาน (Model, Messages, Prompt Templates), Tools & Agents (@tool decorator, bind_tools, create_agent, ReAct Loop), RAG pipeline (Load→Split→Embed→Store→Retrieve→Generate), LangGraph (State, Nodes, Edges, Checkpointing, Human-in-the-Loop), Advanced RAG (Hybrid Search, Re-ranking, Multi-query, Conversational RAG), Multi-Agent Systems (Supervisor, Handoff, Pipeline patterns), Evaluation (Phoenix, LangFuse, Rule-based, LLM-as-Judge), Deployment (LangServe, FastAPI, Docker, Cloud Deploy), Lang Ecosystem (LangChain, LangGraph, LangSmith, LangServe, LangChain.js)
- อัปเดต index.md (16 sources ใหม่)
