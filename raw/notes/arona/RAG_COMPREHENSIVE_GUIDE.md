# คู่มือ RAG ครอบคลุม: ตั้งแต่พื้นฐานไปสู่ขั้นสูง

## สารบัญ

1. [รู้จักกับ RAG](#รู้จักกับ-rag)
2. [แนวคิดพื้นฐาน](#แนวคิดพื้นฐาน)
3. [Embeddings](#embeddings)
4. [Vector Databases](#vector-databases)
5. [การแบ่งเอกสาร](#การแบ่งเอกสาร)
6. [วิธีการค้นหา](#วิธีการค้นหา)
7. [การสร้างคำตอบ](#การสร้างคำตอบ)
8. [เทคนิคขั้นสูง](#เทคนิคขั้นสูง)
9. [ข้อควรพิจารณาสำหรับ Production](#ข้อควรพิจารณาสำหรับ-production)
10. [ตัวอย่างการนำไปใช้](#ตัวอย่างการนำไปใช้)

---

## รู้จักกับ RAG

### RAG คืออะไร?

**RAG (Retrieval-Augmented Generation)** คือสถาปัตยกรรม AI ที่เพิ่มความสามารถให้ Large Language Models (LLMs) โดยนำข้อมูลที่เกี่ยวข้องและเฉพาะเจาะจงมาให้ก่อนที่ AI จะตอบคำถาม

```
LLM แบบดั้งเดิม:
คำถาม → LLM → คำตอบ
              (จำกัดด้วยข้อมูลที่เคยเรียนมา)

LLM แบบ RAG:
คำถาม → ค้นหาเอกสารที่เกี่ยวข้อง → เพิ่ม Context → LLM → คำตอบ
              (เข้าถึงข้อมูลล่าสุดและเฉพาะทาง)
```

### ทำไม RAG สำคัญ

| ปัญหา | ไม่มี RAG | มี RAG |
|---------|-------------|----------|
| **ข้อมูลล้าสมัย** | LLM รู้เฉพาะข้อมูลถึงวันที่เรียน | เข้าถึงข้อมูลล่าสุด |
| **โกหก** | LLM อาจสร้างข้อมูลเท็จ | อ้างอิงจากข้อเท็จจริง |
| **ความเฉพาะทาง** | รู้เฉพาะความรู้ทั่วไป | มีความรู้เฉพาะด้าน |
| **การอ้างอิง** | ไม่สามารถระบุแหล่งที่มา | อ้างอิงแหล่งที่มาได้ |

### สถาปัตยกรรม RAG โดยรวม

```
┌─────────────────────────────────────────────────────────────────────┐
│                         สถาปัตยกรรมระบบ RAG                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  เฟสทำดัชนี (Indexing Phase - Offline)                             │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐     │
│  │เอกสาร  │───▶│ แบ่งส่วน │───▶│Embedding │───▶│ Vector   │     │
│  │Documents │    │ Chunking │    │          │    │   DB     │     │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘     │
│                                                    ▲                │
│  เฟสค้นหา (Retrieval Phase - Online)              │                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    │                │
│  │  คำถาม  │───▶│Embedding │───▶│ Similarity│────┘                │
│  │  Query   │    │          │    │  Search  │                     │
│  └──────────┘    └──────────┘    └──────────┘                     │
│       │                                                   │        │
│       ▼                                                   ▼        │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                    เฟสสร้างคำตอบ (Generation Phase)          │  │
│  │  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐│  │
│  │  │ ส่วนที่  │───▶│ สร้าง   │───▶│   LLM    │───▶│คำตอบ  ││  │
│  │  │ค้นได้   │    │ Context  │    │          │    │        ││  │
│  │  └──────────┘    └──────────┘    └──────────┘    └────────┘│  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## แนวคิดพื้นฐาน

### 1. Vector Embeddings

**Embeddings คืออะไร?**

Embeddings คือการแปลงข้อความ (หรือข้อมูลอื่นๆ) ให้เป็นตัวเลขที่แสดงความหมาย ความคล้ายคลึงกันทางความหมายจะมีค่าเวคเตอร์ใกล้เคียงกัน

```
ข้อความ: "แมวนั่งบนพื้นที่"
      ↓ Embedding Model
เวคเตอร์: [0.0123, -0.4567, 0.7890, ..., 0.0345] (1536 มิติ)

ข้อความ: "แมวนอนอยู่บนพรม"
      ↓ Embedding Model
เวคเตอร์: [0.0118, -0.4612, 0.7854, ..., 0.0338] (1536 มิติ)

คะแนนความคล้าย: 0.94 (คล้ายมาก!)
```

**คุณสมบัติหลัก:**

- **Dimensionality**: จำนวนค่าในเวคเตอร์ (เช่น 1536 สำหรับ OpenAI)
- **Semantic Similarity**: ความหมายคล้ายกัน = เวคเตอร์ใกล้กันในปริภูมิ
- **Language Independent**: แนวคิดคล้ายกันข้ามภาษาจะถูกจัดกลุ่มเข้าด้วยกัน

**Embedding Models ที่นิยม:**

| Model | มิติ | ราคา | ประสิทธิภาพ |
|-------|------------|------|-------------|
| OpenAI text-embedding-3-small | 1536 | ต่ำ | ดีมาก |
| OpenAI text-embedding-3-large | 3072 | ปานกลาง | ดีที่สุด |
| OpenAI text-embedding-ada-002 | 1536 | ต่ำ | ดี |
| Cohere embed-v3 | 1024 | ต่ำ | ดีมาก |
| Sentence Transformers (open source) | 384-768 | ฟรี | ดี |

### 2. Similarity Metrics

**Cosine Similarity** (ใช้บ่อยที่สุด)

วัดโคไซน์ของมุมระหว่างสองเวคเตอร์:

```
cosine_similarity(A, B) = (A · B) / (||A|| × ||B||)

ช่วง: -1 ถึง 1
- 1.0 = ทิศทางเดียวกัน (คล้ายที่สุด)
- 0.0 = ตั้งฉาก (ไม่เกี่ยวข้อง)
- -1.0 = ทิศทางตรงข้าม (ความหมายตรงกันข้าม)
```

**Dot Product**

```
dot_product(A, B) = A · B = Σ(Ai × Bi)

ช่วง: -∞ ถึง ∞
คำนวณเร็ว (ไม่ต้อง normalize)
ไวต่อขนาดเวคเตอร์
```

**Euclidean Distance**

```
euclidean_distance(A, B) = √(Σ(Ai - Bi)²)

ช่วง: 0 ถึง ∞
0 = เหมือนกัน
ค่ามาก = คล้ายน้อย
```

### 3. กลยุทธ์การแบ่งส่วน (Chunking)

**Fixed-Size Chunking** (แบ่งตามขนาด)

```typescript
const chunkSize = 1000  // ตัวอักษร
const overlap = 200     // ตัวอักษรทับซ้อน

// ผลลัพธ์:
Chunk 1: [0:1000]
Chunk 2: [800:1800]   // ทับซ้อน 200 ตัวอักษร
Chunk 3: [1600:2600]
```

**ข้อดี:** ง่าย, คาดเดาได้
**ข้อเสีย:** อาจตัดประโยค/แนวคิด

**Semantic Chunking** (แบ่งตามความหมาย)

```typescript
// แบ่งตามขอบเขตความหมาย
const chunks = text.split(/\n\n|\. |\? |\! /)

// หรือใช้ sentence embeddings เพื่อหาจุดแบ่งความหมาย
```

**ข้อดี:** รักษาความหมาย
**ข้อเสีย:** ขนาด chunk ไม่คงที่

**Hierarchical Chunking** (แบ่งแบบลำดับชั้น)

```typescript
เอกสาร (Document)
├── ส่วนที่ 1 (Section 1)
│   ├── ย่อหน้า 1.1
│   └── ย่อหน้า 1.2
├── ส่วนที่ 2 (Section 2)
│   ├── ย่อหน้า 2.1
│   └── ย่อหน้า 2.2

// เก็บในหลายระดับ:
// - ระดับย่อหน้า (รายละเอียด)
// - ระดับส่วน (ภาพรวม)
```

---

## Embeddings

### Embeddings ทำงานยังไง

**กระบวนการสร้าง Embedding**

```
Input Text: "How do I create a route in Elysia?"

Step 1: Tokenization
["How", "do", "I", "create", "a", "route", "in", "Elysia", "?"]

Step 2: Model Processing
[Neural Network with Millions of Parameters]

Step 3: Vector Output
[0.012, -0.234, 0.567, ..., 0.089]
  1536 dimensions
```

**การมองเห็น Embedding Space**

```
           ภาษาโปรแกรมมิ่ง
                  │
                  │
    Python ───────┼─────── JavaScript
                  │
                  │
       Web ───────┼─────── Mobile
                  │
                  │
```

แนวคิดคล้ายกันจะถูกจัดกลุ่มเข้าด้วยกันในปริภูมิหลายมิตินี้

### เทคนิค Embeddings ในทางปฏิบัติ

**1. Query Normalization** (ปรับคำถามให้เป็นมาตรฐาน)

ปรับคำถามผู้ใช้ให้ดีขึ้น:

```typescript
// คำถามดั้งเดิม (ต่างกันหมด):
"How do I create a route?"
"Help me make a route"
"I need to create an endpoint"

// หลังปรับ (คล้ายกันหมด):
"create route elysia"
```

**การนำไปใช้ของ Arona:**

```typescript
const normalizePromptInstruction = `
ลดคำถามให้เหลือความตั้งใจการค้นหาหลัก
- ลบคำ filler และคำทักทายทั้งหมด
- ลดเหลือหัวข้อหลัก + การกระทำ
- ใช้คำศัพท์ที่สอดคล้องกัน
- ไม่เกิน 12 คำ
`

const normalized = await generateText({
  model: smallModel,
  system: normalizePromptInstruction,
  prompt: "Can you please help me understand how to create a route?"
})
// Output: "create route elysia"
```

**2. Filler Word Stripping** (ลบคำที่ไม่จำเป็น)

```typescript
const fillerPattern =
  /^(?:can you tell me|i would like to|would you kindly|i was wondering|just wondering|quick question|do you know|help me with|i want to|i need to|could you|would you|can you|tell me|show me|help me|please|hello|hey|hi|pls|plz)\s+/gi

export const stripFillers = (q: string) => {
  q = q.toLowerCase()
  while (fillerPattern.test(q)) {
    q = q.replace(fillerPattern, '')
  }
  return q.replace(/[()\[\]{}@#$%^&*!?.,:;&]/g, '').trim()
}
```

**3. Embedding Caching** (เก็บ Embedding ไว้ใช้ซ้ำ)

```typescript
// Cache embeddings เพื่อหลีกเลี่ยงการคำนวณซ้ำ
const embeddingCache = new LRUCache<string, number[]>({
  max: 750,
  ttl: 9 * 60 * 60 * 1000 // 9 ชั่วโมง
})

export const getEmbedding = async (prompt: string) => {
  if (embeddingCache.has(prompt)) {
    return embeddingCache.get(prompt)!
  }

  const embedding = await embed({
    model: openai.embeddingModel('text-embedding-3-small'),
    value: prompt
  })

  embeddingCache.set(prompt, embedding.embedding)
  return embedding.embedding
}
```

---

## Vector Databases

### Vector Database คืออะไร?

ฐานข้อมูลเฉพาะทางที่ออกแบบมาเพื่อเก็บ, จัดดัชนี, และค้นหาเวคเตอร์อย่างมีประสิทธิภาพ

**คุณสมบัติหลัก:**

- **Vector Storage**: เก็บเวคเตอร์หลายมิติ
- **Approximate Nearest Neighbor (ANN) Search**: ค้นหาความคล้ายได้เร็ว
- **Indexing**: โครงสร้างข้อมูลเฉพาะสำหรับเวคเตอร์
- **Metadata Filtering**: ค้นหาด้วยเวคเตอร์ + metadata

### ตัวเลือก Vector Database

**Self-Hosted (ติดตั้งเอง):**

| Database | รายละเอียด | ข้อดี | ข้อเสีย |
|----------|-------------|------|------|
| **ParadeDB** | PostgreSQL extension | BM25 + Vector, SQL | โปรเจ็กต์ใหม่ |
| **pgvector** | PostgreSQL extension | โดดเด่น, SQL | ไม่มี BM25 ในตัว |
| **Qdrant** | Rust-based | เร็ว, ใส่ filter ง่าย | แยกจาก SQL |
| **Weaviate** | Go-based | GraphQL API | Infrastructure แยก |
| **Milvus** | Cloud-native | ขยายง่าย | ตั้งค่าซับซ้อน |

**Managed (บริการ SaaS):**

| Database | รายละเอียด | ราคาเริ่มต้น |
|----------|-------------|----------------|
| **Pinecone** | บริการครบวงจร | $70/เดือน |
| **Weaviate Cloud** | Weaviage แบบบริการ | ราคากำหนดเอง |
| **Chroma Cloud** | Chroma แบบบริการ | มี tier ฟรี |

### ParadeDB (ที่ Arona ใช้)

**ทำไมเลือก ParadeDB?**

```sql
-- BM25 Search (Full-text)
CREATE INDEX idx_documents_content_bm25
ON documents USING bm25 (
  link, title, summary, file_name
) WITH (key_field='link');

-- Vector Search
CREATE TABLE documents (
  embedding VECTOR(1536)  -- OpenAI embeddings
);

-- Hybrid: BM25 + Vector ใน query เดียว!
```

**ข้อดีหลัก:**

1. **ใช้ PostgreSQL เป็นพื้นฐาน**: SQL คุ้นเคย, ระบบนิเวศิงโย่
2. **BM25 + Vector**: ทั้งสองแบบในฐานข้อมูลเดียว
3. **Infrastructure เดียว**: ไม่ต้องมี vector DB แยก
4. **ACID Compliance**: ธุรกรรมเชื่อถือได้

---

## การแบ่งเอกสาร (Document Chunking)

### กลยุทธ์การแบ่งส่วน

**1. Fixed-Size Chunking** (แบ่งตามขนาด)

```typescript
function fixedSizeChunk(text: string, chunkSize: number, overlap: number) {
  const chunks: string[] = []
  let start = 0

  while (start < text.length) {
    const end = Math.min(start + chunkSize, text.length)
    chunks.push(text.slice(start, end))
    start = end - overlap
  }

  return chunks
}

// ตัวอย่าง:
const chunks = fixedSizeChunk(
  "This is a long text that needs to be chunked...",
  100,  // ขนาด chunk
  20    // ทับซ้อน
)
```

**2. Recursive Character Chunking** (แบ่งตามตัวแบ่งซ้ำ)

```typescript
function recursiveChunk(
  text: string,
  separators: string[] = ['\n\n', '\n', '. ', ' ', '']
): string[] {
  for (const separator of separators) {
    const chunks = text.split(separator)
    if (chunks.every(c => c.length <= MAX_CHUNK_SIZE)) {
      return chunks
    }
  }
  // Fallback to fixed-size
  return fixedSizeChunk(text, MAX_CHUNK_SIZE, OVERLAP)
}
```

**3. Semantic Chunking** (แบ่งตามความหมาย)

```typescript
// แบ่งตามหัวข้อ (เหมือน Arona)
function chunkByHeaders(markdown: string) {
  return markdown
    .split('\n## ')
    .map(section => {
      const sep = section.indexOf('\n')
      const title = sep === -1 ? section : section.slice(0, sep)
      const content = sep === -1 ? '' : section.slice(sep + 1)
      return { title, content: content.trim() }
    })
    .filter(x => x.title && !x.title.startsWith('<'))
}
```

**กลยุทธ์ของ Arona:**

```typescript
// จาก structure.ts
const index = (chunks: Chunk[]) =>
  chunks
    .flatMap(doc =>
      doc.content
        .split('\n## ')  // แบ่งตาม H2 headers
        .map(format)
        .filter(x => notTag(x.title))
        .map(section => ({
          ...doc,
          ...section,
          link: headerToLink(doc.file, section.title),
          sequence: doc.file in fileToSequence
            ? ++fileToSequence[doc.file]
            : (fileToSequence[doc.file] = 0)
        }))
    )
    .filter((x, i, a) => a.findIndex(y => y.link === x.link) === i)
```

**ทำไมแบ่งตามหัวข้อ?**

- รักษาโครงสร้างเอกสาร
- แต่ละ chunk เป็นอิสระ
- ลิงก์โดยตรงไปยัง source sections
- ขอบเขตความหมายตามธรรมชาติ

### แนวทางปฏิบัติการแบ่งส่วน

| แนวทาง | รายละเอียด |
|----------|-------------|
| **ขนาด Chunk** | 500-1500 tokens (เล็กไป = ไม่มี context, ใหญ่ไป = มี noise) |
| **ทับซ้อน** | 10-20% (ช่วยแก้ปัญหาขอบเขต) |
| **Metadata** | ระบุ source, title, position |
| **Summarization** | สร้าง summary สำหรับแต่ละ chunk |
| **Multiple Embeddings** | Embed content + title + filename |

---

## วิธีการค้นหา

### 1. BM25 (Full-Text Search)

**BM25 คืออะไร?**

BM25 คือฟังก์ชันจัดอันดับที่ search engines ใช้เพื่อประเมินความเกี่ยวข้องของเอกสารกับคำค้นหา

**สูตร BM25:**

```
score(D, Q) = Σ IDF(qi) × (f(qi, D) × (k1 + 1)) /
                        (f(qi, D) + k1 × (1 - b + b × |D| / avgdl))

โดยที่:
- D = เอกสาร
- Q = query ที่มีคำ qi
- f(qi, D) = ความถี่ของ qi ใน D
- |D| = ความยาวของ D
- avgdl = ความยาวเอกสารเฉลี่ย
- k1, b = พารามิเตอร์ (โดยทั่วไป k1=1.2, b=0.75)
- IDF = inverse document frequency
```

**คุณสมบัติ BM25:**

- **Term Frequency**: มากขึ้น = คะแนนสูงขึ้น
- **Document Frequency**: คำหายาก = สำคัญกว่า
- **Length Normalization**: เอกสารยาวๆ ไม่ครอง
- **Exact Matching**: ต้องมีคำที่ตรงกัน

**Query BM25 ของ Arona:**

```sql
WITH raw AS (
  SELECT
    file, link, title, sequence, summary, weight,
    paradedb.score(link) AS r  -- BM25 score
  FROM documents d
  WHERE link @@@ 'summary:create route'  -- Full-text search
),
normalized AS (
  SELECT *,
    r / NULLIF(MAX(r) OVER (), 0) AS r_norm  -- Normalize 0-1
  FROM raw
)
SELECT * FROM normalized
WHERE (0.775 * r_norm + 0.275 * weight) >= 0.6  -- Hybrid score
ORDER BY score DESC
LIMIT 10;
```

### 2. Vector Similarity Search

**Cosine Similarity Search**

```sql
-- หาเอกสารที่คล้ายกับ query embedding
SELECT
  title,
  summary,
  1 - (embedding <=> $1) AS similarity  -- Cosine distance
FROM documents
WHERE embedding <=> $1 < 0.3  -- Distance threshold
ORDER BY embedding <=> $1
LIMIT 5;
```

**HNSW Index (Hierarchical Navigable Small World)**

```sql
-- สร้างดัชนีสำหรับค้นหาแบบ approximate เร็วๆ
CREATE INDEX ON documents
USING hnsw (embedding vector_cosine_ops);
```

**คุณสมบัติ HNSW:**

- **Speed**: O(log N) vs O(N) brute force
- **Accuracy**: ~95% ของ exact search
- **Memory**: ใช้ memory มากขึ้น
- **Tunable**: ปรับ speed vs accuracy ได้

### 3. Hybrid Search

**รวม BM25 + Vector**

```typescript
// สูตร hybrid scoring ของ Arona
const bm25Score = normalizedBM25Score
const vectorScore = cosineSimilarity(queryEmbedding, docEmbedding)
const docWeight = documentWeight  // essential > blog

// ผสมถ่วง
const finalScore = 0.775 * bm25Score + 0.225 * vectorScore

// ทำไม 77.5% BM25?
// - Exact keyword matching สำคัญสำหรับ docs
// - "route" ต้อง match "route" ไม่ใช่แค่แนวคิดคล้ายๆ กัน
```

**Hybrid Search Flow:**

```
Query: "How do I create a route?"

BM25 Search:
- "create" ✓
- "route" ✓
Results: [Doc A: 0.9, Doc B: 0.7, Doc C: 0.5]

Vector Search:
- "create route" ~ "endpoint setup" ✓
Results: [Doc B: 0.95, Doc D: 0.8, Doc A: 0.6]

Combined:
Doc A: 0.775×0.9 + 0.225×0.6 = 0.83
Doc B: 0.775×0.7 + 0.225×0.95 = 0.76
Doc C: 0.775×0.5 + 0.225×0.3 = 0.45

Final: [Doc A, Doc B, Doc C]
```

### 4. Parent Document Retrieval

**แนวคิด:**

เก็บ chunks สำหรับค้นหา แต่คืน parent documents สำหรับสร้างคำตอบ

```
Document: "Elysia Routes"
├── Chunk 1: "Creating routes" (ค้นหาตัวนี้)
├── Chunk 2: "Route parameters" (ค้นหาตัวนี้)
└── Chunk 3: "Route handlers" (ค้นหาตัวนี้)

Generation: ใช้ parent document เต็มๆ (context มากขึ้น)
```

**การนำไปใช้:**

```sql
CREATE TABLE documents (
  link VARCHAR(255) PRIMARY KEY,
  file VARCHAR(255),           -- Parent identifier
  title VARCHAR(255),
  content TEXT,                -- Full content for generation
  summary TEXT,                -- Chunk for retrieval
  embedding VECTOR(1536),
  sequence SMALLINT
);

-- คืน adjacent chunks สำหรับ context
SELECT
  c.link,
  c.title,
  string_agg(c.summary, E'\n') AS summary  -- รวม chunks
FROM filtered f
JOIN documents c
  ON c.file = f.file
  AND c.sequence BETWEEN f.sequence AND f.sequence + 1  -- Chunk ถัดไป
GROUP BY c.file, c.title, c.link
ORDER BY score DESC;
```

---

## การสร้างคำตอบ

### Context Building

**1. Simple Concatenation** (ต่อกันแบบง่าย)

```typescript
const context = retrievedChunks
  .map(c => `${c.title}\n${c.summary}`)
  .join('\n\n')

const prompt = `
Context:
${context}

Question: ${query}

Answer:
`
```

**2. Weighted Context** (Context แบบถ่วง)

```typescript
const context = retrievedChunks
  .sort((a, b) => b.score - a.score)
  .map((c, i) => {
    const weight = Math.exp(-i * 0.5)  // ลดลงตามลำดับ
    return `[${(weight * 100).toFixed(0)}% relevant] ${c.summary}`
  })
  .join('\n')
```

**3. Dynamic Context Selection** (เลือก Context แบบไดนามิก)

```typescript
// เลือก chunks ตามประเภทคำถาม
if (isHowToQuery(query)) {
  // เน้น content แบบ step-by-step
  return chunks.filter(c => c.summary.includes('step'))
} else if (isReferenceQuery(query)) {
  // เน้น docs ครอบคลุม
  return chunks.filter(c => c.weight > 0.7)
}
```

### เทคนิคการสร้างคำตอบ

**1. Streaming Response**

```typescript
import { streamText } from 'ai'

const result = await streamText({
  model,
  messages: [{ role: 'user', content: prompt }],
  on_finish: (metadata) => {
    // Log token usage, timing
  }
})

// Stream ไปยัง client
return new Response(
  result.textStream,
  { headers: { 'Content-Type': 'text/event-stream' } }
)
```

**2. Tool Calling (Agent Loop)**

```typescript
// กำหนด tools
const tools = {
  search: tool({
    description: 'ค้นหาเอกสาร',
    inputSchema: z.object({
      sentence: z.string()
    }),
    execute: async ({ sentence }) => {
      return await search(sentence)
    }
  }),
  readPage: tool({
    description: 'อ่านทั้งหน้า',
    inputSchema: z.object({
      link: z.string()
    }),
    execute: async ({ link }) => {
      return await readPage(link)
    }
  })
}

// AI ตัดสินใจเลือก tools
const result = await streamText({
  model,
  tools,
  messages: [{ role: 'user', content: query }],
  maxToolRoundtrips: 8,  // Agent steps สูงสุด
  stopWhen: [
    stepCountIs(8),
    () => references.length > 32
  ]
})
```

**Agent Flow:**

```
User: "How do I create a route and add validation?"

Step 1: AI เรียก search({ sentence: "create route" })
  → Returns: [basic routes doc]

Step 2: AI เรียก search({ sentence: "validation" })
  → Returns: [validation doc]

Step 3: AI เรียก readPage({ link: "essential/validation" })
  → Returns: Full validation content

Step 4: AI สรุปคำตอบจากข้อมูลที่รวบรวม
```

**3. Response Verification** (ตรวจสอบคำตอบ)

```typescript
// เพิ่ม checksum เพื่อป้องกันการปลอมแปลง
import { createHash } from 'crypto'

function addChecksum(content: string, secret: string) {
  const hash = createHash('sha256')
    .update(content)
    .update(secret)
    .digest('hex')

  return `${content}\n\n---Elysia-Metadata---\nchecksum:${hash}`
}

// ตรวจสอบที่ client
function verifyChecksum(content: string, checksum: string, secret: string) {
  const hash = createHash('sha256')
    .update(content)
    .update(secret)
    .digest('hex')

  return timingSafeEqual(
    Buffer.from(hash),
    Buffer.from(checksum)
  )
}
```

---

## เทคนิคขั้นสูง

### 1. Semantic Caching

**แนวคิด:** Cache ตาม semantic similarity ไม่ใช่ exact match

```typescript
class SemanticCache {
  async get(query: string): Promise<string | null> {
    const queryEmbedding = await getEmbedding(query)

    // ค้นหา cached queries ที่คล้ายกัน
    const result = await redis.call(
      'FT.SEARCH',
      'idx:cache',
      '*=>[KNN 1 @embedding $vec AS score]',
      'PARAMS', '2', 'vec',
      Buffer.from(new Float32Array(queryEmbedding).buffer),
      'DIALECT', '2',
      'RETURN', '2', 'score', 'response'
    )

    if (!result) return null

    const [, , [, score, , response]] = result
    const similarity = 1 - parseFloat(score)

    // 90% similarity threshold
    if (similarity >= 0.9) {
      console.log(`Semantic cache hit: ${similarity.toFixed(4)}`)
      return response
    }

    return null
  }

  async set(query: string, response: string) {
    const key = `cache:${cyrb53(query)}`
    await redis.hset(key, {
      prompt: query,
      response: response,
      embedding: await getEmbeddingBuffer(query)
    })
    await redis.expire(key, 14400)  // 4 ชั่วโมง
  }
}
```

**ตัวอย่าง Cache Hit:**

```
Cached Query: "How many vacation days do I get?"

คำถามที่คล้ายกันที่ Hit:
- "What's my vacation allowance?" (92% similar)
- "How many days of annual leave?" (91% similar)
- "Tell me about vacation days" (90% similar)

Misses:
- "How do I request vacation?" (76% similar - intent ต่างกัน)
- "What's the sick leave policy?" (65% similar)
```

### 2. Query Expansion

**Multi-Query Retrieval:**

```typescript
// สร้างหลายแบบของคำถาม
const variations = await generateText({
  model: smallModel,
  prompt: `Generate 3 different ways to ask: "${query}"
  Focus on different aspects and keywords.
  Output as JSON array.`,
  response_format: { type: 'json_object' }
})

const queries = [query, ...JSON.parse(variations.text)]

// ค้นหาสำหรับแต่ละคำถาม
const allResults = await Promise.all(
  queries.map(q => search(q))
)

// ลบ duplicate และจัดอันดับ
const uniqueResults = deduplicateByLink(allResults.flat())
return rankByScore(uniqueResults)
```

### 3. Re-ranking

**ทำต้อง Re-rank?**

Vector similarity ไม่ได้ match กับ relevance สำหรับ query เสมอไป

```typescript
// Initial retrieval (เร็วแต่น้อยกว่า)
const candidates = await vectorSearch(query, { topK: 50 })

// Re-ranking (ช้ากว่าแต่แม่นยำกว่า)
const reranked = await rerank({
  query,
  documents: candidates,
  model: 'cohere-rerank-3'  // หรือ cross-encoder
})

// ทำไม Arona ไม่ใช้:
// - เพิ่ม latency (~100-200ms)
// - เพิ่มค่า API
// - Hybrid search (BM25 + Vector) เพียงพอสำหรับ docs
```

### 4. Metadata Filtering

```typescript
// ค้นหาพร้อม metadata constraints
const results = await search({
  query: "authentication",
  filters: {
    category: 'security',    // เฉพาะ security docs
    weight: { $gte: 0.7 },   // เฉพาะ docs สำคัญ
    updated: { $gt: '2024-01-01' }  // เฉพาะอัปเดตล่าสุด
  }
})
```

### 5. Incremental Indexing

**แนวคิด:** อัปเดตเฉพาะเอกสารที่เปลี่ยน ไม่ต้อง reindex ทั้งหมด

```typescript
const incrementalIndex = async () => {
  // ดึงเอกสารที่มี
  const current = await sql`
    SELECT file, content, link FROM documents
  `

  // ดึงเอกสารใหม่
  const fresh = await loadDocuments()

  // เปรียบเทียบ
  const toUpdate = []
  const toRemove = []

  for (const newDoc of fresh) {
    const existing = current.find(c => c.link === newDoc.link)

    if (!existing) {
      toUpdate.push(newDoc)  // เอกสารใหม่
    } else if (existing.content !== newDoc.content) {
      toUpdate.push(newDoc)  // เอกสารที่เปลี่ยน
    }
  }

  for (const oldDoc of current) {
    if (!fresh.find(f => f.link === oldDoc.link)) {
      toRemove.push(oldDoc)  // เอกสารที่ลบ
    }
  }

  // ประมวลผลเฉพาะที่เปลี่ยน
  await processUpdates(toUpdate)
  await processRemovals(toRemove)
}
```

---

## ข้อควรพิจารณาสำหรับ Production

### 1. การปรับปรุงประสิทธิภาพ

**Latency Budget:**

```
เป้าหมาย: < 2 วินาที end-to-end

แบ่งสัดส่วน:
- Cache hit: ~50ms
- Semantic cache miss: ~200ms
- Search + retrieval: ~300ms
- LLM generation: ~1000ms
```

**เทคนิคการปรับปรุง:**

```typescript
// 1. Parallel independent operations
const [cached, searchResults] = await Promise.all([
  semanticCache.get(query),
  search(query, { abortSignal })
])

// 2. Abort on timeout
const controller = new AbortController()
setTimeout(() => controller.abort(), 5000)

// 3. Early termination
if (abortSignal.aborted) return references

// 4. Streaming สำหรับความรู้สึกเร็ว
return streamText({ model, messages }).textStream
```

### 2. การประหยัดค่าใช้จ่าย

**กลยุทธ์ลดค่าใช้จ่าย:**

```typescript
// 1. Aggressive caching
const cacheStrategy = {
  exact: { ttl: 4 * 60 * 60 * 1000 },      // 4 ชั่วโมง
  semantic: { ttl: 14400 },               // 4 ชั่วโมง
  burst: { ttl: 30 * 1000, max: 250 }     // 30 วินาที
}

// 2. ใช้ model เล็กสำหรับ sub-tasks
const models = {
  main: 'openai/gpt-oss-120b',      // Primary generation
  normalize: 'openai/gpt-oss-20b',   // Query normalization
  summary: 'openai/gpt-oss-20b'     // Summarization
}

// 3. Embedding caching
const embeddingCache = new LRUCache({
  max: 750,  // ~4.4MB memory
  ttl: 9 * 60 * 60 * 1000  // 9 ชั่วโมง
})

// 4. ใช้ summary แทน full content
const chunk = {
  summary: await generateSummary(content),  // 200 tokens
  content: fullContent  // เก็บแต่ไม่ส่งให้ LLM
}
```

**เปรียบเทียบค่าใช้จ่าย (Arona vs Off-the-shelf):**

| Metric | Arona (DIY) | Off-the-shelf |
|--------|-------------|---------------|
| ค่าใช้จ่ายต่อเดือน | ~$21 | $250-1000+ |
| ค่าต่อ Query | ~$0.003 | $0.05-0.20 |
| ค่า Setup | $0 (เวลา dev) | $0 |

### 3. ความปลอดภัย

**Rate Limiting:**

```typescript
// Sliding window rate limit
async function checkRateLimit(ip: string, limit = 10, window = 35) {
  const key = `ratelimit:${ip}`
  const now = Date.now()

  await redis.zadd(key, { score: now, member: now })
  await redis.zremrangebyscore(key, 0, now - window * 1000)

  const count = await redis.zcard(key)

  if (count > limit) {
    throw new Error('Rate limit exceeded')
  }

  await redis.expire(key, window)
}
```

**Proof of Work:**

```typescript
// Client ต้องแก้ปริศนาคำนวณ
function verifyProofOfWork(nonce: string, suffix: number, bits: number) {
  const hash = createHash('sha256')
    .update(`${nonce}:${suffix}`)
    .digest('hex')

  const zeros = '0'.repeat(Math.floor(bits / 4))

  return hash.startsWith(zeros)
}

// 19 bits = 4.75 hex zeros
// ใช้เวลา ~2-5 วินาทีที่ client
// ทำให้ bot attacks แพง
```

**Checksum Verification:**

```typescript
// ป้องกันการปลอมแปลง AI responses
function verifyResponse(content: string, checksum: string) {
  const computed = createHash('sha256')
    .update(content)
    .update(process.env.AI_CHECKSUM_SECRET)
    .digest('hex')

  return timingSafeEqual(
    Buffer.from(computed),
    Buffer.from(checksum)
  )
}
```

### 4. Observability

**OpenTelemetry Tracing:**

```typescript
import { record } from '@elysiajs/opentelemetry'

export async function search(query: string) {
  return record('Search Documents', async (span) => {
    span.setAttribute('query', query)

    const results = await sql.query(/* ... */)

    span.setAttribute('result.count', results.length)

    return results
  })
}
```

**Metrics ที่ต้องติดตาม:**

```typescript
const metrics = {
  cache_hit_rate: 'cache hits / total requests',
  avg_response_time: 'p50, p95, p99 latencies',
  token_usage: 'input/output tokens per request',
  error_rate: 'errors / total requests',
  semantic_similarity: 'avg similarity for cache hits',
  retrieval_quality: 'relevant docs / total retrieved'
}
```

---

## ตัวอย่างการนำไปใช้

### RAG Pipeline แบบสมบูรณ์ (TypeScript)

```typescript
import { embed, generateText, streamText } from 'ai'
import { LRUCache } from 'lru-cache'
import { Redis } from 'ioredis'

// Configuration
const config = {
  embeddingModel: 'text-embedding-3-small',
  mainModel: 'gpt-oss-120b',
  vectorDimensions: 1536,
  cacheTTL: 4 * 60 * 60  // 4 ชั่วโมง
}

// Cache layers
const embeddingCache = new LRUCache<string, number[]>({
  max: 750,
  ttl: 9 * 60 * 60 * 1000
})

const redis = new Redis(process.env.REDIS_URL)

// 1. EMBEDDING
async function getEmbedding(text: string): Promise<number[]> {
  if (embeddingCache.has(text)) {
    return embeddingCache.get(text)!
  }

  const { embedding } = await embed({
    model: openai.embeddingModel(config.embeddingModel),
    value: text
  })

  embeddingCache.set(text, embedding)
  return embedding
}

// 2. RETRIEVAL
async function search(query: string, topK = 5) {
  const queryEmbedding = await getEmbedding(query)

  // Hybrid search: BM25 + Vector
  const results = await sql`
    WITH vector_search AS (
      SELECT DISTINCT ON (file)
        link, title, summary,
        (0.1 * title_embedding <#> ${queryEmbedding} +
         0.675 * embedding <#> ${queryEmbedding} +
         0.125 * weight * -1) AS score
      FROM documents
      ORDER BY file, score DESC
      LIMIT ${topK * 2}
    )
    SELECT * FROM vector_search
    WHERE score >= 0.4
    ORDER BY score DESC
    LIMIT ${topK}
  `

  return results
}

// 3. SEMANTIC CACHE
async function semanticCache(query: string): Promise<string | null> {
  const embedding = await getEmbedding(query)

  const result = await redis.call(
    'FT.SEARCH', 'idx:cache',
    '*=>[KNN 1 @embedding $vec AS score]',
    'PARAMS', '2', 'vec',
    Buffer.from(new Float32Array(embedding).buffer),
    'RETURN', '2', 'score', 'response'
  )

  if (!result || result.length < 3) return null

  const [, , [, score, , response]] = result
  const similarity = 1 - parseFloat(score)

  return similarity >= 0.9 ? response : null
}

// 4. GENERATION
async function ask(query: string, history = []) {
  // Check cache
  const cached = await semanticCache(query)
  if (cached) return cached

  // Retrieve context
  const context = await search(query)

  // Build prompt
  const prompt = `
    ตอบตาม context นี้:
    ${context.map(c => c.summary).join('\n\n')}

    คำถาม: ${query}
  `

  // Generate response
  const { text } = await generateText({
    model: openai(config.mainModel),
    prompt
  })

  // Cache response
  await redis.hset(`cache:${hash(query)}`, {
    prompt: query,
    response: text,
    embedding: await getEmbedding(query)
  })

  return text
}
```

### Document Indexing Pipeline

```typescript
async function indexDocuments() {
  // 1. Load documents
  const files = await glob('docs/**/*.md')
  const docs = await Promise.all(
    files.map(async (file) => ({
      path: file,
      content: await readFile(file, 'utf-8')
    }))
  )

  // 2. Chunk documents
  const chunks = docs.flatMap(doc =>
    doc.content.split('\n## ').map((section, i) => ({
      link: generateLink(doc.path, i),
      title: extractTitle(section),
      content: section,
      file: doc.path
    }))
  )

  // 3. Generate embeddings
  const { embeddings } = await embedMany({
    model: openai.embeddingModel('text-embedding-3-small'),
    values: chunks.map(c => c.content)
  })

  // 4. Generate summaries
  const summaries = await Promise.all(
    chunks.map(chunk =>
      generateSummary(chunk.content)
    )
  )

  // 5. Store in database
  await sql`
    INSERT INTO documents (link, title, content, summary, embedding)
    SELECT *
    FROM unnest(
      ${chunks.map(c => c.link)},
      ${chunks.map(c => c.title)},
      ${chunks.map(c => c.content)},
      ${summaries},
      ${embeddings}
    )
    ON CONFLICT (link) DO UPDATE SET
      content = EXCLUDED.content,
      embedding = EXCLUDED.embedding,
      summary = EXCLUDED.summary
  `
}
```

---

## สรุป

### ข้อควรจำ

1. **RAG รวม retrieval + generation** สำหรับคำตอบที่ถูกต้องและมี context
2. **Embeddings** แทะความหมายใน vector space
3. **Vector databases** ทำให้ค้นหาความคล้ายได้เร็ว
4. **Hybrid search** (BM25 + Vector) ให้ผลลัพธ์ดีที่สุดสำหรับ docs
5. **Semantic caching** ลดค่าใช้จ่ายและ latency ได้อย่างมีนัยสำคัญ
6. **Tool calling** ให้ AI ตัดสินใจเองว่าต้องการข้อมูลอะไร
7. **Incremental indexing** ทำให้ข้อมูลเป็นปัจจุบันได้อย่างมีประสิทธิภาพ
8. **Security layers** ป้องกันการใช้งานในทางที่ผิด

### Technology Stack สำหรับ RAG

```
Core Components:
- Embedding Model: OpenAI text-embedding-3-small
- LLM: GPT-OSS-120B หรือคล้ายกัน
- Vector DB: ParadeDB (PostgreSQL + pgvector)
- Cache: DragonflyDB (Redis-compatible)
- Framework: Elysia / Express / Fastify

Optional Enhancements:
- Re-ranking: Cohere Rerank
- Query Expansion: Multi-query retrieval
- Parent Documents: Hierarchical chunking
- Metadata Filtering: Category/date filtering
```

### เมื่อไรควรใช้ RAG

```
✅ ใช้ RAG เมื่อ:
- ต้องการเข้าถึงข้อมูลล่าสุด
- มีความรู้เฉพาะด้าน
- ต้องการอ้างอิงแหล่งที่มา
- อยากลดการโกหก
- มีเอกสารจำนวนมาก

❌ ไม่ควรใช้ RAG เมื่อ:
- Q&A ง่ายๆ พอ
- ความรู้ทั่วไปพอ
- ไม่ต้องการข้อมูล real-time
- ค่าใช้จ่ายสำคัญกว่าความแม่นยำ
```

---

**อ่านเพิ่มเติม:**

- [LangChain RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/)
- [LlamaIndex Documentation](https://docs.llamaindex.ai/)
- [ParadeDB Documentation](https://docs.paradedb.com/)
- [OpenAI Embeddings Guide](https://platform.openai.com/docs/guides/embeddings)
