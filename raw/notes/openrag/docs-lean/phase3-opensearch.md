# Phase 3: OpenSearch — Vector Database & Search Engine

## OpenSearch คืออะไร?

**OpenSearch** คือ Open-Source Search and Analytics Engine ที่ fork มาจาก Elasticsearch โดย Amazon Web Services (AWS) ในปี 2021

**ใน OpenRAG ใช้ OpenSearch เป็น:**
1. **Vector Database** — เก็บ embedding vectors ของเอกสาร
2. **Full-text Search Engine** — ค้นหาด้วย keyword
3. **Semantic Search Engine** — ค้นหาด้วยความหมาย (KNN)
4. **Access Control Store** — เก็บ permissions ระดับ document

---

## Vector Database คืออะไร? ทำไมต้องใช้?

```
ปัญหา: ถ้าถามว่า "ราคาสินค้า" — จะ search หา text "ราคาสินค้า" เท่านั้น
        แต่เอกสารอาจพูดว่า "cost", "price", "มูลค่า" — หา keyword ไม่เจอ

Vector Search แก้ปัญหา:
1. แปลงข้อความเป็น Vector (array ของตัวเลข) ที่แทน "ความหมาย"
2. ข้อความที่มีความหมายใกล้เคียงกัน → Vector ที่ใกล้เคียงกัน
3. ค้นหาด้วย "ระยะห่างระหว่าง Vectors" แทน exact keyword match

ตัวอย่าง:
"ราคาสินค้า" → Vector: [0.12, -0.45, 0.87, ...]
"product price" → Vector: [0.11, -0.43, 0.85, ...]  ← ใกล้กัน!
"สูตรทำขนมปัง" → Vector: [-0.9, 0.2, -0.1, ...]   ← ไกลกัน
```

---

## Architecture ของ OpenSearch ใน OpenRAG

```
┌──────────────────────────────────────────────────────┐
│                  OpenSearch Cluster                   │
│                  (Single Node Mode)                   │
│                                                      │
│  Indices (เหมือน Tables ใน SQL):                     │
│  ┌─────────────────────────────────────────────┐     │
│  │ Index: "documents" (ค่าเริ่มต้น)             │     │
│  │  └── เก็บ document chunks + vectors          │     │
│  └─────────────────────────────────────────────┘     │
│  ┌─────────────────────────────────────────────┐     │
│  │ Index: "knowledge_filters"                   │     │
│  │  └── เก็บ filter definitions                │     │
│  └─────────────────────────────────────────────┘     │
│  ┌─────────────────────────────────────────────┐     │
│  │ Index: "api_keys"                            │     │
│  │  └── เก็บ API keys สำหรับ public API        │     │
│  └─────────────────────────────────────────────┘     │
│                                                      │
│  Ports:                                              │
│    9200 — REST API (HTTPS)                           │
│    9600 — Performance Analyzer                       │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## Index Schema: "documents"

นี่คือโครงสร้างของ data ที่เก็บใน OpenSearch:

```json
{
  "mappings": {
    "properties": {
      "document_id":    { "type": "keyword" },
      "filename":       { "type": "keyword" },
      "page":           { "type": "integer" },
      "text":           { "type": "text" },
      "source_url":     { "type": "keyword" },
      "owner":          { "type": "keyword" },
      "allowed_users":  { "type": "keyword" },
      "allowed_groups": { "type": "keyword" },
      "created_time":   { "type": "date" },

      "embeddings": {
        "type": "knn_vector",
        "dimension": 1536,
        "method": {
          "name": "hnsw",
          "engine": "faiss",
          "parameters": {
            "ef_construction": 100,
            "m": 16
          }
        }
      }
    }
  }
}
```

### ตัวอย่างข้อมูลใน Index:
```json
{
  "document_id": "annual_report_2024_chunk_0042",
  "filename": "annual_report_2024.pdf",
  "page": 12,
  "text": "Revenue for Q3 2024 was $3.2M, representing a 15% increase compared to Q3 2023.",
  "source_url": "",
  "owner": "user@company.com",
  "allowed_users": ["user@company.com", "manager@company.com"],
  "allowed_groups": ["finance-team"],
  "created_time": "2024-11-15T10:30:00Z",
  "embeddings": [0.0234, -0.1892, 0.4521, ..., 0.0012]
}
```

---

## KNN Algorithm — disk_ann

OpenSearch ใช้ **disk_ann** algorithm สำหรับ Approximate Nearest Neighbor (ANN) search:

```
disk_ann คืออะไร?
- DiskANN: Disk-based Approximate Nearest Neighbor
- ออกแบบมาให้จัดการกับ dataset ขนาดใหญ่
- เก็บ index บน disk แทน RAM — ประหยัด memory
- ใช้ L2 distance (Euclidean) ในการวัดความใกล้เคียง

Parameters:
  ef_construction: 100  — ยิ่งสูง = ยิ่ง accurate แต่ build ช้า
  m: 16               — จำนวน connections ต่อ node
