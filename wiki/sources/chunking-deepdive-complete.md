---
title: "Chunking Deep Dive — ทุกวิธี + ข้อมูลจริง"
type: source
source_file: raw/notes/rag-knowledge/04-chunking-deepdive.md
tags: [rag, chunking, data-prep, embedding, optimization]
related: [wiki/concepts/rag-chunking-strategies]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/rag-knowledge/04-chunking-deepdive.md|Original file]]

## สรุป

คู่มือ Deep Dive ทุกวิธี Chunking พร้อมตัวอย่าง Output จริง — ครอบคลุม 10 วิธี, โครงสร้าง database, การเชื่อมโยง chunks, best practices 2025, decision matrix, และ common mistakes

## ประเด็นสำคัญ

### ทำไมต้อง Chunk

1. **Embedding Model มี limit**: text-embedding-3-small รับได้สูงสุด 8,191 tokens
2. **ยิ่งข้อความยาว embedding ยิ่ง "เบลอ"**: Embed ทั้งเอกสาร 50 หน้า → vector กลายเป็น "ค่าเฉลี่ย"
3. **ประหยัด LLM cost**: ส่ง 3 chunks (1,500 tokens) = $0.000225 vs ทั้งเอกสาร = $0.00375 (ประหยัด 16 เท่า)
4. **LLM "หลง" ถ้า context ยาวเกิน**: งานวิจัย "Lost in the Middle" — LLM มักลืมข้อมูลกลาง context

### องค์ประกอบของ Chunk ที่ดี

- **Self-contained**: อ่านเดี่ยวๆ แล้วเข้าใจ
- **Focused**: พูดเรื่องเดียว
- **Right size**: 200-1000 ตัวอักษร (sweet spot)
- **Has context**: รู้ว่ามาจากไหน (metadata)

## 10 วิธี Chunking ทั้งหมด

### วิธี 1: Fixed Size Chunking
- **วิธี**: ตัดทุกๆ N ตัวอักษร ไม่สน content
- **ง่าย**: ★★★★★
- **คุณภาพ**: ★★☆☆☆
- **ปัญหา**: ตัดกลางประโยค, ผสมหลายเรื่องใน chunk เดียว
- **ใช้เมื่อ**: แทบไม่มีใครใช้แล้ว

### วิธี 2: Recursive Character Splitting
- **วิธี**: พยายามตัดที่ "\n\n" → "\n" → "." → " " → ตัวอักษร
- **ง่าย**: ★★★★☆
- **คุณภาพ**: ★★★★☆
- **ข้อดี**: ตัดที่ paragraph break, แต่ละ chunk พูดเรื่องเดียว
- **ข้อเสีย**: ไม่มี header ติดมาด้วย
- **ใช้เมื่อ**: เอกสารที่ไม่มี structure ชัดเจน ← **ใช้บ่อยที่สุด**

