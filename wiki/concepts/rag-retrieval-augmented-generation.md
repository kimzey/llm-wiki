---
title: "RAG — Retrieval-Augmented Generation"
type: concept
tags: [rag, ai, retrieval, embeddings, vector-search, llm]
sources: [arona/README.md, arona/RAG_COMPARISON.md, arona/ARONA_RAG_COMPREHENSIVE_GUIDE.md]
related: [wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/semantic-caching.md, wiki/concepts/stateless-rag-design.md]
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
- **By header** (Arona): แบ่งตาม `##` headings — เหมาะกับ Markdown docs
- **Recursive character**: แบ่งตาม chunk size — ทั่วไป
- **Semantic**: แบ่งตามความหมาย — แม่นยำที่สุดแต่ช้า

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

### แนวทางสร้าง RAG ระบบ

| วิธี | Cost | Flexibility | Time-to-Market | เหมาะกับ |
|------|------|-------------|----------------|---------|
| DIY (เช่น Arona) | ต่ำสุด (~$21/เดือน) | 100% | 3-4 สัปดาห์ | Long-term, cost-sensitive |
| LangChain/LlamaIndex | ปานกลาง | 70-80% | 1-2 สัปดาห์ | POC, ทีมใหม่ |
| Off-the-shelf | สูงสุด ($250-1000/เดือน) | 30% | 1-3 วัน | ไม่มี dev team |

### Cost Optimization Techniques
- **Semantic caching**: ใช้คำตอบเก่าถ้า similarity ≥ 90%
- **Incremental indexing**: embed เฉพาะที่เปลี่ยน
- **Summarized retrieval**: index summary แทน full content, อ่าน full page เมื่อต้องการ
- **Small model** สำหรับ query normalization, **large model** สำหรับ generation

## ตัวอย่าง / กรณีศึกษา

**Arona** — RAG สำหรับ Elysia documentation
- Cost: ~$21/เดือน (เทียบกับ $250-1,000 ของ SaaS)
- Throughput: 750 req/day, cache hit rate 85%+
- Stack: ParadeDB (BM25+Vector) + DragonflyDB (Cache) + GPT OSS 120B via OpenRouter

**Sellsuki RAG** — Internal company knowledge base
- ตอบคำถาม HR policy, business rules, procedures
- รองรับ LINE Bot + Slack + Web chat
- เพิ่ม role-based access control บน base architecture ของ Arona

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]] — วิธีค้นหาในขั้นตอน Retrieval
- [[wiki/concepts/semantic-caching|Semantic Caching]] — การ cache ด้วย vector similarity
- [[wiki/concepts/stateless-rag-design|Stateless RAG Design]] — ปรัชญาการออกแบบสำหรับ documentation search

## แหล่งที่มา

- [[wiki/sources/arona-overview|Arona System Overview]]
- [[wiki/sources/arona-rag-techniques|RAG Techniques & Comparison]]
- [[wiki/sources/arona-stateless-rag|Stateless RAG Explained]]
