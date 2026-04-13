---
title: "OpenRAG — Cache System Analysis"
type: source
source_file: raw/notes/openrag/docs-lean/openrag-cache-analysis.md
url: ""
published: 2026-03-17
tags: [openrag, cache, redis, semantic-cache, cost-optimization, rag]
related: [wiki/concepts/semantic-caching.md, wiki/concepts/openrag-platform.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/openrag-cache-analysis.md|Cache Analysis]]

## สรุป
OpenRAG มี cache แค่ส่วน utility (conversation memory, OpenSearch client, OAuth tokens) — ไม่มี cache สำหรับ Query, Embedding, หรือ LLM Response ซึ่งเป็นส่วนที่กิน cost และเวลามากที่สุด

## ประเด็นสำคัญ

### Cache ที่มีอยู่แล้ว (Built-in)
| Layer | ประเภท | TTL | หมายเหตุ |
|-------|--------|-----|---------|
| Conversation history | Memory + Disk | ไม่มี | metadata เท่านั้น |
| OpenSearch client | Memory dict | ไม่มี | per-user connection pool |
| Flow file paths | Memory dict | ไม่มี | utility only |
| Connector instances | Memory dict | token expiry | Google/OneDrive/SharePoint |
| Config YAML | Memory object | on save | |
| OAuth tokens | MSAL file | token expiry | |

### Cache ที่ขาดไป (Missing) — จุดสำคัญ
- ❌ **Search/Query Result Cache** — user 100 คนถามซ้ำ → ทุกคน embed + KNN + LLM
- ❌ **Embedding Cache** — embed query ใหม่ทุก request (~100-300ms ต่อครั้ง)
- ❌ **LLM Response Cache** — "ลาป่วยได้กี่วัน" vs "ลาป่วยได้กี่วัน?" = 2 LLM calls
- ❌ **Docling Parse Cache** — ลบแล้ว re-upload → Docling ทำงานซ้ำ

### ผลกระทบจากการไม่มี Cache
```
50 users × 20 queries/วัน × 22 วัน = 22,000 queries/เดือน
สมมติ unique queries จริง 20% (ซ้ำ 80%)

แบบไม่มี cache:  LLM: 22,000 × $0.01 = $220/เดือน
แบบมี cache 80% hit rate: LLM: 4,400 × $0.01 = $44/เดือน
ประหยัด: ~80% = ~$176/เดือน, latency ลด 300-700x (cache hit ~5ms vs 1.5-3.5s)
```

### แผนเพิ่ม Cache (เรียงจากง่ายไปยาก)

**Level 1: In-Memory Cache (1 วัน)**
```python
class QueryCache:
    def __init__(self, ttl_seconds=3600, max_size=1000):
        self._cache = {}
    def _make_key(self, query, user_id): return sha256(f"{query}:{user_id}")
    # หายเมื่อ restart, ไม่ต้องติดตั้งอะไรเพิ่ม
```

**Level 2: Redis Cache (1-2 วัน)**
```yaml
# docker-compose.yml
redis:
  image: redis:7-alpine
  command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
```
- TTL, persist, shared across workers
- Cache embedding 24h, LLM response 1h

**Level 3: Semantic Cache (1 สัปดาห์)**
```python
# คำถามใกล้เคียงกัน → ใช้ answer เดิม
if cosine_similarity(query_embedding, cached_embedding) >= 0.95:
    return cached_answer
```
- "ลาพักร้อนได้กี่วัน" ≈ "พักร้อนประจำปีมีกี่วัน" (similarity 0.97) → cache hit

### Cache Invalidation Strategy
```
เมื่อ upload document ใหม่ → invalidate cache ของ user นั้น
เมื่อลบ document → invalidate cache ของ user นั้น
เมื่อเปลี่ยน LLM model → invalidate ทุก cache
TTL expiry → default 1 ชั่วโมง
```

## ข้อมูล / หลักฐาน ที่น่าสนใจ
- Conversation memory ใช้ Python dict ไม่มี eviction policy → memory รั่วถ้า conversation เยอะมาก
- OpenSearch client cached per-user — ไม่ต้อง init connection ทุก request
- ไม่มี Semantic Cache หรือ Exact Match Cache สำหรับ LLM เลย

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว
- Arona (wiki ก่อนหน้า) มี Semantic Cache ด้วย DragonflyDB ส่วน OpenRAG ยังไม่มี — OpenRAG เน้น features ครบกว่าแต่ optimization ยังด้อยกว่า

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/semantic-caching.md|Semantic Caching]]
- [[wiki/concepts/openrag-platform.md|OpenRAG Platform]]
