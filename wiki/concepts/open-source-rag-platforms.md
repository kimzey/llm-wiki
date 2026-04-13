---
title: "Open Source RAG Platforms"
type: concept
tags: [rag, open-source, ragflow, dify, anythingllm, danswer, privatgpt, langflow, flowise, comparison]
sources: [wiki/sources/open-source-rag-platforms-comparison, wiki/sources/openrag-platform-overview, wiki/sources/open-source-rag-platforms-research]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/openrag-platform.md, wiki/concepts/langflow-visual-workflow.md, wiki/concepts/langchain-framework.md, wiki/concepts/llamaindex-framework.md]
created: 2026-04-13
updated: 2026-04-14
---

## สรุปสั้น

Open Source RAG Platforms แบ่งเป็น 3 ประเภท: RAG Application (พร้อมใช้เลย), Low-Code Builder (visual pipeline), และ Framework (เขียน code) — แต่ละประเภทมี trade-off ระหว่างความง่ายและความยืดหยุ่น

## อธิบาย

```
ง่าย/พร้อมใช้ ←────────────────────→ ยืดหยุ่น/ต้องเขียน

RAG Application     Low-Code Builder    Framework
(RAGFlow, Dify,     (Flowise, Langflow) (LangChain, LlamaIndex,
AnythingLLM, etc.)                       Haystack)

deploy ได้เลย       ลาก วาง            เขียน code เต็มที่
custom จำกัด        custom ปานกลาง      custom ได้ทุกอย่าง
```

## ประเด็นสำคัญ

### Type A: RAG Applications

**RAGFlow** (30k+ stars, by InfiniFlow)
- จุดเด่น: **Document Layout Analysis ดีที่สุด** — ตาราง, multi-column PDF, รูปภาพ
- Template chunking: Naive/Book/Paper/QA/Table/Law/Resume/Presentation/One
- Hybrid search (Vector + BM25) + Reranking ในตัว
- ข้อเสีย: Docker image ~10GB+, RAM สูง, documentation ยังไม่สมบูรณ์

**Dify** (55k+ stars, by LangGenius — ใหญ่ที่สุด)
- จุดเด่น: **ครบที่สุด** — Chatbot + Agent + Workflow + Analytics + Cloud hosted
- Parent-Child chunking (v0.8+), หลาย Vector DB (Qdrant/Weaviate/pgvector/Milvus)
- ข้อเสีย: ซับซ้อน, overengineered สำหรับ use case เล็ก

**AnythingLLM**
- จุดเด่น: **Desktop App** — ติดตั้งเหมือน app ปกติ, privacy 100% local
- ข้อเสีย: Retrieval เบาสุด (Vector only, ไม่มี BM25)

**Danswer / Onyx** (Enterprise-grade)
- จุดเด่น: **Connector เยอะที่สุด** — Confluence, Jira, Slack, GitHub, Notion, Salesforce
- Hybrid search + Reranking, Architecture: Vespa + PostgreSQL
- ข้อเสีย: Setup ซับซ้อนมาก

**PrivateGPT**
- จุดเด่น: **100% local** — ไม่ส่งข้อมูลออกเลย
- ข้อเสีย: Retrieval เบาสุด, ไม่มีจุดเด่นด้าน quality

**Kotaemon** (Research-grade)
- Hybrid search + Reranking + GraphRAG optional
- เหมาะ researcher ที่ต้องการ transparency ใน RAG pipeline

### Type B: Low-Code Builders

**Flowise & Langflow**
- สร้าง RAG/Agent pipeline แบบ visual (drag & drop)
- Export เป็น API ได้
- Langflow built on LangChain, Flowise ยืดหยุ่นกว่า

### เปรียบเทียบ Chunking

