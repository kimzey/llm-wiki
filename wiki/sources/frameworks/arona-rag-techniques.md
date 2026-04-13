---
title: "Arona RAG — เทคนิคและการเปรียบเทียบแนวทาง"
type: source
source_file: raw/notes/arona/RAG_COMPARISON.md
url: ""
published: 2026-04-13
tags: [rag, comparison, diy, langchain, llamaindex, off-the-shelf, cost-optimization]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/frameworks/arona/RAG_COMPARISON.md|Original file]] | [[../../../raw/notes/frameworks/arona/ARONA_RAG_COMPREHENSIVE_GUIDE.md|ARONA_RAG_COMPREHENSIVE_GUIDE]] | [[../../../raw/notes/frameworks/arona/RAG_RECOMMENDATION.md|RAG_RECOMMENDATION]]

## สรุป

เปรียบเทียบ 4 แนวทางในการสร้างระบบ RAG: DIY (Arona), LangChain, LlamaIndex, Off-the-shelf พร้อมวิเคราะห์เทคนิคเฉพาะที่ Arona ใช้เพื่อลด cost และเพิ่มประสิทธิภาพ

## ประเด็นสำคัญ

### เปรียบเทียบ 4 แนวทาง

| วิธี | Cost | Flexibility | Time-to-Market | Score |
|------|------|-------------|----------------|-------|
| DIY (Arona) | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | 4.2/5 |
| LangChain | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 3.5/5 |
| LlamaIndex | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | 3.8/5 |
| Off-the-shelf | ⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | 2.5/5 |

### Cost Comparison (ต่อเดือน)
```
Off-the-shelf:    $250 - $1,000+
LangChain (DIY):  $50 - $100
LlamaIndex (DIY): $50 - $100
DIY (Arona):      $20 - $30
```

### เทคนิคที่ Arona ใช้ลด Cost

**1. Semantic Caching (Vector Similarity)**
- เก็บ embedding ของคำถามเก่า + คำตอบ
- ถ้า cosine similarity ≥ 90% ใช้คำตอบเก่าเลย
- ตัวอย่าง: "ลาพักร้อนกี่วัน" กับ "สิทธิ์ลาพักผ่อน" = cache hit

**2. Summarized Content (ลด Token)**
- Index: สร้าง summary ของเนื้อหา (200 tokens) ไว้แทน full content (2000 tokens)
- AI ใช้ summary ก่อน → ถ้าต้องการรายละเอียด เรียก `readPage` tool

**3. Tool Calling (ไม่ใช้ Full Agent)**
- AI ตัดสินใจเลือก tool เอง (search, readPage, tableOfContents)
- ไม่ต้องมี Agent state machine ซับซ้อน
- หยุดเมื่อ: ครบ 8 steps หรือ references > 32

**4. Hybrid Search (BM25 + Vector)**
- BM25 score (77.5%) + Document weight (27.5%)
- Vector search เป็น fallback สำหรับ semantic queries
- ไม่ใช้ Re-ranking (ประหยัด latency + cost)

**5. Incremental Indexing (Diff-aware)**
- Hash เนื้อหาทุก chunk
- ถ้า similarity > 98% = เปลี่ยนแค่ metadata
- ถ้า similarity ≤ 98% = embed ใหม่

### Decision Roadmap
```
Phase 1: POC (1-2 สัปดาห์)    → ใช้ LangChain / LlamaIndex
Phase 2: MVP (4-6 สัปดาห์)    → เริ่ม DIY ถ้าต้องการ control
Phase 3: Production            → Full DIY เหมือน Arona
```

### เมื่อไรเลือกอะไร
- **DIY**: traffic > 10,000/เดือน, long-term, cost-sensitive, มี dev team
- **LangChain**: POC/MVP, ทีมใหม่, < 3 เดือน
- **LlamaIndex**: เอกสาร 10,000+ หน้า, หลาย data source, enterprise search
- **Off-the-shelf**: ไม่มี dev team, ต้องการใช้ทันที, budget ไม่จำกัด

### Cost 3-ปี (มุมมอง ROI)
| แนวทาง | รวม 3 ปี | หมายเหตุ |
|---------|---------|---------|
| DIY (Arona) | $6,000 | dev cost สูงปีแรก, maintenance ต่ำ |
| LangChain DIY | $5,000 | dev ถูกกว่า, overhead สูงกว่า |
| Off-the-shelf | $9,000 | ไม่มี dev cost แต่ subscription แพง |

## ข้อมูลที่น่าสนใจ

- ผู้สร้าง Arona บอกว่าแนวคิดหลักคือ "Cut cost เยอะๆ เลยเล่นหลายๆ ท่า เพื่อลด cost ให้เข้า AI น้อยสุด"
- Arona ใช้เวลา 3-4 สัปดาห์สร้าง แต่หลังจากนั้น maintenance cost ต่ำมาก
- LangChain มี 50,000+ บรรทัด ส่วน Arona มีแค่ ~2,000 บรรทัดสำหรับ core RAG

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG (Retrieval-Augmented Generation)]]
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search: BM25 + Vector]]
- [[wiki/concepts/semantic-caching|Semantic Caching]]
