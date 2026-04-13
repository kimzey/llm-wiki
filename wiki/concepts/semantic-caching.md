---
title: "Semantic Caching"
type: concept
tags: [caching, semantic, vector-search, redis, cost-optimization, rag]
sources: [arona/README.md, arona/RAG_COMPARISON.md, arona/ARONA_GUIDE.md, rag-decision-guide.md, llamaindex-full-guide.md]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/hybrid-search-bm25-vector.md, wiki/concepts/llamaindex-framework.md, wiki/concepts/langchain-framework.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Semantic Caching คือการ cache คำตอบโดยใช้ vector similarity แทน exact string match ถ้าคำถามใหม่มีความหมายใกล้เคียงกับคำถามที่เคยตอบแล้ว (similarity ≥ 90%) ให้ส่งคำตอบเก่ากลับไปเลย โดยไม่ต้องเรียก LLM ใหม่

## อธิบาย

Exact match cache ธรรมดาจะ miss ถ้าผู้ใช้ถามด้วยถ้อยคำต่างกัน:
- "ลาพักร้อนกี่วัน" ≠ "สิทธิ์ลาพักผ่อนประจำปีกี่วัน" (ในสายตา string comparison)
- แต่สองคำถามนี้ต้องการ "คำตอบเดียวกัน"

Semantic Cache แก้ปัญหาด้วยการ:
1. แปลงคำถามเป็น embedding
2. เปรียบเทียบ cosine similarity กับ embeddings ของคำถามที่เคยถามก่อน
3. ถ้า similarity ≥ threshold → ส่งคำตอบเก่ากลับไป

### Implementations

**Arona (TypeScript + DragonflyDB):**
```typescript
// เก็บคำถาม + คำตอบ + embedding
await redis.hset(key, {
  prompt: normalizedQuestion,    // คำถามที่ normalize แล้ว
  response: aiResponse,          // คำตอบ
  embedding: questionEmbedding   // vector ของคำถาม
})

// ค้นหาด้วย KNN (K-Nearest Neighbors)
const result = await redis.call(
  'FT.SEARCH', 'idx:cache',
  '*=>[KNN 1 @embedding $vec AS score]',
  'PARAMS', '2', 'vec', embeddingBuffer
)

// ตรวจสอบ threshold
const similarity = 1 - parseFloat(result[2][0][1])
return similarity >= 0.9 ? cachedResponse : null     // 90% threshold
```

**LangChain (Python + Redis/Dragonfly):**
```python
from langchain_community.cache import RedisSemanticCache

langchain.llm_cache = RedisSemanticCache(
    redis_url="redis://dragonfly:6379",
    embedding=OpenAIEmbeddings(),
    score_threshold=0.95
)
```

**LlamaIndex (Python + RedisKVStore):**
```python
from llama_index.storage.kvstore.redis import RedisKVStore

cache = IngestionCache(
    cache=RedisKVStore.from_host_and_port("dragonfly", 6379),
    collection="ingestion_cache"
)
```

### Query Normalization (ก่อน Cache Lookup)

Arona ใช้ small model (GPT OSS 20B) normalize คำถามก่อน:
- ลบ filler words ("ช่วยบอกหน่อยนะครับ" → ตัดออก)
- Canonicalize intent ("สร้าง route ยังไง" = "วิธีสร้าง route")
- ไม่เกิน 12 คำ

วิธีนี้ช่วยให้ semantic cache hit rate สูงขึ้นด้วย

## ประเด็นสำคัญ

### Multi-Layer Cache ใน Arona

| Layer | Storage | TTL | วัตถุประสงค์ |
|-------|---------|-----|-------------|
| L1: LRU In-Memory | RAM | 30s–9hr | Burst traffic, exact/near-exact |
| L2: Semantic Cache | DragonflyDB (Redis Vector) | 4hr | Semantic similarity ≥ 90% |
| L3: Persistent Redis | DragonflyDB | 3hr–10hr | Exact match persistent |

