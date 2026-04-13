# RAG Deep Dive — คำศัพท์ทั้งหมด + การเตรียม Data เข้า Vector Database

---

## Part 1: คำศัพท์ทั้งหมดที่ต้องรู้

### 1.1 Core Concepts

#### RAG (Retrieval-Augmented Generation)

```
RAG = "ค้นหาก่อน แล้วค่อยตอบ"

ปกติ LLM ตอบจากความรู้ที่ train มาเท่านั้น:
  User: "นโยบายลาของ Sellsuki?"
  LLM:  "ผมไม่มีข้อมูลเรื่องนี้" ❌

RAG เพิ่มขั้นตอน "ค้นหา" ก่อน:
  User: "นโยบายลาของ Sellsuki?"
  ↓
  [ค้นหา] → พบ: "พนักงานลาพักร้อนได้ 10 วัน/ปี..."
  ↓
  [ส่งให้ LLM] → "ตามนโยบาย Sellsuki พนักงานลาพักร้อนได้ 10 วัน/ปี" ✅

ทำไมไม่ fine-tune LLM แทน?
  Fine-tune = สอน LLM ใหม่ (แพง, ช้า, update ยาก)
  RAG = ค้นจาก database (ถูก, เร็ว, update ได้ทันที)
```

#### LLM (Large Language Model)

```
LLM = โมเดล AI ที่เข้าใจและสร้างภาษาได้

ลองนึกภาพ:
  LLM เหมือน "คนฉลาดมาก" ที่อ่านหนังสือมาเป็นล้านเล่ม
  ถามอะไรก็ตอบได้ แต่ไม่รู้ข้อมูลเฉพาะบริษัทคุณ

ตัวอย่าง LLM:
  GPT-4o, Claude, Gemini, Llama, Mistral

ศัพท์ที่เกี่ยวข้อง:
  Token    = หน่วยเล็กสุดที่ LLM อ่าน (ประมาณ 1 คำ ≈ 1-2 tokens)
  Prompt   = ข้อความที่ส่งให้ LLM
  Context  = ข้อมูลทั้งหมดที่ LLM เห็นใน 1 ครั้ง
  Context Window = ขนาด context สูงสุดที่รับได้ (เช่น 128K tokens)
```

#### Embedding

```
Embedding = การแปลงข้อความเป็นตัวเลข (vector)
เพื่อให้คอมพิวเตอร์ "เข้าใจ" ความหมายได้

ตัวอย่าง:
  "ลาพักร้อน"  → [0.82, -0.15, 0.43, ..., 0.07]  (768 ตัวเลข)
  "หยุดพักผ่อน" → [0.80, -0.14, 0.41, ..., 0.09]  (768 ตัวเลข)
  "ซื้อข้าว"    → [0.12, 0.67, -0.33, ..., 0.55]  (768 ตัวเลข)

สังเกต:
  "ลาพักร้อน" กับ "หยุดพักผ่อน" → ตัวเลขใกล้กัน (ความหมายคล้ายกัน)
  "ลาพักร้อน" กับ "ซื้อข้าว"     → ตัวเลขต่างกันมาก (ความหมายต่างกัน)

Embedding Model = AI ที่ทำหน้าที่แปลงข้อความเป็น vector
  เช่น: text-embedding-3-small, BGE-M3, Gemini text-embedding-004
```

#### Vector

```
Vector = ชุดตัวเลขที่เรียงกัน ใช้แทนตำแหน่งในมิติสูง

ง่ายสุด:
  2 มิติ: [3, 5]         → จุดบนกราฟ x=3, y=5
  3 มิติ: [3, 5, 2]      → จุดในพื้นที่ 3D

Embedding ก็คือ vector ในมิติสูงมาก:
  768 มิติ:  [0.82, -0.15, 0.43, ..., 0.07]  (768 ตัวเลข)
  1536 มิติ: [0.02, 0.91, -0.33, ..., 0.18]  (1536 ตัวเลข)

ทำไมต้องมิติเยอะ?
  ยิ่งมิติมาก → แทนความหมายที่ซับซ้อนได้มากขึ้น
  เหมือนอธิบายคน: ถ้ามีแค่ "สูง" กับ "หนัก" → แยกคนได้น้อย
  แต่ถ้ามี 768 ลักษณะ → แยกคนได้ละเอียดมาก
```

#### Vector Database

```
Vector Database = ฐานข้อมูลที่เก็บ vectors และค้นหา "vector ที่ใกล้กัน" ได้เร็ว

Database ปกติ:
  SELECT * FROM users WHERE name = 'สมชาย'  → ค้นหาตรงตัว

Vector Database:
  SELECT * FROM docs ORDER BY embedding <=> [0.82, -0.15, ...] LIMIT 5
  → ค้นหา "5 อันที่ความหมายใกล้เคียงที่สุด"

ตัวอย่าง:
  pgvector    = PostgreSQL extension (ใช้ SQL ปกติ + vector search)
  ParadeDB    = PostgreSQL ที่มี pgvector + BM25 ในตัว
  Pinecone    = Cloud vector DB (managed)
  Qdrant      = Rust-based vector DB (เร็วมาก)
  ChromaDB    = Lightweight vector DB (เหมาะ prototype)
  Milvus      = Enterprise vector DB (scale ใหญ่)
  Weaviate    = Vector DB + graph
  LanceDB     = Embedded vector DB (ไม่ต้อง server)
```

### 1.2 Search & Retrieval

#### Cosine Similarity

```
Cosine Similarity = วัดว่า 2 vectors "ชี้ไปทิศทางเดียวกัน" แค่ไหน

ค่า 1.0  = เหมือนกันเป๊ะ (ทิศเดียวกัน)
ค่า 0.0  = ไม่เกี่ยวกันเลย (ตั้งฉาก)
ค่า -1.0 = ตรงข้ามกัน

ตัวอย่าง:
  "ลาพักร้อน" vs "หยุดพักผ่อน"  = 0.95  (คล้ายมาก)
  "ลาพักร้อน" vs "สวัสดิการ"     = 0.72  (เกี่ยวข้อง)
  "ลาพักร้อน" vs "ซื้อข้าว"      = 0.15  (ไม่เกี่ยว)

สูตร (ไม่ต้องจำ):
  cos(θ) = (A · B) / (|A| × |B|)
  
  คือวัด "มุม" ระหว่าง 2 vectors
  มุมน้อย = ทิศเดียวกัน = ความหมายคล้ายกัน
```

#### Cosine Distance

```
Cosine Distance = 1 - Cosine Similarity

ใน pgvector ใช้ operator <=> สำหรับ cosine distance:
  ค่า 0.0 = เหมือนกัน
  ค่า 2.0 = ตรงข้ามกัน

  SELECT * FROM docs ORDER BY embedding <=> query_vector LIMIT 5
  → เรียงจาก distance น้อยสุด (คล้ายสุด) ก่อน

Similarity vs Distance:
  similarity = 1 - distance
  distance   = 1 - similarity
  
  ถ้า similarity = 0.95 → distance = 0.05
  ถ้า similarity = 0.30 → distance = 0.70
```

#### BM25 (Best Match 25)

