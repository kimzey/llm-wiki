# Phase 2: Docling — AI Document Parser

## Docling คืออะไร?

**Docling** คือ Open-Source Library จาก IBM Research ที่ใช้ AI ในการ **แปลงเอกสารทุกรูปแบบให้เป็น Structured Text** เพื่อนำไปใช้กับ LLM และ RAG ได้อย่างมีประสิทธิภาพ

ชื่อมาจาก **"Doc" + "ling"** (linguistics) = การทำความเข้าใจภาษาของเอกสาร

---

## ทำไมต้องมี Docling? ปัญหาของ PDF ปกติ

```
PDF ธรรมดา:
- เป็น binary format ที่ไม่ได้ออกแบบมาให้ machine อ่าน
- Table อาจถูกบันทึกเป็น floating text ไม่มีโครงสร้าง
- Multi-column layout ทำให้อ่านผิดลำดับ
- Header/Footer ปะปนกับเนื้อหา
- ภาพที่มีข้อความ (scan) อ่านไม่ได้เลย

Docling แก้:
- ใช้ AI model วิเคราะห์ layout ของเอกสาร
- แยก: heading, paragraph, table, figure, list, code
- OCR สำหรับภาพที่มีข้อความ
- Export เป็น Markdown/JSON พร้อม structure
```

---

## เอกสารที่ Docling รองรับ

| ประเภทไฟล์ | รองรับ | หมายเหตุ |
|------------|--------|---------|
| PDF | ✅ ดีมาก | รวม scanned PDF ด้วย OCR |
| DOCX (Word) | ✅ ดีมาก | รักษา structure ได้ดี |
| PPTX (PowerPoint) | ✅ ดี | แยก slide content |
| XLSX (Excel) | ✅ ดี | แยก table data |
| HTML | ✅ ดี | Parse web content |
| Markdown | ✅ ดี | Pass-through |
| ภาพ (PNG, JPG, TIFF) | ✅ ดี | OCR ล้วน |
| AsciiDoc | ✅ ดี | |
| TXT | ✅ พอใช้ | |

### ข้อจำกัดตามประเภทเอกสาร

```
เอกสารที่ Docling ทำงานได้ดีมาก:
✅ PDF ที่มี text layer (digital native)
✅ DOCX จาก Word ปกติ
✅ เอกสารที่มี table ชัดเจน

เอกสารที่ต้องพึ่ง OCR (ผลลัพธ์อาจไม่สมบูรณ์):
⚠️  Scanned PDF ที่มีคุณภาพต่ำ (DPI ต่ำ, เบี้ยว)
⚠️  ภาพที่มีแสง/เงา ไม่สม่ำเสมอ
⚠️  ลายมือเขียน (handwriting) — OCR ทั่วไปอ่านได้แต่ accuracy ต่ำ
⚠️  เอกสารภาษาที่ไม่ใช่ Latin script อาจต้องตั้งค่า OCR engine

เอกสารที่ยาก:
❌ PDF ที่ protect/encrypt ด้วย password (ต้อง decrypt ก่อน)
❌ เอกสารที่เป็น vector graphics ล้วน (เช่น CAD)
```

---

## OCR Engine ที่ Docling รองรับ

Docling มี OCR engine หลายตัวให้เลือก:

```python
# 1. EasyOCR (Default) — ใช้งานได้กว้าง รองรับหลายภาษา
DOCLING_OCR_ENGINE=easyocr

# 2. Tesseract — OCR classic ติดตั้งง่าย
DOCLING_OCR_ENGINE=tesseract

# 3. RapidOCR — เร็วกว่า เหมาะกับ CPU
DOCLING_OCR_ENGINE=rapidocr

# 4. TesseractCLI — ใช้ Tesseract ผ่าน command line
DOCLING_OCR_ENGINE=tesseract_cli
```

**ตัวอย่างการตั้งค่าใน .env:**
```bash
# สำหรับเอกสารภาษาไทย แนะนำ EasyOCR หรือ Tesseract
DOCLING_OCR_ENGINE=easyocr
```

> **หมายเหตุ:** EasyOCR รองรับภาษาไทย แต่อาจต้องดาวน์โหลด model เพิ่มเติม

---

## วิธีที่ Docling วิเคราะห์เอกสาร (Document Understanding Pipeline)

