# Log

บันทึกการดำเนินการทั้งหมดใน wiki นี้ — เรียงตามเวลา ไม่ลบ ไม่แก้ไข

---

## [2026-04-13] query | central-component คืออะไร — Sellsuki Web Components Library

- ตอบคำถามจาก: wiki/concepts/web-components-lit, wiki/concepts/design-tokens, wiki/sources/sellsuki-components-docs
- ครอบคลุม: Tech Stack, 3 Core Systems (Theme/I18n/Toast), Design Tokens, Shadow DOM, Component Catalog

## [2026-04-13] ingest | raw/notes/network-fundamentals — พื้นฐาน Network & Infrastructure

- สร้าง source pages: network-fundamentals (1 ไฟล์)
- สร้าง concept pages: ip-address-networking, port-networking, dns, vpn-tailscale, cloudflare-tunnel, ssl-tls, reverse-proxy (7 ไฟล์ใหม่)
- อัปเดต concept pages: docker-containers (เพิ่ม Docker Network section + sources/related), server-security (เพิ่ม security levels comparison + related links) (2 ไฟล์)
- ครอบคลุมเนื้อหา: IP Address (Public/Private, Private ranges 192.168/10/172.16/100.64), Port & Well-known Ports, DNS, NAT, Port Forwarding (วิธีการ + ความเสี่ยง), 3 วิธีเข้า server จากนอกบ้าน, Tailscale Mesh VPN (WireGuard), Cloudflare Tunnel (Reverse Tunnel), SSL/HTTPS (Let's Encrypt + Cloudflare), Reverse Proxy (Nginx), Docker Network isolation, UFW Firewall, Security level comparison (2-10/10)
- อัปเดต index.md (1 source ใหม่, 7 concepts ใหม่)

## [2026-04-13] ingest | raw/notes/homeserver-knowledge — Home Server Admin

- สร้าง source pages: homeserver-admin-knowledge (1 ไฟล์)
- สร้าง concept pages: docker-containers, linux-system-administration, server-security, backup-strategy, systemd-services (5 ไฟล์)
- ครอบคลุมเนื้อหา: Linux Command Line (files, permissions, processes, networking), Docker (containers, compose, images, volumes), Systemd (services, journalctl, timers), Security (SSH hardening, UFW, Fail2ban, automatic updates), Backup Strategy (3-2-1 rule, scripts, rclone, cloud sync), Log Management, Performance Tuning, Crontab, Environment Variables, Git for Config Management, SSL Certificates, Troubleshooting, Monitoring, Update Strategy, Gotchas, Production Checklist
- อัปเดต index.md (1 source, 5 concepts ใหม่)

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

## [2026-04-13] ingest | raw/notes/openrag — remaining files (round 2)

- สร้าง source pages ใหม่ 8 ไฟล์:
  - openrag-document-update-guide (จาก docs-lean/data-update-guide.md)
  - openrag-ingestion-paths (จาก docs-lean/ingestion-flow-explained.md + ingestion-two-paths-explained.md + md-file-and-splitter-analysis.md)
  - openrag-access-control-rbac (จาก docs-lean/openrag-allowed-groups-google.md + pipeline-and-rbac-deep-dive.md)
  - openrag-data-ingestion-channels (จาก docs-lean/openrag-data-ingestion-guide.md)
  - openrag-flow-payloads (จาก docs-lean/openrag-flows-payload-permission.md + openrag-langflow-flows-guide.md)
  - openrag-organization-deployment (จาก docs-lean/openrag-organization-guide.md + org-rag-guide.md)
  - openrag-sdk-reference (จาก docs-lean/sdk.md)
  - open-source-rag-platforms-comparison (จาก open-rag-deep-dive.md)
- สร้าง concept pages ใหม่ 1 ไฟล์:
  - open-source-rag-platforms (RAGFlow, Dify, AnythingLLM, Danswer, PrivateGPT, Kotaemon — comparison tables)
- อัปเดต concept pages:
  - openrag-platform (เพิ่ม 8 Ingestion Channels, Document Update Mechanism, SDK overview, RBAC Gap analysis)
  - rag-chunking-strategies (เพิ่ม CharacterTextSplitter analysis)
- ครอบคลุมเนื้อหา: SHA256 hash deduplication, Delete-All-Reindex-All update mechanism, CharacterTextSplitter (separator="\n") analysis vs RecursiveCharacterTextSplitter, MD file table/code cutting issues, 2 ingestion paths (Langflow vs Backend Python), 8 ingestion channels (UI/S3/Google Drive/OneDrive/SharePoint/URL/REST API), 4 Langflow flows payload structure, DLS policy gap (allowed_groups ไม่ถูก enforce), 3 RBAC fix options, Organization setup (.env/Onboarding Wizard/maintenance), SDK 0.2.0 (TypeScript/Python), Open Source RAG platform comparison (3 types + decision guide)
- ข้ามไฟล์ที่ซ้ำกัน (LangChain): langchain_basics.md, langchain_component_explained.md, langchain_full.md, langchain-llamaindex-deep-dive.md (เนื้อหาซ้ำกับ wiki ที่มีอยู่แล้ว)
- อัปเดต index.md (8 sources + 1 concept ใหม่)

## [2026-04-13] ingest | raw/notes/openrag — OpenRAG Platform (31 files)

- สร้าง source pages ใหม่ 9 ไฟล์:
  - openrag-platform-overview (จาก README + openrag_guide.md + phase1-overview.md)
  - openrag-docling-parser (จาก phase2-docling.md)
  - openrag-opensearch-vector-db (จาก phase3-opensearch.md)
  - openrag-langflow-workflow (จาก phase4-langflow.md + openrag-backend-vs-langflow.md)
  - openrag-integration-scenarios (จาก phase5-integration.md)
  - openrag-config-cost-models (จาก advanced-config.md + cost-and-models.md)
  - openrag-chunking-ingestion (จาก chunking.md + document-best-practices.md)
  - openrag-cache-analysis (จาก openrag-cache-analysis.md)
  - openrag-rag-spike-research (จาก spike-rag-research-summary.md)
- สร้าง concept pages ใหม่ 3 ไฟล์:
  - openrag-platform, docling-document-parser, langflow-visual-workflow
- อัปเดต concept pages: rag-chunking-strategies (เพิ่ม OpenRAG multi-layer chunking), rag-retrieval-augmented-generation (เพิ่ม OpenRAG example)
- ครอบคลุมเนื้อหา: OpenRAG architecture (5 services: FastAPI/Langflow/OpenSearch/Docling/Next.js), Document ingestion flow, Chat/Query flow, Access Control (RBAC + Document-level DLS), KNN vector search (disk_ann), Langflow Visual Workflow, multi-layer chunking (Docling page-aware + SplitText + Token Batching), cost analysis (gpt-4o-mini ~$9/เดือน/50 users), Ollama local LLM setup, config.yaml structure, Spike research (pgvector vs OpenSearch vs Qdrant), cache gap analysis
- ไฟล์ที่ยังไม่ได้ ingest ใน folder นี้ (remaining): openrag-allowed-groups-google.md, pipeline-and-rbac-deep-dive.md, org-rag-guide.md, openrag-data-ingestion-guide.md, ingestion-two-paths-explained.md, sdk.md, open-rag-deep-dive.md, langchain_basics.md (LangChain files — likely overlap กับ wiki ที่มีอยู่)
- อัปเดต index.md (9 sources + 3 concepts ใหม่)

## [2026-04-13] ingest | raw/notes/openrag — complete large files (round 3)

- อ่านส่วนที่เหลือของ 4 ไฟล์ขนาดใหญ่ที่อ่านได้ไม่ครบใน round 2:
  - `openrag-langflow-flows-guide.md` (บรรทัด 200–1096) — 3 custom UI methods, OAuth flow, API Key setup, Custom Flow creation steps, React Chat App example, Python integration script
  - `org-rag-guide.md` (บรรทัด 100–1215) — 5 ingestion methods, RAG usage via API, Public API v1 endpoints ครบ, Data-Level ACL (3 levels), Group management, LLM/Embedding selection, Chunk size guide, Backup & Recovery, Security checklist, Scaling, 5 enterprise use cases
  - `pipeline-and-rbac-deep-dive.md` (บรรทัด 300–540) — Custom JWT enrichment code, Google Admin API requirements, Google Workspace integration step-by-step, RBAC checklist
  - `sdk.md` (บรรทัด 230–1669) — Streaming patterns ครบ (ChatStream, text-only, finalText), Multi-turn conversation, Conversation management API, All parameter tables, Settings API, Knowledge Filters CRUD, Models API, Error handling hierarchy + retry pattern, 5 use cases, Complete Type Reference (Pydantic + TypeScript)
- อัปเดต source pages:
  - openrag-flow-payloads.md (เพิ่ม 3 custom UI methods + auth modes + custom flow creation)
  - openrag-sdk-reference.md (เพิ่ม streaming patterns, conversation management, settings/models/error handling, type reference)
  - openrag-organization-deployment.md (เพิ่ม 5 ingestion methods, permissions table, LLM/embedding selection, backup, security checklist, scaling, enterprise use cases)
  - openrag-access-control-rbac.md (เพิ่ม Google Admin API code, Workspace integration steps, RBAC checklist)
- ไม่ต้องอัปเดต index.md (ไม่มี page ใหม่ — เป็นการอัปเดต page ที่มีอยู่แล้ว)

## [2026-04-14] ingest | Sellsuki RAG Agent - Complete Implementation Guide

- สร้าง: wiki/sources/sellsuki-rag-complete-guide.md
- สร้าง concept ใหม่: wiki/concepts/pgvector.md
- อัปเดต concepts: rag-retrieval-augmented-generation.md, rag-chunking-strategies.md
- Concepts: pgvector, RAG pipeline, chunking, embedding, pgvector indexes (HNSW/IVFFlat), LangChain, LlamaIndex, FastAPI Agent, deployment, cost estimation

## [2026-04-14] ingest | 06-sellsuki-agent-guide.md

- สร้าง: wiki/sources/sellsuki-agent-guide-06.md
- หมายเหตุ: ไฟล์นี้มีเนื้อหาเหมือนกันกับ sellsuki-rag-complete-guide ทุกประการ
- ไม่มี concept ใหม่หรืออัปเดตเพิ่มเติม

## [2026-04-14] ingest | 07-sellsuki-agent-plan-v2.md

- สร้าง: wiki/sources/sellsuki-agent-plan-v2.md
- อัปเดต concepts: hybrid-search-bm25-vector.md, semantic-caching.md
- Concepts: cost optimization (3 strategies), semantic cache 0.95 threshold, diff-based incremental indexing, ParadeDB (BM25+pgvector), Dragonfly cache, multi-provider LLM fallback (Groq→Gemini→GPT), co-location strategy, anti-abuse (PoW)

## [2026-04-14] ingest | rag-complete-guide.md

- สร้าง: wiki/sources/rag-complete-guide-comprehensive.md
- อัปเดต concepts: rag-evaluation.md
- Concepts: RAG pipeline 8 ขั้นตอน, chunking 7 strategies spectrum, retrieval methods (vector/BM25/hybrid/reranking/multi-query), RAGAS 4 metrics, deployment options (no-code/low-code/framework/DIY), framework comparison (LangChain/LlamaIndex/Haystack)

## [2026-04-14] ingest | rag-deep-dive.md (LangChain)

- สร้าง: wiki/sources/rag-deep-dive-langchain.md
- อัปเดต concepts: langchain-framework.md, agentic-rag.md
- Concepts: LangChain document loaders, advanced retrievers (Multi-Query/ContextualCompression/ParentDocument/SelfQuery/CrossEncoderReranker/EnsembleRetriever), MMR search, Conversational RAG pattern, RAG Agent ด้วย LangGraph, LangSmith evaluation, FAQ ห้าม split, Gemini embedding ฟรี

## [2026-04-14] ingest | langchain_component_explained.md (LangFlow Custom Component Guide)

- สร้าง: wiki/sources/langflow-custom-component-guide.md
- อัปเดต concepts: langflow-visual-workflow.md (เพิ่ม Component Lifecycle detail, auto-binding rule, Input types table, debug tips, OpenRouter+Redis integration)
- Concepts touched: langflow-visual-workflow, langchain-framework, semantic-caching

## [2026-04-14] ingest | langchain_full.md (LangChain คู่มือสมบูรณ์)

- สร้าง: wiki/sources/langchain-full-reference.md
- สร้าง concept ใหม่: wiki/concepts/lcel-langchain-expression-language.md
- อัปเดต concepts: langchain-framework.md (เพิ่ม Package structure, LLM methods table, Output Parsers table, Memory options, Document Loaders)
- Concepts: LCEL (RunnablePassthrough/Lambda/Parallel/.assign()/.bind()), Output Parser types, Memory backends (InMemory/Redis/SQL)

## [2026-04-14] ingest | langchain_basics.md (LangChain พื้นฐาน)

- สร้าง: wiki/sources/langchain-basics-openrag.md
- Concepts updated: langchain-framework (via LCEL concept cross-link), lcel-langchain-expression-language (added as source)
- Concepts touched: langchain-framework, lcel-langchain-expression-language, rag-retrieval-augmented-generation, ai-agent

## [2026-04-14] ingest | langchain-llamaindex-deep-dive.md (OpenRAG version)

- สร้าง: wiki/sources/openrag-langchain-llamaindex-deepdive.md
- อัปเดต concepts: llamaindex-framework.md (เพิ่ม QueryEngine types table, Postprocessors เพิ่มเติม, SentenceTransformerRerank)
- Concepts touched: langchain-framework, llamaindex-framework, lcel-langchain-expression-language, rag-chunking-strategies

## [2026-04-14] ingest | advanced-evaluation-deep-dive.md (LangChain/LangSmith)

- สร้าง: wiki/sources/rag-advanced-evaluation-deepdive.md
- สร้าง concept ใหม่: wiki/concepts/langsmith-tracing-evaluation.md
- อัปเดต concepts: rag-evaluation.md (เพิ่ม LLM-as-Judge section: Rule-based vs LLM-as-Judge, Judge Prompt Design, RAG Triad, Multi-dimension grading, Pitfalls, Agent evaluators)
- Concepts: LLM-as-Judge, RAG Triad (Context Relevance + Faithfulness + Answer Relevance), Pairwise Comparison, Position/Verbosity/Self-preference bias, CI/CD eval pipeline, LangSmith evaluator contract
