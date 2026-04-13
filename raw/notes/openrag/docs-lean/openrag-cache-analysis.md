# OpenRAG — วิเคราะห์ระบบ Cache (ฉบับละเอียด)

> **คำตอบสั้น:** OpenRAG มี cache บางส่วนอยู่แล้ว แต่เป็นแค่ **internal utility cache** —
> ไม่มี cache สำหรับ Query, Embedding, หรือ LLM Response เลย ซึ่งเป็นจุดที่กิน cost และเวลามากที่สุด

---

## สารบัญ

- [Cache ที่มีอยู่แล้ว (Built-in)](#cache-ที่มีอยู่แล้ว-built-in)
- [Cache ที่ขาดไป (Missing)](#cache-ที่ขาดไป-missing)
- [ผลกระทบถ้าไม่มี Cache](#ผลกระทบถ้าไม่มี-cache)
- [แผนการทำ Cache เพิ่มเอง](#แผนการทำ-cache-เพิ่มเอง)
- [ตัวอย่างโค้ด Implementation](#ตัวอย่างโค้ด-implementation)

---

## Cache ที่มีอยู่แล้ว (Built-in)

### 1. Conversation Memory Cache — `agent.py`

**ประเภท:** In-memory Python dict
**ไฟล์:** `src/agent.py` (line 11)

```python
# In-memory storage for active conversation threads
active_conversations = {}
# structure: { user_id: { response_id: conversation_state } }
```

**สิ่งที่ cache:** Full conversation thread (messages + function call history)
**TTL:** ไม่มี — อยู่จนกว่า process จะ restart
**Persist:** บันทึก metadata ลงไฟล์ `data/conversations.json` อัตโนมัติ
**ข้อจำกัด:** ไม่มี eviction policy → memory รั่วได้ถ้า conversation เยอะมาก

---

### 2. OpenSearch Client Cache — `session_manager.py`

**ประเภท:** In-memory dict
**ไฟล์:** `src/session_manager.py` (line 253)

```python
# Check if we have a cached client for this user
if user_id not in self.user_opensearch_clients:
    # สร้าง client ใหม่พร้อม JWT token
    ...
```

**สิ่งที่ cache:** OpenSearch client instance ต่อ user
**TTL:** ไม่มี
**ประโยชน์:** ไม่ต้อง init connection ใหม่ทุก request

---

### 3. Flow File Path Cache — `flows_service.py`

**ประเภท:** In-memory dict
**ไฟล์:** `src/services/flows_service.py` (line 26)

```python
# Cache for flow file mappings to avoid repeated filesystem scans
self._flow_file_cache = {}
```

**สิ่งที่ cache:** path ของ Langflow flow file จาก flow_id
**TTL:** ไม่มี แต่ invalidate เมื่อไฟล์ถูกลบ
**ประโยชน์:** ไม่ต้อง scan filesystem ซ้ำทุกครั้ง

---

### 4. Connector Instance Cache — `connection_manager.py`

**ประเภท:** In-memory dict
**ไฟล์:** `src/connectors/connection_manager.py` (line 301)

```python
# Return cached connector if available
if connection_id in self.active_connectors:
    connector = self.active_connectors[connection_id]
    if connector.is_authenticated:
        return connector  # ✓ ใช้ cached
    else:
        del self.active_connectors[connection_id]  # ✗ ลบถ้า expired
```

**สิ่งที่ cache:** Google Drive / OneDrive / SharePoint connector instances
**Invalidation:** ลบออกเมื่อ token หมดอายุ

---

### 5. Config Cache — `config_manager.py`

**ประเภท:** In-memory object
**ไฟล์:** `src/config/config_manager.py` (line 307)

```python
# Update cached config to reflect the edited flags
self._config = config
```

**สิ่งที่ cache:** parsed config.yaml (เก็บใน `_config` attribute)
**ประโยชน์:** ไม่ต้องอ่านไฟล์ YAML ทุก request

---

### 6. Anonymous JWT Cache — `session_manager.py`

```python
# anonymous JWT (cached)
if not hasattr(self, "_anonymous_jwt"):
    self._anonymous_jwt = self._create_anonymous_jwt()
return self._anonymous_jwt
```

**สิ่งที่ cache:** JWT token สำหรับ anonymous user (no-auth mode)

---

### 7. OAuth Token Cache — SharePoint / OneDrive

**ประเภท:** MSAL SerializableTokenCache (disk-persisted)
**ไฟล์:** `src/connectors/sharepoint/oauth.py`, `src/connectors/onedrive/oauth.py`

```python
self.token_cache = msal.SerializableTokenCache()
# persist ไปที่ไฟล์ token file (MSAL format)
await self.save_cache()
```

**สิ่งที่ cache:** OAuth access tokens + refresh tokens
**ประโยชน์:** ไม่ต้อง re-authenticate ทุกครั้ง

---

## สรุป Cache ที่มี vs ไม่มี

| Layer | Cache มีอยู่? | ประเภท | TTL | Notes |
|-------|-------------|--------|-----|-------|
| Conversation history | ✅ | Memory + Disk | ไม่มี | metadata เท่านั้น |
| OpenSearch client | ✅ | Memory dict | ไม่มี | per-user |
| Flow file paths | ✅ | Memory dict | ไม่มี | utility only |
| Connector instances | ✅ | Memory dict | token expiry | |
| Config YAML | ✅ | Memory object | on save | |
| OAuth tokens | ✅ | MSAL file | token expiry | |
| **Search query results** | ❌ | — | — | **ขาด** |
| **Embedding vectors** | ❌ | — | — | **ขาด** |
| **LLM responses** | ❌ | — | — | **ขาด** |
| **Docling parse output** | ❌ | — | — | **ขาด** |

---

## Cache ที่ขาดไป (Missing)

### 1. Search/Query Result Cache ❌

**สถานการณ์:** User 100 คนถามคำถามซ้ำกัน

```
คำถาม: "นโยบายลาพักร้อนคือกี่วัน?"

User A ถาม → embed query → KNN search OpenSearch → LLM → ตอบ
User B ถามเหมือนกัน → embed query → KNN search OpenSearch → LLM → ตอบ  ← ซ้ำ!
User C ถามเหมือนกัน → embed query → KNN search OpenSearch → LLM → ตอบ  ← ซ้ำ!
```

**ผลกระทบ:** เรียก embedding API + LLM API ซ้ำฟรี ทุกครั้ง

---

### 2. Embedding Cache ❌

ทุกครั้งที่มี query → ส่งไป OpenAI Embeddings API ทุกครั้ง:

```python
# ใน agent.py ทุก request
resp = await clients.patched_embedding_client.embeddings.create(
    model=embedding_model,
    input=query_text  # ← ไม่มี cache ก่อนส่ง
)
```

**ผลกระทบ:** ต้นทุน API calls + latency เพิ่ม ~100-300ms ต่อ query

---

### 3. LLM Response Cache ❌

ไม่มี **Semantic Cache** หรือ **Exact Match Cache** สำหรับ LLM:

```
"ลาป่วยได้กี่วัน" → เรียก LLM ทุกครั้ง
"ลาป่วยได้กี่วัน?" → เรียก LLM อีกครั้ง (ต่างแค่ ?)
"สามารถลาป่วยได้กี่วัน" → เรียก LLM อีกครั้ง (ความหมายเหมือนกัน)
```

---

### 4. Docling Parse Cache ❌

เอกสารที่ผ่าน Docling แล้วไม่มีการ cache output:

```
Document.pdf → Docling (30 วินาที) → chunks → OpenSearch  ✓ (hash check ป้องกัน re-index)

แต่ถ้า delete แล้ว re-upload:
Document.pdf → Docling อีกครั้ง (30 วินาที)  ← ไม่มี cache ผลลัพธ์
```

---

## ผลกระทบถ้าไม่มี Cache

### ต้นทุน API (ตัวอย่าง)

```
สมมติ: 50 users ถาม 20 คำถาม/วัน = 1,000 queries/วัน
คำถาม unique จริง = 200 คำถาม (ซ้ำ 80%)

แบบไม่มี cache:
  - Embedding calls: 1,000 ครั้ง × $0.0001 = $0.10/วัน
  - LLM calls: 1,000 ครั้ง × $0.01 = $10/วัน
  - รวม: ~$10.10/วัน = ~$3,600/ปี

แบบมี cache (hit rate 80%):
  - Embedding calls: 200 ครั้ง × $0.0001 = $0.02/วัน
  - LLM calls: 200 ครั้ง × $0.01 = $2/วัน
  - รวม: ~$2.02/วัน = ~$737/ปี
  - ประหยัด: ~80% = ~$2,863/ปี
```

### Latency

```
แบบไม่มี cache (ต่อ query):
  Embed query:    100-300ms
  KNN Search:     50-200ms
  LLM call:       1,000-3,000ms
  รวม:            ~1.5-3.5 วินาที

แบบมี cache (cache hit):
  Cache lookup:   1-5ms
  รวม:            ~5ms  ← เร็วกว่า 300-700x
```

---

## แผนการทำ Cache เพิ่มเอง

### ระดับที่ทำได้ (เรียงจากง่ายไปยาก)

```
Level 1: In-Memory Cache (วันเดียวทำได้)
  → ใช้ Python dict + hashlib
  → ไม่ต้องติดตั้งอะไรเพิ่ม
  → หาย เมื่อ restart

Level 2: Redis Cache (1-2 วัน)
  → เพิ่ม Redis container ใน docker-compose
  → TTL, persist, shared across workers
  → แนะนำสำหรับ production

Level 3: Semantic Cache (1 สัปดาห์)
  → cache โดยใช้ vector similarity ของ query
  → คำถามใกล้เคียงกัน → ใช้ answer เดิม
  → ต้องการ embedding + vector store สำหรับ cache keys
```

---

## ตัวอย่างโค้ด Implementation

### Option A: Simple In-Memory Cache (ง่ายที่สุด)

สร้างไฟล์ `src/utils/query_cache.py`:

```python
import hashlib
import time
from typing import Optional, Any

class QueryCache:
    """Simple in-memory cache with TTL for query results."""

    def __init__(self, ttl_seconds: int = 3600, max_size: int = 1000):
        self._cache: dict[str, dict] = {}
        self.ttl = ttl_seconds
        self.max_size = max_size

    def _make_key(self, query: str, user_id: str, filter_id: str = None) -> str:
        """สร้าง cache key จาก query + context"""
        raw = f"{query.lower().strip()}:{user_id}:{filter_id or ''}"
        return hashlib.sha256(raw.encode()).hexdigest()

    def get(self, query: str, user_id: str, filter_id: str = None) -> Optional[Any]:
        key = self._make_key(query, user_id, filter_id)
        entry = self._cache.get(key)
        if entry and time.time() - entry["ts"] < self.ttl:
            return entry["value"]
        return None

    def set(self, query: str, user_id: str, value: Any, filter_id: str = None):
        if len(self._cache) >= self.max_size:
            # ลบ entry เก่าที่สุด (FIFO)
            oldest = min(self._cache, key=lambda k: self._cache[k]["ts"])
            del self._cache[oldest]
        key = self._make_key(query, user_id, filter_id)
        self._cache[key] = {"value": value, "ts": time.time()}

# Global instance
query_cache = QueryCache(ttl_seconds=3600, max_size=500)
```

ใช้งานใน `src/services/chat_service.py`:

```python
from utils.query_cache import query_cache

async def chat(self, prompt, user_id, filter_id=None, ...):
    # ลองดู cache ก่อน
    cached = query_cache.get(prompt, user_id, filter_id)
    if cached:
        return cached  # ← return ทันที ไม่เรียก LLM

    # ไม่มีใน cache → เรียก LLM ปกติ
    response = await async_chat(...)

    # เก็บลง cache
    query_cache.set(prompt, user_id, response, filter_id)
    return response
```

---

### Option B: Redis Cache (แนะนำสำหรับ Production)

**Step 1:** เพิ่ม Redis ใน `docker-compose.yml`:

```yaml
services:
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes --maxmemory 512mb --maxmemory-policy allkeys-lru

volumes:
  redis-data:
```

**Step 2:** สร้าง `src/utils/redis_cache.py`:

```python
import hashlib
import json
import redis.asyncio as aioredis
from typing import Optional, Any

class RedisCache:
    def __init__(self, url: str = "redis://redis:6379", ttl: int = 3600):
        self.redis = aioredis.from_url(url, decode_responses=True)
        self.ttl = ttl

    def _key(self, namespace: str, *parts) -> str:
        raw = ":".join(str(p) for p in parts)
        return f"openrag:{namespace}:{hashlib.md5(raw.encode()).hexdigest()}"

    async def get_query(self, query: str, user_id: str, filter_id: str = None) -> Optional[dict]:
        key = self._key("query", query.lower().strip(), user_id, filter_id or "")
        value = await self.redis.get(key)
        return json.loads(value) if value else None

    async def set_query(self, query: str, user_id: str, result: dict, filter_id: str = None):
        key = self._key("query", query.lower().strip(), user_id, filter_id or "")
        await self.redis.setex(key, self.ttl, json.dumps(result, ensure_ascii=False))

    async def get_embedding(self, text: str) -> Optional[list]:
        key = self._key("embed", text)
        value = await self.redis.get(key)
        return json.loads(value) if value else None

    async def set_embedding(self, text: str, vector: list, ttl: int = 86400):
        key = self._key("embed", text)
        await self.redis.setex(key, ttl, json.dumps(vector))

    async def invalidate_user(self, user_id: str):
        """ลบ cache ทั้งหมดของ user (เช่น เมื่อ document ถูก update)"""
        pattern = f"openrag:query:*{user_id}*"
        keys = await self.redis.keys(pattern)
        if keys:
            await self.redis.delete(*keys)

cache = RedisCache()
```

**Step 3:** Embed cache ใน `src/models/processors.py`:

```python
from utils.redis_cache import cache

# แทนที่การเรียก embedding โดยตรง
async def get_embedding_with_cache(text: str, model: str) -> list:
    # ลองดูจาก cache ก่อน
    cached = await cache.get_embedding(f"{model}:{text}")
    if cached:
        return cached

    # ไม่มีใน cache → เรียก API
    resp = await clients.patched_embedding_client.embeddings.create(
        model=model, input=[text]
    )
    vector = resp.data[0].embedding

    # เก็บลง cache 24 ชั่วโมง
    await cache.set_embedding(f"{model}:{text}", vector, ttl=86400)
    return vector
```

---

### Option C: Semantic Cache (คำถามใกล้เคียงกัน → ใช้ answer เดิม)

แนวคิด: แทนที่จะ exact match → ใช้ cosine similarity ของ query embedding

```python
import numpy as np

class SemanticCache:
    """Cache ที่ match คำถามที่มีความหมายใกล้เคียงกัน"""

    def __init__(self, similarity_threshold: float = 0.95):
        self.threshold = similarity_threshold
        # เก็บ: [(query_embedding, answer, metadata), ...]
        self._entries: list[dict] = []

    def _cosine_similarity(self, a: list, b: list) -> float:
        a, b = np.array(a), np.array(b)
        return float(np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b)))

    def find_similar(self, query_embedding: list, user_id: str) -> Optional[str]:
        for entry in self._entries:
            if entry["user_id"] != user_id:
                continue
            sim = self._cosine_similarity(query_embedding, entry["embedding"])
            if sim >= self.threshold:
                return entry["answer"]  # คำถามใกล้เคียงพอ → ใช้ answer เดิม
        return None

    def store(self, query_embedding: list, answer: str, user_id: str):
        self._entries.append({
            "embedding": query_embedding,
            "answer": answer,
            "user_id": user_id,
            "ts": time.time()
        })
```

ตัวอย่างที่ semantic cache จับได้:

```
คำถาม 1: "ลาพักร้อนได้กี่วัน"          → เรียก LLM → เก็บ cache
คำถาม 2: "พักร้อนประจำปีมีกี่วัน"       → similarity 0.97 → ใช้ cache ✓
คำถาม 3: "annual leave policy คือ?"    → similarity 0.92 → ใช้ cache ✓
คำถาม 4: "ลาป่วยได้กี่วัน"              → similarity 0.71 → เรียก LLM ใหม่ ✗
```

---

## แผนการ Implement แนะนำสำหรับองค์กร

### Phase 1: Quick Win (ทำได้ใน 1-2 วัน)

```
1. เพิ่ม Redis ใน docker-compose.yml
2. Cache embedding vectors (24h TTL)
   → ลด embedding API calls ได้ทันที
3. Cache LLM responses สำหรับ exact match (1h TTL)
   → คำถามซ้ำ → ตอบทันที
```

### Phase 2: Smart Cache (1 สัปดาห์)

```
4. Semantic cache สำหรับ similar queries
5. Cache invalidation เมื่อ document เปลี่ยน
   → เมื่อ upload เอกสารใหม่ → flush cache ที่เกี่ยวข้อง
```

### Phase 3: Advanced (1 เดือน)

```
6. Per-knowledge-filter cache partition
7. Cache warming (pre-compute คำถามที่พบบ่อย)
8. Cache hit/miss metrics dashboard
```

---

## Cache Invalidation Strategy

```
เมื่อ Upload เอกสารใหม่ → invalidate cache ทั้งหมดของ user นั้น
เมื่อลบเอกสาร          → invalidate cache ทั้งหมดของ user นั้น
เมื่อเปลี่ยน LLM model   → invalidate ทุก cache (response format อาจต่าง)
TTL expiry              → cache หมดอายุตาม config (default: 1 ชั่วโมง)
```

---

## สรุป

| สิ่งที่ต้องทำ | Priority | ความยาก | ผลประโยชน์ |
|------------|---------|---------|-----------|
| Redis container | สูง | ง่าย | ลด latency + cost ทันที |
| Embedding cache | สูง | ง่าย | ลด API cost ~60-80% |
| LLM response cache | สูง | ปานกลาง | ลด cost + ตอบเร็วขึ้น |
| Semantic cache | กลาง | ยาก | UX ดีขึ้นมาก |
| Cache invalidation | สูง | ปานกลาง | ป้องกัน stale data |

---

*วิเคราะห์จาก source code โดยตรง:*
*`src/agent.py`, `src/session_manager.py`, `src/services/flows_service.py`,*
*`src/connectors/connection_manager.py`, `src/config/config_manager.py`*