```
Input File (PDF/DOCX/etc.)
          │
          ▼
┌─────────────────────────────────────────────┐
│           Docling Processing Pipeline        │
│                                             │
│  1. Format Detection                        │
│     └── ตรวจสอบว่าไฟล์ชนิดอะไร             │
│                                             │
│  2. Pre-processing                          │
│     └── ปรับ image quality, DPI, rotation  │
│                                             │
│  3. Layout Analysis (AI Model)              │
│     └── DocLayNet model วิเคราะห์ layout   │
│         แยก: text, table, figure, header   │
│                                             │
│  4. Table Structure Recognition             │
│     └── TABLEFORMER model แยก rows/cols    │
│                                             │
│  5. OCR (ถ้าจำเป็น)                        │
│     └── EasyOCR/Tesseract แปลงภาพ→ข้อความ │
│                                             │
│  6. Reading Order Detection                 │
│     └── เรียงลำดับ content ให้ถูกต้อง      │
│                                             │
│  7. Export                                  │
│     └── Markdown, JSON, HTML                │
└─────────────────────────────────────────────┘
          │
          ▼
Structured Output (Markdown/JSON)
```

---

## AI Models ที่ Docling ใช้ภายใน

| Model | หน้าที่ |
|-------|---------|
| **DocLayNet** | Layout Analysis — แยก region ต่างๆ ในหน้า |
| **TableFormer** | Table Structure Recognition — แยก row/column |
| **EasyOCR/Tesseract** | Text extraction จากภาพ |
| **GLPN/LayoutLM** (optional) | Document understanding เพิ่มเติม |

---

## Output ที่ Docling ส่งกลับ

### ตัวอย่าง Input: PDF มีตาราง

```
Original PDF content:
┌──────────────────────────┐
│ Product Catalog           │
│                           │
│ ┌──────┬────────┬──────┐  │
│ │Name  │Price   │Stock │  │
│ ├──────┼────────┼──────┤  │
│ │Widget│ $10.00 │  100 │  │
│ │Gadget│ $25.00 │   50 │  │
│ └──────┴────────┴──────┘  │
│                           │
│ Contact: sales@co.com     │
└──────────────────────────┘
```

### Output เป็น Markdown:
```markdown
# Product Catalog

| Name   | Price   | Stock |
|--------|---------|-------|
| Widget | $10.00  | 100   |
| Gadget | $25.00  | 50    |

Contact: sales@co.com
```

### Output เป็น JSON (DoclingDocument):
```json
{
  "schema_name": "DoclingDocument",
  "version": "1.3.0",
  "name": "product_catalog",
  "body": {
    "children": [
      {
        "cref": "#/texts/0",
        "label": "page_header"
      },
      {
        "cref": "#/tables/0",
        "label": "table"
      }
    ]
  },
  "texts": [
    {
      "self_ref": "#/texts/0",
      "text": "Product Catalog",
      "label": "title",
      "prov": [{"page_no": 1, "bbox": {...}}]
    }
  ],
  "tables": [
    {
      "self_ref": "#/tables/0",
      "data": {
        "table_cells": [
          {"row_span": 1, "col_span": 1, "text": "Name", ...},
          {"row_span": 1, "col_span": 1, "text": "Price", ...}
        ],
        "num_rows": 3,
        "num_cols": 3
      }
    }
  ]
}
```

---

## Docling Serve — REST API Mode

ใน OpenRAG, Docling ถูก run ในโหมด **docling-serve** ซึ่งเปิดเป็น HTTP API:

```
Docling Serve รัน port 5001
─────────────────────────────
Endpoint: POST /convert/source
Input: ไฟล์ + options
Output: Markdown/JSON
```

### ตัวอย่าง API Request:
```bash
curl -X POST "http://localhost:5001/convert/source" \
  -F "file=@document.pdf" \
  -F "options={\"to_formats\":[\"md\"]}"
```

### ตัวอย่าง API Response:
```json
{
  "document": {
    "md_content": "# Title\n\n## Section 1\n\nContent here...\n\n| Col1 | Col2 |\n...",
    "status": "success",
    "timings": {
      "total": 2.3
    }
  }
}
```

---

## Docling ใน OpenRAG — Integration

### การตรวจสอบ Health:
```python
# /src/api/docling.py
GET /docling/health
# proxy ไปที่ DOCLING_SERVE_URL/health
```

### การตั้งค่า URL:
```bash
# .env
# ถ้าไม่ตั้ง — จะ auto-detect container host
DOCLING_SERVE_URL=http://host.docker.internal:5001

# ถ้ารันใน Docker network เดียวกัน:
DOCLING_SERVE_URL=http://docling:5001
```

