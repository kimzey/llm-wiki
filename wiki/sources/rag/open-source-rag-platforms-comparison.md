---
title: "Open Source RAG Platforms — Deep Dive Comparison"
type: source
source_file: raw/notes/openrag/open-rag-deep-dive.md
published: 2026-04-13
tags: [rag, ragflow, dify, anythingllm, quivr, danswer, privatgpt, open-source, comparison]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/langflow-visual-workflow.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/frameworks/openrag/open-rag-deep-dive.md|Original file]]

## สรุป

เปรียบเทียบ Open Source RAG Platforms ทั้งหมดที่พร้อม deploy — RAGFlow, Dify, AnythingLLM, Quivr, Danswer/Onyx, PrivateGPT, Kotaemon, Verba, Flowise, Langflow — พร้อม framework อย่าง LangChain/LlamaIndex/Haystack

## ประเด็นสำคัญ

### 3 ประเภท Open Source RAG

**Type A: RAG Application (พร้อมใช้เลย)**
— มี UI + API + RAG Pipeline ครบ ไม่ต้องเขียน code
RAGFlow, Dify, AnythingLLM, Quivr, Danswer/Onyx, PrivateGPT, Kotaemon, Verba

**Type B: Low-Code Builder**
— UI visual pipeline, ลาก-วาง component
Flowise, Langflow

**Type C: Framework (เขียน Code)**
— Library เต็มรูปแบบ ยืดหยุ่นสูงสุด
LangChain, LlamaIndex, Haystack

### Platform Deep Dive

**RAGFlow** (GitHub: infiniflow/ragflow, 30k+ stars)
- จุดเด่น: **Deep Document Understanding** — Layout analysis, table extraction, multi-column PDF
- Template-based chunking: Naive/Book/Paper/Manual/QA/Table/Resume/Law/Presentation
- Retrieval: Vector + BM25 + Hybrid + Reranking
- Architecture: Elasticsearch + InfinityDB + MinIO + MySQL
- ข้อเสีย: image ~10GB+, documentation ไม่สมบูรณ์, Agent ไม่แข็งแรง

**Dify** (GitHub: langgenius/dify, 55k+ stars — ใหญ่ที่สุด)
- จุดเด่น: **ครบที่สุด** — Chatbot + Agent + Workflow + Analytics
- LLM หลายตัว, Vector DB หลายตัว (Qdrant/Weaviate/pgvector/Milvus)
- Parent-Child chunking ตั้งแต่ v0.8+
- มี Cloud hosted (dify.ai)
- ข้อเสีย: ซับซ้อน, overengineered สำหรับ use case เล็กๆ

**AnythingLLM** (ง่ายที่สุด)
- จุดเด่น: **Desktop App** — ติดตั้งเหมือน app ปกติ ไม่ต้อง Docker
- Privacy 100% (ทำงาน local ทั้งหมด)
- Vector DB: LanceDB/Chroma/Pinecone/Qdrant
- ข้อเสีย: Retrieval เบาสุด (Vector only, ไม่มี BM25/Hybrid)

**Danswer / Onyx** (Enterprise-grade)
- จุดเด่น: **connector เยอะที่สุด** — Confluence, Jira, Slack, GitHub, Notion, Salesforce, etc.
- Hybrid search (Vector + BM25), Reranking
- Architecture: Vespa + PostgreSQL
- ข้อเสีย: Setup ซับซ้อนมาก

**PrivateGPT** (Privacy-first)
- จุดเด่น: **100% local** — ไม่ส่งข้อมูลออก internet เลย
- ง่ายสุดสำหรับ local setup
- ข้อเสีย: Retrieval เบาสุด, Chunking พื้นฐาน, UI น้อย

**Kotaemon** (Research-grade)
- จุดเด่น: Hybrid search, Reranking, GraphRAG optional
- เหมาะกับ researcher ที่ต้องการ transparency ใน RAG pipeline

### เปรียบเทียบ Chunking Support

| Platform | Fixed | Semantic | Hierarchical | Structure | Custom |
|----------|-------|----------|-------------|-----------|--------|
| RAGFlow | ✅ | ❌ | ❌ | ✅ Best (template) | ❌ |
| Dify | ✅ | ❌ | ✅ v0.8+ | ❌ | ❌ |
| LlamaIndex | ✅ | ✅ | ✅ | ✅ | ✅ (ครบที่สุด) |
| LangChain | ✅ | ✅ | ❌ | ✅ | ✅ |
| Langflow | ✅ | ✅ | ❌ | ✅ | ❌ |

### เปรียบเทียบ Retrieval

| Platform | Vector | BM25 | Hybrid | Reranking | Multi-Query | Metadata |
|----------|--------|------|--------|-----------|-------------|---------|
| RAGFlow | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| Dify | ✅ | ✅ | ✅ | ✅ Cohere/Jina | ❌ | ✅ |
| Danswer/Onyx | ✅ | ✅ | ✅ | ✅ | ❌ | ✅ |
| LangChain | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| LlamaIndex | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |

### Decision Guide

```
ไม่เขียน code + เอกสารซับซ้อน (ตาราง, layout)  → RAGFlow
ไม่เขียน code + เอกสารทั่วไป + UI ดีสุด       → Dify
Privacy 100% บน desktop                       → AnythingLLM
Enterprise หลายแหล่งข้อมูล (Slack/Confluence) → Danswer/Onyx

ต้อง custom ปานกลาง (visual)                  → Flowise / Langflow
ต้อง custom เต็มที่ + RAG คุณภาพสูง           → LlamaIndex + FastAPI
ต้อง custom เต็มที่ + Agent + Tools           → LangChain + LangServe
Enterprise production + compliance            → Haystack / เขียนเอง
```

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

- Langflow ไม่มี Hierarchical chunking ในตัว (ต้องใช้ LlamaIndex ถ้าต้องการ)
- Dify ใหญ่กว่า LangChain ใน GitHub stars (55k vs ~90k แต่ Dify โตเร็วกว่า)

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — core concept
- [[wiki/concepts/openrag-platform|OpenRAG Platform]] — one of the platforms compared
- [[wiki/concepts/langflow-visual-workflow|Langflow]] — Type B platform