```
BM25 = อัลกอริทึมจัดอันดับข้อความ (text ranking algorithm)
ใช้ในการค้นหาแบบ full-text search มานานมาก (Google ยุคแรกๆ ก็ใช้หลักการนี้)

หลักการ:
  1. Term Frequency (TF) — คำนั้นปรากฏในเอกสารกี่ครั้ง
     "ลาพักร้อน" ปรากฏ 5 ครั้ง → score สูง
  
  2. Inverse Document Frequency (IDF) — คำนั้นหายากแค่ไหน
     "ลาพักร้อน" ปรากฏใน 2 เอกสารจาก 1000 → คำนี้สำคัญ (IDF สูง)
     "การ" ปรากฏใน 900 เอกสารจาก 1000 → คำนี้ไม่สำคัญ (IDF ต่ำ)
  
  3. Document Length — เอกสารยาวแค่ไหน
     เอกสารสั้นที่มีคำตรง → น่าจะเกี่ยวข้องมากกว่าเอกสารยาว

เทียบกับ Vector Search:
  BM25:           "ลาพักร้อน" → ค้นหาคำว่า "ลาพักร้อน" ตรงตัว
                  ✅ แม่นกับชื่อเฉพาะ, คำ technical
                  ❌ ไม่เข้าใจว่า "หยุดพัก" = "ลาพักร้อน"

  Vector Search:  "ลาพักร้อน" → ค้นหา "ความหมายคล้ายกัน"
                  ✅ เข้าใจ synonyms, paraphrase
                  ❌ อาจพลาดคำเฉพาะทาง

  Hybrid (BM25 + Vector): ได้ข้อดีทั้งสอง → แม่นที่สุด
```

#### FTS (Full-Text Search)

```
FTS = การค้นหาข้อความแบบเต็ม

แตกต่างจาก LIKE:
  LIKE '%ลาพักร้อน%' → ค้นหา substring ตรงตัว, ช้า, ไม่ smart
  FTS               → tokenize, จัดอันดับ, เข้าใจคำที่เกี่ยวข้อง

PostgreSQL มี FTS ในตัว:
  ts_vector + ts_query → ใช้ได้แต่ไม่ใช่ BM25 จริงๆ
  
ParadeDB เพิ่ม pg_search extension:
  → BM25 จริงๆ, แม่นกว่า ts_vector มาก
```

#### Hybrid Search

```
Hybrid Search = รวม Vector Search + BM25 เข้าด้วยกัน

วิธีรวม (Reciprocal Rank Fusion - RRF):
  1. ค้นด้วย BM25 → ได้ ranking: [doc_A, doc_C, doc_B, ...]
  2. ค้นด้วย Vector → ได้ ranking: [doc_B, doc_A, doc_D, ...]
  3. รวม score:
     doc_A: 1/(60+1) + 1/(60+2) = 0.0163 + 0.0161 = 0.0324  ← สูงสุด
     doc_B: 1/(60+3) + 1/(60+1) = 0.0159 + 0.0163 = 0.0322
     doc_C: 1/(60+2) + 0        = 0.0161
     doc_D: 0        + 1/(60+3) = 0.0159
  4. เรียงใหม่: [doc_A, doc_B, doc_C, doc_D]

  k=60 เป็นค่า constant มาตรฐาน (ทำให้ ranking อันดับท้ายๆ ไม่มีผลมาก)

ผลวิจัย 2024-2025:
  Hybrid search ดีกว่า vector อย่างเดียว 15-30% ในเกือบทุก benchmark
```

#### Re-ranking

```
Re-ranking = ใช้ AI model ตัวที่ 2 มาจัดอันดับผลลัพธ์ search ใหม่

Flow:
  1. Search ได้ 20 ผลลัพธ์ (rough, เร็ว)
  2. ส่ง 20 ผลลัพธ์ให้ Re-ranker (แม่น, ช้ากว่า)
  3. Re-ranker จัดอันดับใหม่ → เอา top 5

Re-ranking Models:
  Cohere Rerank       → $1/1000 queries (cloud)
  bge-reranker-v2-m3  → ฟรี (self-host)
  Jina Reranker       → มี free tier

ใช้เมื่อ:
  ✅ เอกสารเยอะมาก (> 1000 pages)
  ✅ ต้องการความแม่นยำสูงสุด
  
ไม่ใช้เมื่อ:
  ❌ เอกสารน้อย (< 100 pages) → hybrid search พอ
  ❌ ต้องการ latency ต่ำ
  ❌ Budget จำกัด
```

### 1.3 Chunking & Indexing

#### Chunk / Chunking

```
Chunk = ท่อนข้อความเล็กๆ ที่ถูกตัดมาจากเอกสารใหญ่
Chunking = กระบวนการตัดเอกสารเป็น chunks

ทำไมต้อง chunk:
  1. Embedding model มี limit (เช่น 8192 tokens)
  2. ข้อความสั้น → embedding แม่นกว่า
  3. ส่งแค่ chunk ที่เกี่ยวข้องให้ LLM → ประหยัด tokens

ตัวอย่าง:
  เอกสาร 10 หน้า → chunk เป็น 25 ท่อน (ท่อนละ ~500-1000 ตัวอักษร)
  แต่ละท่อนมี embedding ของตัวเอง
  ตอน search → หา "ท่อนที่เกี่ยวข้อง" ไม่ใช่ "เอกสารทั้งเล่ม"
```

#### Chunk Overlap

```
Overlap = ส่วนที่ซ้อนกันระหว่าง chunk ต่อเนื่อง

ทำไมต้อง overlap:
  ถ้าไม่ overlap → ข้อมูลอาจหายตรง "รอยตัด"

ตัวอย่าง (ไม่มี overlap):
  Chunk 1: "พนักงานลาพักร้อนได้ 10 วัน โดยต้อง"
  Chunk 2: "แจ้งล่วงหน้า 3 วัน และต้องได้รับอนุมัติ"
  → ถ้า search "ต้องแจ้งล่วงหน้ากี่วัน" อาจได้ Chunk 2
    แต่ไม่มีบริบทว่าพูดถึง "ลาพักร้อน"

ตัวอย่าง (overlap 100 ตัวอักษร):
  Chunk 1: "พนักงานลาพักร้อนได้ 10 วัน โดยต้อง"
  Chunk 2: "ได้ 10 วัน โดยต้องแจ้งล่วงหน้า 3 วัน และต้องได้รับอนุมัติ"
  → Chunk 2 มีบริบทครบ ✅

ค่าแนะนำ:
  chunk_size = 800 ตัวอักษร
  overlap = 150-200 ตัวอักษร (15-25% ของ chunk_size)
```

#### Content Hash

```
Content Hash = ค่า fingerprint ของข้อความ ใช้เทียบว่าเนื้อหาเปลี่ยนหรือไม่

ใช้สำหรับ Diff-based Indexing:
  SHA256("พนักงานลาได้ 10 วัน") = "a3f2b8c1..."
  
  ตอน re-index:
    hash เก่า "a3f2b8c1..." == hash ใหม่ "a3f2b8c1..." → ข้าม (ไม่ต้อง embed ใหม่)
    hash เก่า "a3f2b8c1..." != hash ใหม่ "d7e9f0a2..." → เปลี่ยนแล้ว → embed ใหม่
```

### 1.4 Database & Index

#### HNSW (Hierarchical Navigable Small World)

