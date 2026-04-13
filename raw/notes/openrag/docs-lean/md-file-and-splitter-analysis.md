# ไฟล์ .md ต้องผ่าน Docling ไหม? และ CharacterTextSplitter รับได้ทุก doc จริงหรือ?

---

## คำถาม 1: ไฟล์ .md ต้องผ่าน Docling ไหม?

คำตอบ: **ขึ้นกับ path ที่ใช้**

### Langflow Path (default) — .md ผ่าน Docling ✅

```
.md file
  → LangflowFileProcessor.process_item()
      → .txt ถูก rename เป็น .md ก่อน (workaround)  ← (line ~756 processors.py)
      → langflow_file_service.upload_and_ingest_file()
          → Langflow flow (ingestion_flow.json)
              → [DoclingRemote] ← ผ่าน Docling เสมอ ไม่มีข้อยกเว้น
              → [ExportDoclingDocument] → Markdown text
              → [Split Text]
              → [OpenSearch]
```

Docling จัดการ .md ได้โดย parse Markdown structure → แปลงเป็น DoclingDocument → Export กลับเป็น Markdown อีกที
ดูเหมือน "ทำงานซ้ำ" แต่ Docling ทำให้ format มาตรฐานและ extract structure

### Backend Path (DISABLE_INGEST_WITH_LANGFLOW=true) — .md ข้าม Docling ✅

```python
# src/models/processors.py:207
file_ext = os.path.splitext(file_path)[1].lower()

if file_ext in ('.txt', '.md'):   # ← bypass Docling
    slim_doc = process_text_file(file_path)
else:
    full_doc = await convert_file(file_path, ...)   # ← ผ่าน Docling
    slim_doc = extract_relevant(full_doc)
```

Backend path ถือว่า .md/.txt เป็น plain text ไม่ต้องให้ Docling parse

### สรุป: .md กับ Docling

| Path | .md ผ่าน Docling? | วิธีตัด |
|---|---|---|
| **Langflow (default)** | **ใช่** — ผ่านทุกไฟล์ | Split Text (CharacterTextSplitter, `"\n"`) |
| **Backend** | **ไม่** — bypass | process_text_file (`"\n\n"`, ~1000 chars, no overlap) |

---

## คำถาม 2: CharacterTextSplitter ได้ทุกเอกสารเลยหรอ?

**คำตอบ: ได้ทุก "text" แต่ไม่ได้ทุก "structure"** — มีจุดอ่อนสำคัญ

### ก่อนเข้า Split Text: ข้อความถูก normalize หมดแล้ว

```
PDF/DOCX/PPT/MD/etc.
    → Docling Serve    (parse ทุก format)
    → ExportDoclingDocument  (→ Markdown text)
    → DataFrame Operations   (เพิ่ม metadata)
    → Split Text   ← รับ Markdown plain text เสมอ
```

Split Text ไม่เคยเห็น "ไฟล์ต้นฉบับ" — เห็นแค่ Markdown string

### Separator ที่ใช้จริงใน OpenRAG: `"\n"` (single newline)

```json
// flows/ingestion_flow.json node SplitText-QIKhg
"separator": {
    "value": "\n"   ← single newline
}
```

### ปัญหา: `CharacterTextSplitter` + `"\n"` + Markdown ไม่ ideal

```
Markdown ที่ได้จาก ExportDoclingDocument:
──────────────────────────────────────────────────────────────
# Introduction to Python

Python is a programming language.
It is easy to learn.

## Data Structures

| Name   | Type  | Size |
|--------|-------|------|
| list   | seq   | n    |
| dict   | map   | n    |

```python
def hello():
    print("Hello")
```
──────────────────────────────────────────────────────────────

CharacterTextSplitter ตัดด้วย "\n":
splits = [
  "# Introduction to Python",        ← header อยู่คนเดียว
  "",                                 ← empty line
  "Python is a programming language.",
  "It is easy to learn.",
  "",
  "## Data Structures",
  "",
  "| Name   | Type  | Size |",
  "|--------|-------|------|",         ← แยกจาก table header
  "| list   | seq   | n    |",        ← table row แยกออกไป
  "| dict   | map   | n    |",
  "",
  "```python",                        ← code block เปิด
  "def hello():",
  "    print(\"Hello\")",
  "```"                               ← code block ปิด
]

