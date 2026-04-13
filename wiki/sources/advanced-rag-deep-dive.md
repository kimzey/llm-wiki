---
title: "Advanced RAG Deep Dive — Hybrid Search, Re-ranking, Multi-query"
type: source
source_file: raw/notes/LangChain/advanced-rag-deep-dive.md
tags: [rag, advanced, hybrid-search, reranking, multi-query]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/advanced-rag-deep-dive.md|Original file]]

## สรุป

เอกสาร Deep Dive สำหรับ Advanced RAG ครอบคลุมทุกเทคนิค Advanced RAG ที่ใช้ใน Production จริง ทุกตัวอย่างรันได้จริง ใช้ Python + LangChain + Gemini

## ประเด็นสำคัญ

### ปัญหาของ Naive RAG

```
❌ ปัญหาของ Naive RAG:

1. Retrieval ไม่แม่น
   - Semantic search อย่างเดียวพลาดคำเฉพาะทาง (ชื่อคน, รหัสสินค้า, ตัวย่อ)
   - ดึง chunks ที่ซ้ำๆ กันมา (ขาดความหลากหลาย)
   - คำถามซับซ้อน → ค้นหาไม่ครอบคลุม

2. Context คุณภาพต่ำ
   - Chunks ที่ดึงมามีส่วนที่ไม่เกี่ยวข้องปนอยู่
   - ลำดับ chunks ไม่เหมาะ (อันที่ดีที่สุดอาจไม่ได้อยู่ต้นๆ)

3. Generation ไม่ดี
   - LLM หลอน (Hallucinate) ตอบนอกเหนือ context
   - ไม่ตรวจสอบว่าคำตอบสอดคล้องกับ context จริงไหม
   - ไม่สามารถบอกได้ว่า "ไม่มีข้อมูล"

✅ Advanced RAG แก้ปัญหาเหล่านี้ด้วย:
1. Hybrid Search → แม่นทั้งคำเฉพาะทางและความหมาย
2. Re-ranking → จัดลำดับผลลัพธ์ใหม่ให้แม่นยำ
3. Multi-query → ค้นหาหลายมุมมอง
4. Compression → ตัดส่วนไม่เกี่ยวข้องออก
5. Self-RAG / CRAG → ตรวจสอบคุณภาพก่อนตอบ
```

### Hybrid Search

**แนวคิด**
```
คำถาม: "นโยบาย WFH ปี 2025"

Keyword Search (BM25):                    Semantic Search (Vector):
✅ เก่งเรื่องคำเฉพาะ "WFH", "2025"       ✅ เก่งเรื่องความหมาย
❌ ไม่เข้าใจ "ทำงานจากบ้าน" = "WFH"      ❌ อาจพลาดคำเฉพาะเจาะจง
                        ↘               ↙
                     Hybrid Search
                  ✅ ได้ทั้งสองอย่าง
```

**ติดตั้ง**
```bash
pip install -U langchain-chroma langchain-google-genai rank_bm25
```

**EnsembleRetriever**
- รวม BM25 + Semantic ด้วย weights
- `weights=[0.4, 0.6]` = ให้ semantic มากกว่า

### Re-ranking

**แนวคิด**
1. ดึง chunks มาเยอะๆ (เช่น 15 อัน) → เร็ว แต่หยาบ
2. จัดอันดับใหม่ด้วย model ที่แม่นกว่า → ช้าหน่อย แต่แม่น
3. เลือก top 3 → ส่งให้ AI

**Cross-Encoder**
- Model พิเศษที่เทียบ "คำถาม" กับ "เอกสาร" ทีละคู่
- แม่นกว่า embedding similarity มาก
- แต่ช้ากว่า → เลยใช้ตอน rerank (เอกสารน้อยแล้ว)

### Multi-Query Retrieval

**แนวคิด**
- คำถามเดียวอาจไม่ครอบคลุม
- ให้ LLM สร้างคำถามหลายเวอร์ชัน → ค้นหาทุกเวอร์ชัน → รวมผลลัพธ์

**ตัวอย่าง**
- คำถามเดิม: "สวัสดิการด้านสุขภาพมีอะไรบ้าง"
- LLM สร้าง:
  1. "ประกันสุขภาพบริษัทครอบคลุมอะไร"
  2. "วงเงินค่ารักษาพยาบาลเท่าไหร่"
  3. "มีสวัสดิการทันตกรรมไหม"
- ค้นหาทั้ง 3 → ได้ผลลัพธ์ครอบคลุมมากขึ้น

### เทคนิคขั้นสูงอื่นๆ

1. **Query Transformation** — ปรับคำถามก่อนค้นหา
2. **Parent Document Retriever** — ค้นเล็ก ดึงใหญ่
3. **Self-Query Retriever** — แปลงคำถามเป็น Filter
4. **Contextual Compression** — บีบอัด Context
5. **RAPTOR** — Recursive Abstractive Processing
6. **Corrective RAG (CRAG)**
7. **Self-RAG** — AI ตรวจสอบตัวเอง
8. **Adaptive RAG** — เลือก Strategy อัตโนมัติ

### หัวข้อที่ครอบคลุม

1. **ปัญหาของ Naive RAG**
2. **Hybrid Search** — รวม Keyword + Semantic
3. **Re-ranking** — จัดอันดับผลลัพธ์ใหม่
4. **Multi-Query Retrieval** — ถามหลายมุม
5. **Query Transformation** — ปรับคำถามก่อนค้นหา
6. **Parent Document Retriever** — ค้นเล็ก ดึงใหญ่
7. **Self-Query Retriever** — แปลงคำถามเป็น Filter
8. **Contextual Compression** — บีบอัด Context
9. **RAPTOR** — Recursive Abstractive Processing
10. **Corrective RAG (CRAG)**
11. **Self-RAG** — AI ตรวจสอบตัวเอง
12. **Adaptive RAG** — เลือก Strategy อัตโนมัติ
13. **รวมทุกเทคนิค — Production RAG Pipeline**
14. **เปรียบเทียบเทคนิคทั้งหมด**

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- **Hybrid Search** ใช้ `EnsembleRetriever` รวม BM25 + Semantic
- **Re-ranking** ใช้ `CrossEncoderReranker` หรือ LLM
- **Multi-query** ใช้ `MultiQueryRetriever` สร้างคำถาม variation อัตโนมัติ

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/agentic-rag|Agentic RAG]]