```
HNSW = อัลกอริทึมสร้าง index สำหรับ vector search

ลองนึกภาพ:
  สมมติมี 100,000 vectors (จุดในอวกาศ)
  ถ้าค้นทีละจุด → ช้ามาก (O(n))
  
  HNSW สร้าง "กราฟหลายชั้น":
  
  ชั้น 3 (บนสุด, น้อยจุดมาก):  A ────── B
                                    \   |
  ชั้น 2 (กลาง):                 A ─ C ─ B ─ D
                                  |   |   |   |
  ชั้น 1 (ล่าง, เยอะจุดมาก):    A-C-E-B-D-F-G-H

  ค้นหา:
  1. เริ่มจากชั้นบนสุด → หาจุดใกล้คร่าวๆ
  2. ลงมาชั้นกลาง → หาละเอียดขึ้น
  3. ลงมาชั้นล่าง → หาแม่นยำ
  = เร็วมาก O(log n)

Parameters:
  m = จำนวน connections ต่อ node (default 16)
      ยิ่งมาก = แม่นกว่า แต่ใช้ memory มากกว่า
  ef_construction = ความกว้างตอนสร้าง index (default 64)
      ยิ่งมาก = index ดีกว่า แต่สร้างนานกว่า
```

#### IVFFlat (Inverted File Flat)

```
IVFFlat = อัลกอริทึม index อีกแบบ แบ่ง vectors เป็นกลุ่ม

ลองนึกภาพ:
  แบ่งพื้นที่เป็น 100 กลุ่ม (clusters/lists)
  แต่ละ vector ถูกจัดเข้ากลุ่มที่ใกล้ที่สุด

  ค้นหา:
  1. หาว่า query vector ใกล้กลุ่มไหนที่สุด
  2. ค้นเฉพาะใน 5-10 กลุ่มที่ใกล้ (probes)
  = ไม่ต้องค้นทั้ง 100,000 จุด

Parameters:
  lists = จำนวนกลุ่ม (แนะนำ sqrt(จำนวน rows))
  probes = กี่กลุ่มที่จะค้น (ยิ่งมาก = แม่นกว่า แต่ช้ากว่า)

HNSW vs IVFFlat:
  HNSW:    แม่นกว่า, ไม่ต้อง train, เพิ่ม data ได้เรื่อยๆ ← แนะนำ
  IVFFlat: เร็วกว่าเล็กน้อย, ใช้ memory น้อยกว่า, ต้อง rebuild เมื่อ data เปลี่ยนเยอะ
```

#### Reciprocal Rank Fusion (RRF)

```
RRF = วิธีรวมผลลัพธ์จากหลาย search เข้าด้วยกัน

สูตร: score(doc) = Σ 1 / (k + rank_i)

ตัวอย่าง (k=60):
  BM25 results:    [doc_A(อันดับ1), doc_C(อันดับ2), doc_B(อันดับ3)]
  Vector results:  [doc_B(อันดับ1), doc_A(อันดับ2), doc_D(อันดับ3)]

  doc_A: 1/(60+1) + 1/(60+2) = 0.01639 + 0.01613 = 0.03252  ← ชนะ
  doc_B: 1/(60+3) + 1/(60+1) = 0.01587 + 0.01639 = 0.03226
  doc_C: 1/(60+2) + 0        = 0.01613
  doc_D: 0        + 1/(60+3) = 0.01587

  ผลลัพธ์สุดท้าย: [doc_A, doc_B, doc_C, doc_D]

  doc_A ชนะเพราะติดอันดับต้นๆ ใน "ทั้งสอง" search
```

### 1.5 Caching

#### Semantic Cache

```
Semantic Cache = cache ที่ดูความหมาย ไม่ใช่แค่ข้อความตรงตัว

Cache ปกติ:
  "ลาพักร้อนกี่วัน" → cache hit ✅
  "ลาพักร้อนได้กี่วัน" → cache miss ❌ (ข้อความต่างกัน!)

Semantic Cache:
  "ลาพักร้อนกี่วัน" → cache hit ✅
  "ลาพักร้อนได้กี่วัน" → cosine similarity 0.97 > threshold 0.95 → cache hit ✅
  "หยุดพักผ่อนได้กี่วัน" → cosine similarity 0.91 < threshold 0.95 → cache miss ❌

วิธีทำ:
  1. User ถาม → embed คำถาม
  2. เทียบกับ embeddings ของคำถามเก่าใน cache
  3. ถ้า similarity > threshold → ใช้คำตอบเก่า (ไม่ต้องเรียก LLM)
  4. ถ้า < threshold → เรียก LLM → เก็บคำถาม+คำตอบใหม่ลง cache

ประหยัด: 40-60% ของ LLM calls สำหรับคำถามซ้ำๆ
```

#### TTFT (Time to First Token)

```
TTFT = เวลาตั้งแต่ส่ง request จนได้ token แรกกลับมา

ทำไมสำคัญ:
  User ส่งคำถาม → รอ......... → เห็นคำตอบทีเดียว (TTFT สูง = UX แย่)
  User ส่งคำถาม → เห็นคำตอบเริ่มพิมพ์ทันที (TTFT ต่ำ = UX ดี)

ตัวอย่าง:
  GPT-4o:     TTFT ~300-500ms
  Groq:       TTFT ~50-150ms   ← เร็วกว่า 3-5x
  Cerebras:   TTFT ~30-100ms   ← เร็วที่สุด

TTFT ต่ำ + Streaming = user เห็นคำตอบเริ่ม type เกือบทันที
```

#### Flex Mode

```
Flex Mode = โหมดที่ API request มีโอกาสล้มเหลวได้ แลกกับราคาถูกลง

Groq Flex Mode:
  ปกติ:    $0.05/1M input tokens, รับประกัน response
  Flex:    ถูกกว่า ~50%, แต่มีโอกาส error ได้ (busy hours)

วิธีใช้:
  1. ส่ง request แบบ flex
  2. ถ้า success → ได้คำตอบ (ส่วนใหญ่จะ success)
  3. ถ้า fail → retry ด้วย exponential backoff
     retry 1: รอ 1 วินาที
     retry 2: รอ 2 วินาที
     retry 3: รอ 4 วินาที
     ...

เหมาะกับ:
  ✅ Non-realtime (user รอได้ 2-3 วินาที)
  ✅ มี retry mechanism
  ❌ ไม่เหมาะกับ voice/real-time ที่ต้องตอบทันที
```

### 1.6 Anti-abuse

#### PoW (Proof of Work)

```
PoW = ให้ client "ทำงาน" (คำนวณ) ก่อนจึงจะส่ง request ได้

หลักการ:
  Server: "หาค่า x ที่ SHA256('challenge_abc' + x) เริ่มต้นด้วย '0000'"
  Client: ต้องลอง x ไปเรื่อยๆ จนเจอ (ใช้เวลา ~100-500ms)
  Server: ตรวจว่าถูกจริง → อนุญาตให้ถาม

ทำไมป้อง abuse:
  คนจริง:  คำนวณ 1 ครั้ง 300ms → ไม่รู้สึก
  Bot:     ต้องคำนวณทุก request → ส่ง spam ได้ช้ามาก
           100 requests × 300ms = 30 วินาที (แทนที่จะเป็น 0.1 วินาที)
```

#### Turnstile (Cloudflare)

```
Turnstile = CAPTCHA ของ Cloudflare แต่ไม่ต้องให้ user ทำอะไร

ปกติ: "กดรูปภาพที่มีรถยนต์" (น่ารำคาญ)
Turnstile: วิเคราะห์ behavior อัตโนมัติ (mouse movement, timing)
           → ตัดสินว่าเป็นคนจริงหรือ bot
           → ใช้เวลาคำนวณ ~1-3 วินาที (ใช้ชะลอ request ได้ด้วย)
```

### 1.7 Infrastructure

#### Co-location