### วิธี 3: Markdown/HTML Header Splitting
- **วิธี**: ตัดตามโครงสร้าง headers (# ## ###)
- **ง่าย**: ★★★★☆
- **คุณภาพ**: ★★★★★
- **ข้อดี**: แต่ละ chunk มี topic ชัดเจน, metadata เก็บ hierarchy
- **ใช้เมื่อ**: Markdown docs, HTML pages ← **แนะนำ**

### วิธี 4: Hierarchical (File → Header → Sub-chunk)
- **วิธี**: แบ่งตาม file → ตาม header → ถ้ายังยาวก็ recursive split
- **ง่าย**: ★★★☆☆
- **คุณภาพ**: ★★★★★
- **ข้อดี**: จัดการเอกสารใหญ่ได้ดีกว่า
- **ใช้เมื่อ**: Documentation ที่มี structure, คู่มือ, wiki

### วิธี 5: Sentence Splitting
- **วิธี**: ตัดเป็นประโยค แล้วรวม N ประโยคเป็น 1 chunk
- **ง่าย**: ★★★★☆
- **คุณภาพ**: ★★★☆☆
- **ปัญหา**: ภาษาไทยตัดประโยคยาก
- **ใช้เมื่อ**: ภาษาอังกฤษที่มี period ชัดเจน (ไม่แนะนำภาษาไทย)

### วิธี 6: Sentence Window
- **วิธี**: แต่ละ chunk = 1 ประโยค แต่เก็บ "ประโยครอบข้าง" ไว้เป็น context
- **ง่าย**: ★★★☆☆
- **คุณภาพ**: ★★★★☆
- **ข้อดี**: Search แม่น (embed สั้น) + LLM ได้บริบทครบ (window ยาว)
- **ข้อเสีย**: ต้องเก็บข้อมูล 2 ส่วน
- **ใช้เมื่อ**: ต้องการ precision สูงมาก (legal docs, technical specs)

### วิธี 7: Semantic Chunking
- **วิธี**: ใช้ embedding ตัดที่ "ความหมายเปลี่ยน"
- **ง่าย**: ★★☆☆☆
- **คุณภาพ**: ★★★★★
- **ข้อดี**: แม่นที่สุด — AI ตัดสินใจว่า "เรื่องเปลี่ยน" ตรงไหน
- **ข้อเสีย**: ช้า + แพง — ต้อง embed ทุกประโยคก่อน
- **ใช้เมื่อ**: เอกสารไม่มี structure เลย (meeting transcripts, notes)

### วิธี 8: Parent-Child (Small-to-Big)
- **วิธี**: Child chunks (เล็ก, search) + Parent chunks (ใหญ่, context)
- **ง่าย**: ★★★☆☆
- **คุณภาพ**: ★★★★★
- **ข้อดี**: Search แม่น + Context ครบ
- **ใช้เมื่อ**: เอกสาร technical ยาวๆ

### วิธี 9: Context-Enriched
- **วิธี**: Chunk + enrich ด้วย context เพิ่ม (title, summary, keywords)
- **ง่าย**: ★★☆☆☆
- **คุณภาพ**: ★★★★☆
- **ข้อดี**: Search แม่นขึ้น
- **ใช้เมื่อ**: ต้องการความแม่นยำสูง

### วิธี 10: RAG-specific
- **วิธี**: ออกแบบ chunk สำหรับ RAGโดยเฉพาะ (คำถาม-คำตอบ)
- **ง่าย**: ★★☆☆☆
- **คุณภาพ**: ★★★★★
- **ใช้เมื่อ**: FAQ, Q&A datasets

## โครงสร้าง Chunk ใน Database

### Schema ที่ดี
```python
{
    "id": 1,
    "content": "พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน...",
    "summary": "ลาพักร้อน 10วัน/ปี แจ้ง3วัน เกิน3วันแจ้ง1สัปดาห์",
    "embedding": [0.034, -0.128, ..., 0.045],  # 768/1536 ตัวเลข
    "content_hash": "a3f2b8c1d4e5...",  # SHA256

    # Metadata
    "source": "handbook_2024.md",
    "category": "hr",
    "department": "HR",
    "doc_type": "policy",
    "title": "ลาพักร้อน",
    "header_1": "นโยบายการลา",
    "header_2": "ลาพักร้อน",
    "chunk_index": 0,
    "chunk_total": 6,
    "parent_doc_id": "doc_handbook_2024",
    "prev_chunk_id": null,
    "next_chunk_id": 2,
    "last_updated": "2024-01-15",
    "created_at": "2024-12-01T10:30:00Z"
}
```

## Decision Matrix

| เอกสาร | วิธีที่แนะนำ | เหตุผล |
|---------|----------------|----------|
| Markdown docs | **Markdown Header Split** | มี structure ชัดเจน |
| Meeting notes | **Semantic Chunking** | ไม่มี structure, semantic เปลี่ยนบ่อย |
| FAQ | **RAG-specific** | Q&A format |
| Legal docs | **Sentence Window** หรือ **Parent-Child** | ต้อง precision สูง |
| Wiki | **Hierarchical** | มี hierarchy ชัดเจน |
| Plain text | **Recursive Character** | ไม่มี structure |
| PDF reports | **Parent-Child** | ยาว, ต้อง context ครบ |

## Common Mistakes

1. ❌ **Chunk size ใหญ่เกินไป (>2000 chars)**: Embedding เบลอ, search ไม่แม่น
2. ❌ **ไม่มี overlap**: ข้อมูลหายตรง "รอยตัด"
3. ❌ **ไม่มี metadata**: ไม่สามารถ filter หรือ trace กลับ source ได้
4. ❌ **ตัดกลางประโยค**: Chunk ไม่ self-contained
5. ❌ **ใช้ Fixed Size**: ยุคนี้ไม่มีเหตุผลใช้แล้ว
6. ❌ **ไม่ได้ summary**: Search ยากขึ้น
7. ❌ **ไม่ได้ content hash**: ไม่สามารถ incremental update ได้

## Best Practices 2025

1. **Hybrid approach**: Markdown Header + Recursive (สำหรับ section ยาว)
2. **Metadata enrichment**: เก็บ title, summary, keywords
3. **Overlap 15-25%**: ช่วยไม่ให้ข้อมูลหาย
4. **Chunk size 500-1000**: Sweet spot สำหรับเอกสารทั่วไป
5. **Content hash**: Enable diff-based indexing
6. **Link chunks**: prev_chunk_id, next_chunk_id

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG — Retrieval-Augmented Generation]]
- [[wiki/concepts/embedding|Embedding]]
- [[wiki/concepts/vector-database|Vector Database]]
