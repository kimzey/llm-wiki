---
title: "RAG ครอบคลุม: ตั้งแต่พื้นฐานไปสู่ขั้นสูง"
type: source
source_file: raw/notes/arona/RAG_COMPREHENSIVE_GUIDE.md
tags: [rag, comprehensive-guide, embeddings, vector-database]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุป

คู่มือ RAG (Retrieval-Augmented Generation) อย่างครอบคลุมตั้งแต่พื้นฐานไปสู่เทคนิคขั้นสูง ครอบคลุม embeddings, vector databases, document chunking, search methods, generation techniques และ production considerations

## ประเด็นสำคัญ

### RAG คืออะไร?

**RAG (Retrieval-Augmented Generation)** คือสถาปัตยกรรม AI ที่เพิ่มความสามารถให้ Large Language Models (LLMs) โดยนำข้อมูลที่เกี่ยวข้องและเฉพาะเจาะจงมาให้ก่อนที่ AI จะตอบคำถาม

```
LLM แบบดั้งเดิม:
คำถาม → LLM → คำตอบ (จำกัดด้วยข้อมูลที่เคยเรียนมา)

LLM แบบ RAG:
คำถาม → ค้นหาเอกสาร → Context → LLM → คำตอบ (เข้าถึงข้อมูลล่าสุด)
```

### ทำไม RAG สำคัญ

| ปัญหา | ไม่มี RAG | มี RAG |
|---------|-------------|----------|
| ข้อมูลล้าสมัย | LLM รู้เฉพาะข้อมูลถึงวันที่เรียน | เข้าถึงข้อมูลล่าสุด |
| โกหก | LLM อาจสร้างข้อมูลเท็จ | อ้างอิงจากข้อเท็จจริง |
| ความเฉพาะทาง | รู้เฉพาะความรู้ทั่วไป | มีความรู้เฉพาะด้าน |
| การอ้างอิง | ไม่สามารถระบุแหล่งที่มา | อ้างอิงแหล่งที่มาได้ |

### Vector Embeddings

**Embeddings** คือการแปลงข้อความให้เป็นตัวเลขที่แสดงความหมาย
- **Dimensionality**: 1536 สำหรับ OpenAI text-embedding-3-small
- **Semantic Similarity**: ความหมายคล้ายกัน = เวคเตอร์ใกล้กัน
- **Language Independent**: แนวคิดคล้ายกันข้ามภาษา

**Embedding Models ที่นิยม:**
| Model | มิติ | ราคา | ประสิทธิภาพ |
|-------|------------|------|-------------|
| OpenAI text-embedding-3-small | 1536 | ต่ำ | ดีมาก |
| OpenAI text-embedding-3-large | 3072 | ปานกลาง | ดีที่สุด |
| Cohere embed-v3 | 1024 | ต่ำ | ดีมาก |

### Similarity Metrics

**Cosine Similarity** (ใช้บ่อยที่สุด):
```
cosine_similarity(A, B) = (A · B) / (||A|| × ||B||)
ช่วง: -1 ถึง 1
- 1.0 = ทิศทางเดียวกัน (คล้ายที่สุด)
- 0.0 = ตั้งฉาก (ไม่เกี่ยวข้อง)
- -1.0 = ทิศทางตรงข้าม (ความหมายตรงกันข้าม)
```

### Document Chunking Strategies

**1. Fixed-Size Chunking** (แบ่งตามขนาด)
```typescript
const chunkSize = 1000  // ตัวอักษร
const overlap = 200     // ตัวอักษรทับซ้อน
```

**2. Semantic Chunking** (แบ่งตามความหมาย)
```typescript
const chunks = text.split(/\n\n|\. |\? |\! /)
```

**3. Hierarchical Chunking** (แบ่งแบบลำดับชั้น)
```
Document
├── Section 1
│   ├── 1.1
│   └── 1.2
└── Section 2
```

### Search Methods

#### 1. BM25 (Full-Text Search)
ฟังก์ชันจัดอันดับที่ search engines ใช้:
```sql
WHERE link @@@ 'summary:create route'
score = 0.775 * r_norm + 0.275 * weight
```

#### 2. Vector Similarity Search
```sql
SELECT 1 - (embedding <=> $1) AS similarity
FROM documents
WHERE embedding <=> $1 < 0.3
```

#### 3. Hybrid Search
รวม BM25 + Vector:
```typescript
const finalScore = 0.775 * bm25Score + 0.225 * vectorScore
```

### Advanced Techniques

#### 1. Semantic Caching
Cache ตาม semantic similarity (ไม่ใช่ exact match):
```typescript
if (similarity >= 0.9) {
  return cachedResponse  // Cache hit
}
```

#### 2. Query Expansion
สร้างหลายแบบของคำถาม:
```typescript
const variations = await generateText({
  prompt: `Generate 3 different ways to ask: "${query}"`
})
```

#### 3. Re-ranking
```typescript
const candidates = await vectorSearch(query, { topK: 50 })
const reranked = await rerank({ query, documents: candidates })
```

#### 4. Metadata Filtering
```typescript
const results = await search({
  query: "authentication",
  filters: {
    category: 'security',
    weight: { $gte: 0.7 }
  }
})
```

### Production Considerations

**Latency Budget:**
- Cache hit: ~50ms
- Semantic cache miss: ~200ms
- Search + retrieval: ~300ms
- LLM generation: ~1000ms

**Cost Reduction Strategies:**
1. Aggressive caching
2. Small models for sub-tasks
3. Embedding caching
4. Use summary instead of full content

**Security:**
- Rate limiting (sliding window)
- Proof of Work
- Checksum verification

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/rag]] - RAG Architecture
- [[wiki/concepts/embeddings]] - Vector Embeddings
- [[wiki/concepts/hybrid-search]] - BM25 + Vector Search
- [[wiki/concepts/semantic-caching]] - Semantic Similarity Cache