```
Co-location = วาง services ทั้งหมดไว้ที่เดียวกัน/ใกล้กัน

ปัญหา (services กระจาย):
  App (US) → Database (EU) → Cache (Asia) → LLM (US)
  latency: 100ms + 200ms + 150ms + 100ms = 550ms 😱

Co-location:
  App + Database + Cache อยู่ server เดียวกัน
  latency: 0.1ms + 0.1ms + 0.1ms = 0.3ms 🚀
  + LLM API (same region): 50ms
  = รวม ~50ms

คอขวดของ RAG คือ Database latency
→ App กับ DB ต้องอยู่ใกล้กันที่สุด
```

#### Coolify

```
Coolify = self-hosted PaaS (Platform as a Service)
เหมือน Heroku/Railway/Vercel แต่ลงใน server ของตัวเอง

ทำให้:
  - Deploy ด้วย git push
  - จัดการ Docker containers
  - Auto SSL certificates
  - ดู logs, metrics
  - ฟรี (self-host)

แทน: Heroku ($25/mo), Railway ($5-20/mo), Vercel
```

#### Dragonfly

```
Dragonfly = Redis-compatible in-memory database

เหมือน Redis ทุกอย่าง แต่:
  - เร็วกว่า Redis 25x ในบาง workload
  - ใช้ memory น้อยกว่า
  - Single binary, deploy ง่าย
  - รองรับ Redis commands ทั้งหมด
  - ใช้ redis client library เดิมได้เลย

ใช้สำหรับ:
  - Semantic cache
  - Session store
  - Rate limiting
  - Embedding cache
  - DB query cache
```

#### ParadeDB

```
ParadeDB = PostgreSQL distribution ที่ setup มาพร้อมใช้:

รวมมาให้แล้ว:
  - pgvector (vector search)
  - pg_search (BM25 full-text search)
  - pg_analytics (columnar storage)

คือ PostgreSQL + extensions ที่ต้องใช้กับ RAG ทั้งหมด
ไม่ต้อง setup เอง ลง Docker image เดียวจบ

เทียบกับ PostgreSQL ปกติ:
  PostgreSQL: ต้อง install pgvector เอง, ไม่มี BM25 จริงๆ
  ParadeDB:   มี pgvector + BM25 ในตัว, config มาแล้ว
```

### 1.8 Techniques

#### Tool Call / Function Calling

```
Tool Call = ให้ LLM "เรียกใช้ function" ได้เอง

แทนที่จะ:
  1. Search 5 chunks ← เราตัดสินใจ
  2. ยัดทุก chunk เข้า prompt ← waste tokens
  3. LLM ตอบ

Tool Call:
  1. บอก LLM: "คุณมี tools: search_hr, search_product, search_sop"
  2. LLM ตัดสินใจเอง: "คำถามนี้ต้อง search_hr('ลาพักร้อน')"
  3. เราเรียก search_hr → ได้ผลลัพธ์
  4. ส่งผลลัพธ์กลับให้ LLM → ตอบ

ข้อดี:
  ✅ LLM เลือก context เอง = แม่นกว่า
  ✅ เรียกเท่าที่จำเป็น = ประหยัด tokens
  ✅ loop ได้ถ้าต้องการข้อมูลเพิ่ม
```

#### Exponential Backoff

```
Exponential Backoff = retry ที่เว้นระยะเพิ่มขึ้นเท่าตัว

retry 1: รอ 1 วินาที  → ลองอีก
retry 2: รอ 2 วินาที  → ลองอีก
retry 3: รอ 4 วินาที  → ลองอีก
retry 4: รอ 8 วินาที  → ลองอีก
retry 5: รอ 16 วินาที → ยอมแพ้

ทำไมไม่ retry ทันที:
  ถ้า server busy แล้ว 1000 clients retry พร้อมกัน → busy หนักกว่าเดิม
  Exponential backoff → กระจาย retry ออกไป → server ฟื้นตัวได้

มักใช้กับ:
  + jitter (random delay เล็กน้อย) เพื่อไม่ให้ retry พร้อมกัน
```

#### Streaming / SSE (Server-Sent Events)

```
Streaming = ส่งคำตอบทีละ token แทนที่จะรอทั้งหมด

ไม่ streaming:
  User ถาม → รอ 3 วินาที → เห็นคำตอบทั้งหมดทีเดียว

Streaming (SSE):
  User ถาม → 0.1s เห็น "ตาม" → 0.15s เห็น "ตามนโยบาย" → 0.2s เห็น "ตามนโยบายบริษัท"...
  = เหมือน ChatGPT ที่ type ทีละคำ

เทคนิค:
  SSE (Server-Sent Events) = protocol สำหรับ server ส่ง data ทีละชิ้นให้ client
  ใช้ HTTP ปกติ แค่ Content-Type: text/event-stream
```

#### MCP (Model Context Protocol)

```
MCP = มาตรฐานให้ AI tools เข้าถึง data sources ภายนอก

Anthropic สร้างขึ้น เป็นเหมือน "USB ของ AI":
  USB: เสียบ device อะไรก็ได้กับคอมไหนก็ได้
  MCP: ต่อ data source อะไรก็ได้กับ AI tool ไหนก็ได้

สร้าง MCP Server 1 ตัว (RAG Agent) → ใช้ได้กับ:
  Claude Desktop, Claude Code, Cursor, VS Code, Windsurf, Cline, Zed...
  ไม่ต้องเขียน integration แยกสำหรับแต่ละ tool
```

---

## Part 2: การเตรียม Data เข้า Vector Database — Deep Dive

### 2.1 ภาพรวม Pipeline ทั้งหมด

```
                    Data Preparation Pipeline
                    
  ┌─────────┐    ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
  │ Extract  │───▶│  Clean  │───▶│  Chunk   │───▶│  Enrich  │───▶│  Index   │
  │          │    │         │    │          │    │          │    │          │
  │ PDF      │    │ ลบ noise│    │ หั่นเป็น  │    │ + embed  │    │ เข้า DB  │
  │ Docs     │    │ format  │    │ ชิ้นเล็กๆ │    │ + summary│    │ + index  │
  │ MD       │    │ normalize│   │          │    │ + hash   │    │          │
  │ HTML     │    │         │    │          │    │ + meta   │    │          │
  └─────────┘    └─────────┘    └──────────┘    └──────────┘    └──────────┘
  
  ทุกขั้นตอนสำคัญ ถ้าขั้นตอนไหนทำไม่ดี → ผลลัพธ์ search จะแย่
  "Garbage in, garbage out"
```

### 2.2 Step 1: Extract — ดึงข้อมูลจาก Source