**Order ของการเช็ค:** L1 → L2 → L3 → AI call

### ประสิทธิผล

- Cache hit rate ของ Arona: 85%+ (รวมทุก layer)
- LangChain ทั่วไป: ~70%
- Semantic cache ช่วย hit คำถามที่ "คล้ายกันแต่ต่างคำ"

### ข้อดี-ข้อเสีย

**ข้อดี:**
- ลดค่าใช้จ่าย LLM อย่างมีนัย
- ลด latency (cache hit ~10ms vs cache miss ~1300ms)
- ไม่ต้องการ exact match

**ข้อเสีย:**
- ต้องการ embedding generation ทุกคำถาม (cost เล็กน้อย)
- TTL ที่ดีต้องหาเอง (ข้อมูลเปลี่ยนแค่ไหน?)
- False positive อาจเกิดถ้า threshold ต่ำเกินไป

### DragonflyDB (Redis-compatible + Vector)

Arona ใช้ DragonflyDB แทน Redis ปกติเพราะ:
- Redis-compatible API ใช้ code เดิมได้
- รองรับ Vector Search (FT.SEARCH + KNN)
- ประสิทธิภาพสูงกว่า Redis ในบาง workload

## ตัวอย่าง

```
คำถาม 1: "ลาพักร้อนกี่วัน?" → AI ตอบ, เก็บ cache
คำถาม 2: "ลาพักผ่อนประจำปีกี่วัน?" → similarity 0.94 ≥ 0.9 → cache hit!
คำถาม 3: "สิทธิ์การลาของพนักงาน" → similarity 0.91 ≥ 0.9 → cache hit!
คำถาม 4: "เงินเดือนได้เมื่อไหร่?" → similarity 0.32 < 0.9 → cache miss → AI call
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — ใช้ semantic cache เป็น optimization layer
- [[wiki/concepts/hybrid-search-bm25-vector|Hybrid Search]] — vector similarity ใช้หลักการเดียวกัน
- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]] — RedisKVStore สำหรับ IngestionCache
- [[wiki/concepts/langchain-framework|LangChain Framework]] — RedisSemanticCache built-in

## แหล่งที่มา

- [[wiki/sources/arona-overview|Arona System Overview]]
- [[wiki/sources/arona-rag-techniques|RAG Techniques & Comparison]]
- [[wiki/sources/rag-decision-guide|RAG Framework Decision Guide — Sellsuki]]
- [[wiki/sources/llamaindex-full-guide|LlamaIndex Full Guide — Sellsuki RAG System]]

## จาก Sellsuki RAG Agent Plan v2

### Semantic Cache เป็น Top-1 Cost Optimization

ประหยัด 40-60% ของ LLM cost สำหรับ FAQ-style queries

**Threshold แนะนำ: cosine > 0.95** (แม่นกว่า 0.9 ลด false positive)

**Implementation Pattern:**

```python
async def check_semantic_cache(query, threshold=0.95):
    query_embedding = await get_embedding_cached(query)
    # Search ใน Dragonfly sorted set ที่เก็บ cached embeddings
    # Return cached answer ถ้า similarity > threshold
```

**สิ่งที่ต้องระวัง:**
- ต้องมีปุ่ม "ข้าม cache" ให้ user กรณีคำตอบไม่ตรง
- ตั้ง TTL ให้เหมาะสม (ข้อมูลเปลี่ยนบ่อย → TTL สั้น)

### Multi-layer Cache Strategy (Plan v2)

1. **Semantic Cache**: cache คำตอบ (ลด LLM calls)
2. **Embedding Cache**: cache embedding vectors (ลด embedding API calls)
3. **DB Query Cache**: cache ผล vector search (ลด DB latency)
4. **Summarized Content**: ลด token count ตอน generate (ลด cost per call)

ใช้ Dragonfly (Redis-compatible) เป็น cache layer — 25x เร็วกว่า Redis

- [[wiki/sources/sellsuki-agent-plan-v2|Sellsuki RAG Agent Plan v2]]
