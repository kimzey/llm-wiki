---
title: "RAG — Retrieval-Augmented Generation"
type: concept
tags: [rag, ai, retrieval, embeddings, vector-search, llm]
sources: [arona/README.md, arona/RAG_COMPARISON.md, arona/ARONA_RAG_COMPREHENSIVE_GUIDE.md, rag-decision-guide.md, llamaindex-deep-dive.md, langchain-rag-guide.md, openrag_guide.md, spike-rag-research-summary.md]
related: [wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/semantic-caching.md, wiki/concepts/stateless-rag-design.md, wiki/concepts/llamaindex-framework.md, wiki/concepts/langchain-framework.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/openrag-platform.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

RAG (Retrieval-Augmented Generation) คือเทคนิคที่ให้ AI ค้นหาข้อมูลจากเอกสารที่เรากำหนดก่อน แล้วนำมาประกอบการตอบคำถาม ทำให้ได้คำตอบที่ถูกต้องและอ้างอิงแหล่งที่มาได้จริง แทนที่จะพึ่ง training knowledge ของ model เพียงอย่างเดียว

## อธิบาย

AI models มีข้อจำกัด 2 อย่างหลัก:
1. **Knowledge cutoff** — ไม่รู้ข้อมูลหลัง training date
2. **Hallucination** — อาจสร้างข้อมูลที่ไม่มีจริง

RAG แก้ปัญหาด้วยการแบ่งเป็น 2 ระบบ:
- **Retrieval system** — ค้นหาเอกสารที่เกี่ยวข้องกับคำถาม
- **Generation** — AI อ่านเอกสารที่ค้นมาแล้วตอบคำถาม

### Pipeline หลัก

**1. Ingestion (การนำข้อมูลเข้า)**
```
เอกสาร → Chunk → Embed → Store ใน Vector DB
```

**2. Retrieval (การค้นหา)**
```
คำถาม → Embed → ค้นใน Vector DB → เอกสารที่เกี่ยวข้อง
```

**3. Generation (การสร้างคำตอบ)**
```
คำถาม + เอกสาร → LLM → คำตอบ + แหล่งอ้างอิง
```

## ประเด็นสำคัญ

### Chunking Strategies

ดูรายละเอียดที่ [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]

- **Fixed-Size**: แบ่งตาม chunk size (RecursiveCharacterTextSplitter) — เร็ว, ง่าย
- **Semantic**: แบ่งตามความหมาย — แม่นยำที่สุด แต่ช้า
- **Hierarchical/Parent-Child**: ค้นหา child เล็ก, ส่ง parent ใหญ่ให้ LLM — ดีที่สุดสำหรับ HR Policy
- **Sentence Window**: เก็บ context รอบข้างไว้ใน metadata — เหมาะกับ Q&A แบบ precise
- **Markdown Header**: แบ่งตาม `#`, `##`, `###` — เหมาะกับ docs

### Embedding
- แปลงข้อความเป็น vector (ชุดตัวเลข) ใน high-dimensional space
- ข้อความที่มีความหมายใกล้เคียงกันจะมี vector ใกล้กัน
- Arona ใช้ `text-embedding-3-small` (1536 มิติ) สร้าง 3 embeddings ต่อ chunk (content, title, filename)

### Search Methods
- **BM25 (Full-text)**: ค้นหาด้วย keyword — เร็ว, แม่นยำสำหรับคำที่ตรงกันพอดี
- **Vector (Semantic)**: ค้นหาด้วยความหมาย — ยืดหยุ่น, จับ paraphrase ได้
- **Hybrid**: รวมทั้งสอง — ดีที่สุดในทางปฏิบัติ

### RAG Architectures
| แบบ | ลักษณะ | เหมาะกับ |
|-----|--------|---------|
| Stateless | ไม่เก็บ session, client จัดการ history | Documentation search |
| Stateful | เก็บ conversation memory ที่ server | Conversational chatbot |
| Agentic | AI เลือก tools เอง, หลาย steps | Complex queries |

### RAG Implementation Approaches

ดูรายละเอียดที่ [[wiki/concepts/llamaindex-framework|LlamaIndex]] และ [[wiki/concepts/langchain-framework|LangChain]]

| วิธี | Time to Production | RAG Quality | Incremental Update | เหมาะกับ |
|------|-------------------|-------------|-------------------|---------|
| **LlamaIndex** | 2-4 สัปดาห์ | ดีที่สุด | ✅ Built-in Cache | RAG core use case, ต้องการ metadata filter ดี |
| **LangChain** | 3-6 สัปดาห์ | ดี | Manual | ต้องการ Agent ซับซ้อน, LangSmith observability |
| **DIY** | 3-4 เดือน | ขึ้นกับทีม | เขียนเอง | Long-term, cost-sensitive, custom requirement |
| **Off-the-shelf** | 1-3 วัน | ปานกลาง | ตาม platform | ไม่มี dev team, จ้าย SaaS |

### แนวทางสร้าง RAG ระบบ

| วิธี | Cost | Flexibility | Time-to-Market | เหมาะกับ |
|------|------|-------------|----------------|---------|
| DIY (เช่น Arona) | ต่ำสุด (~$21/เดือน) | 100% | 3-4 สัปดาห์ | Long-term, cost-sensitive |
| LangChain/LlamaIndex | ปานกลาง | 70-80% | 1-2 สัปดาห์ | POC, ทีมใหม่ |
| Off-the-shelf | สูงสุด ($250-1000/เดือน) | 30% | 1-3 วัน | ไม่มี dev team |

### Cost Optimization Techniques
- **Semantic caching**: ใช้คำตอบเก่าถ้า similarity ≥ 90% — ดู [[wiki/concepts/semantic-caching|Semantic Caching]]
- **Incremental indexing**: embed เฉพาะที่เปลี่ยน — LlamaIndex IngestionCache ทำ auto
- **Summarized retrieval**: index summary แทน full content, อ่าน full page เมื่อต้องการ
- **Small model** สำหรับ query normalization, **large model** สำหรับ generation
- **Embedding cache**: ใช้ Dragonfly cache embedding vectors แทนเรียก OpenAI ซ้ำ

## ตัวอย่าง / กรณีศึกษา

**OpenRAG** — Complete self-hosted RAG platform (Sellsuki use case)
- Stack: FastAPI + Langflow + OpenSearch + Docling + Next.js
- ฟีเจอร์: Document-level RBAC, multi-LLM (OpenAI/Claude/Ollama), Knowledge Filters, Google Drive connector
- Evaluation: built-in faithfulness/relevancy/precision/recall metrics
- ดู [[wiki/concepts/openrag-platform|OpenRAG Platform]] สำหรับรายละเอียด

**Arona** — RAG สำหรับ Elysia documentation
- Cost: ~$21/เดือน (เทียบกับ $250-1,000 ของ SaaS)
- Throughput: 750 req/day, cache hit rate 85%+
- Stack: ParadeDB (BM25+Vector) + DragonflyDB (Cache) + GPT OSS 120B via OpenRouter

**Sellsuki RAG (ด้วย LlamaIndex)** — Internal company knowledge base
- Stack: ParadeDB + Dragonfly + Docs Sellsuki (Outline) + LlamaIndex + GPT-4o
- ตอบคำถาม HR policy, business rules, procedures
- รองรับ LINE Bot + Slack + Web chat + Cursor MCP
- IngestionPipeline + Cache → incremental update auto
- Metadata filter: department, access_level (public/internal/confidential)
- Hybrid search: HNSW (vector) + BM25 (full-text) + RRF fusion

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]] — วิธีค้นหาในขั้นตอน Retrieval (BM25 + Vector)
- [[wiki/concepts/semantic-caching|Semantic Caching]] — การ cache ด้วย vector similarity
- [[wiki/concepts/stateless-rag-design|Stateless RAG Design]] — ปรัชญาการออกแบบสำหรับ documentation search
- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]] — Data framework สำหรับ RAG ที่ดีที่สุด
- [[wiki/concepts/langchain-framework|LangChain Framework]] — General-purpose LLM framework
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]] — วิธีแบ่ง chunks ทั้งหมด