```python
# ==========================================
# ดึงข้อมูลจากแต่ละ format
# ==========================================

# --- PDF ---
# pip install pymupdf (เร็วกว่า PyPDF2, รองรับ layout ดีกว่า)
import fitz  # pymupdf

def extract_pdf(path: str) -> str:
    doc = fitz.open(path)
    text = ""
    for page in doc:
        text += page.get_text("text") + "\n"
    doc.close()
    return text

# output:
# "นโยบายการลา\n\nลาพักร้อน\nพนักงานประจำ..."


# --- PDF ที่เป็นรูป (scanned) ---
# pip install pytesseract pillow
# ต้องติดตั้ง Tesseract OCR ก่อน
import pytesseract
from PIL import Image

def extract_scanned_pdf(path: str) -> str:
    doc = fitz.open(path)
    text = ""
    for page in doc:
        pix = page.get_pixmap()
        img = Image.frombytes("RGB", [pix.width, pix.height], pix.samples)
        text += pytesseract.image_to_string(img, lang='tha+eng') + "\n"
    doc.close()
    return text


# --- Markdown ---
def extract_markdown(path: str) -> str:
    with open(path, 'r', encoding='utf-8') as f:
        return f.read()
    # Markdown เก็บ raw ไว้เลย เพราะ headers มีประโยชน์ตอน chunk


# --- Google Docs (via API) ---
# pip install google-api-python-client
from googleapiclient.discovery import build

def extract_google_doc(doc_id: str, credentials) -> str:
    service = build('docs', 'v1', credentials=credentials)
    doc = service.documents().get(documentId=doc_id).execute()
    
    text = ""
    for element in doc.get('body', {}).get('content', []):
        if 'paragraph' in element:
            for elem in element['paragraph'].get('elements', []):
                if 'textRun' in elem:
                    text += elem['textRun']['content']
    return text


# --- Notion ---
# pip install notion-client
from notion_client import Client

def extract_notion_page(token: str, page_id: str) -> str:
    notion = Client(auth=token)
    blocks = notion.blocks.children.list(block_id=page_id)
    
    text = ""
    for block in blocks["results"]:
        block_type = block["type"]
        
        if block_type in ["paragraph", "heading_1", "heading_2", "heading_3",
                          "bulleted_list_item", "numbered_list_item"]:
            rich_texts = block[block_type].get("rich_text", [])
            line = "".join([rt["plain_text"] for rt in rich_texts])
            
            # เพิ่ม header markers
            if block_type == "heading_1": line = f"# {line}"
            elif block_type == "heading_2": line = f"## {line}"
            elif block_type == "heading_3": line = f"### {line}"
            
            text += line + "\n"
    
    return text


# --- CSV/Excel (FAQ format) ---
import pandas as pd

def extract_faq(path: str) -> list[dict]:
    if path.endswith('.csv'):
        df = pd.read_csv(path)
    else:
        df = pd.read_excel(path)
    
    # สมมติ columns: question, answer, category
    faqs = []
    for _, row in df.iterrows():
        faqs.append({
            "content": f"คำถาม: {row['question']}\nคำตอบ: {row['answer']}",
            "metadata": {
                "title": row["question"][:100],
                "category": row.get("category", "faq"),
                "doc_type": "faq",
            }
        })
    return faqs

# output:
# [
#   {
#     "content": "คำถาม: ลาพักร้อนกี่วัน?\nคำตอบ: 10 วันต่อปี",
#     "metadata": {"title": "ลาพักร้อนกี่วัน?", "category": "hr", "doc_type": "faq"}
#   },
#   ...
# ]
```

### 2.3 Step 2: Clean — ทำความสะอาดข้อมูล

```python
import re

def clean_text(text: str) -> str:
    """ทำความสะอาดข้อมูลก่อน chunk"""
    
    # 1. ลบ null characters
    text = text.replace('\x00', '')
    
    # 2. ลบ control characters (ยกเว้น \n \t)
    text = re.sub(r'[\x01-\x08\x0b\x0c\x0e-\x1f\x7f]', '', text)
    
    # 3. Normalize whitespace
    text = re.sub(r'[ \t]+', ' ', text)       # หลาย spaces → 1 space
    text = re.sub(r'\n{3,}', '\n\n', text)    # หลาย newlines → 2 newlines
    
    # 4. ลบ page numbers / headers ซ้ำ (PDF)
    text = re.sub(r'(?m)^Page \d+ of \d+$', '', text)
    text = re.sub(r'(?m)^-\s*\d+\s*-$', '', text)
    
    # 5. ลบ URLs ที่ยาวมาก (เก็บสั้นๆ ไว้)
    text = re.sub(r'https?://\S{200,}', '[long-url-removed]', text)
    
    # 6. Normalize quotes
    text = text.replace('"', '"').replace('"', '"')
    text = text.replace(''', "'").replace(''', "'")
    
    # 7. Trim
    text = text.strip()
    
    return text


# ตัวอย่าง input:
input_text = """
Page 1 of 5

      นโยบายการลา      
   
   
   
   ลาพักร้อน
   
พนักงานประจำ   มีสิทธิ์ลาพักร้อน   ปีละ 10 วัน

Page 2 of 5
"""

# output หลัง clean:
# "นโยบายการลา\n\nลาพักร้อน\n\nพนักงานประจำ มีสิทธิ์ลาพักร้อน ปีละ 10 วัน"
```

### 2.4 Step 3: Chunk — หั่นข้อมูล (ส่วนสำคัญที่สุด)

#### ประเภท Chunking ทั้งหมด

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Chunking Strategies                              │
├───────────────────┬─────────────────────────────────────────────────────┤
│                   │                                                     │
│ 1. Fixed Size     │ หั่นตามจำนวน characters/tokens ตายตัว               │
│                   │ ง่ายที่สุด แต่อาจตัดกลางประโยค                      │
│                   │                                                     │
│ 2. Recursive      │ พยายามตัดที่ paragraph → sentence → word            │
│                   │ ดีกว่า fixed size มาก ← ใช้บ่อยที่สุด               │
│                   │                                                     │
│ 3. Document       │ ตัดตามโครงสร้างเอกสาร (headers, sections)           │
│    Structure      │ ดีมากสำหรับ Markdown/HTML ที่มี headers             │
│                   │                                                     │
│ 4. Semantic       │ ใช้ embedding ตัดที่ "ความหมายเปลี่ยน"               │
│                   │ แม่นที่สุด แต่ช้าและแพง (ต้องเรียก embedding model) │
│                   │                                                     │
│ 5. Sentence       │ หั่นเป็นประโยคเดียว + เก็บ context รอบข้าง          │
│    Window         │ ดีสำหรับ precise retrieval                         │
│                   │                                                     │
│ 6. File + Header  │ แบ่งตาม file ก่อน แล้วซอยตาม headers ← Elysia ใช้ │
│    (Hierarchical) │ ดีสำหรับ documentation ที่มี structure              │
│                   │                                                     │
│ 7. Agentic        │ ให้ LLM ตัดสินใจเองว่าจะตัดตรงไหน                  │
│    Chunking       │ แม่นมาก แต่แพงมาก (เรียก LLM ทุก chunk)            │
│                   │                                                     │
│ 8. Late Chunking  │ Embed ทั้งเอกสารก่อน แล้วค่อยตัด                   │
│    (2024 ใหม่)     │ vector ยังมีบริบทของเอกสารทั้งหมด                   │
│                   │                                                     │
└───────────────────┴─────────────────────────────────────────────────────┘
```

#### ตัวอย่างแต่ละวิธี

```python
# ==========================================
# สมมติ document นี้:
# ==========================================
sample_doc = """# นโยบายการลา

## ลาพักร้อน
พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ การลาพักร้อนต้องได้รับอนุมัติจากหัวหน้างานโดยตรง หากลาเกิน 3 วันติดต่อกันต้องแจ้งล่วงหน้า 1 สัปดาห์ พนักงานที่ทำงานไม่ครบ 1 ปี จะได้สิทธิ์ลาพักร้อนตามสัดส่วน

## ลาป่วย
พนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน โดยได้รับค่าจ้าง หากลาป่วยเกิน 3 วันติดต่อกัน ต้องมีใบรับรองแพทย์ประกอบ กรณีป่วยหนักอาจขอลาพิเศษเพิ่มได้โดยพิจารณาเป็นรายกรณี

## ลากิจ
พนักงานมีสิทธิ์ลากิจปีละ 5 วัน สำหรับธุระส่วนตัวที่จำเป็น เช่น งานแต่งงาน งานศพ ย้ายบ้าน ต้องแจ้งล่วงหน้าอย่างน้อย 1 วัน ยกเว้นกรณีฉุกเฉิน"""