### การ Auto-detect Container Host:
```python
# /src/utils/container_utils.py
# ตรวจสอบว่ารันอยู่ใน Docker หรือ Podman
# ถ้าใน Docker: ใช้ host.docker.internal
# ถ้าใน Podman: ใช้ host.containers.internal
# ถ้าไม่ใช่: ใช้ localhost
```

---

## การติดตั้ง Docling แยก (Local)

Docling ไม่รวมอยู่ใน docker-compose.yml ต้องรันแยก:

```bash
# Option 1: ใช้ pip
pip install docling-serve
docling-serve run --host 0.0.0.0 --port 5001

# Option 2: ใช้ Docker
docker run -p 5001:5001 \
  -e DOCLING_OCR_ENGINE=easyocr \
  quay.io/docling-project/docling-serve:latest

# Option 3: ใช้ GPU (เร็วกว่ามาก)
docker run --gpus all -p 5001:5001 \
  quay.io/docling-project/docling-serve:latest-gpu
```

---

## คำถามที่มักถาม

### Q: Docling ตัดเอกสารได้ทุกแบบเลยไหม?

**A: เกือบทุกแบบ แต่มีข้อจำกัด:**

```
✅ เอกสาร digital native (สร้างจาก Word, PDF ที่มี text layer)
   → ผลลัพธ์ดีมาก 95%+

⚠️  Scanned document คุณภาพดี (300 DPI+)
   → OCR ทำงาน accuracy ~85-95%

⚠️  เอกสารซับซ้อน (multi-column, watermark, เบี้ยว)
   → อาจต้อง custom preprocessing

❌ Handwriting
   → OCR ทั่วไปไม่ดี ต้องใช้ Handwriting-specific model

❌ เอกสารที่ encrypt ด้วย password
   → ต้อง decrypt ก่อน
```

### Q: ต้อง Custom ไหม?

**สำหรับการใช้งานทั่วไป: ไม่ต้อง**
- Docling มี default settings ที่ดีพอ
- เปลี่ยแค่ `DOCLING_OCR_ENGINE` ถ้าต้องการ OCR ที่ต่างกัน

**ต้อง Custom เมื่อ:**
- เอกสารมีรูปแบบ custom เฉพาะทาง (เช่น form แบบพิเศษ)
- ต้องการ extract specific fields (ใช้ post-processing)
- ต้องการ accuracy สูงมากสำหรับตาราง → ปรับ TableFormer settings

---

## Docling vs ทางเลือกอื่น

| Tool | จุดแข็ง | จุดอ่อน |
|------|---------|---------|
| **Docling** | AI-powered layout, table structure, Open-source | ต้องรันแยก service |
| **PyMuPDF** | เร็ว, ง่าย | ไม่มี layout analysis |
| **pdfplumber** | table extraction ดี | เฉพาะ PDF |
| **Unstructured.io** | รองรับหลายรูปแบบ | cloud-based (ค่าใช้จ่าย) |
| **Azure Form Recognizer** | accuracy สูงมาก | ราคาแพง, cloud-only |

**Docling เหมาะสำหรับ OpenRAG เพราะ:**
- Open-source = ไม่มีค่าใช้จ่ายเพิ่มเติม
- AI-powered = เข้าใจ structure ได้ดี
- Self-hosted = ข้อมูลไม่ออกไปไหน
- REST API = integrate ได้ง่ายกับ Langflow

---

## ตัวอย่างการทำงานจริงใน OpenRAG

```
1. User อัปโหลด "annual_report.pdf" (50 หน้า มีตาราง + กราฟ)

2. Backend ส่งไปที่ Langflow ผ่าน:
   POST /api/v1/files/upload → Langflow รับไฟล์

3. Langflow เรียก Docling Serve:
   POST http://host.docker.internal:5001/convert/source
   Body: { file: annual_report.pdf }

4. Docling ประมวลผล:
   - Layout Analysis: ตรวจ 50 หน้า
   - พบ: 5 ตาราง, 10 กราฟ (ข้ามกราฟ), 45 section
   - OCR: ไม่จำเป็น (digital PDF)
   - เวลา: ~5-15 วินาที

5. Return Markdown:
   # Annual Report 2024
   ## Financial Highlights
   | Metric | 2023 | 2024 |
   |--------|------|------|
   | Revenue| $10M | $12M |
   ...

6. Langflow ส่ง Markdown ต่อไปที่ Text Splitter
```

---

*กลับไป: [Phase 1 — Overview](./phase1-overview.md)*
*ต่อไป: [Phase 3 — OpenSearch](./phase3-opensearch.md)*
