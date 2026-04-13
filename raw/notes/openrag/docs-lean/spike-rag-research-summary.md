# Spike Research: RAG System with Vector Database

**Spike Card**: ศึกษา Vector Database สำหรับระบบ Q&A อิงจากเอกสาร (RAG)
**ผู้ศึกษา**: —
**วันที่**: 2026-03-18

---

## TL;DR

RAG (Retrieval-Augmented Generation) คือการให้ LLM ตอบคำถามโดยดึงข้อมูลจาก Document จริงๆ แทนที่จะพึ่ง training data อย่างเดียว ระบบประกอบด้วย 3 ส่วนหลัก: **Ingestion** (นำเอกสารเข้า), **Retrieval** (ค้นหา chunk ที่เกี่ยวข้อง), และ **Generation** (LLM สร้างคำตอบ)

pgvector ทำได้จริง แต่มีทางเลือกที่ครบกว่า — ดู [Tech Stack](#4-tech-stack)

---

## 1. Data Organization

### ปัญหา
ถ้า dump เอกสารทั้งหมดลง vector database โดยไม่จัดระเบียบ — search จะดึง chunk จากเอกสารที่ไม่เกี่ยวข้องขึ้นมาปน

### วิธีจัดระเบียบ: Metadata + Filter

ทุก chunk ที่เก็บใน database ควรมี metadata ติดไปด้วย:

```json
{
  "text": "...chunk content...",
  "embedding": [0.12, -0.45, ...],

  "document_id": "abc123",
  "filename":    "hr-policy-2026.pdf",
  "mimetype":    "application/pdf",
  "page":        3,
  "file_size":   204800,

  "owner":          "user-id",
  "allowed_users":  ["alice", "bob"],
  "allowed_groups": ["hr-team"],

  "source":         "google-drive",
  "indexed_time":   "2026-03-18T10:00:00",
  "embedding_model":"text-embedding-3-small"
}
```

### Pattern การจัดหมวด

| วิธี | ตัวอย่าง | เหมาะกับ |
|---|---|---|
| **By source** | `source = "confluence"` | หลาย data source |
| **By owner/team** | `owner = "hr-team"` | access control |
| **By document type** | `mimetype = "application/pdf"` | filter เอกสารประเภทเดียวกัน |
| **By tag/category** | `category = "policy"` | จัดหมวดด้วย business logic |

> **ข้อสำคัญ**: ออกแบบ metadata schema ให้ดีตั้งแต่ต้น เพราะ chunk ที่ index ไปแล้วต้อง re-index ใหม่ถ้าอยากเปลี่ยน

---

## 2. Data Chunking

### ทำไมต้อง Chunk?
- LLM มี context window จำกัด (ไม่สามารถยัด document ทั้งหมดลงไปได้)
- Vector embedding ทำงานดีกับ text ที่มีความหมายเดียว ไม่ใช่ document ยาวๆ
- Chunk เล็กกว่า = search แม่นยำกว่า แต่ context น้อยกว่า

### เทคนิค Chunking ที่มี

#### Fixed-Size Chunking ← ที่ใช้อยู่ใน OpenRAG
```
ตัดตามจำนวนตัวอักษร + overlap
chunk_size = 1000 chars, overlap = 200 chars

[====chunk1====]
         [====chunk2====]
                  [====chunk3====]
```
- ✅ เร็ว ง่าย คาดเดาได้
- ❌ อาจตัดกลางประโยค กลางความหมาย
- ❌ ต้องมี overlap เพื่อรักษา context ข้าม chunk

#### Recursive Chunking ← แนะนำสำหรับ plain text ทั่วไป
```
ลอง separator หลายระดับ ถ้า chunk ยังใหญ่เกิน → ลง level ต่อไป
1. "\n\n" (paragraph)
2. "\n"   (line)
3. ". "   (sentence)
4. " "    (word)
```
- ✅ เหมาะกับเอกสารที่มีรูปแบบหลากหลาย
- ✅ ตัดที่ natural boundary ก่อน

#### Semantic Chunking ← แพงสุด ดีสุด
```
ตัดตาม "ความหมาย" ไม่ใช่ตัวอักษร
วิเคราะห์ว่าตรงไหนที่หัวข้อเปลี่ยน → ตัดตรงนั้น
```
- ✅ chunk มีความหมายสมบูรณ์ที่สุด
- ❌ ช้า ต้อง embed ก่อน chunk
- ❌ ค่าใช้จ่าย embedding สูงกว่า

#### Structural Chunking ← เหมาะกับ structured docs
```
1 chunk = 1 หน้า PDF / 1 section / 1 table
```
- ✅ ไม่ตัดกลางความหมาย
- ❌ chunk size ไม่สม่ำเสมอ ขึ้นกับขนาด page

### ค่า Default ที่เหมาะสม (starting point)

| ประเภทเอกสาร | Chunk Size | Overlap | วิธี |
|---|---|---|---|
| เอกสารทั่วไป | 1000 chars | 200 chars | Fixed/Recursive |
| เอกสาร technical | 500–800 chars | 100–150 chars | Recursive |
| PDF หน้าเยอะ | per-page | none | Structural |
| FAQ / Q&A | per-question | none | Structural |

> **Rule of thumb**: เริ่มที่ 1000/200 ก่อน แล้ว tune ตาม retrieval quality จริงๆ

---

## 3. Q&A Mechanism: Flow ทั้งหมด

### Phase 1: Ingestion (เตรียมข้อมูล)

```
Document (PDF/DOCX/MD/etc.)
    │
    ▼
[Document Parser]          ← แปลง binary → text
    e.g. Docling, PyPDF, Unstructured
    │
    ▼
[Text Cleaner]             ← ตัด header/footer noise ออก
    │
    ▼
[Chunker]                  ← ตัดเป็น chunks
    │ chunks[]
    ▼
[Embedding Model]          ← แปลง text → vector (float[])
    e.g. text-embedding-3-small, nomic-embed-text (Ollama)
    │ (chunk, vector, metadata)[]
    ▼
[Vector Database]          ← เก็บ chunk + vector + metadata
    e.g. pgvector, OpenSearch, Qdrant
```

### Phase 2: Query (ตอบคำถาม)

```
User: "นโยบายลาคลอดของบริษัทมีกี่วัน?"
    │
    ▼
[Embedding Model]          ← แปลง คำถาม → query vector
    │
    ▼
[Vector Search]            ← หา chunks ที่ vector ใกล้เคียงกับคำถาม
    SELECT * FROM docs
    ORDER BY embedding <-> query_vector  (cosine similarity)
    LIMIT 5
    │ top-k chunks
    ▼
[Context Builder]          ← รวม chunks เป็น context
    "chunk1 content\nchunk2 content\n..."
    │
    ▼
[LLM]                      ← ส่ง context + คำถาม
    System: "ตอบคำถามโดยอิงจากข้อมูลต่อไปนี้เท่านั้น: {context}"
    User:   "นโยบายลาคลอดของบริษัทมีกี่วัน?"
    │
    ▼
"ตามเอกสาร HR Policy บริษัทมีวันลาคลอด 98 วัน (ตาม พ.ร.บ.คุ้มครองแรงงาน)
 อ้างอิง: hr-policy-2026.pdf หน้า 12"
```

### Hybrid Search (แนะนำ)
Vector search อย่างเดียวอาจพลาด keyword สำคัญ — ใช้ทั้งสองแบบรวมกัน:

```
Query → [Vector Search]  ─┐
                           ├→ [Rank Fusion (RRF)] → Top results → LLM
Query → [Keyword Search] ─┘
```

---

## 4. Tech Stack

### pgvector (Spike Target)

| หัวข้อ | รายละเอียด |
|---|---|
| **ติดตั้ง** | Extension บน PostgreSQL (`CREATE EXTENSION vector`) |
| **Data type** | `vector(1536)` — ระบุ dimension ตอนสร้าง column |
| **Index** | IVFFlat หรือ HNSW |
| **Search** | `embedding <-> $1` (L2), `embedding <#> $1` (cosine) |
| **Hybrid** | ทำได้ — รวม full-text search (`tsvector`) กับ vector search |

```sql
-- Schema ตัวอย่าง
CREATE TABLE documents (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    document_id TEXT,
    filename    TEXT,
    page        INT,
    text        TEXT,
    embedding   vector(1536),
    owner       TEXT,
    metadata    JSONB,
    indexed_at  TIMESTAMP DEFAULT NOW()
);

CREATE INDEX ON documents USING hnsw (embedding vector_cosine_ops);

-- Query
SELECT text, filename, page,
       1 - (embedding <=> $1) AS score
FROM documents
WHERE owner = $2
ORDER BY embedding <=> $1
LIMIT 5;
```

### เปรียบเทียบ Options

| | **pgvector** | **OpenSearch** | **Qdrant** |
|---|---|---|---|
| Setup | ใช้ Postgres ที่มีอยู่ได้เลย | ต้องติดตั้ง cluster แยก | service แยก |
| Hybrid Search | ทำเองผ่าน SQL | built-in (BM25 + vector) | built-in |
| Scalability | vertical scaling ตาม PG | horizontal scaling ดี | horizontal scaling ดี |
| Metadata Filter | SQL WHERE clause | query DSL | filter objects |
| Ops Complexity | ต่ำ — ทีมรู้ Postgres อยู่แล้ว | กลาง | ต่ำ-กลาง |
| Use Case | ข้อมูลน้อย-กลาง, ทีมชอบ SQL | เอกสารเยอะ + full-text สำคัญ | vector-first |

### ข้อสรุป Tech Stack

**pgvector เหมาะถ้า:**
- ทีมใช้ Postgres อยู่แล้ว ไม่อยาก manage service เพิ่ม
- ข้อมูลยังไม่เยอะมาก (< 10M documents)
- ต้องการ transactional consistency กับข้อมูลอื่นในระบบ

**OpenSearch/Qdrant เหมาะกว่าถ้า:**
- ต้องการ hybrid search แบบ built-in พร้อมใช้
- เอกสารเยอะ ต้องการ horizontal scale
- ต้องการ relevance tuning ละเอียด

---

## 5. สิ่งที่ต้องตัดสินใจก่อน Implement

- [ ] **Embedding Model**: OpenAI `text-embedding-3-small` (cloud, ค่าใช้จ่าย) หรือ Ollama `nomic-embed-text` (on-premise, ฟรี)?
- [ ] **Chunk Strategy**: Fixed-size 1000/200 เป็น default หรือ Recursive?
- [ ] **Document Parser**: จะใช้ Docling (รองรับ PDF/DOCX/PPT ครบ) หรือ library อื่น?
- [ ] **Vector DB**: pgvector บน Postgres เดิม หรือ service แยก?
- [ ] **Access Control**: ทุกคนเห็น document เดียวกันหมด หรือต้องมี per-user/per-team filter?
- [ ] **Re-ingestion**: ถ้า document เปลี่ยน จะ re-index ทั้งหมดหรือ upsert เฉพาะที่เปลี่ยน?

---

## 6. สิ่งที่รู้จากการ Research จริง (OpenRAG)

ได้ศึกษา OpenRAG ซึ่งเป็น production RAG system จริงๆ พบว่า:

| ประเด็น | สิ่งที่พบ |
|---|---|
| Vector DB | ใช้ **OpenSearch** ไม่ใช่ pgvector |
| Chunker | `CharacterTextSplitter` — separator `"\n"`, size 1000, overlap 200 |
| Doc Parser | **Docling Serve** — รองรับ PDF/DOCX/PPT/MD/Images ครบ |
| Embedding | multi-model — `text-embedding-3-small` default |
| Ingestion pipeline | 2 path: Langflow (UI-configurable) / Python backend |
| Hybrid Search | Vector + BM25 keyword ใน OpenSearch |
| Metadata | document_id, filename, page, owner, allowed_users, indexed_time ฯลฯ |

**Lesson learned**: chunk size ที่ดีต้องทดสอบกับ data จริง — ไม่มี one-size-fits-all

---

## References

- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [LangChain Text Splitters](https://python.langchain.com/docs/concepts/text_splitters/)
- [Docling](https://docling-project.github.io/docling/)
- [OpenRAG](https://github.com/langflow-ai/openrag)