```

### ตัวอย่างการ Search:
```python
# Search หา documents ที่ใกล้เคียงกับ user query
query = {
  "knn": {
    "embeddings": {
      "vector": [0.023, -0.189, 0.452, ...],  # embedding ของ user query
      "k": 10  # return top 10 results
    }
  }
}
```

---

## Search Modes ที่ OpenRAG รองรับ

### 1. Semantic Search (KNN)
```python
# ค้นหาด้วยความหมาย — ดีที่สุดสำหรับ natural language
query_vector = embed("อะไรคือรายได้ไตรมาส 3?")

# OpenSearch หา chunks ที่ vector ใกล้เคียงที่สุด
results = opensearch.search({
  "knn": {
    "embeddings": {
      "vector": query_vector,
      "k": 10
    }
  }
})
```

### 2. Keyword Search (BM25)
```python
# ค้นหาด้วย keyword แบบ traditional
results = opensearch.search({
  "match": {
    "text": "Q3 revenue 2024"
  }
})
```

### 3. Hybrid Search (KNN + BM25)
```python
# รวมทั้งสอง approach
results = opensearch.search({
  "query": {
    "hybrid": {
      "queries": [
        {
          "knn": {
            "embeddings": {"vector": query_vector, "k": 10}
          }
        },
        {
          "match": {"text": "Q3 revenue 2024"}
        }
      ]
    }
  }
})
```

---

## Access Control — Row-Level Security

OpenSearch ใน OpenRAG implement document-level permissions:

```python
# ทุก document chunk มี fields เหล่านี้:
{
  "owner": "alice@company.com",         # เจ้าของ
  "allowed_users": [                     # user ที่เข้าถึงได้
    "alice@company.com",
    "bob@company.com"
  ],
  "allowed_groups": ["finance-team"]     # group ที่เข้าถึงได้
}

# เวลา search — backend เพิ่ม filter อัตโนมัติ:
filter = {
  "bool": {
    "should": [
      {"term": {"owner": current_user}},
      {"term": {"allowed_users": current_user}},
      {"terms": {"allowed_groups": user_groups}}
    ]
  }
}
```

### ตัวอย่าง: Alice upload เอกสาร, Bob ค้นหา

```
Alice uploads "salary_info.pdf" → owner: alice@company.com

Bob ค้นหา "เงินเดือน":
  → filter: allowed_users includes bob@company.com?
  → No → Bob ไม่เห็นเอกสารนี้ ✅

HR Manager ค้นหา "เงินเดือน":
  → filter: allowed_groups includes "hr-team"?
  → Yes → HR Manager เห็นเอกสาร ✅
```

---

## OpenSearch กับ Embedding Models

OpenRAG รองรับ embedding models หลายตัว โดยแต่ละตัวมี dimension ต่างกัน:

```python
EMBEDDING_MODELS = {
  # OpenAI
  "text-embedding-3-small": {"dimension": 1536, "provider": "openai"},
  "text-embedding-3-large": {"dimension": 3072, "provider": "openai"},
  "text-embedding-ada-002":  {"dimension": 1536, "provider": "openai"},

  # IBM WatsonX
  "ibm/slate-125m-english-rtrvr": {"dimension": 768, "provider": "watsonx"},
  "ibm/slate-30m-english-rtrvr":  {"dimension": 384, "provider": "watsonx"},

  # Ollama (Local)
  "nomic-embed-text":    {"dimension": 768, "provider": "ollama"},
  "mxbai-embed-large":   {"dimension": 1024, "provider": "ollama"},
}
```

**สำคัญ:** dimension ของ embedding ต้องตรงกับ index ที่สร้างไว้ใน OpenSearch
ถ้าเปลี่ยน model → ต้อง re-index เอกสารใหม่ทั้งหมด

---

## OpenSearch Client ใน Python

```python
# /src/config/settings.py
from opensearchpy import AsyncOpenSearch

# สร้าง client หลัก
os_client = AsyncOpenSearch(
    hosts=[{
        "host": "opensearch",
        "port": 9200
    }],
    http_auth=("admin", "YourPassword"),
    use_ssl=True,
    verify_certs=False,
    ssl_assert_hostname=False,
    ssl_show_warn=False
)
```

### การ Index Document:
```python
# บันทึก chunk ลง OpenSearch
await os_client.index(
    index="documents",
    id="annual_report_chunk_042",
    body={
        "document_id": "annual_report_chunk_042",
        "filename": "annual_report.pdf",
        "page": 5,
        "text": "Revenue in Q3 2024 was $3.2 million...",
        "embeddings": [0.023, -0.189, 0.452, ...],
        "owner": "user@example.com",
        "allowed_users": ["user@example.com"],
        "allowed_groups": [],
        "created_time": "2024-11-15T10:30:00Z"
    }
)
```

### การ Search:
```python
# /src/services/search_service.py
response = await os_client.search(
    index="documents",
    body={
        "size": 10,
        "query": {
            "bool": {
                "must": {
                    "knn": {
                        "embeddings": {
                            "vector": query_embedding,
                            "k": 10
                        }
                    }
                },
                "filter": [
                    {
                        "bool": {
                            "should": [
                                {"term": {"owner": current_user}},
                                {"term": {"allowed_users": current_user}}
                            ]
                        }
                    }
                ]
            }
        }
    }
)
```

---

## OpenSearch Security Configuration

ใน docker-compose.yml:
```yaml
opensearch:
  environment:
    - plugins.security.disabled=false
    - OPENSEARCH_INITIAL_ADMIN_PASSWORD=${OPENSEARCH_PASSWORD}
  volumes:
    - opensearch-data:/usr/share/opensearch/data
