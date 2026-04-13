# Phase 4: Langflow — Visual Workflow Builder

## Langflow คืออะไร?

**Langflow** คือ Open-Source platform สำหรับสร้าง **AI Workflow และ Agent** แบบ Visual (Drag-and-Drop) โดยไม่ต้องเขียน code มาก

คิดว่า Langflow เหมือน **"น้ำ flow ผ่าน pipe"**:
- **Node** = component แต่ละอัน (เช่น Docling, Text Splitter, OpenAI)
- **Edge** = การเชื่อมต่อระหว่าง component
- **Flow** = pipeline ทั้งหมด

---

## ทำไม OpenRAG ถึงใช้ Langflow?

```
ไม่ใช้ Langflow: Backend ต้องเขียน code สำหรับทุก step
  - เรียก Docling API
  - Split text
  - เรียก Embedding API
  - บันทึกลง OpenSearch
  - Handle errors
  - ...ซับซ้อนมาก

ใช้ Langflow:
  - สร้าง pipeline ด้วย UI
  - เปลี่ยน component ได้ง่าย (เช่น เปลี่ยน LLM Provider)
  - ทดสอบ flow ได้โดยตรงใน UI
  - ใช้ API เรียก flow ได้ง่าย
  - Community มี component สำเร็จรูปมากมาย
```

---

## Flows ใน OpenRAG

OpenRAG มี 4 Flows หลัก:

```
flows/
├── ingestion_flow.json      # 1. Document Ingestion Pipeline
├── openrag_agent.json       # 2. Chat/RAG Agent
├── openrag_url_mcp.json     # 3. URL Ingestion
└── openrag_nudges.json      # 4. Contextual Suggestions
```

### Flow IDs (จาก .env):
```bash
LANGFLOW_INGEST_FLOW_ID=5488df7c-b93f-4f87-a446-b67028bc0813
LANGFLOW_CHAT_FLOW_ID=1098eea1-6649-4e1d-aed1-b77249fb8dd0
LANGFLOW_URL_INGEST_FLOW_ID=72c3d17c-2dac-4a73-b48a-6518473d7830
NUDGES_FLOW_ID=ebc01d31-1976-46ce-a385-b0240327226c
```

---

## Flow 1: Ingestion Flow

Pipeline สำหรับ **ประมวลผลและ index เอกสาร**

```
┌──────────────────────────────────────────────────────────────┐
│                    Ingestion Flow                             │
│                                                              │
│  [File Input]                                                │
│       │                                                      │
│       ▼                                                      │
│  [Docling Component]                                         │
│  ├── URL: http://host.docker.internal:5001                  │
│  ├── OCR Engine: configurable                               │
│  └── Output: Markdown text                                  │
│       │                                                      │
│       ▼                                                      │
│  [Split Text Component]                                      │
│  ├── Chunk Size: ~1000 tokens                               │
│  ├── Chunk Overlap: ~200 tokens                             │
│  └── Method: recursive character splitting                  │
│       │                                                      │
│       ▼                                                      │
│  [Embedding Component]                                       │
│  ├── Provider: OpenAI / WatsonX / Ollama                   │
│  ├── Model: text-embedding-3-small                         │
│  └── Output: float array [1536 dimensions]                 │
│       │                                                      │
│       ▼                                                      │
│  [OpenSearch Store Component]                                │
│  ├── Host: opensearch:9200                                  │
│  ├── Index: documents                                       │
│  ├── Auth: admin credentials                               │
│  └── Fields: text, embeddings, metadata                    │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Tweak Parameters ที่ Backend ส่งไปให้ Flow:
```python
# /src/services/flows_service.py
tweaks = {
    "OpenSearchVectorStore-xxxx": {
        "index_name": "documents",
        "opensearch_url": "https://opensearch:9200",
        "username": "admin",
        "password": settings.OPENSEARCH_PASSWORD,
    },
    "DoclingComponent-xxxx": {
        "docling_serve_url": "http://host.docker.internal:5001",
    },
    "MetadataInjector-xxxx": {
        "owner": current_user,
        "filename": filename,
        "document_id": document_id,
    }
}
```

---

## Flow 2: RAG Agent Flow (Chat)

Pipeline สำหรับ **ตอบคำถามจากเอกสาร**

```
┌──────────────────────────────────────────────────────────────┐
│                    OpenRAG Agent Flow                         │
│                                                              │
│  [Chat Input]                                                │
│  └── User message: "Q3 revenue คือเท่าไร?"                 │
│       │                                                      │
│       ▼                                                      │
│  [OpenSearch Retriever]                                      │
│  ├── Embed user query                                       │
│  ├── KNN search in OpenSearch                              │
│  ├── Apply access control filter                           │
│  ├── k=10 results                                          │
│  └── Output: relevant chunks + metadata                    │
│       │                                                      │
│       ▼                                                      │
│  [Context Builder]                                           │
│  └── ประกอบ chunks เป็น context string                     │
│       │                                                      │
│       ▼                                                      │
│  [LLM Component]                                             │
│  ├── Provider: OpenAI / Claude / WatsonX / Ollama           │
│  ├── System Prompt: "You are a helpful assistant..."       │
│  ├── Input: user query + context                           │
│  └── Output: Generated response                            │
│       │                                                      │
│       ▼                                                      │
│  [Chat Output]                                               │
│  └── Response + citations                                   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## Flow 3: URL Ingestion (MCP)

