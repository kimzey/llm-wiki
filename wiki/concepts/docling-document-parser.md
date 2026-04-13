---
title: "Docling — AI Document Parser"
type: concept
tags: [docling, document-parser, ocr, pdf, ibm-research, rag, layout-analysis]
sources: [phase2-docling.md]
related: [wiki/concepts/openrag-platform.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น
Docling เป็น Open-Source Library จาก IBM Research ที่ใช้ AI แปลงเอกสารทุกรูปแบบ (PDF, DOCX, PPTX, ภาพ) ให้เป็น Structured Text/Markdown ที่ LLM นำไปใช้ได้ โดยรักษา layout, table structure, และ reading order

## อธิบาย
PDF ธรรมดาเป็น binary format ที่เขียนเพื่อแสดงผลบนหน้าจอ ไม่ใช่เพื่อให้ machine อ่าน Docling แก้ปัญหานี้ด้วย AI:

**Processing Pipeline:**
```
Input → Format Detection → Pre-processing → Layout Analysis (DocLayNet) →
Table Structure Recognition (TableFormer) → OCR (ถ้าจำเป็น) → Reading Order → Markdown/JSON
```

**AI Models ภายใน:**
- **DocLayNet** — วิเคราะห์ layout แยก text, table, figure, heading
- **TableFormer** — แยก rows/columns ในตาราง
- **EasyOCR/Tesseract/RapidOCR** — Text extraction จากภาพ

**ไฟล์ที่รองรับ:** PDF, DOCX, PPTX, XLSX, HTML, Markdown, ภาพ (PNG/JPG/TIFF)

## ประเด็นสำคัญ
- **Digital native PDF** (สร้างจาก Word, Google Docs) → ผลดีมาก 95%+
- **Scanned PDF 300 DPI+** → OCR accuracy ~85-95%
- **ข้อจำกัด**: ไม่รองรับ password-protected PDF, ลายมือเขียน, vector-only graphics (CAD)
- **ภาษาไทย**: EasyOCR รองรับแต่อาจต้องดาวน์โหลด model เพิ่ม
- รันในโหมด **docling-serve** เป็น REST API ที่ port 5001
- ไม่รวมใน docker-compose.yml → ต้องรันแยก (มี Docker image)

## ตัวอย่าง / กรณีศึกษา
**PDF มีตาราง → Markdown:**
```markdown
# Product Catalog

| Name   | Price   | Stock |
|--------|---------|-------|
| Widget | $10.00  | 100   |
| Gadget | $25.00  | 50    |

Contact: sales@co.com
```

**GPU version เร็วกว่ามาก:**
```bash
docker run --gpus all -p 5001:5001 quay.io/docling-project/docling-serve:latest-gpu
```

## ความสัมพันธ์กับ concept อื่น
- [[wiki/concepts/openrag-platform.md|OpenRAG]] — ใช้ Docling เป็น document parser หลัก
- [[wiki/concepts/rag-chunking-strategies.md|RAG Chunking]] — Docling output (page/table chunks) เป็น input ให้ chunking layer ต่อ
- [[wiki/concepts/rag-retrieval-augmented-generation.md|RAG]] — Docling อยู่ใน Ingestion phase ของ RAG pipeline

## แหล่งที่มา
- phase2-docling.md (OpenRAG Phase 2 documentation)