```

**Security Features:**
- HTTPS (TLS/SSL) บน port 9200
- Admin password authentication
- Role-based access control (RBAC)
- Audit logging

---

## OpenSearch Dashboards

OpenSearch Dashboards เปิดที่ `http://localhost:5601` ใช้ monitor:

```
Dashboards ทำอะไรได้:
├── Dev Tools — ทดสอบ queries โดยตรง
├── Index Management — ดู/แก้ไข index settings
├── Discover — Browse documents ใน index
├── Visualize — สร้าง charts จาก data
└── Security — จัดการ users/roles
```

### ตัวอย่างการใช้ Dev Tools:

```json
// ดู documents ทั้งหมดใน index
GET documents/_search
{
  "query": {"match_all": {}},
  "size": 5
}

// นับจำนวน documents
GET documents/_count

// ดู mapping ของ index
GET documents/_mapping

// ลบ index ทั้งหมด (ระวัง!)
DELETE documents
```

---

## Performance Tuning

### Index Settings ที่สำคัญ:
```json
{
  "settings": {
    "index": {
      "knn": true,
      "knn.algo_param.ef_search": 100,
      "number_of_shards": 1,
      "number_of_replicas": 0
    }
  }
}
```

**ef_search:** ยิ่งสูง = ยิ่งค้นหาแม่นยำ แต่ช้ากว่า (default: 100)

### เมื่อ Index ใหญ่ขึ้น:
```bash
# ปรับ JVM heap size ใน docker-compose.yml
environment:
  - OPENSEARCH_JAVA_OPTS=-Xms1g -Xmx1g  # 1GB (ค่าเริ่มต้น)
  # เพิ่มเป็น:
  - OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g  # 4GB สำหรับ production
```

---

## ตัวอย่าง End-to-End Search Flow

```
User: "บริษัทมีนโยบายลาป่วยกี่วัน?"
                │
                ▼
[Backend] รับ query → สร้าง embedding
        query_vector = embed("บริษัทมีนโยบายลาป่วยกี่วัน?")
        # → [0.12, -0.34, 0.78, ...]
                │
                ▼
[OpenSearch] KNN Search
        หา chunks ที่ similarity สูงสุดกับ query_vector
        + กรองด้วย access control filter
                │
                ▼
[Results] Top 5 chunks:
  1. "ลาป่วย: พนักงานมีสิทธิลาป่วยได้ 30 วันต่อปี..." (score: 0.94)
  2. "การลาป่วยต้องมีใบรับรองแพทย์สำหรับการลาเกิน 3 วัน..." (score: 0.89)
  3. "นโยบาย HR: การลาประเภทต่างๆ..." (score: 0.82)
                │
                ▼
[LLM] ส่ง query + chunks → Generate answer
        "ตามนโยบายบริษัท พนักงานมีสิทธิลาป่วยได้ 30 วันต่อปี
         โดยต้องมีใบรับรองแพทย์หากลาเกิน 3 วัน
         (อ้างอิง: HR Policy Manual หน้า 12)"
```

---

## เปรียบเทียบ OpenSearch กับ ทางเลือกอื่น

| Feature | OpenSearch | Pinecone | Weaviate | pgvector |
|---------|-----------|---------|---------|---------|
| Self-hosted | ✅ | ❌ (cloud) | ✅ | ✅ |
| Open-source | ✅ | ❌ | ✅ | ✅ |
| Full-text search | ✅ ดีมาก | ❌ | ⚠️ | ⚠️ |
| Vector search | ✅ ดี | ✅ ดีมาก | ✅ ดีมาก | ⚠️ |
| Access control | ✅ Row-level | ✅ Namespace | ✅ | ❌ |
| Scale | ✅ Large scale | ✅ | ✅ | ⚠️ |
| Learning curve | ⚠️ สูง | ✅ ง่าย | ⚠️ | ✅ ง่าย |

**OpenSearch เหมาะสำหรับ OpenRAG เพราะ:**
- Full-text + Vector search ในที่เดียว
- Row-level security ที่ mature
- Open-source + Self-hosted = ไม่มีค่าใช้จ่าย cloud
- รองรับ scale ใหญ่ได้

---

*กลับไป: [Phase 2 — Docling](./phase2-docling.md)*
*ต่อไป: [Phase 4 — Langflow](./phase4-langflow.md)*