merge จนถึง chunk_size=1000:
  Chunk 1: "# Introduction to Python\n\nPython is a programming language.\n..."
           (merged หลาย splits จนถึง 1000 chars)
  Chunk 2: overlap 200 chars + ต่อไป
```

### ปัญหาที่เกิดจริง

| กรณี | ผลกระทบ |
|---|---|
| **Table ถูกตัดกลาง** | Header row กับ data rows อาจอยู่คนละ chunk |
| **Code block ถูกตัด** | ` ```python ` และ ` ``` ` อาจอยู่คนละ chunk → context หาย |
| **Heading อยู่คนเดียว** | `# Introduction` merge กับข้อความถัดไปปกติ (OK) แต่ถ้า heading อยู่ท้าย chunk → chunk ถัดไปไม่มี context |
| **Single split ใหญ่กว่า chunk_size** | เก็บทั้งก้อน ไม่ตัดต่อ → chunk ใหญ่กว่าที่ตั้งใจ |
| **Empty lines** | `""` นับเป็น splits แต่ไม่มี content → overhead เล็กน้อย |

### สิ่งที่ CharacterTextSplitter ไม่รู้

```
❌ ไม่รู้ Markdown structure (header, list, table, code block)
❌ ไม่รู้ semantic boundary (จบ section, จบ concept)
❌ ตัด separator ได้แค่ 1 ตัว  (ต่างจาก RecursiveCharacterTextSplitter)
❌ split เดี่ยวที่ใหญ่กว่า chunk_size → ไม่ตัดต่อ (เก็บทั้งก้อน)
✅ เร็ว
✅ predictable
✅ overlap ช่วย preserve context ข้าม chunk boundary
```

---

## เปรียบเทียบ: CharacterTextSplitter vs RecursiveCharacterTextSplitter

LangChain มี splitter หลายตัว แต่ OpenRAG เลือกใช้แค่ `CharacterTextSplitter`:

```
CharacterTextSplitter (ที่ OpenRAG ใช้)
  └── ตัวดียว: "\n"
  └── merge splits จนถึง chunk_size
  └── ง่าย คาดเดาได้

RecursiveCharacterTextSplitter (ไม่ได้ใช้)
  └── ลอง separators ตามลำดับ: ["\n\n", "\n", " ", ""]
  └── ถ้า "\n\n" ตัดแล้วยังใหญ่ → ลอง "\n"
  └── ถ้า "\n" ยังใหญ่ → ลอง " "
  └── เหมาะกับ plain text ทั่วไปมากกว่า
  └── LangChain เองแนะนำให้ใช้ตัวนี้เป็น default

MarkdownTextSplitter (ไม่ได้ใช้)
  └── separators = ["# ", "## ", "### ", "\n\n", "\n", " ", ""]
  └── เข้าใจ Markdown structure
  └── เหมาะสุดสำหรับ output ของ ExportDoclingDocument
```

---

## สรุปภาพรวม

```
ไฟล์ .md (Langflow path)
  ↓
Docling Serve (parse Markdown → DoclingDocument)
  ↓
ExportDoclingDocument (→ Markdown text อีกรอบ)
  ↓
Split Text: CharacterTextSplitter(separator="\n", chunk_size=1000, overlap=200)
  ↓ ตัดทุก \n → merge จนถึง 1000 → overlap 200
  ↓ ⚠️ tables / code blocks อาจถูกตัดกลาง
  ↓
OpenSearch


ไฟล์ .md (Backend path)
  ↓
process_text_file() — ข้าม Docling
  ↓ split("\n\n") → merge จนถึง 1000
  ↓ ⚠️ ไม่มี overlap เลย
  ↓
OpenSearch
```

### ถ้าต้องการ chunking ที่ดีกว่าสำหรับ .md

เปลี่ยน separator ใน Langflow UI จาก `\n` เป็น `\n\n` หรือแก้ flow ให้ใช้ `MarkdownTextSplitter` แทน
→ จะตัดที่ paragraph boundary แทนที่จะตัดทุก newline
