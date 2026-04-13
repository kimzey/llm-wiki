---
title: "OpenRAG — Docling AI Document Parser"
type: source
source_file: raw/notes/openrag/docs-lean/phase2-docling.md
url: ""
published: 2026-03-17
tags: [docling, document-parser, ocr, pdf, rag, ibm]
related: [wiki/concepts/docling-document-parser.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/phase2-docling.md|Phase 2 — Docling]]

## สรุป
Docling คือ Open-Source Library จาก IBM Research ที่ใช้ AI แปลงเอกสารทุกรูปแบบ (PDF, DOCX, PPTX, ภาพ) ให้เป็น Structured Text/Markdown/JSON พร้อมรักษา layout, table structure, และ reading order

## ประเด็นสำคัญ

### ทำไมต้องมี Docling?
PDF ธรรมดาเป็น binary format ที่ machine อ่านยาก:
- Table ถูกบันทึกเป็น floating text ไม่มีโครงสร้าง
- Multi-column layout ทำให้อ่านผิดลำดับ
- ภาพที่มีข้อความ (scan) อ่านไม่ได้เลย

### AI Models ภายใน Docling
| Model | หน้าที่ |
|-------|---------|
| **DocLayNet** | Layout Analysis — แยก region ต่างๆ ในหน้า |
| **TableFormer** | Table Structure Recognition — แยก row/column |
| **EasyOCR/Tesseract** | Text extraction จากภาพ |

### Processing Pipeline ของ Docling
```
Input → Format Detection → Pre-processing → Layout Analysis (AI) →
Table Structure Recognition → OCR (ถ้าจำเป็น) → Reading Order → Export
```

### ไฟล์ที่รองรับ
| ประเภท | รองรับ | หมายเหตุ |
|--------|--------|---------|
| PDF | ✅ ดีมาก | รวม scanned PDF ด้วย OCR |
| DOCX | ✅ ดีมาก | รักษา structure ได้ดี |
| PPTX | ✅ ดี | แยก slide content |
| HTML | ✅ ดี | Parse web content |
| ภาพ (PNG/JPG/TIFF) | ✅ ดี | OCR ล้วน |

### OCR Engine ที่รองรับ
```bash
DOCLING_OCR_ENGINE=easyocr      # Default — หลายภาษา รวมภาษาไทย
DOCLING_OCR_ENGINE=tesseract    # Classic OCR
DOCLING_OCR_ENGINE=rapidocr     # เร็วกว่า เหมาะกับ CPU
```

### ข้อจำกัด
- ❌ PDF ที่ password protected
- ❌ เอกสาร vector graphics ล้วน (CAD)
- ❌ ลายมือเขียน (handwriting)
- ⚠️ Scanned PDF คุณภาพต่ำ (DPI < 300)

### Output Format
Docling ส่งออกเป็น Markdown หรือ JSON (DoclingDocument):
```markdown
# Product Catalog

| Name   | Price   | Stock |
|--------|---------|-------|
| Widget | $10.00  | 100   |
```

### การ Run ใน OpenRAG
Docling รันในโหมด **docling-serve** เปิด HTTP API ที่ port 5001:
```bash
# Docker (แนะนำ)
docker run -p 5001:5001 quay.io/docling-project/docling-serve:latest

# GPU version (เร็วกว่ามาก)
docker run --gpus all -p 5001:5001 quay.io/docling-project/docling-serve:latest-gpu
```

### เปรียบเทียบกับทางเลือกอื่น
| Tool | จุดแข็ง | จุดอ่อน |
|------|---------|---------|
| **Docling** | AI-powered layout, table structure, Open-source | ต้องรันแยก service |
| **PyMuPDF** | เร็ว, ง่าย | ไม่มี layout analysis |
| **Unstructured.io** | รองรับหลายรูปแบบ | cloud-based (ค่าใช้จ่าย) |
| **Azure Form Recognizer** | accuracy สูงมาก | ราคาแพง, cloud-only |

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- เอกสาร digital native → ผลลัพธ์ดีมาก 95%+
- Scanned document 300 DPI+ → OCR accuracy ~85-95%
- PDF 50 หน้า ใช้เวลา ~5-15 วินาทีในการ process
- ไม่รวมอยู่ใน docker-compose.yml ต้องรันแยก

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/docling-document-parser.md|Docling Document Parser]]
- [[wiki/concepts/openrag-platform.md|OpenRAG Platform]]
