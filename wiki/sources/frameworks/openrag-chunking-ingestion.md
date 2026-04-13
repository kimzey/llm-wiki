---
title: "OpenRAG — Chunking System & Document Ingestion"
type: source
source_file: raw/notes/openrag/docs-lean/chunking.md
url: ""
published: 2026-03-17
tags: [openrag, chunking, ingestion, document-processing, rag, docling]
related: [wiki/concepts/rag-chunking-strategies.md, wiki/concepts/openrag-platform.md, wiki/concepts/docling-document-parser.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/frameworks/openrag/docs-lean/chunking.md|Original file]]
> Additional sources: [[../../../raw/notes/frameworks/openrag/docs-lean/document-best-practices.md|Document Best Practices]]

## สรุป
OpenRAG มีระบบ Chunking หลายชั้น: Docling-based (page-aware + table-aware), Plain Text (paragraph-aware), Langflow SplitText (recursive character), และ Token-based Batching สำหรับ Embedding API. เอกสารคุณภาพดี + chunk strategy ที่ถูกต้อง = RAG คุณภาพสูง

## ประเด็นสำคัญ

### 4 ชั้นของ Chunking ใน OpenRAG

**1. Docling-based Chunking** (PDF, DOCX, PPTX)
- แต่ละ **หน้า** = 1 chunk
- แต่ละ **ตาราง** = 1 chunk แยกต่างหาก
- รองรับ OCR สำหรับ scanned PDF

**2. Plain Text Chunking** (.txt, .md)
- เป้าหมาย ~1,000 characters ต่อ chunk
- แบ่งตาม paragraph (`\n\n`) ก่อน
- ไม่มี overlap ในระดับนี้

**3. Langflow SplitText** (ชั้นที่ 2 บน text จาก Docling)
- Recursive Character Splitting ผ่าน LangChain
- `chunk_size: 1000`, `chunk_overlap: 200`, `separator: "\n\n"`
- ปรับได้ผ่าน UI, API, หรือ per-request

**4. Token-based Batching** (สำหรับ Embedding API)
- ใช้ `tiktoken` นับ token จริง
- Limit: **8,000 tokens ต่อ batch**
- ป้องกัน API error จาก token overflow

### Config Priority
```
Per-request tweaks > CHUNK_SIZE env var > config.yaml defaults
```

### Chunk Settings ตามประเภทเอกสาร
| ประเภท | chunk_size | overlap | วิธี |
|--------|-----------|---------|------|
| Policy/Procedure | 1000 | 200 | Default |
| FAQ/Q&A | 400 | 50 | เล็กลง → 1 Q&A ต่อ chunk |
| Technical Docs | 1500 | 300 | ใหญ่ขึ้น → context ยาว |
| Product Catalog | 500 | 100 | เล็ก → แต่ละ item ชัดเจน |

### Document Best Practices

**Tier 1: ดีมาก (95%+)**
- PDF ที่สร้างจาก Word/Google Docs (digital native)
- DOCX จาก Word
- HTML / Web pages

**Tier 2: ดี แต่ต้องระวัง**
- Scanned PDF 300 DPI+
- PPTX (content ใน slide มักน้อย เพิ่ม speaker notes)
- XLSX (ควร export เป็น CSV แทน)

**Tier 3: มีปัญหา**
- Scanned PDF คุณภาพต่ำ
- PDF ที่ encrypt
- ลายมือเขียน

**Tips สำคัญ:**
- ใช้ Heading structure ชัดเจน (`# H1`, `## H2`)
- ตารางต้องมี Header row
- อธิบาย acronym ครั้งแรกที่ใช้ เพราะ user อาจถามด้วยคำเต็ม
- ชื่อไฟล์สื่อความหมาย: `hr_leave_policy_2024.pdf` ไม่ใช่ `scan001.pdf`

### จุดอ่อนที่ควรรู้
- Plain text chunking ไม่มี overlap → อาจเสีย context ที่ขอบ chunk
- ไม่รองรับ semantic chunking หรือ multi-level separators
- เปลี่ยน chunk_size หลัง index → ต้อง re-ingest ทั้งหมด
- Langflow SplitText ใช้ separator เดียวต่อครั้ง (ไม่ใช่ recursive list)

### Checklist ก่อน Go-Live
```
เอกสาร:
[ ] ชื่อไฟล์สื่อความหมาย
[ ] ลบไฟล์ duplicate / version เก่า
[ ] ตรวจว่า PDF ไม่ encrypted
[ ] Scanned PDF DPI ≥ 300

Access Control:
[ ] ตั้ง allowed_groups ทุกไฟล์
[ ] ทดสอบว่า user เห็น/ไม่เห็นเอกสารที่ถูกต้อง

RAG Quality:
[ ] ทดสอบถามคำถามทุก topic หลัก
[ ] ตรวจ citation ถูกต้อง
[ ] ทดสอบถามสิ่งที่ไม่อยู่ในเอกสาร (ควรบอกว่าไม่พบ)
```

### การ Update เอกสาร
```bash
# 1. ลบเก่าก่อน (สำคัญ!)
curl POST /documents/delete-by-filename  { "filename": "hr_policy_2024.pdf" }
# 2. Upload ใหม่
curl POST /v1/documents/ingest  -F "file=@hr_policy_2025.pdf"
# ถ้ามีทั้งปี 2024 และ 2025 → AI อาจตอบสับสน
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- Token batching: chunk ใหญ่กว่า 8,000 tokens จะถูกแบ่งอีกชั้นด้วย token boundary
- Docling table chunk ใช้ TSV format (tab-separated) ใน text field
- `picture_descriptions: true` → AI อธิบายภาพใน PDF แต่เพิ่ม cost มาก
- Embedding model ชื่อถูก sanitize ใน OpenSearch field name: `text-embedding-3-small` → `chunk_embedding_text_embedding_3_small`

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]]
- [[wiki/concepts/openrag-platform|OpenRAG Platform]]
- [[wiki/concepts/docling-document-parser|Docling]]