สำหรับ ingest เนื้อหาจาก URL โดยตรง:

```
[URL Input] → [Web Scraper/MCP] → [Docling] → [Split] → [Embed] → [OpenSearch]
```

ตัวอย่างการใช้:
```
POST /langflow/ingest
Body: {
  "url": "https://docs.company.com/policy",
  "owner": "user@example.com"
}
```

---

## Flow 4: Nudges Flow

สำหรับ generate **"คำแนะนำ"** ให้ user รู้ว่าควรถามอะไรต่อ:

```
[Chat History] → [LLM] → [Suggested Questions]

ตัวอย่าง output:
"คุณอาจสนใจถามเรื่อง:
  - นโยบายลาป่วยมีข้อยกเว้นอะไรบ้าง?
  - ลาป่วยยาวต้องทำเอกสารอะไร?"
```

---

## Langflow API — วิธีที่ Backend เรียกใช้

### 1. Upload File ไปที่ Langflow:
```python
# POST /api/v1/files/upload
import httpx

async with httpx.AsyncClient() as client:
    response = await client.post(
        f"{LANGFLOW_URL}/api/v1/files/upload/{flow_id}",
        files={"file": (filename, file_content, content_type)},
        headers={"Authorization": f"Bearer {langflow_api_key}"}
    )
    file_path = response.json()["file_path"]
```

### 2. Run Flow (Ingest):
```python
# POST /api/v1/run/{flow_id}
response = await client.post(
    f"{LANGFLOW_URL}/api/v1/run/{INGEST_FLOW_ID}",
    json={
        "input_value": file_path,
        "input_type": "text",
        "output_type": "text",
        "tweaks": {
            "DoclingComponent-xxxx": {
                "docling_serve_url": docling_url
            },
            "OpenSearchVectorStore-xxxx": {
                "index_name": index_name,
                "username": os_user,
                "password": os_pass
            }
        }
    },
    headers={"Authorization": f"Bearer {langflow_api_key}"},
    timeout=INGESTION_TIMEOUT  # 3600 seconds
)
```

### 3. Run Chat Flow (Streaming):
```python
# POST /api/v1/run/{flow_id}?stream=true
async with client.stream(
    "POST",
    f"{LANGFLOW_URL}/api/v1/run/{CHAT_FLOW_ID}",
    json={
        "input_value": user_message,
        "input_type": "chat",
        "output_type": "chat",
        "session_id": session_id,
        "tweaks": {
            "OpenSearchRetriever-xxxx": {
                "filter_owner": current_user,
                "k": 10
            }
        }
    }
) as response:
    async for chunk in response.aiter_text():
        yield chunk  # Stream ไปยัง Frontend
```

---

## Langflow Component Types ที่ใช้ใน OpenRAG

| Component | หน้าที่ |
|-----------|---------|
| **DoclingFileLoader** | เรียก Docling API เพื่อ parse เอกสาร |
| **RecursiveCharacterTextSplitter** | แบ่ง text เป็น chunks |
| **OpenAIEmbeddings** | สร้าง embeddings ด้วย OpenAI |
| **WatsonXEmbeddings** | สร้าง embeddings ด้วย IBM WatsonX |
| **OllamaEmbeddings** | สร้าง embeddings ด้วย Ollama |
| **OpenSearchVectorStore** | บันทึก/ค้นหาใน OpenSearch |
| **ChatOpenAI** | LLM component (OpenAI) |
| **ChatAnthropic** | LLM component (Claude) |
| **ChatWatsonX** | LLM component (IBM WatsonX) |
| **Prompt** | System prompt template |
| **ConversationChain** | จัดการ conversation history |

---

## Langflow Configuration ใน OpenRAG

```bash
# .env settings
LANGFLOW_URL=http://langflow:7860
LANGFLOW_SECRET_KEY=your-secret-key
LANGFLOW_AUTO_LOGIN=False          # ต้อง login ด้วย API key
LANGFLOW_TIMEOUT=2400              # 40 นาที (สำหรับ large file)
LANGFLOW_CONNECT_TIMEOUT=30        # Connection timeout

# Backend สร้าง API key อัตโนมัติ
# เก็บไว้ใน: ./keys/langflow_api_key
```

### Auto-initialization:
```python
# /src/main.py — เมื่อ Backend start ขึ้น:
# 1. รอ Langflow พร้อม (health check)
# 2. สร้าง Langflow API key อัตโนมัติ
# 3. โหลด Flows จาก /flows/ directory
# 4. บันทึก Flow IDs ลง .env
```