## แหล่งที่มา

- [[wiki/sources/arona-overview|Arona System Overview]]
- [[wiki/sources/arona-rag-techniques|RAG Techniques & Comparison]]
- [[wiki/sources/arona-stateless-rag|Stateless RAG Explained]]
- [[wiki/sources/rag-decision-guide|RAG Framework Decision Guide — Sellsuki]]
- [[wiki/sources/llamaindex-deep-dive|LlamaIndex & RAG Deep Dive]]
- [[wiki/sources/langchain-rag-guide|LangChain, RAG & Sellsuki Knowledge Base]]
- [[wiki/sources/openrag-platform-overview|OpenRAG Platform Overview]]
- [[wiki/sources/openrag-rag-spike-research|RAG Spike Research]]

## จาก Sellsuki RAG Agent - Complete Implementation Guide

### Implementation Timeline แบบ 6 สัปดาห์

```
Week 1-2: เตรียม Data + Chunking
Week 3:   Setup pgvector + Insert
Week 4:   สร้าง Agent + API
Week 5:   Deploy + Testing
Week 6:   Fine-tune + UAT + Launch
```

### Cost Estimation (Internal Use ~$20-70/เดือน)
- Embedding (text-embedding-3-small): ~$0.5-2/เดือน
- LLM (gpt-4o-mini, 1000 queries): ~$5-20/เดือน
- Database (self-hosted): ~$5-10/เดือน
- Hosting (Cloud Run/VPS): ~$5-15/เดือน

### pgvector เป็น Vector DB ทางเลือก

นอกจาก ParadeDB, OpenSearch ที่เคยพูดถึง — pgvector บน PostgreSQL ปกติ ก็เป็นตัวเลือกที่เหมาะสมสำหรับทีมที่ใช้ PostgreSQL อยู่แล้ว ดู [[wiki/concepts/pgvector|pgvector]]

### Deployment Options
- **Google Cloud Run**: แนะนำ — auto scale, integrate กับ Cloud SQL ได้
- **Docker Compose + VPS**: ถูกสุด self-hosted
- **Railway/Render**: ง่ายสุด สำหรับทีมเล็ก

- [[wiki/sources/sellsuki-rag-complete-guide|Sellsuki RAG Agent - Complete Implementation Guide]]