# ==========================================
# วิธี 1: Fixed Size Chunking
# ==========================================
def chunk_fixed(text: str, size: int = 200, overlap: int = 50) -> list[str]:
    chunks = []
    start = 0
    while start < len(text):
        end = start + size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks

fixed_chunks = chunk_fixed(sample_doc, size=200, overlap=50)

# ผลลัพธ์:
# Chunk 1: "# นโยบายการลา\n\n## ลาพักร้อน\nพนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ การลาพักร้อนต้องได้รับอนุมัติจากหัวหน้างานโดยตรง หากลาเกิ"
# Chunk 2: "เกิน 3 วันติดต่อกันต้องแจ้งล่วงหน้า 1 สัปดาห์ พนักงานที่ทำงานไม่ครบ 1 ปี จะได้สิทธิ์ลาพักร้อนตามสัดส่วน\n\n## ลาป่วย\nพนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน โดยได้รับค่าจ้าง"
# 
# ❌ ปัญหา: ตัดกลางคำ "เกิ|น" ที่ chunk 1


# ==========================================
# วิธี 2: Recursive Chunking ← ใช้บ่อยที่สุด
# ==========================================
from langchain_text_splitters import RecursiveCharacterTextSplitter

def chunk_recursive(text: str, size: int = 500, overlap: int = 100) -> list[str]:
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=size,
        chunk_overlap=overlap,
        separators=["\n\n", "\n", "。", ". ", " ", ""],
        # พยายามตัดที่ "\n\n" ก่อน (paragraph break)
        # ถ้าอยังใหญ่เกิน → ตัดที่ "\n" (line break)
        # ถ้าอยังใหญ่เกิน → ตัดที่ ". " (จบประโยค)
        # สุดท้ายตัดที่ " " (space)
    )
    return splitter.split_text(text)

recursive_chunks = chunk_recursive(sample_doc, size=300, overlap=50)

# ผลลัพธ์:
# Chunk 1: "# นโยบายการลา\n\n## ลาพักร้อน\nพนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ การลาพักร้อนต้องได้รับอนุมัติจากหัวหน้างานโดยตรง"
# Chunk 2: "หากลาเกิน 3 วันติดต่อกันต้องแจ้งล่วงหน้า 1 สัปดาห์ พนักงานที่ทำงานไม่ครบ 1 ปี จะได้สิทธิ์ลาพักร้อนตามสัดส่วน"
# Chunk 3: "## ลาป่วย\nพนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน โดยได้รับค่าจ้าง หากลาป่วยเกิน 3 วันติดต่อกัน ต้องมีใบรับรองแพทย์ประกอบ"
# 
# ✅ ดีขึ้น: ตัดที่ paragraph break ไม่ตัดกลางประโยค


# ==========================================
# วิธี 3: Document Structure (Header-based)
# ==========================================
from langchain_text_splitters import MarkdownHeaderTextSplitter

def chunk_by_headers(text: str) -> list[dict]:
    splitter = MarkdownHeaderTextSplitter(
        headers_to_split_on=[
            ("#", "header_1"),
            ("##", "header_2"),
            ("###", "header_3"),
        ]
    )
    chunks = splitter.split_text(text)
    return [
        {
            "content": chunk.page_content,
            "metadata": chunk.metadata,
        }
        for chunk in chunks
    ]

header_chunks = chunk_by_headers(sample_doc)

# ผลลัพธ์:
# Chunk 1: {
#   "content": "พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน โดยต้องแจ้งล่วงหน้า...",
#   "metadata": {"header_1": "นโยบายการลา", "header_2": "ลาพักร้อน"}
# }
# Chunk 2: {
#   "content": "พนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน โดยได้รับค่าจ้าง...",
#   "metadata": {"header_1": "นโยบายการลา", "header_2": "ลาป่วย"}
# }
# Chunk 3: {
#   "content": "พนักงานมีสิทธิ์ลากิจปีละ 5 วัน...",
#   "metadata": {"header_1": "นโยบายการลา", "header_2": "ลากิจ"}
# }
#
# ✅ ดีมาก: แต่ละ chunk มี topic ชัดเจน + metadata บอกว่าอยู่ section ไหน


# ==========================================
# วิธี 4: File + Header (Hierarchical) ← Elysia ใช้
# ==========================================
def chunk_hierarchical(filepath: str, source_meta: dict) -> list[dict]:
    """
    1. อ่าน file ทั้งหมด
    2. แบ่งตาม headers เป็น sections
    3. ถ้า section ยังยาวเกิน → recursive split ต่อ
    4. เก็บ hierarchy ทั้งหมดใน metadata
    """
    text = extract_markdown(filepath)
    
    # Step 1: Split by headers
    sections = []
    current_h1 = ""
    current_h2 = ""
    current_content = ""
    
    for line in text.split("\n"):
        if line.startswith("# ") and not line.startswith("##"):
            if current_content.strip():
                sections.append({
                    "content": current_content.strip(),
                    "h1": current_h1,
                    "h2": current_h2,
                })
            current_h1 = line[2:].strip()
            current_h2 = ""
            current_content = ""
        elif line.startswith("## "):
            if current_content.strip():
                sections.append({
                    "content": current_content.strip(),
                    "h1": current_h1,
                    "h2": current_h2,
                })
            current_h2 = line[3:].strip()
            current_content = ""
        else:
            current_content += line + "\n"
    
    # อย่าลืม section สุดท้าย
    if current_content.strip():
        sections.append({
            "content": current_content.strip(),
            "h1": current_h1,
            "h2": current_h2,
        })
    
    # Step 2: Sub-chunk ถ้ายาวเกิน
    chunks = []
    for section in sections:
        content = section["content"]
        
        if len(content) > 1000:
            sub_chunks = chunk_recursive(content, size=800, overlap=150)
        else:
            sub_chunks = [content]
        
        for i, sub in enumerate(sub_chunks):
            chunks.append({
                "content": sub,
                "metadata": {
                    **source_meta,
                    "header_1": section["h1"],
                    "header_2": section["h2"],
                    "sub_chunk_index": i,
                    "title": section["h2"] or section["h1"] or source_meta.get("title", ""),
                }
            })
    
    return chunks

# ผลลัพธ์ (เหมือนวิธี 3 แต่จัดการ sub-chunking ด้วย):
# [
#   {"content": "พนักงานประจำ...", "metadata": {"header_1": "นโยบายการลา", "header_2": "ลาพักร้อน", ...}},
#   {"content": "พนักงานมีสิทธิ์ลาป่วย...", "metadata": {"header_1": "นโยบายการลา", "header_2": "ลาป่วย", ...}},
#   {"content": "พนักงานมีสิทธิ์ลากิจ...", "metadata": {"header_1": "นโยบายการลา", "header_2": "ลากิจ", ...}},
# ]


# ==========================================
# วิธี 5: Semantic Chunking (ตัดตามความหมาย)
# ==========================================
# pip install langchain-experimental

from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