---

## Flow Management — Backup & Reset

Backend มีระบบ backup/reset flows:

```python
# /src/services/flows_service.py

# Backup current flows
POST /flows/backup

# Reset flow เป็น default
POST /reset-flow/{flow_type}
# flow_type: "nudges", "retrieval", "ingest", "url_ingest"

# การ reset จะ:
# 1. โหลด flow จาก /flows/*.json
# 2. Delete flow เก่าใน Langflow
# 3. Import flow ใหม่
# 4. Update Flow ID ใน settings
```

---

## การ Customize Langflow Flows

### เปิด Langflow UI:
```
http://localhost:7860
```

### ตัวอย่างการแก้ไข Ingestion Flow:
1. เข้า Langflow UI → เปิด ingestion_flow
2. คลิก Text Splitter component
3. แก้ไข `chunk_size` จาก 1000 → 2000 (สำหรับ document ยาว)
4. Save flow
5. Flow จะมีผลทันทีเมื่อ ingest เอกสารใหม่

### เพิ่ม Custom Component:
```python
# สร้าง Python file ใน Langflow
# เช่น: custom_preprocessor.py

from langflow.custom import CustomComponent
from langflow.schema import Document

class CustomPreprocessor(CustomComponent):
    display_name = "Custom Preprocessor"
    description = "Custom text preprocessing"

    def build(self, text: str) -> str:
        # ทำการ preprocessing
        cleaned = text.replace("\x00", "")  # ลบ null bytes
        cleaned = " ".join(cleaned.split())  # normalize whitespace
        return cleaned
```

---

## Langflow vs ทางเลือกอื่น

| Feature | Langflow | LangChain (code) | n8n | Flowise |
|---------|---------|-----------------|-----|---------|
| Visual UI | ✅ | ❌ | ✅ | ✅ |
| Python-native | ✅ | ✅ | ❌ | ❌ |
| LLM-focused | ✅ | ✅ | ⚠️ | ✅ |
| Self-hosted | ✅ | N/A | ✅ | ✅ |
| API export | ✅ | N/A | ✅ | ✅ |
| Custom components | ✅ | ✅ | ⚠️ | ⚠️ |
| Streaming | ✅ | ✅ | ❌ | ✅ |

---

## ตัวอย่าง: Trace การทำงานเต็ม

```
User อัปโหลด "product_manual.pdf"
            │
            ▼
Backend: POST /upload
  └── สร้าง task_id: "task-abc123"
  └── บันทึกไฟล์ชั่วคราว
            │
            ▼
Backend: POST /langflow/upload_ingest
  ├── Upload ไฟล์ไปที่ Langflow Files API
  │   POST http://langflow:7860/api/v1/files/upload/{ingest_flow_id}
  │   Response: { "file_path": "/tmp/langflow/product_manual.pdf" }
  │
  └── Run Ingestion Flow
      POST http://langflow:7860/api/v1/run/{ingest_flow_id}
      Body: {
        "input_value": "/tmp/langflow/product_manual.pdf",
        "tweaks": {
          "DoclingComponent": {"docling_serve_url": "http://...:5001"},
          "OpenSearchStore": {"index": "documents"},
          "MetadataInjector": {"owner": "user@example.com", "filename": "product_manual.pdf"}
        }
      }
            │
            ▼
Langflow Ingestion Flow:
  [File] → [Docling] → [TextSplit] → [Embed] → [OpenSearch]

  Step 1: Docling แปลง PDF → Markdown
    "# Product Manual\n## Chapter 1: Installation\n..."

  Step 2: Text Split → 45 chunks
    chunk_0: "# Product Manual\n## Chapter 1: Installation\n..."
    chunk_1: "...Step 3: Connect the power cable to the..."
    ...

  Step 3: Embed each chunk → vectors [1536 dims]

  Step 4: Index 45 chunks ใน OpenSearch
    document_id: "product_manual_chunk_000" .. "product_manual_chunk_044"
            │
            ▼
Backend: อัปเดต task status → "completed"
  └── Task: { "status": "success", "chunks_indexed": 45 }
```

---

## Timeouts ที่สำคัญ

```bash
# .env
LANGFLOW_TIMEOUT=2400        # 40 นาที — รอ flow ประมวลผล
LANGFLOW_CONNECT_TIMEOUT=30  # 30 วินาที — รอ connection
INGESTION_TIMEOUT=3600       # 60 นาที — timeout สำหรับ ingestion task
```

> **หมายเหตุ:** ไฟล์ขนาดใหญ่ (100+ หน้า) อาจใช้เวลาหลายนาที — timeout ควรตั้งให้สูงพอ

---

*กลับไป: [Phase 3 — OpenSearch](./phase3-opensearch.md)*
*ต่อไป: [Phase 5 — Integration & Examples](./phase5-integration.md)*
