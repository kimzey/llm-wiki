---
title: "Sellsuki RAG Agent - Complete Implementation Guide"
type: source
source_file: raw/notes/rag-knowledge/Sellsuki RAG Agent - Complete Implementation Guide.md
url: ""
published: 2026-01-01
tags: [rag, sellsuki, pgvector, agent, implementation, langchain, llamaindex, fastapi, embedding]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/agentic-rag.md, wiki/concepts/langchain-framework.md]
created: 2026-04-14
updated: 2026-04-14
---

**Full source**: [[../../raw/notes/rag-knowledge/Sellsuki RAG Agent - Complete Implementation Guide.md|Original file]]

## สรุป

คู่มือการสร้าง RAG Agent สำหรับบริษัท Sellsuki แบบครบวงจร ครอบคลุมตั้งแต่การเตรียม data, chunking & embedding, การตั้งค่า pgvector, การเปรียบเทียบ RAG frameworks, การสร้าง Agent พร้อม API, ไปจนถึงการ deploy บน production — พร้อม code ตัวอย่างสำหรับทุก phase

## ประเด็นสำคัญ

### Architecture ภาพรวม

```
Company Documents → Chunking & Embedding → pgvector (PostgreSQL) → RAG Agent
                                                                         ↓
                                    User ถาม → Agent ค้น Vector DB → LLM ตอบ
```

Flow 5 ขั้นตอน: รวบรวมเอกสาร → chunk → embed → เก็บใน pgvector → ค้นหาตอบคำถาม

### Phase 1: เตรียม Data

ประเภทเอกสารที่รองรับ: Company Policy (PDF/Docs), Product Info (Markdown/HTML), FAQ (CSV/JSON), SOP/Process, HR Info, Meeting Notes, Knowledge Base

Extractors: PyPDF2 (PDF), python-docx (Word), BeautifulSoup (HTML), pandas (CSV/Excel), notion-client (Notion)

Metadata สำคัญ: `source`, `category`, `department`, `doc_type`, `last_updated`, `title`

### Phase 2: Chunking & Embedding

**Chunking Strategies:**
- Fixed Size: หั่นตาม characters/tokens — ง่าย เร็ว
- Recursive: หั่นตาม `\n\n` → `\n` → space → char — **แนะนำสำหรับ Sellsuki**
- Semantic: หั่นตามความหมาย (ใช้ embedding) — แม่นกว่า แต่ช้ากว่า
- Markdown Header: หั่นตาม `#`, `##`, `###` — เหมาะกับ docs ที่มีโครงสร้าง

**Chunk Size Guidelines:**
- 200-500 chars: precision สูง เหมาะ FAQ
- 500-1000 chars: สมดุล — **แนะนำเริ่มต้น**
- 1000-2000 chars: context ครบ เหมาะ technical docs
- Overlap 100-200 chars: ช่วยรักษา context ที่รอยต่อ

**Embedding Models:**
| Model | Dimensions | ราคา | หมายเหตุ |
|-------|-----------|------|----------|
| text-embedding-3-small | 1536 | $0.02/1M tok | แนะนำ — ราคาถูก ดี |
| text-embedding-3-large | 3072 | $0.13/1M tok | ดีมาก |
| Cohere embed-multilingual-v3 | 1024 | $0.10/1M tok | ดีมากสำหรับภาษาไทย |
| BGE-M3 (open source) | 1024 | ฟรี | self-host |

### Phase 3: pgvector

**ทำไมเลือก pgvector:**
- ใช้ PostgreSQL ที่คุ้นเคย ไม่ต้องเรียน DB ใหม่
- SQL + vector search ในคำสั่งเดียวกันได้
- Filter metadata ด้วย WHERE clause
- ไม่ต้อง deploy service แยก
- รองรับ index: IVFFlat, HNSW

**Vector Index Types:**
- **IVFFlat**: แบ่ง vectors เป็นกลุ่ม ค้นเฉพาะกลุ่มใกล้สุด — เร็ว memory น้อย เหมาะ data ไม่ update บ่อย
- **HNSW**: กราฟหลายชั้น — แม่นยำสูง ไม่ต้อง train เพิ่ม data ได้เรื่อยๆ **แนะนำ**

