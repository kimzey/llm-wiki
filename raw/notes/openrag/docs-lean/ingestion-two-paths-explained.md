# ทำไมต้องมี 2 Path? Langflow vs Backend Ingestion

## คำตอบสั้น: มันเป็น "สวิตช์" ตัวเดียว

```
DISABLE_INGEST_WITH_LANGFLOW=false  →  ใช้ Langflow pipeline  (default)
DISABLE_INGEST_WITH_LANGFLOW=true   →  ใช้ Backend Python pipeline
```

ไม่ใช่ว่าทำงาน "ร่วมกัน" พร้อมกัน — แต่เป็นการเลือกว่าจะใช้ path ไหน **ต่อ request เดียวกัน**

---

## ทำไมต้องมีสองแบบ?

| เหตุผล | อธิบาย |
|---|---|
| **Legacy vs Modern** | Backend path เกิดก่อน ตอน OpenRAG ยังไม่ใช้ Langflow เต็มรูปแบบ |
| **Fallback** | ถ้า Langflow มีปัญหา หรือ deploy ในสภาพแวดล้อมที่ไม่มี Langflow service ใช้ Backend ได้เลย |
| **Feature parity ต่างกัน** | Langflow path รองรับ multi-model embedding, metadata ที่ละเอียดกว่า, ปรับ chunk settings ผ่าน UI ได้ |
| **Performance tradeoff** | Backend path เร็วกว่าสำหรับไฟล์ง่าย (.txt, .md) เพราะข้าม Langflow overhead |

---

## Request แต่ละ Type ใช้ Path ไหน?

```
src/api/router.py  ← ตัวตัดสิน
│
├── DISABLE_INGEST_WITH_LANGFLOW=true
│     └── api/upload.py  → DocumentFileProcessor → processors.py (Backend path)
│
└── DISABLE_INGEST_WITH_LANGFLOW=false  (default)
      └── LangflowFileProcessor → langflow_file_service.upload_and_ingest_file()
            → Langflow flow (ingestion_flow.json)
```

ทั้ง UI upload, connector sync, และ default documents startup ล้วนผ่านสวิตช์นี้ทั้งหมด

---

## เปรียบเทียบ Chunking: Langflow Split Text vs Backend

### ภาพรวม

| Feature | Langflow Split Text | Backend: extract_relevant() | Backend: process_text_file() |
|---|---|---|---|
| ใช้กับไฟล์ประเภทไหน | ทุกไฟล์ (ผ่าน Docling แล้ว) | PDF, DOCX, PPT, etc. (complex) | .txt, .md เท่านั้น |
| Library | `CharacterTextSplitter` (LangChain) | Python custom code | Python custom code |
| Logic การตัด | separator → merge จนถึง chunk_size | ตามหน้า (page_no) + tables | ตาม paragraph `\n\n` |
| Chunk Size | ปรับได้ (default: 1000 chars) | ไม่จำกัด — 1 หน้า = 1 chunk | ~1000 chars |
| Overlap | ปรับได้ (default: 200 chars) | **ไม่มี overlap** | **ไม่มี overlap** |
| Separator | ปรับได้ (default: `\n`) | N/A | `\n\n` (hardcoded) |
| ผล | uniform size chunks | chunks ไม่สม่ำเสมอ (ขึ้นกับความยาว page) | chunks ตาม paragraph |

---

## วิเคราะห์ Langflow Split Text Component

### Code Path

```python
# SplitTextComponent.split_text() →
CharacterTextSplitter(
    chunk_overlap = self.chunk_overlap,   # default 200
    chunk_size    = self.chunk_size,      # default 1000
    separator     = separator,            # default "\n"
    keep_separator= keep_sep,             # default False
)
splitter.split_documents(documents)
```

### Algorithm: CharacterTextSplitter (LangChain)

```
Input text (ตัวอย่าง):
─────────────────────────────────────────────────────────
"การใช้งาน Python\nสำหรับ Data Science\nเรียนรู้ Pandas\n
NumPy เป็นไลบรารี่หลัก\nใช้ใน machine learning\n..."
─────────────────────────────────────────────────────────

Step 1: SPLIT by separator ("\n")
  → ["การใช้งาน Python",
     "สำหรับ Data Science",
     "เรียนรู้ Pandas",
     "NumPy เป็นไลบรารี่หลัก",
     "ใช้ใน machine learning", ...]

Step 2: MERGE splits จนถึง chunk_size=1000
  → รวม splits ต่อๆ กันจนกว่าจะเกิน 1000 chars
  → Chunk 1: "การใช้งาน Python\nสำหรับ Data Science\n..." (≤1000 chars)
  → Chunk 2: overlap 200 chars จาก Chunk 1 + text ต่อไป

Step 3: ถ้า split เดี่ยวยาวกว่า chunk_size → เก็บทั้งหมดเป็น 1 chunk (ไม่ตัดอีก)
```

### จุดสำคัญ: ตัดได้กี่วิธี?

**ตัดได้ 1 separator เท่านั้น** — `CharacterTextSplitter` ไม่ใช่ `RecursiveCharacterTextSplitter`

```
separator options (ปรับได้ใน Langflow UI):
  "\n"    → ตัดทุก newline  (default)
  "\n\n"  → ตัดเฉพาะ paragraph breaks
  "."     → ตัดทุก sentence
  " "     → ตัดทุก word
  ""      → ตัดทุก character
  ใส่อะไรก็ได้ เช่น "###" สำหรับ markdown headers
```

ต่างจาก `RecursiveCharacterTextSplitter` ที่ลอง separator หลายตัวตามลำดับ

### รองรับทุกเอกสารไหม?

