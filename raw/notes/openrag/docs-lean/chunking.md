# Chunking System Documentation

> เอกสารนี้อธิบายระบบ Chunking ของ OpenRAG อย่างละเอียด ครอบคลุมกลไก ตำแหน่งของ code ค่า default การ customize และตัวอย่าง output จริง

---

## สารบัญ

1. [ภาพรวม](#ภาพรวม)
2. [Chunking Strategies ที่รองรับ](#chunking-strategies-ที่รองรับ)
3. [ตำแหน่ง Code ทั้งหมด](#ตำแหน่ง-code-ทั้งหมด)
4. [Pipeline การทำงาน](#pipeline-การทำงาน)
5. [ค่า Default และ Configuration](#ค่า-default-และ-configuration)
6. [การ Customize](#การ-customize)
7. [ตัวอย่าง Output จริง](#ตัวอย่าง-output-จริง)
8. [ประเมินคุณภาพ](#ประเมินคุณภาพ)
9. [Limitations](#limitations)

---

## ภาพรวม

OpenRAG ใช้ระบบ Chunking หลายชั้นขึ้นอยู่กับประเภทไฟล์และ Pipeline ที่ใช้งาน โดยมี **3 กลไกหลัก**:

```
ไฟล์ที่รับเข้ามา
       │
       ▼
┌─────────────────────────────────────┐
│         ประเภทไฟล์?                 │
├────────────────┬────────────────────┤
│  .txt / .md   │  PDF / DOCX / etc. │
│               │                    │
▼               ▼                    │
Plain Text   Docling                  │
Chunker      Extraction               │
│               │                    │
└───────────────┘                    │
       │                             │
       ▼                             │
  Chunks (text)                      │
       │                             │
       ▼                             │
Token-based Batching  ◄──────────────┘
(for Embeddings API)
       │
       ▼
  OpenSearch Index
```

---

## Chunking Strategies ที่รองรับ

### 1. Docling-based Chunking (PDF, DOCX, PPTX, ฯลฯ)

**ไฟล์**: `src/utils/document_processing.py`

กลยุทธ์: **Page-aware + Table-aware**

- แต่ละ **หน้า** ของเอกสาร = 1 chunk
- แต่ละ **ตาราง** ในเอกสาร = 1 chunk แยกต่างหาก
- รองรับ OCR สำหรับ scanned PDF
- รองรับการสกัด `picture_descriptions`

```
Document (10 pages, 2 tables)
├── chunk 1: page 1 text
├── chunk 2: page 1 table_0
├── chunk 3: page 2 text
├── chunk 4: page 3 text
...
├── chunk 11: page 5 table_1
└── chunk 12: page 10 text
```

---

### 2. Plain Text Chunking (.txt, .md)

**ไฟล์**: `src/utils/document_processing.py` (lines 9–82)

กลยุทธ์: **Character-based + Paragraph-aware**

- เป้าหมาย: ~1,000 characters ต่อ chunk
- แบ่งตาม paragraph (`\n\n`) ก่อน
- Merge paragraphs จนกว่าจะถึง chunk size
- **ไม่มี overlap** ในระดับนี้

Algorithm:
```
paragraphs = content.split('\n\n')
current_chunk = ""

for para in paragraphs:
    if len(current_chunk) + len(para) > 1000:
        save(current_chunk)
        current_chunk = para
    else:
        current_chunk += "\n\n" + para
```

---

### 3. Langflow Split Text Component

**ไฟล์**: `flows/ingestion_flow.json` (node: `SplitText-QIKhg`)

กลยุทธ์: **Recursive Character Splitting** (ผ่าน LangChain)

เป็น chunking layer ที่ทำงานบน text ที่ผ่าน Docling มาแล้ว ก่อนที่จะ embed

| Parameter | Default | ความหมาย |
|-----------|---------|-----------|
| `chunk_size` | 1000 | ขนาด chunk สูงสุด (characters) |
| `chunk_overlap` | 200 | จำนวน characters ที่ overlap ระหว่าง chunks |
| `separator` | `\n\n` | ตัวคั่นหลักที่ใช้แบ่ง |
| `keep_separator` | `false` | เก็บ separator ไว้ใน chunk หรือไม่ |

Options ของ `keep_separator`:
- `False` — ตัด separator ออก (default)
- `True` — เก็บ separator ไว้
- `Start` — วาง separator ที่ต้น chunk
- `End` — วาง separator ที่ท้าย chunk

---

### 4. Token-based Batching (สำหรับ Embedding API)

**ไฟล์**: `src/services/document_service.py` (lines 31–87)

กลยุทธ์: **Token-aware Batching** ไม่ใช่การ chunk เนื้อหา แต่เป็นการจัดกลุ่ม chunks ให้พอดีกับ token limit ของ Embedding API

- ใช้ `tiktoken` นับ token จริง
- Limit: **8,000 tokens ต่อ batch** (buffer จาก 8,191)
- ถ้า chunk เดี่ยวเกิน 8,000 tokens → แบ่งอีกชั้นด้วย token boundary

```python
text_batches = chunk_texts_for_embeddings(texts, max_tokens=8000)
# → [[chunk1, chunk2], [chunk3], [chunk4, chunk5, chunk6], ...]
```

Fallback encoding: ถ้าไม่รู้จัก model → ใช้ `cl100k_base`

---

## ตำแหน่ง Code ทั้งหมด

| ส่วนงาน | ไฟล์ | Lines |
|---------|------|-------|
| ค่า default config | `src/config/config_manager.py` | 66–77 |
| Config schema (API) | `src/api/settings.py` | 40–51, 133–141, 148–152 |
| Plain text chunker | `src/utils/document_processing.py` | 9–82 |
| Docling extractor | `src/utils/document_processing.py` | 85–143 |
| Token batching | `src/services/document_service.py` | 31–87 |
| Token counting | `src/services/document_service.py` | 19–28 |
| Ingestion processor | `src/models/processors.py` | 149–292 |
| Flow update (chunk_size) | `src/services/flows_service.py` | 607–616 |
| Flow update (chunk_overlap) | `src/services/flows_service.py` | 618–627 |
| Settings API handler | `src/api/settings.py` | 595–634 |
| Langflow tweaks integration | `src/services/langflow_file_service.py` | 284–309 |
| OpenSearch schema | `src/config/settings.py` | 115–155 |
| Embedding field naming | `src/utils/embedding_fields.py` | — |
| Langflow flow JSON | `flows/ingestion_flow.json` | node: SplitText-QIKhg |

---

## Pipeline การทำงาน

### Standard Ingestion Flow (ผ่าน Langflow)

```
1. ไฟล์เข้า API
        │
2. ตรวจประเภทไฟล์
   ├─ .txt/.md → process_text_file()    [document_processing.py:9]
   └─ อื่นๆ   → docling convert_file() [document_processing.py:85]
        │
3. ได้ slim_doc["chunks"] = [{page, type, text}, ...]
        │
4. ส่งเข้า Langflow ingestion_flow.json
   └─ SplitText-QIKhg (chunk_size=1000, overlap=200, sep="\n\n")
        │
5. chunk_texts_for_embeddings()        [document_service.py:31]
   └─ แบ่ง batches ไม่เกิน 8,000 tokens
        │
6. Embedding API call (ต่อ batch)
        │
7. Index แต่ละ chunk เข้า OpenSearch
   └─ fields: document_id, filename, page, text, chunk_embedding_*, ...
```

### Settings Update Flow

```
PATCH /api/settings
  { chunk_size: 500, chunk_overlap: 100 }
        │
1. validate (gt:0, ge:0)
2. บันทึกใน config/config.yaml
3. อัพเดต Langflow flow node SplitText-QIKhg
   ├─ flows_service.update_ingest_flow_chunk_size(500)
   └─ flows_service.update_ingest_flow_chunk_overlap(100)
4. Telemetry event: ORB_SETTINGS_CHUNK_UPDATED
```

---

## ค่า Default และ Configuration

### ค่า Default

```yaml
# config/config.yaml (KnowledgeConfig)
chunk_size: 1000       # characters
chunk_overlap: 200     # characters
table_structure: true  # แยก table เป็น chunk ต่างหาก
ocr: false             # OCR สำหรับ scanned images
picture_descriptions: false
```

### Environment Variables (override config)

```bash
CHUNK_SIZE=1500
CHUNK_OVERLAP=300
```

ถูกอ่านใน `src/config/config_manager.py` lines 249–252

### API Body (per-request override)

```json
{
  "chunkSize": 800,
  "chunkOverlap": 150,
  "separator": "\n"
}
```

ถูก map ใน `src/services/langflow_file_service.py` lines 284–309:

```python
final_tweaks["SplitText-QIKhg"]["chunk_size"] = settings["chunkSize"]
final_tweaks["SplitText-QIKhg"]["chunk_overlap"] = settings["chunkOverlap"]
final_tweaks["SplitText-QIKhg"]["separator"] = settings["separator"]
```

### Priority ของ Config

```
Per-request tweaks  >  Environment Variables  >  config.yaml defaults
```

---

## การ Customize

### ระดับ Global (ถาวร)

**ผ่าน API:**
```http
PATCH /api/settings
Content-Type: application/json

{
  "chunk_size": 500,
  "chunk_overlap": 50
}
```

**ผ่าน Environment:**
```bash
# docker-compose.yml หรือ .env
CHUNK_SIZE=500
CHUNK_OVERLAP=50
```

### ระดับ Per-request

ส่ง `settings` object ตอน ingest:
```json
{
  "files": [...],
  "settings": {
    "chunkSize": 300,
    "chunkOverlap": 30,
    "separator": "\n"
  }
}
```

### ระดับ Advanced (แก้ code)

**เปลี่ยน separator strategy** (`flows/ingestion_flow.json`):
```json
"separator": {
  "value": "\n"
}
```

**เปลี่ยน `keep_separator`** (เก็บ separator ไว้ใน chunk):
```json
"keep_separator": {
  "value": "Start"
}
```

**เปลี่ยน token limit สำหรับ embedding batch** (`src/models/processors.py` line 228):
```python
text_batches = chunk_texts_for_embeddings(texts, max_tokens=4000)  # ลดลง
```

**เปลี่ยน plain text chunk size** (`src/utils/document_processing.py` line 16):
```python
chunk_size = 500  # จาก 1000
```

---

## ตัวอย่าง Output จริง

### Input: ไฟล์ PDF 3 หน้า พร้อม 1 ตาราง

**Original content (หน้า 1):**
```
Introduction to Machine Learning

Machine learning is a subset of artificial intelligence that enables
systems to learn and improve from experience without being explicitly
programmed. It focuses on developing computer programs that can access
data and use it to learn for themselves.

The process begins with observations or data, such as examples, direct
experience, or instruction, so that computers can learn to perform tasks
without human intervention.
```

**Docling Output** (`slim_doc`):
```json
{
  "id": "a1b2c3d4e5f6...",
  "filename": "ml_intro.pdf",
  "mimetype": "application/pdf",
  "chunks": [
    {
      "page": 1,
      "type": "text",
      "text": "Introduction to Machine Learning\n\nMachine learning is a subset of artificial intelligence that enables systems to learn and improve from experience without being explicitly programmed. It focuses on developing computer programs that can access data and use it to learn for themselves.\n\nThe process begins with observations or data, such as examples, direct experience, or instruction, so that computers can learn to perform tasks without human intervention."
    },
    {
      "page": 2,
      "type": "text",
      "text": "Types of Machine Learning\n\nSupervised Learning: The algorithm learns from labeled training data...\nUnsupervised Learning: Finds patterns in data without labels...\nReinforcement Learning: Learns through trial and error with rewards..."
    },
    {
      "page": 2,
      "type": "table",
      "table_index": 0,
      "text": "Algorithm\tType\tUse Case\tAccuracy\nLinear Regression\tSupervised\tPrediction\tHigh\nK-Means\tUnsupervised\tClustering\tMedium\nQ-Learning\tReinforcement\tGames\tVariable"
    },
    {
      "page": 3,
      "type": "text",
      "text": "Conclusion\n\nMachine learning continues to transform industries..."
    }
  ]
}
```

**หลัง Langflow SplitText** (chunk_size=1000, overlap=200):

Chunk ที่ยาวน้อยกว่า 1000 chars จะไม่ถูกแบ่งอีก ถ้ายาวกว่าจะถูก split และมี overlap:

```
chunk[0]: "Introduction to Machine Learning\n\nMachine learning is a subset..."
          [450 chars - ไม่ถูก split]

chunk[1]: "Types of Machine Learning\n\nSupervised Learning: The algorithm..."
          [320 chars - ไม่ถูก split]

chunk[2]: "Algorithm\tType\tUse Case\t..."  ← table chunk
          [185 chars - ไม่ถูก split]

chunk[3]: "Conclusion\n\nMachine learning continues..."
          [95 chars - ไม่ถูก split]
```

**Indexed ใน OpenSearch** (ตัวอย่าง 1 document ใน index):
```json
{
  "_index": "documents",
  "_id": "a1b2c3d4e5f6_chunk_0",
  "_source": {
    "document_id": "a1b2c3d4e5f6...",
    "filename": "ml_intro.pdf",
    "mimetype": "application/pdf",
    "page": 1,
    "text": "Introduction to Machine Learning\n\nMachine learning is a subset of artificial intelligence...",
    "chunk_embedding_text_embedding_3_small": [0.0123, -0.0456, 0.0789, ...],
    "embedding_model": "text-embedding-3-small",
    "embedding_dimensions": 1536,
    "file_size": 102400,
    "connector_type": "upload",
    "indexed_time": "2026-03-17T10:30:00.000Z",
    "owner": "user_abc123",
    "allowed_users": ["user_abc123"],
    "allowed_groups": []
  }
}
```

---

### Input: ไฟล์ .txt ขนาดใหญ่

**Original content:**
```
Paragraph 1 (600 chars)...

Paragraph 2 (500 chars)...

Paragraph 3 (800 chars)...

Paragraph 4 (200 chars)...
```

**Plain Text Chunker Output** (chunk_size=1000):
```json
[
  {
    "page": 1,
    "type": "text",
    "text": "Paragraph 1...\n\nParagraph 2..."
  },
  {
    "page": 2,
    "type": "text",
    "text": "Paragraph 3..."
  },
  {
    "page": 3,
    "type": "text",
    "text": "Paragraph 4..."
  }
]
```

- P1 (600) + P2 (500) = 1100 > 1000 → แบ่ง
- P1 (600) ไว้ใน chunk 1 เพียงลำพัง? ไม่ใช่ — P1 เริ่ม chunk แรก, P2 เกินจึง save P1 แล้วเริ่ม chunk ใหม่ด้วย P2
- P2 (500) + P3 (800) = 1300 > 1000 → P2 อยู่ chunk 2, P3 อยู่ chunk 3
- P4 (200) + ไม่มีอะไรต่อ → chunk 4

---

### Token Batching สำหรับ Embedding

**Input:** 5 chunks, token counts: [3000, 4000, 5000, 2000, 1000]

```
max_tokens = 8000

chunk[0]: 3000 tokens → batch_0 = [chunk0]
chunk[1]: 4000 tokens → 3000+4000=7000 ≤ 8000 → batch_0 = [chunk0, chunk1]
chunk[2]: 5000 tokens → 7000+5000=12000 > 8000 → save batch_0, batch_1 = [chunk2]
chunk[3]: 2000 tokens → 5000+2000=7000 ≤ 8000 → batch_1 = [chunk2, chunk3]
chunk[4]: 1000 tokens → 7000+1000=8000 ≤ 8000 → batch_1 = [chunk2, chunk3, chunk4]

Result: [
  [chunk0, chunk1],       → API call 1
  [chunk2, chunk3, chunk4] → API call 2
]
```

---

## ประเมินคุณภาพ

### จุดแข็ง

| จุด | รายละเอียด |
|-----|-----------|
| **Table-aware** | ตาราง PDF/Word ถูกแยกเป็น chunk ต่างหาก ไม่ปนกับ text body — ดีมากสำหรับ structured data retrieval |
| **Page-aware** | รู้ว่า chunk มาจากหน้าไหน ช่วย traceability |
| **Token-accurate** | ใช้ `tiktoken` นับ token จริง ไม่ใช่แค่ character count → หลีกเลี่ยง API error |
| **Customizable** | ปรับได้ผ่าน env, API global, หรือ per-request |
| **Multi-layer** | มี fallback logic สำหรับแต่ละ file type |
| **OCR support** | รองรับ scanned documents ผ่าน Docling |

### จุดอ่อน / ข้อควรระวัง

| จุด | รายละเอียด |
|-----|-----------|
| **Plain text ไม่มี overlap** | `process_text_file()` ไม่ทำ overlap → อาจเสีย context ที่ขอบ chunk |
| **Plain text ใช้ character count** | ไม่นับ token จริง ขนาด chunk จริงอาจต่างจาก NLP perspective |
| **Docling chunks = 1 page** | ถ้าหน้ามีเนื้อหายาวมาก → chunk ใหญ่เกินไป อาจเกิน embedding token limit |
| **Separator เดียว** | Langflow SplitText รองรับ separator เดียวต่อครั้ง (ไม่ใช่ recursive list) |
| **ไม่มี semantic chunking** | ไม่ได้แบ่งตาม meaning/topic แบบ advanced chunking strategies |
| **Flow update อาจ fail แบบ silent** | ถ้า Langflow flow update ล้มเหลว settings จะถูก save แต่ flow จะไม่อัพเดต (logged แต่ไม่ raise error) |

---

## Limitations

1. **ไม่รองรับ**: Semantic chunking, Sliding window chunking, Sentence-based chunking
2. **ไม่รองรับ**: Multi-level separators (เช่น `["\n\n", "\n", " "]` แบบ LangChain `RecursiveCharacterTextSplitter`)
3. **ไม่รองรับ**: Chunk size ในหน่วย token โดยตรงที่ Langflow layer (ใช้ characters)
4. **ขึ้นกับ Langflow**: Chunking หลักผ่าน Langflow component — ถ้า Langflow service down จะไม่สามารถ ingest ได้
5. **ไม่มี re-chunking**: ถ้าเปลี่ยน chunk_size หลังจาก index เอกสารแล้ว จะต้อง re-ingest ทั้งหมดด้วยตัวเอง

---

## Quick Reference

```bash
# ดู chunk settings ปัจจุบัน
GET /api/settings

# เปลี่ยน chunk size (global)
PATCH /api/settings
{"chunk_size": 500, "chunk_overlap": 50}

# Ingest พร้อม custom chunk settings (per-request)
POST /api/ingest
{
  "files": [...],
  "settings": {
    "chunkSize": 300,
    "chunkOverlap": 30,
    "separator": "\n"
  }
}

# Environment override
CHUNK_SIZE=800 CHUNK_OVERLAP=100 docker-compose up
```

---

*Generated: 2026-03-17 | OpenRAG Chunking System Documentation*