**Distance Functions:**
- Cosine (`<=>`) — วัดมุม ไม่สนขนาด **แนะนำ**
- L2 (`<->`) — Euclidean ปกติ
- Inner Product (`<#>`) — เร็วสุด เหมาะกับ normalized vectors

Schema: `id`, `content`, `embedding vector(1536)`, `source`, `category`, `department`, `doc_type`, `title`, `last_updated`, `chunk_index`, `chunk_total`, `metadata JSONB`

### Phase 4: RAG Frameworks Comparison

| Framework | ความง่าย | Flexibility | RAG Support | Agent Support | แนะนำกรณี |
|-----------|---------|-------------|-------------|---------------|-----------|
| LangChain | ★★★☆☆ | ★★★★★ | ★★★★★ | ★★★★★ | tools เยอะ, multi-agent |
| LlamaIndex | ★★★★☆ | ★★★★☆ | ★★★★★ | ★★★★☆ | เน้น RAG/query |
| Google ADK | ★★★★☆ | ★★★☆☆ | ★★★☆☆ | ★★★★★ | ใช้ GCP/Gemini |
| CrewAI | ★★★★☆ | ★★★☆☆ | ★★★☆☆ | ★★★★★ | multi-agent ร่วมมือ |
| DIY | ★★☆☆☆ | ★★★★★ | ขึ้นกับทีม | ขึ้นกับทีม | เข้าใจทุกส่วน, simple |

**แนะนำสำหรับ Sellsuki**: LangChain หรือ LlamaIndex เพราะ community ใหญ่ มี pgvector support ดี

### Phase 5: Agent Architecture (LangChain)

```python
# Pattern: Retriever Tool + AgentExecutor
general_tool = create_retriever_tool(retriever, "search_company_info", "...")
hr_tool = create_retriever_tool(hr_retriever, "search_hr_info", "...")

agent = create_openai_functions_agent(llm, [general_tool, hr_tool], agent_prompt)
agent_executor = AgentExecutor(agent=agent, tools=[...], verbose=True)
```

API: FastAPI + session management (Redis สำหรับ production), `/chat`, `/ingest`, `/health` endpoints

### Phase 6: Deployment Options

| Platform | ราคา | ความง่าย | เหมาะกับ |
|---------|------|---------|---------|
| Google Cloud Run | กลาง | ★★★★☆ | ใช้ GCP |
| Docker + K8s | ถูก-กลาง | ★★★☆☆ | มี K8s |
| VPS + Docker Compose | ถูก | ★★★☆☆ | self-hosted |
| Railway/Render | กลาง | ★★★★★ | ง่ายสุด |

**Cost Estimation (ต่อเดือน, internal use):**
- Embedding: ~$0.5-2
- LLM (gpt-4o-mini): ~$5-20
- Database (self-hosted): ~$5-10
- Hosting: ~$5-15
- **รวม: ~$20-70/เดือน**

### Implementation Timeline

```
Week 1-2: เตรียม Data + Chunking
Week 3:   Setup pgvector + Insert
Week 4:   สร้าง Agent + API
Week 5:   Deploy + Testing
Week 6:   Fine-tune + UAT + Launch
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- ใช้ `chunk_size=800, chunk_overlap=150` เป็นค่าเริ่มต้นแนะนำ
- `similarity_threshold=0.7` เป็น threshold ที่เหมาะสม
- HNSW index: `m=16, ef_construction=64` เป็น default ที่ใช้ได้ดี
- ควรเก็บ metadata ทั้งหมดใน JSONB column เพิ่มเติมนอกจาก column ที่แยกไว้ (สำหรับ flexible query)
- In-memory chat history สำหรับ prototype, Redis สำหรับ production

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบความขัดแย้งกับเนื้อหาใน wiki ที่มีอยู่แล้ว — คู่มือนี้สอดคล้องกับ framework comparisons ก่อนหน้า

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]]
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/agentic-rag|Agentic RAG]]
- [[wiki/concepts/langchain-framework|LangChain Framework]]
- [[wiki/concepts/pgvector|pgvector]]