def chunk_semantic(text: str) -> list[str]:
    """
    วิธีทำงาน:
    1. แบ่งเป็นประโยค
    2. Embed แต่ละประโยค
    3. เทียบ cosine similarity ระหว่างประโยคที่ติดกัน
    4. ถ้า similarity drop ลงมาก = "ความหมายเปลี่ยน" = จุดตัด
    
    ตัวอย่าง:
    ประโยค 1: "ลาพักร้อน 10 วัน"           ─┐
    ประโยค 2: "ต้องแจ้งล่วงหน้า 3 วัน"     ─┤ similarity สูง → กลุ่มเดียว
    ประโยค 3: "ต้องได้รับอนุมัติจากหัวหน้า"  ─┘
                                              ← similarity drop!
    ประโยค 4: "ลาป่วยได้ 30 วัน"           ─┐
    ประโยค 5: "ต้องมีใบรับรองแพทย์"         ─┘ กลุ่มใหม่
    """
    embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
    splitter = SemanticChunker(
        embeddings,
        breakpoint_threshold_type="percentile",  # หรือ "standard_deviation"
        breakpoint_threshold_amount=75,           # ตัดที่ percentile 75
    )
    return splitter.split_text(text)

# ✅ แม่นที่สุด
# ❌ ช้า + แพง (ต้อง embed ทุกประโยค)
# ใช้เมื่อ: เอกสารที่ structure ไม่ชัด (เช่น meeting notes, เอกสารยาวๆ ไม่มี headers)
```

#### เลือกวิธีไหนดี

```
เอกสารมี headers ชัดเจน (Markdown, HTML, Docs ที่มี heading):
  → วิธี 3 (Header-based) หรือ วิธี 4 (Hierarchical) ✅

เอกสารไม่มี structure (plain text, meeting notes):
  → วิธี 2 (Recursive) ✅
  → หรือ วิธี 5 (Semantic) ถ้าอยากแม่นมาก

FAQ / Q&A pairs:
  → ไม่ต้อง chunk! แต่ละ Q&A เป็น 1 chunk เลย ✅

เอกสารสั้นๆ (< 500 ตัวอักษร):
  → ไม่ต้อง chunk! เก็บทั้งหมดเป็น 1 chunk ✅

Documentation (เช่น Sellsuki docs):
  → วิธี 4 (Hierarchical) ← แนะนำ ✅
  → file-level → header-level → sub-chunk ถ้ายาวเกิน
```

### 2.5 Step 4: Enrich — เพิ่มคุณค่าให้ Chunk

```python
import hashlib

async def enrich_chunk(chunk: dict, embed_model, llm_client) -> dict:
    """เพิ่ม embedding, summary, hash ให้ chunk"""
    
    content = chunk["content"]
    
    # 1. Content Hash (สำหรับ diff-based indexing)
    chunk["content_hash"] = hashlib.sha256(content.encode()).hexdigest()
    
    # 2. Embedding
    chunk["embedding"] = await get_embedding(content, model=embed_model)
    
    # 3. Summary (ใช้ LLM สรุป — ทำตอน index ครั้งเดียว)
    summary_response = await llm_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": "สรุปข้อความนี้ให้สั้นกระชับ 1-2 ประโยค เก็บ key facts ทั้งหมด ภาษาไทย"
            },
            {"role": "user", "content": content}
        ],
        max_tokens=150,
    )
    chunk["summary"] = summary_response.choices[0].message.content
    
    return chunk


# ตัวอย่าง input:
chunk_input = {
    "content": "พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ การลาพักร้อนต้องได้รับอนุมัติจากหัวหน้างานโดยตรง หากลาเกิน 3 วันติดต่อกันต้องแจ้งล่วงหน้า 1 สัปดาห์",
    "metadata": {
        "source": "hr_policy.md",
        "header_1": "นโยบายการลา",
        "header_2": "ลาพักร้อน",
        "category": "hr",
    }
}

# ตัวอย่าง output หลัง enrich:
chunk_output = {
    "content": "พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน โดยต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการ...",
    "content_hash": "a3f2b8c1d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1",
    "embedding": [0.034, -0.128, 0.891, ..., 0.045],  # 768 dimensions
    "summary": "พนักงานลาพักร้อนได้ 10 วัน/ปี แจ้งล่วงหน้า 3 วัน (เกิน 3 วันแจ้ง 1 สัปดาห์) ต้องหัวหน้าอนุมัติ",
    "metadata": {
        "source": "hr_policy.md",
        "header_1": "นโยบายการลา",
        "header_2": "ลาพักร้อน",
        "category": "hr",
        "title": "ลาพักร้อน",
    }
}
```

### 2.6 Step 5: Index — เข้า Database

```python
import asyncpg
import json

async def insert_chunks(pool: asyncpg.Pool, chunks: list[dict]):
    """Insert enriched chunks เข้า ParadeDB"""
    
    async with pool.acquire() as conn:
        # Batch insert สำหรับ performance
        records = []
        for chunk in chunks:
            meta = chunk["metadata"]
            records.append((
                chunk["content"],
                chunk.get("summary", ""),
                str(chunk["embedding"]),     # vector as string
                chunk["content_hash"],
                meta.get("source", ""),
                meta.get("category", ""),
                meta.get("department", ""),
                meta.get("title", ""),
                meta.get("doc_type", "document"),
                json.dumps(meta),
            ))
        
        await conn.executemany("""
            INSERT INTO documents 
            (content, summary, embedding, content_hash,
             source, category, department, title, doc_type, metadata)
            VALUES ($1, $2, $3::vector, $4, $5, $6, $7, $8, $9, $10::jsonb)
        """, records)
    
    print(f"✅ Inserted {len(chunks)} chunks")


# ==========================================
# สิ่งที่ถูกเก็บใน Database (ตัวอย่าง)
# ==========================================
#
# id | content                                    | summary                        | embedding           | content_hash | source        | category | title      |
# ---|--------------------------------------------|---------------------------------|---------------------|-------------|---------------|----------|------------|
# 1  | พนักงานประจำมีสิทธิ์ลาพักร้อนปีละ 10 วัน...   | ลาพักร้อน 10วัน/ปี แจ้ง3วัน...   | [0.034,-0.128,...]  | a3f2b8c1... | hr_policy.md  | hr       | ลาพักร้อน   |
# 2  | พนักงานมีสิทธิ์ลาป่วยปีละ 30 วัน...           | ลาป่วย30วัน เกิน3วันใช้ใบแพทย์   | [0.012,0.445,...]   | b7d3e9f2... | hr_policy.md  | hr       | ลาป่วย      |
# 3  | พนักงานมีสิทธิ์ลากิจปีละ 5 วัน...             | ลากิจ5วัน แจ้ง1วัน ฉุกเฉินได้    | [-0.091,0.234,...]  | c4a8b1d6... | hr_policy.md  | hr       | ลากิจ       |
# 4  | วิธีสร้าง Order ใน Sellsuki...              | สร้างorder: เมนู→กรอกข้อมูล→ยืนยัน | [0.567,-0.321,...] | f2c9a7e3... | manual.md     | product  | สร้าง Order  |
```

### 2.7 Full Pipeline — รวมทุกขั้นตอน

```python
import asyncio
from pathlib import Path