| Platform | Fixed | Semantic | Hierarchical | Structure | Custom |
|----------|-------|----------|-------------|-----------|--------|
| RAGFlow | ✅ | ❌ | ❌ | ✅ Best | ❌ |
| Dify | ✅ | ❌ | ✅ v0.8 | ❌ | ❌ |
| AnythingLLM | ✅ | ❌ | ❌ | ❌ | ❌ |
| Danswer | ✅ | ✅ | ❌ | ❌ | ❌ |
| LlamaIndex | ✅ | ✅ | ✅ | ✅ | ✅ |
| LangChain | ✅ | ✅ | ❌ | ✅ | ✅ |

### เปรียบเทียบ Retrieval

| Platform | Vector | BM25 | Hybrid | Reranking | Multi-Query |
|----------|--------|------|--------|-----------|-------------|
| RAGFlow | ✅ | ✅ | ✅ | ✅ | ❌ |
| Dify | ✅ | ✅ | ✅ | ✅ | ❌ |
| Danswer | ✅ | ✅ | ✅ | ✅ | ❌ |
| LangChain | ✅ | ✅ | ✅ | ✅ | ✅ |
| LlamaIndex | ✅ | ✅ | ✅ | ✅ | ✅ |

### เปรียบเทียบ Architecture (Vector Store)

| Platform | Vector Store | Database |
|----------|-------------|----------|
| RAGFlow | Elasticsearch + InfinityDB | MySQL |
| Dify | Qdrant/Weaviate/pgvector/Milvus | PostgreSQL |
| AnythingLLM | LanceDB/Chroma/Pinecone | SQLite |
| Quivr | pgvector (Supabase) | Supabase |
| Danswer/Onyx | Vespa | PostgreSQL |
| OpenRAG | OpenSearch | — |

## ตัวอย่าง / กรณีศึกษา

### Decision Guide

```
เอกสารซับซ้อน (ตาราง, multi-column PDF)  → RAGFlow
เอกสารทั่วไป + UI ดี + cloud hosting    → Dify
Privacy 100% บน desktop                 → AnythingLLM
Enterprise connector เยอะ (Slack/Confluence) → Danswer/Onyx

ต้อง custom pipeline + visual builder   → Flowise / Langflow
ต้อง custom เต็มที่ + RAG quality สูง  → LlamaIndex + FastAPI
ต้อง custom เต็มที่ + Agent complex    → LangChain + LangServe
Production enterprise + compliance      → Haystack / เขียนเอง
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — core concept ที่ทุก platform implement
- [[wiki/concepts/openrag-platform|OpenRAG Platform]] — Sellsuki's choice (Type B + custom)
- [[wiki/concepts/langflow-visual-workflow|Langflow]] — Low-code builder (Type B)
- [[wiki/concepts/langchain-framework|LangChain]] — Framework (Type C)
- [[wiki/concepts/llamaindex-framework|LlamaIndex]] — Framework (Type C) — chunking ครบที่สุด

## เปรียบเทียบ RAGFlow vs Dify vs AnythingLLM vs Flowise (2026)

| | RAGFlow | Dify | AnythingLLM | Flowise |
|--|---------|------|-------------|---------|
| RAG ซับซ้อน | ⭐⭐⭐ (RAPTOR, self-RAG) | ⭐⭐ | ⭐ | ⭐⭐ |
| ใช้งานง่าย | ⭐ | ⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Agent support | ⭐⭐ | ⭐⭐⭐ | ⭐ | ⭐⭐ |
| Complex docs | ⭐⭐⭐ | ⭐⭐ | ⭐ | ⭐ |

Decision: เอกสาร PDF ซับซ้อน → RAGFlow · Ship เร็ว → Dify · Non-tech team → AnythingLLM · Custom workflows → Flowise

## แหล่งที่มา

- [[wiki/sources/rag/open-source-rag-platforms-comparison|Open Source RAG Platforms Comparison]]
- [[wiki/sources/rag/open-source-rag-platforms-research|Open Source RAG Platforms Research 2026]]
