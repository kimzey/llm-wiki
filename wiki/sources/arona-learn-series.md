---
title: "Arona Learn Series — ชุดบทเรียน Ingest Pipeline"
type: source
source_file: raw/notes/arona/learn/LEARN.md
url: ""
published: 2026-04-13
tags: [arona, ingestion, chunking, embedding, vectorization, learning]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/sources/arona-deepdive.md, wiki/sources/arona-overview.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/arona/learn/LEARN.md|LEARN]] | [[../../raw/notes/arona/learn/LEARN2.md|LEARN2]] – [[../../raw/notes/arona/learn/LEARN14.md|LEARN14]]

## สรุป

ชุดบทเรียน 14 ตอนที่อธิบาย Arona Ingest Service อย่างละเอียด สำหรับผู้ที่ต้องการเข้าใจว่าระบบแปลงเอกสารเป็น vector embeddings และเก็บลง database อย่างไร

## ประเด็นสำคัญ

### Ingest Service คืออะไร

Ingest Service คือ "พนักงานจัดเก็บเอกสาร" ที่ทำหน้าที่:
1. **Extract** — อ่านไฟล์ (MD, PDF, Text)
2. **Chunk** — ตัดแบ่งตามหัวข้อ `##`
3. **Plan** — เช็คกับ DB ว่าเนื้อหาซ้ำไหม
4. **Embed** — แปลงเป็น vector ด้วย OpenAI
5. **Save** — บันทึกลง PostgreSQL (ParadeDB)

### ทำไมต้องมี Chunking

AI มีข้อจำกัดด้าน Context Window ไม่สามารถโยนไฟล์ 1,000 หน้าได้ การ chunk ช่วยให้:
- ค้นหาได้แม่นยำขึ้น (เนื้อหาชัดเจนต่อ chunk)
- ลด token cost ในการ retrieval
- เพิ่ม recall (เจอเนื้อหาที่เกี่ยวข้องมากขึ้น)

### โครงสร้างไฟล์หลัก
```
src/modules/ingest/
├── index.ts     # API endpoint
├── service.ts   # Business logic
└── model.ts     # Type definitions

src/libs/
├── chunker.ts      # ตัดแบ่งข้อความ
├── ai.ts           # เชื่อม OpenAI
└── extractors/     # อ่านไฟล์ประเภทต่างๆ
    ├── markdown.ts
    └── text.ts
```

### Vectorization (3 Embeddings per Chunk)
ทุก chunk สร้าง embedding 3 ชนิด:
1. **Content embedding** — เนื้อหาหลัก (ใช้ weight 67.5%)
2. **Title embedding** — หัวข้อ (weight 10%)
3. **Filename embedding** — ชื่อไฟล์ (weight 10%)

### Planning Step (ประหยัด Cost)
ก่อน embed จะเช็ค:
```
ถ้า hash เหมือนเดิม → ข้าม embedding
ถ้า content similarity > 98% → update metadata only
ถ้าต่างกันมาก → embed ใหม่ทั้งหมด
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG (Retrieval-Augmented Generation)]]
- [[wiki/sources/arona-deepdive|Arona Deep Dive]]
- [[wiki/sources/arona-overview|Arona System Overview]]