async def full_pipeline(
    docs_dir: str,
    db_pool: asyncpg.Pool,
    embed_model: str = "text-embedding-004",
    chunk_size: int = 800,
):
    """Pipeline เต็ม: Extract → Clean → Chunk → Enrich → Index"""
    
    all_chunks = []
    docs_dir = Path(docs_dir)
    
    # ==========================================
    # Step 1-2: Extract + Clean
    # ==========================================
    print("📂 Extracting documents...")
    
    for filepath in docs_dir.rglob("*"):
        if filepath.suffix in ['.md', '.txt']:
            text = extract_markdown(str(filepath))
        elif filepath.suffix == '.pdf':
            text = extract_pdf(str(filepath))
        elif filepath.suffix == '.docx':
            text = extract_docx(str(filepath))
        elif filepath.suffix in ['.csv', '.xlsx']:
            faqs = extract_faq(str(filepath))
            all_chunks.extend(faqs)  # FAQs ไม่ต้อง chunk
            continue
        else:
            continue
        
        text = clean_text(text)
        
        # ==========================================
        # Step 3: Chunk
        # ==========================================
        source_meta = {
            "source": filepath.name,
            "category": filepath.parent.name,  # folder name = category
            "doc_type": "document",
        }
        
        if filepath.suffix == '.md':
            # Markdown → ใช้ header-based chunking
            chunks = chunk_hierarchical(str(filepath), source_meta)
        else:
            # อื่นๆ → ใช้ recursive chunking
            raw_chunks = chunk_recursive(text, size=chunk_size, overlap=150)
            chunks = [
                {"content": c, "metadata": {**source_meta, "title": filepath.stem}}
                for c in raw_chunks
            ]
        
        all_chunks.extend(chunks)
        print(f"  ✅ {filepath.name} → {len(chunks)} chunks")
    
    print(f"\n📊 Total chunks: {len(all_chunks)}")
    
    # ==========================================
    # Step 4: Enrich (embed + summarize + hash)
    # ==========================================
    print("\n🔄 Enriching chunks...")
    
    # Check existing hashes (diff-based)
    existing_hashes = set()
    async with db_pool.acquire() as conn:
        rows = await conn.fetch("SELECT content_hash FROM documents")
        existing_hashes = {row["content_hash"] for row in rows}
    
    new_chunks = []
    for chunk in all_chunks:
        content_hash = hashlib.sha256(chunk["content"].encode()).hexdigest()
        if content_hash not in existing_hashes:
            chunk["content_hash"] = content_hash
            new_chunks.append(chunk)
    
    print(f"  ⏭️  Skipped: {len(all_chunks) - len(new_chunks)} (unchanged)")
    print(f"  🆕 New/changed: {len(new_chunks)}")
    
    if not new_chunks:
        print("✅ Nothing to update!")
        return
    
    # Batch embed
    print("  📐 Generating embeddings...")
    contents = [c["content"] for c in new_chunks]
    embeddings = await get_embeddings_batch(contents, model=embed_model)
    
    for chunk, emb in zip(new_chunks, embeddings):
        chunk["embedding"] = emb
    
    # Batch summarize
    print("  📝 Generating summaries...")
    for chunk in new_chunks:
        chunk["summary"] = await generate_summary(chunk["content"])
    
    # ==========================================
    # Step 5: Index
    # ==========================================
    print("\n💾 Inserting into database...")
    await insert_chunks(db_pool, new_chunks)
    
    print(f"\n🎉 Done! Indexed {len(new_chunks)} new chunks")


# ==========================================
# รัน Pipeline
# ==========================================
# โครงสร้าง folder:
# company_docs/
# ├── hr/
# │   ├── leave_policy.md
# │   ├── benefits.md
# │   └── faq.csv
# ├── product/
# │   ├── sellsuki_manual.md
# │   └── api_docs.md
# └── sop/
#     ├── order_management.md
#     └── customer_support.pdf

asyncio.run(full_pipeline(
    docs_dir="./company_docs",
    db_pool=pool,
    embed_model="text-embedding-004",
    chunk_size=800,
))

# Output:
# 📂 Extracting documents...
#   ✅ leave_policy.md → 8 chunks
#   ✅ benefits.md → 12 chunks
#   ✅ faq.csv → 25 chunks (FAQ, no chunking needed)
#   ✅ sellsuki_manual.md → 45 chunks
#   ✅ api_docs.md → 30 chunks
#   ✅ order_management.md → 15 chunks
#   ✅ customer_support.pdf → 20 chunks
#
# 📊 Total chunks: 155
#
# 🔄 Enriching chunks...
#   ⏭️  Skipped: 120 (unchanged)
#   🆕 New/changed: 35
#   📐 Generating embeddings...
#   📝 Generating summaries...
#
# 💾 Inserting into database...
#
# 🎉 Done! Indexed 35 new chunks
```

### 2.8 วิธีที่ปัจจุบัน (2025) ใช้กัน

```
Industry Best Practices 2025:

1. Hybrid Search เป็น default
   → Vector + BM25 ไม่ใช่ vector อย่างเดียวแล้ว
   → ทุก vector DB เริ่มรองรับ hybrid

2. Chunking ที่ดีสำคัญกว่า model ที่ดี
   → chunk ไม่ดี + GPT-4o = ผลลัพธ์แย่
   → chunk ดี + Llama 70B = ผลลัพธ์ดี
   → "ลงทุนกับ chunking ก่อน"

3. Metadata filtering ก่อน vector search
   → filter category/department ก่อน → แล้วค่อย vector search
   → เร็วกว่า + แม่นกว่า search ทั้ง DB

4. Summarized content ลด cost
   → เก็บทั้ง full + summary
   → ส่ง summary เข้า LLM ก่อน ถ้าต้อง detail ค่อยดึง full

5. Diff-based indexing เป็นมาตรฐาน
   → ไม่มีใคร re-index ทั้งหมดทุกครั้งแล้ว
   → hash-based diff + incremental update

6. Late Chunking (เทรนด์ใหม่ 2024-2025)
   → Embed ทั้งเอกสารก่อน แล้วค่อยหั่น
   → vector ของแต่ละ chunk ยังมีบริบทของทั้งเอกสาร
   → ต้องใช้ long-context embedding model

7. Evaluation-driven development
   → สร้าง test set (50-100 คำถาม + คำตอบที่ถูกต้อง)
   → วัดผลทุกครั้งที่เปลี่ยน config (chunk size, model, etc.)
   → ใช้ metrics: precision@k, recall@k, MRR
```

### 2.9 Checklist สรุป

```
การเตรียม Data เข้า Vector Database:

□ Step 1: Extract
  □ รวบรวมเอกสารทั้งหมด
  □ เขียน extractor สำหรับแต่ละ format
  □ ทดสอบ extract แล้วอ่านได้ครบถ้วน

□ Step 2: Clean
  □ ลบ noise (page numbers, headers ซ้ำ, special chars)
  □ Normalize whitespace
  □ ตรวจสอบ encoding (UTF-8)

□ Step 3: Chunk
  □ เลือก strategy ที่เหมาะ (header-based สำหรับ docs มี structure)
  □ ทดสอบ chunk size (เริ่มที่ 800 chars)
  □ ตรวจว่าแต่ละ chunk มีความหมายครบ อ่านแล้วเข้าใจ
  □ ไม่ตัดกลางประโยค/กลางความหมาย

□ Step 4: Enrich
  □ สร้าง content_hash ทุก chunk
  □ สร้าง embedding ทุก chunk
  □ สร้าง summary ทุก chunk (optional แต่แนะนำ)
  □ เพิ่ม metadata ให้ครบ (source, category, title, etc.)

□ Step 5: Index
  □ Insert เข้า database
  □ สร้าง HNSW index สำหรับ vector
  □ สร้าง BM25 index สำหรับ full-text
  □ สร้าง metadata indexes

□ Step 6: Verify
  □ ทดสอบ search ด้วย 20+ คำถามจริง
  □ ตรวจว่า top results เกี่ยวข้องจริง
  □ ปรับ chunk size / threshold ถ้าจำเป็น
  □ วัด latency (target < 100ms สำหรับ search)
```