```
Input types ที่ Split Text รับได้:
  ✅ DataFrame  → แปลงเป็น LangChain Documents ผ่าน to_lc_documents()
  ✅ Data       → to_lc_document()
  ✅ Message    → แปลงเป็น Data ก่อน แล้ว recursive call
  ✅ list[Data]

⚠️  แต่มันรับ text ที่ผ่าน Docling แล้วเท่านั้น
    ไม่ได้อ่านไฟล์โดยตรง — ต้องผ่าน Docling Serve → ExportDoclingDocument ก่อน
```

---

## วิเคราะห์ Backend Chunking

### Path 1: `extract_relevant()` (สำหรับ PDF/DOCX/etc.)

```python
# src/utils/document_processing.py

# Docling ส่งกลับมาเป็น DoclingDocument dict
# มี "texts" (รายการ text fragments พร้อม prov.page_no)
# มี "tables" (รายการตาราง)

# ★ CHUNKING STRATEGY: 1 page = 1 chunk
page_texts = defaultdict(list)
for txt in doc_dict.get("texts", []):
    page_no = prov[0].get("page_no")
    page_texts[page_no].append(txt.get("text", ""))

for page in sorted(page_texts):
    chunks.append({
        "page": page,
        "type": "text",
        "text": "\n".join(page_texts[page])  # รวมทุก fragment ใน page เดียวกัน
    })

# Tables: แยก chunk ต่างหาก (1 table = 1 chunk)
```

**ปัญหา**: ถ้าหน้าหนึ่งยาวมาก (เช่น PDF หน้าที่มีข้อความเยอะ) → chunk ใหญ่มาก อาจเกิน token limit

นั่นคือเหตุผลที่ `chunk_texts_for_embeddings()` ต้องช่วย batch ก่อนส่ง embedding API:

```python
# src/services/document_service.py
def chunk_texts_for_embeddings(texts, max_tokens=8000):
    # ★ นี่คือ "safety net" สำหรับ embedding API limit เท่านั้น
    # ไม่ใช่ chunking strategy จริงๆ
    # ถ้า text เดี่ยวใหญ่เกิน max_tokens → ตัดทิ้ง (truncate)
```

### Path 2: `process_text_file()` (สำหรับ .txt, .md)

```python
# src/utils/document_processing.py
chunk_size = 1000  # hardcoded ไม่ปรับได้

paragraphs = content.split('\n\n')  # ตัดตาม double newline
current_chunk = ""

for para in paragraphs:
    if len(current_chunk) + len(para) + 2 > chunk_size and current_chunk:
        # บันทึก chunk และเริ่มใหม่
        chunks.append({"page": chunk_index + 1, "text": current_chunk})
        current_chunk = para
    else:
        current_chunk += "\n\n" + para
```

**จุดต่างจาก Langflow**:
- `\n\n` hardcoded (ปรับไม่ได้)
- chunk_size 1000 hardcoded (ปรับไม่ได้)
- **ไม่มี overlap เลย** ← ต่างจาก Langflow ที่ default 200

---

## ตาราง: ผลลัพธ์จาก Input เดียวกัน

สมมติ document มีข้อความ 3000 characters, ตัดด้วย separator `\n`:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Langflow Split Text (chunk_size=1000, overlap=200)                 │
├─────────────────────────────────────────────────────────────────────┤
│  Chunk 1: chars 0-999    (1000 chars)                               │
│  Chunk 2: chars 800-1799 (overlap 200 + 800 new = 1000 chars)       │
│  Chunk 3: chars 1600-2599                                           │
│  Chunk 4: chars 2400-2999                                           │
│  Total: 4 chunks (uniform size, overlapping)                        │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Backend extract_relevant() (PDF, 3 pages of ~1000 chars each)      │
├─────────────────────────────────────────────────────────────────────┤
│  Chunk 1: page 1 text (อาจ 800 chars)                               │
│  Chunk 2: page 2 text (อาจ 1500 chars — ยาวมากถ้าหน้านั้นยาว)      │
│  Chunk 3: page 3 text (อาจ 700 chars)                               │
│  Total: 3 chunks (ขนาดไม่สม่ำเสมอ, ไม่มี overlap)                 │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│  Backend process_text_file() (.txt, 3000 chars, 3 paragraphs)       │
├─────────────────────────────────────────────────────────────────────┤
│  Chunk 1: paragraphs จนถึง ~1000 chars                              │
│  Chunk 2: paragraphs ต่อไป                                          │
│  Total: 2-3 chunks (ตาม paragraph boundary, ไม่มี overlap)          │
└─────────────────────────────────────────────────────────────────────┘
```

---

## สรุป: ใช้ Path ไหนดี?

```
ต้องการ...                              → ใช้
───────────────────────────────────────────────────────────
ปรับ chunk size/overlap ผ่าน UI          Langflow (default)
Multi-model embeddings                   Langflow (default)
Metadata ละเอียด (owner, acl, etc.)      Langflow (default)
รองรับ PDF/DOCX/PPT ครบถ้วน             Langflow (default)
Deploy โดยไม่มี Langflow service         Backend (DISABLE=true)
Simple .txt ingest เร็วๆ                 Backend พอ แต่ ไม่มี overlap
Debug / custom pipeline                 Backend แก้ Python code ได้โดยตรง
```

**แนะนำ**: ใช้ Langflow path (default) เสมอ เว้นแต่มีเหตุผลจำเป็นต้อง disable

---

## Key Files Reference

```
src/api/router.py                         ← สวิตช์หลัก DISABLE_INGEST_WITH_LANGFLOW
src/models/processors.py                 ← Backend processor classes
src/utils/document_processing.py         ← extract_relevant(), process_text_file()
src/services/document_service.py         ← chunk_texts_for_embeddings() (embedding batching)
flows/ingestion_flow.json                ← Langflow flow (SplitText node = QIKhg)
```
