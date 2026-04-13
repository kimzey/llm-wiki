# ระบบ Arona RAG - คู่มือครอบคลุม

## สารบัญ

1. [ภาพรวม](#ภาพรวม)
2. [สถาปัตยกรรม](#สถาปัตยกรรม)
3. [คอมโพเนนต์หลัก](#คอมโพเนนต์หลัก)
4. [การไหลของข้อมูล](#การไหลของข้อมูล)
5. [รายละเอียดการนำไปใช้](#รายละเอียดการนำไปใช้)
6. [เทคนิคการปรับแต่ง](#เทคนิคการปรับแต่ง)
7. [ความปลอดภัยและการจำกัดอัตรา](#ความปลอดภัยและการจำกัดอัตรา)
8. [โครงสร้างพื้นฐาน](#โครงสร้างพื้นฐาน)
9. [การวิเคราะห์ต้นทุน](#การวิเคราะห์ต้นทุน)

---

## ภาพรวม

**Arona** คือระบบ RAG (Retrieval-Augmented Generation) ที่พร้อมใช้งานใน Production สร้างขึ้นสำหรับเอกสารของ Elysia ถูกสร้างขึ้นเพื่อเป็นทางเลือกค้นหาเอกสารด้วย AI ที่ราคาย่อมเมื่อเทียบกับ SaaS แพงๆ เช่น Mintlify ($250/เดือน), Kapa, และ Mendable

### ฟีเจอร์หลัก
- **Hybrid Search**: รวม BM25 (Full-Text Search) กับ Vector (Semantic) Search
- **Multi-Layer Caching**: Semantic, Redis, LRU, และ Burst caches
- **Security-First**: Proof of Work + Cloudflare Turnstile
- **Cost-Optimized**: ~$21-27/เดือนทั้งหมด (server + AI costs)
- **Stateless Design**: ไม่ต้องมี session management, ขยายได้เต็มที่

---

## สถาปัตยกรรม

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              คำขอจาก USER                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SECURITY LAYER                                      │
│  ┌──────────────────┐           ┌──────────────────┐                        │
│  │  Proof of Work  │           │   Cloudflare     │                        │
│  │  (19 bits SHA256)│──────────▶│   Turnstile      │                        │
│  └──────────────────┘           └──────────────────┘                        │
│                                           │                                  │
│                                           ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │              Rate Limiting (10 req / 35 sec)             │               │
│  └──────────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CACHE LAYER                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐    │
│  │ Redis Cache  │  │   Semantic   │  │   LRU Embed  │  │  Burst Cache │    │
│  │ (Exact Match)│  │   Cache      │  │   Cache      │  │              │    │
│  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │               Cache Miss?          │
                    └─────────────────┬─────────────────┘
                                      │ Yes
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         QUERY PROCESSING                                    │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │  Query Normalization (GPT-OSS-20B)                       │               │
│  │  - ลบคำ filler                                          │               │
│  │  - ลดเหลือ canonical search intent                     │               │
│  │  - Output ไม่เกิน 12 คำ                                │               │
│  └──────────────────────────────────────────────────────────┘               │
│                                     │                                         │
│                                     ▼                                         │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │  Embedding Generation (OpenAI text-embedding-3-small)    │               │
│  │  - 1536 dimensions                                       │               │
│  │  - 3 embeddings per chunk: content, title, filename      │               │
│  └──────────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RETRIEVAL LAYER                                      │
│  ┌────────────────────────┐         ┌────────────────────────┐              │
│  │   BM25 Search          │         │   Vector Search        │              │
│  │   (ParadeDB/pg_search) │         │   (Cosine Similarity)  │              │
│  └────────────────────────┘         └────────────────────────┘              │
│                │                                    │                         │
│                └────────────┬───────────────────────┘                         │
│                             ▼                                                  │
│              ┌─────────────────────────────┐                                  │
│              │  Score Fusion & Filtering   │                                  │
│              │  BM25: 77.5% | Vector: RRF  │                                  │
│              └─────────────────────────────┘                                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GENERATION LAYER                                     │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │  LLM (GPT-OSS-120B via OpenRouter/Groq)                   │               │
│  │  - Tool Calling (search, readPage, tableOfContents)       │               │
│  │  - Streaming Response                                     │               │
│  │  - Max 8-12 steps (12 if think mode)                      │               │
│  │  - Max 32 references                                      │               │
│  └──────────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RESPONSE & CACHING                                  │
│  ┌──────────────────────────────────────────────────────────┐               │
│  │  Stream Response + Append Sources                        │               │
│  │  Add Checksum for Integrity                              │               │
│  │  Cache to All Layers                                     │               │
│  │  Log to Axiom (OpenTelemetry)                            │               │
│  └──────────────────────────────────────────────────────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## คอมโพเนนต์หลัก

### 1. Embedding System (`src/libs/embedding.ts`)

**วัตถุประสงค์**: แปลงข้อความเป็นเวคเตอร์สำหรับ semantic search

**Model**: OpenAI `text-embedding-3-small` (1536 dimensions)

**ฟีเจอร์หลัก**:
- **Filler Word Stripping**: ลบคำ filler ทางการสนทนาออกเพื่อ embeddings ที่ดีขึ้น
- **LRU Cache**: สูงสุด 750 embeddings, TTL 9 ชั่วโมง (~4.4MB memory)
- **Pending Cache**: ป้องกันการเรียก embedding ซ้ำพร้อมกัน

```typescript
// Filler pattern ที่ถูกลบออกจาก queries
const fillerPattern = /^(?:can you tell me|i would like to|would you kindly|...)/gi

export const stripFillers = (q: string) => {
    q = q.toLowerCase()
    while (fillerPattern.test(q)) q = q.replace(fillerPattern, '')
    return q.replace(/[()\[\]{}@#$%^&*!?.,:;&]/g, '').trim()
}
```

**Multiple Embeddings per Document**:
1. **Content Embedding**: ความหมายหลัก
2. **Title Embedding**: context ของหัวข้อ section
3. **Filename Embedding**: context ของไฟล์/เอกสาร

---

### 2. Vector Search (`src/modules/ai/const.ts`)

**วัตถุประสงค์**: หาเอกสารที่คล้ายกันทางความหมายโดยใช้ cosine similarity

**สูตรคำนวณคะแนน**:
```sql
score = ABS(
    0.1 * (title_embedding <#> query_embedding) +
    0.675 * (content_embedding <#> query_embedding) +
    0.1 * (file_name_embedding <#> query_embedding) +
    0.125 * weight * -1
)
```

- **Content**: น้ำหนัก 67.5% (ความเกี่ยวข้องหลัก)
- **Title**: น้ำหนัก 10% (การ match section)
- **Filename**: น้ำหนัก 10% (การ match เอกสาร)
- **Manual Weight**: 12.5% (essential=1.0, blog=0.8, patterns=0.5, ฯลฯ)

**SQL Operations หลัก**:
1. **KNN Search**: `KNN 1 @embedding $vec AS score`
2. **Score Threshold**: `>= 0.4` (กรองความเกี่ยวข้องต่ำออก)
3. **Distinct on File**: ผลลัพธ์หนึ่งต่อไฟล์, คะแนนสูงสุด
4. **Chunk Expansion**: เพิ่ม chunks ข้างเคียงเพื่อ context

---

### 3. BM25 Full-Text Search (`src/modules/ai/service.ts`)

**วัตถุประสงค์**: ค้นหาตาม keyword โดยใช้ ParadeDB's pg_search extension

**โครงสร้าง Query**:
```typescript
// BM25 พร้อม normalization
WHERE link @@@ `summary:${value.replace(/\(\)/g, '\\$1')}`

// Score normalization
r_norm = r / NULLIF(MAX(r) OVER (), 0)

// Hybrid scoring
score = 0.775 * r_norm + 0.275 * weight
```

**Filtering**: `score >= 0.6` (เข้มงวดกว่า vector search)

---

### 4. Semantic Cache (`src/modules/ai/libs/semantic-cache.ts`)

**วัตถุประสงค์**: คืน response ที่ cache ไว้สำหรับ queries ที่คล้ายกันทางความหมาย

**วิธีทำงาน**:
1. สร้าง embedding สำหรับ user query
2. ทำ KNN search ใน Redis สำหรับ cached queries ที่คล้ายกัน
3. ถ้า similarity >= 0.9, คืน cached response
4. ถ้าไม่, สร้าง response ใหม่และ cache มัน

**Normalization**:
- ใช้ GPT-OSS-20B เพื่อทำให้ queries เป็นมาตรฐาน
- "Can you help me with routing?" → "routing"
- ทำให้ cache hits ข้าม phrasings ที่ต่างกัน

**Cache Configuration**:
- **TTL**: 4 ชั่วโมง (14,400 วินาที)
- **Similarity Threshold**: 0.9
- **Max Query Length**: 192 ตัวอักษร (สำหรับ normalization)
- **Exclusions**: ตัวเลข, code blocks, queries สั้น/ยาวมาก

```typescript
const normalizePromptInstruction = `Reduce the user query to its canonical search intent.
- Remove all filler words, pleasantries
- Reduce to core topic + action
- Use consistent terminology
- Keep it under 12 words`
```

---

### 5. Document Indexing (`src/libs/structure.ts`)

**วัตถุประสงค์**: ประมวลผลและจัดดัชนีเอกสารสำหรับ retrieval

**Indexing Flow**:
```
1. Clone Documentation Repository (git clone)
2. Parse Markdown Files (extract title, content)
3. Chunk by Headers (## sections)
4. Generate 3 Embeddings per Chunk
5. Summarize Content (LLM - SKILLS.md style)
6. Insert/Update in Database (ON CONFLICT UPDATE)
7. Cleanup Temporary Files
```

**Chunking Strategy**:
- แบ่งตาม `##` (H2 headers)
- รักษาลำดับ
- กรอง: assignments, playground, blogs บางส่วน
- สร้าง unique link: `file#section`

**Weight System**:
```typescript
const titleWeight = {
    essential: 1.0,      // เอกสารหลัก
    blog: 0.8,           // Blog posts
    eden: 0.7,           // Type system
    patterns: 0.5,       // Patterns
    unknown: 0.4,        // Uncategorized
    migrate: 0.3,        // Migration guides
    integrations: 0.3,   // Integrations
    tutorial: 0.3        // Tutorials
}
```

**Summary Generation**:
```
Prompt: "Would you kindly summarize this into a SKILLS.md section..."
- Be concise
- Sacrifice grammar for concision
- Retain as much context as possible
```

**Diff Awareness**:
- เปรียบเทียบ chunks ใหม่กับ database ที่มีอยู่
- อัปเดตเฉพาะ chunks ที่เปลี่ยน
- ลบ chunks ที่ถูกลบ
- ลดการเรียก embedding API

---

### 6. Tool System (`src/modules/ai/libs/tool.ts`)

**วัตถุประสงค์**: มอบ tools แบบ deterministic ให้ LLM สำหรับ retrieval ข้อมูล

**Tools ที่มี**:

1. **search**: ค้นหาเอกสารด้วย keyword/sentence
   - Deterministic (ไม่มีการเรียกซ้ำ)
   - คืน content บางส่วน
   - ใช้ `readPage` สำหรับ content เต็ม

2. **readPage**: อ่านหน้าเฉพาะพร้อมรายละเอียดเต็ม
   - รับ link เช่น `patterns/openapi`
   - สามารถมี hash สำหรับ sections: `essential/life-cycle#transform`

3. **tableOfContents**: แสดงรายการเอกสารทั้งหมด
   - Table of contents แบบ static
   - ช่วย AI ค้นพบ topics ที่มี

4. **readHistory**: อ่านประวัติการสนทนา
   - มีเฉพาะถ้า history > 3 ข้อความ
   - คืน 5 ข้อความล่าสุดที่ถูกบีบอัด

---

### 7. AI Service (`src/modules/ai/service.ts`)

**Main Model**: GPT-OSS-120B via OpenRouter/Groq

**Configuration**:
```typescript
{
    topP: 0.75,
    presencePenalty: 0.4,
    maxOutputTokens: 2560,
    maxRetries: 3,
    stopWhen: [
        stepCountIs(think ? 12 : 8),
        () => references.length > 32
    ]
}
```

**Think Mode**:
- เปิดใช้สำหรับข้อความที่มี ``` หรือ > 1024 chars
- เพิ่ม reasoning effort เป็น "medium"
- อนุญาตถึง 12 steps แทน 8
- ใช้ model ที่มีความสามารถมากกว่า

---

### 8. Prompt Engineering (`src/libs/ai.ts`)

**Persona**: "Elysia chan" - ผู้ช่วยสาวน้อยจิ้งจอก

**Instructions หลัก**:
```
- Be concise. Sacrifice grammar for concision
- Truth is paramount and integrity is second to none
- Don't say something you can't cite or verify
- Label unverified information as speculation
- All tools are deterministic, don't call twice
- History limited to previous 3 messages
```

---

## การไหลของข้อมูล

### Request Flow (User Input → Response)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. USER SUBMITS REQUEST                                                      │
│    - message: "How do I create a route?"                                     │
│    - PoW solution: nonce + suffix                                           │
│    - Turnstile token: captcha verification                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. SECURITY VERIFICATION                                                      │
│    - Verify PoW (SHA256 starts with 4 zeros)                                 │
│    - Verify Turnstile token with Cloudflare                                 │
│    - Check rate limits (10 req / 35 sec)                                     │
│    - Verify IP matches challenge                                            │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. CACHE LOOKUP (ขนานกัน)                                                   │
│    a) Redis Cache (exact match)                                              │
│    b) Semantic Cache (similarity >= 0.9)                                     │
│    c) LRU Burst Caches                                                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                         ┌────────────┴────────────┐
                         │      Cache Hit?         │
                         └────────────┬────────────┘
                         Yes         │          No
                         │           │           │
                    Return Response  │           ▼
                                      │   ┌───────────────────────────┐
                                      │   │ 4. QUERY NORMALIZATION    │
                                      │   │    (if message length     │
                                      │   │     12-192 chars)         │
                                      │   └───────────────────────────┘
                                      │                │
                                      │                ▼
                                      │   ┌───────────────────────────┐
                                      │   │ 5. EMBEDDING GENERATION   │
                                      │   │    (stripFillers + LRU)    │
                                      │   └───────────────────────────┘
                                      │                │
                                      ▼                ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. HYBRID SEARCH (ขนานกัน + ผสาน)                                          │
│    a) BM25 Search (ParadeDB)                                                 │
│       - Keyword matching                                                     │
│       - Score: 0.775 * r_norm + 0.275 * weight                              │
│       - Filter: score >= 0.6                                                 │
│    b) Vector Search (Cosine Similarity)                                      │
│       - Semantic matching                                                    │
│       - Score: weighted combination of 3 embeddings                          │
│       - Filter: score >= 0.4                                                 │
│    c) Merge Results (deduplicate by link)                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 7. LLM GENERATION (Streaming)                                                │
│    - Model: GPT-OSS-120B                                                     │
│    - Tools: search, readPage, tableOfContents, readHistory                   │
│    - System Prompt: Elysia chan persona                                      │
│    - Max Steps: 8 (12 if think mode)                                        │
│    - Max References: 32                                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 8. RESPONSE PROCESSING                                                        │
│    a) Stream response chunks to client                                       │
│    b) Collect all references used                                            │
│    c) Append sources to response                                             │
│    d) Generate checksum for integrity                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 9. CACHE STORAGE (Async, non-blocking)                                       │
│    a) Redis Cache (exact match key)                                          │
│    b) Semantic Cache (normalized query + embedding)                          │
│    c) LRU Burst Caches                                                       │
│    d) Embedding Cache                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 10. LOGGING                                                                   │
│     - OpenTelemetry Spans to Axiom                                           │
│     - Token usage tracking                                                   │
│     - Reference tracking                                                     │
│     - Performance metrics                                                    │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Indexing Flow (Webhook → Database)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ 1. WEBHOOK TRIGGER                                                            │
│    - Source: GitHub (Elysia documentation repo)                              │
│    - Endpoint: /{REINDEX_ENDPOINT}                                           │
│    - Verification: headers['reindex_secret'] === REINDEX_SECRET              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 2. PREPARATION                                                                │
│    - Clear docs directory (if exists)                                        │
│    - Clone repository: git clone --depth 1 --single-branch                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 3. FILE PARSING                                                               │
│    a) Scan all .md files in docs/                                            │
│    b) Skip unwanted files:                                                   │
│       - index.md                                                              │
│       - /playground                                                          │
│       - /blog (except openapi-type-gen)                                      │
│       - /cheat-sheet.md                                                      │
│    c) Extract:                                                                │
│       - title (from frontmatter or filename)                                 │
│       - content (remove frontmatter, scripts, components)                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 4. CHUNKING                                                                   │
│    For each file:                                                             │
│    a) Split by ## (H2 headers)                                               │
│    b) Filter out sections starting with < (Vue components)                   │
│    c) Create unique link: file#section                                       │
│    d) Assign sequence number (per file)                                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 5. DIFF CALCULATION                                                           │
│    a) Fetch existing chunks from database                                    │
│    b) Compare content: new vs existing                                       │
│    c) Identify:                                                              │
│       - New chunks (not in DB)                                               │
│       - Changed chunks (content differs)                                     │
│       - Removed chunks (in DB but not new)                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 6. EMBEDDING GENERATION (Batched)                                             │
│    Queue: concurrency=4, 20 ops/second                                       │
│    For each changed chunk:                                                    │
│    a) Generate 3 embeddings via OpenAI API:                                  │
│       - content_embedding (1536 dims)                                        │
│       - title_embedding (1536 dims)                                          │
│       - file_name_embedding (1536 dims)                                      │
│    b) Generate summary via LLM (SKILLS.md style)                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 7. DATABASE UPDATE                                                            │
│    INSERT INTO documents (...) VALUES (...)                                 │
│    ON CONFLICT (link) DO UPDATE SET ...                                      │
│                                                                              │
│    DELETE FROM documents WHERE link IN (removed_chunks)                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ 8. CLEANUP                                                                    │
│    - Remove docs directory                                                   │
│    - Set inProcess = false                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## รายละเอียดการนำไปใช้

### Database Schema

```sql
CREATE TABLE documents (
    link VARCHAR(255) PRIMARY KEY,           -- Unique identifier
    file VARCHAR(255) NOT NULL,              -- Source file
    file_name VARCHAR(255) GENERATED ALWAYS AS (
        regexp_replace(file, '.*/|\\.[^.]+$', '', 'g')
    ) STORED,                                 -- Extracted filename
    title VARCHAR(255) NOT NULL,             -- Section title
    content TEXT NOT NULL,                   -- Full content
    summary TEXT NOT NULL,                   -- LLM-generated summary
    weight FLOAT NOT NULL DEFAULT 0.5,       -- Manual weight
    embedding VECTOR(1536) NOT NULL,         -- Content embedding
    title_embedding VECTOR(1536) NOT NULL,   -- Title embedding
    file_name_embedding VECTOR(1536) NOT NULL, -- Filename embedding
    sequence smallint NOT NULL DEFAULT 0     -- Order within file
);

-- BM25 Index
CREATE INDEX idx_documents_content_bm25 ON documents
USING bm25 (link, title, summary, file_name)
WITH (key_field='link');

-- Link Index
CREATE INDEX idx_documents_link ON documents (link);
```

### Redis Cache Index

```redis
FT.CREATE idx:cache
ON HASH
PREFIX 1 "cache:"
SCORABLE 1
FILTER "@embedding != ''"
SCHEMA
  prompt TEXT
  response TEXT
  embedding VECTOR
    KNN 20
    TYPE FLOAT32
    DIM 1536
    DISTANCE_METRIC COSINE
    INITIAL_CAP 250
    BLOCK_SIZE 1024
```

### API Endpoints

#### 1. POST /model/pow/request
ข้อท้า PoW challenge

**Response**:
```typescript
{
  nonce: string,      // Random 32 bytes
  bits: 19,           // Difficulty (4 leading zeros)
  expires: number     // Timestamp
}
```

#### 2. POST /model/ai/ask
AI endpoint หลักสำหรับถามคำถาม

**Request Body**:
```typescript
{
  message: string,      // User question (max 4096 chars)
  seed?: number,        // Random seed (skip cache if set)
  history?: Array<{     // Conversation history (max 8)
    role: 'user' | 'assistant',
    content: string,
    checksum: string    // SHA256 for integrity
  }>,
  reference?: string,   // Specific page to reference
  think?: boolean       // Enable reasoning mode
}
```

**Headers**:
- `x-turnstile-token`: Cloudflare Turnstile verification
- `x-api-key`: Optional API key bypass

**Body**:
```typescript
{
  pow: {
    suffix: string     // PoW solution
  }
}
```

**Response**: Stream ของ text chunks พร้อม sources

#### 3. PATCH /{REINDEX_ENDPOINT}
Trigger document reindexing

**Headers**:
- `reindex_secret`: Webhook secret

**Response**: "Indexing..." → "Done"

---

## เทคนิคการปรับแต่ง

### 1. Multi-Layer Caching Strategy

```
Layer 1: Redis Cache (Exact Match)
  ├─ Key: q:{hash(message[:page@think))}
  ├─ TTL: 3 ชั่วโมง (สั้นกว่าสำหรับข้อความยาว)
  └─ Hit Rate: ~30-40%

Layer 2: Semantic Cache (Similarity)
  ├─ Threshold: 0.9 cosine similarity
  ├─ TTL: 4 ชั่วโมง
  └─ Hit Rate: ~20-30%

Layer 3: LRU Embedding Cache
  ├─ Size: 750 embeddings (~4.4MB)
  ├─ TTL: 9 ชั่วโมง
  └─ Prevents duplicate API calls

Layer 4: Burst Caches (In-Memory)
  ├─ VolatileHistoryCache: 30s TTL, 250 items
  ├─ NoHistoryWithSeedCache: 4h TTL, 750 items
  └─ FallbackCache: 5s TTL, 1500 items

Layer 5: Database Query Cache
  ├─ TTL: 3 ชั่วโมง
  └─ Cached search results
```

### 2. Cost Reduction Techniques

1. **No Re-ranking Model**: ประหยัด ~$0.002/req
2. **OpenRouter Flex Mode**: ยอมรับ errors, retry แบบ exponential
3. **GPT-OSS Models**: ~500 tps, ถูกกว่า GPT-4 มาก
4. **Small Embedding Model**: text-embedding-3-small (ถูกที่สุด)
5. **Batch Embedding**: ประมวลผลหลาย chunks ใน API call เดียว
6. **Diff-Aware Indexing**: อัปเดตเฉพาะ content ที่เปลี่ยน
7. **Query Normalization**: ลด semantic searches ที่ซ้ำซ้อน

### 3. Latency Optimization

1. **Single Server Deployment**: บริการทั้งหมดบนเครื่องเดียว
2. **Dragonfly vs Redis**: เร็วกว่า ~20% สำหรับ cache operations
3. **Streaming Response**: เริ่มส่งทันที
4. **Parallel Tool Execution**: AI สามารถเรียก tools หลายตัวพร้อมกัน
5. **Connection Pooling**: ใช้ database connections ซ้ำ
6. **LRU Over Redis**: สำหรับ embeddings ที่เข้าถึงบ่อย

### 4. Quality Optimization

1. **Triple Embedding**: Content + Title + Filename
2. **Weighted Scoring**: Manual weights สำหรับประเภทเอกสาร
3. **Chunk Expansion**: เพิ่ม adjacent chunks สำหรับ context
4. **Parent Document Retrieval**: คืนหน้าเต็มเมื่ออ่าน
5. **Summary Generation**: SKILLS.md style สำหรับ AI self-reference

---

## ความปลอดภัยและการจำกัดอัตรา

### Proof of Work (PoW)

**วัตถุประสงค์**: ป้องกันการใช้งานโดยอัตโนมัติโดยการบังคับให้ทำงานคำนวณ

**Configuration**:
- **Difficulty**: 19 bits (4 leading zeros ใน SHA-256)
- **Algorithm**: SHA-256(nonce:suffix)
- **Expiry**: 15 นาที
- **Per-IP**: challenge แต่ละอันผูกกับ IP ของผู้ขอ

**Client Process**:
1. Request challenge จาก `/pow/request`
2. รับ nonce และ difficulty
3. Brute-force compute suffix จนกว่า hash จะตรงเงื่อนไข
4. Submit solution พร้อม request ดั้งเดิม

```typescript
// Verification
const hash = crypto.createHash('sha256')
    .update(`${nonce}:${suffix}`)
    .digest('hex')

const requiredPrefix = '0'.repeat(bits / 4) // '0000'

if (!hash.startsWith(requiredPrefix))
    return status(403, 'Invalid proof of work')
```

### Cloudflare Turnstile

**วัตถุประสงค์**: Bot detection และ CAPTCHA verification

**Configuration**:
- **Endpoint**: `https://challenges.cloudflare.com/turnstile/v0/siteverify`
- **Rate Limit**: 8 requests per 35 seconds per IP
- **Bypass**: API key หรือ development mode

### Rate Limiting

**Implementation**: Sliding window ใช้ Redis sorted sets

**Limits**:
- **Global**: 10 requests / 35 seconds
- **Turnstile Check**: 8 requests / 35 seconds
- **PoW Request**: 10 requests / 35 seconds

### Checksum Verification

**วัตถุประสงค์**: Verify conversation history integrity

```typescript
export abstract class Checksum {
    static generate(content: string) {
        const hash = new Bun.CryptoHasher('sha256', AI_CHECKSUM_SECRET)
            .update(content)
            .digest('hex')
        return hash
    }

    static verify(content: string, checksum: string) {
        return timingSafeEqual(
            Buffer.from(this.generate(content)),
            Buffer.from(checksum)
        )
    }
}
```

---

## โครงสร้างพื้นฐาน

### Stack Overview

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Backend** | Elysia (Bun) | API framework |
| **Database** | ParadeDB | BM25 + Vector search |
| **Cache** | DragonflyDB | Redis-compatible cache |
| **LLM** | GPT-OSS-120B | Main reasoning model |
| **Small LLM** | GPT-OSS-20B | Query normalization |
| **Embedding** | OpenAI text-embedding-3-small | Vector generation |
| **Monitoring** | Axiom + OpenTelemetry | Logging and traces |
| **Security** | Cloudflare Turnstile | Bot detection |
| **Server** | Hetzner (Europe) | Low-cost hosting |
| **Deployment** | Coolify | Self-hosted Vercel alternative |

### Docker Configuration

```yaml
# docker-compose.yml
services:
  pg:
    container_name: arona_paradedb
    image: paradedb/paradedb
    environment:
      POSTGRES_DB: arona
      POSTGRES_USER: plana
      POSTGRES_PASSWORD: 12345678

  dragonfly:
    container_name: arona_dragonfly
    image: docker.dragonflydb.io/dragonflydb/dragonfly
    ulimits:
      memlock: -1
    environment:
      requirepass: 12345678
```

### Environment Variables

```bash
# Database
DATABASE_URL=postgres://user:pass@host:5432/db

# Cache
REDIS_URL=redis://host:6379

# AI
OPENAI_API_KEY=sk-...
OPENROUTER_API_KEY=skr-...
GROQ_API_KEY=gsk_...  # Alternative

# Security
TURNSTILE_SECRET=0x...
CHALLENGE_SECRET=...
AI_CHECKSUM_SECRET=...
REINDEX_SECRET=...
REINDEX_ENDPOINT=/webhook/reindex

# Monitoring
AXIOM_DATASET=arona
AXIOM_TOKEN=...
```

---

## การวิเคราะห์ต้นทุน

### แตกต้นทุนรายเดือน

| หมวด | รายการ | ค่าใช้จ่าย |
|----------|------|------|
| **Infrastructure** | Hetzner Server | ~$12 |
| **AI** | 750 requests/day @ ~$0.02/req | ~$15/เดือน |
| **Total** | | **~$27/เดือน** |

**หมายเหตุ**: โพสต์ดั้งเดิมกล่าวถึง ~$9-15/วัน สำหรับค่า AI ซึ่งจะเป็น ~$270-450/เดือน

### ต้นทุนต่อ Request

```
สมมติ 750 requests/day:

Main Model (GPT-OSS-120B):
  - Input: ~2000 tokens × $0.10/1M = $0.0002
  - Output: ~500 tokens × $0.40/1M = $0.0002
  - Per request: ~$0.0004

Embedding (text-embedding-3-small):
  - 3 embeddings/chunk × 10 chunks × $0.02/1M = $0.0006
  - Amortized over cache hits: ~$0.0001/req

Small Model (GPT-OSS-20B):
  - Normalization: ~50 tokens × $0.10/1M = $0.000005
  - 30% cache hit rate: ~$0.000003/req

Total per request: ~$0.0005 - $0.001

750 requests/day × 30 days × $0.0005 = ~$11/month (base case)
With cache misses and tool calls: ~$15-30/month actual
```

### ผลกระทบจากการปรับแต่งต้นทุน

| Optimization | Savings |
|--------------|---------|
| Semantic Cache (30% hit rate) | ลดค่า AI 30% |
| Redis Cache (20% hit rate) | ลดค่า AI 20% |
| No Re-ranking | ประหยัด ~$0.002/req |
| Small Embedding Model | ถูกกว่า model ใหญ่ 80% |
| Diff-Aware Indexing | ลดการเรียก embed 90% |

---

## ข้อควรจำสำหรับการสร้าง RAG

1. **Hybrid Search Wins**: BM25 + Vector > ใช้อย่างเดียว
2. **Cache Everything**: Semantic, exact, embeddings, queries
3. **Strip Fillers**: Conversational queries ต้องการ normalization
4. **Multiple Embeddings**: Content + Title + Context ปรับปรุง relevance
5. **Streaming Matters**: เริ่ม response ทันที, ไม่ต้องรอ
6. **Tools > Prompts**: ให้ AI call deterministic tools
7. **Stateless Scales**: ไม่มี sessions = scale แบบ horizontal ง่าย
8. **Security Layers**: PoW + Turnstile + Rate Limiting
9. **Monitor Everything**: OpenTelemetry สำหรับ debugging
10. **Accept Flexibility**: Retry > blocking on errors

---

## อ้างอิงไฟล์

| Feature | File |
|---------|------|
| **RAG Core** | `src/modules/ai/service.ts` |
| **Vector Search SQL** | `src/modules/ai/const.ts` |
| **Semantic Cache** | `src/modules/ai/libs/semantic-cache.ts` |
| **Tools** | `src/modules/ai/libs/tool.ts` |
| **Embeddings** | `src/libs/embedding.ts` |
| **Indexing** | `src/libs/structure.ts` |
| **AI Config** | `src/libs/ai.ts` |
| **PoW** | `src/modules/pow/index.ts` |
| **Turnstile** | `src/libs/turnstile.ts` |
| **Rate Limiting** | `src/libs/rate-limit.ts` |
| **Cache** | `src/libs/cache.ts` |
| **Models** | `src/modules/ai/model.ts` |
| **Server** | `src/server.ts` |

---

*Generated from Arona codebase analysis*
*Project: https://github.com/elysiajs/arona*
