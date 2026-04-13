# OpenRAG SDK Documentation

> คู่มือการใช้งาน OpenRAG SDK สำหรับ **TypeScript/JavaScript** และ **Python** อย่างละเอียด ครอบคลุมทุก API, ตัวอย่าง input/output จริง และ pattern การใช้งาน

---

## สารบัญ

1. [ภาพรวม](#ภาพรวม)
2. [การติดตั้ง](#การติดตั้ง)
3. [Authentication](#authentication)
4. [การสร้าง Client](#การสร้าง-client)
5. [Chat API](#chat-api)
6. [Search API](#search-api)
7. [Documents API](#documents-api)
8. [Settings API](#settings-api)
9. [Knowledge Filters API](#knowledge-filters-api)
10. [Models API](#models-api)
11. [Error Handling](#error-handling)
12. [ตัวอย่าง Use Cases จริง](#ตัวอย่าง-use-cases-จริง)
13. [Type Reference](#type-reference)

---

## ภาพรวม

OpenRAG มี official SDK สองภาษา:

| | TypeScript/JS | Python |
|---|---|---|
| **Package** | `openrag-sdk` | `openrag-sdk` |
| **Version** | 0.2.0 | 0.2.0 |
| **Runtime** | Node.js ≥18, Browser | Python ≥3.10 |
| **HTTP Client** | Native `fetch` | `httpx` (async) |
| **Validation** | TypeScript types | Pydantic v2 |
| **Streaming** | `AsyncIterable` + `using` syntax | `async with` context manager |

### Sub-clients ที่มี

```
OpenRAGClient
├── .chat              → Chat + Conversation management
├── .search            → Semantic search
├── .documents         → Ingest + Delete documents
├── .settings          → Configuration
├── .models            → List available LLM/embedding models
└── .knowledgeFilters  → Knowledge filter management
     (TS: knowledgeFilters | Python: knowledge_filters)
```

---

## การติดตั้ง

### TypeScript / JavaScript

```bash
# npm
npm install openrag-sdk

# yarn
yarn add openrag-sdk

# pnpm
pnpm add openrag-sdk
```

**Requirements:**
- Node.js ≥ 18.0.0 (ใช้ native `fetch`)
- TypeScript ≥ 4.9 (optional แต่แนะนำ)
- ไม่มี production dependencies

### Python

```bash
pip install openrag-sdk

# หรือด้วย uv
uv add openrag-sdk
```

**Requirements:**
- Python ≥ 3.10
- `httpx >= 0.25.0`
- `pydantic >= 2.0.0`

---

## Authentication

OpenRAG ใช้ **API Key** ผ่าน HTTP header `X-API-Key`

### รูปแบบ API Key

```
orag_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### วิธีตั้งค่า (แนะนำ: Environment Variables)

```bash
# .env
OPENRAG_API_KEY=orag_your_api_key_here
OPENRAG_URL=http://localhost:3000     # optional, default: http://localhost:3000
```

### การสร้าง API Key

ต้องทำผ่าน REST API (ต้องมี JWT token จาก login ก่อน):

```bash
curl -X POST http://localhost:3000/keys \
  -H "Authorization: Bearer <jwt_token>" \
  -H "Content-Type: application/json"
```

**Response:**
```json
{
  "key": "orag_abc123def456...",
  "id": "key_xyz789",
  "created_at": "2026-03-17T10:00:00Z"
}
```

---

## การสร้าง Client

### TypeScript

```typescript
import { OpenRAGClient } from "openrag-sdk";

// วิธีที่ 1: อ่านจาก environment variables อัตโนมัติ
// ต้องตั้ง OPENRAG_API_KEY และ OPENRAG_URL ไว้
const client = new OpenRAGClient();

// วิธีที่ 2: ระบุค่าโดยตรง
const client = new OpenRAGClient({
  apiKey: "orag_your_api_key_here",
  baseUrl: "http://localhost:3000",
  timeout: 30000,  // milliseconds, default: 30000
});

// วิธีที่ 3: ใช้ใน browser (ผ่าน File object)
const client = new OpenRAGClient({
  apiKey: process.env.OPENRAG_API_KEY!,
  baseUrl: process.env.OPENRAG_URL ?? "http://localhost:3000",
});
```

### Python

```python
import asyncio
from openrag_sdk import OpenRAGClient

# วิธีที่ 1: อ่านจาก environment variables อัตโนมัติ
async def main():
    async with OpenRAGClient() as client:
        # ใช้ client ที่นี่
        pass

asyncio.run(main())

# วิธีที่ 2: ระบุค่าโดยตรง
async def main():
    async with OpenRAGClient(
        api_key="orag_your_api_key_here",
        base_url="http://localhost:3000",
        timeout=30.0,  # seconds, default: 30.0
    ) as client:
        pass

# วิธีที่ 3: สร้างแบบ manual (ไม่ใช้ context manager)
client = OpenRAGClient(api_key="orag_...")
try:
    response = await client.chat.create(message="Hello")
finally:
    await client.close()
```

> **หมายเหตุ Python**: ต้องใช้ `async with` หรือเรียก `await client.close()` เสมอ เพื่อปิด HTTP connection pool อย่างถูกต้อง

---

## Chat API

### Non-Streaming Chat

ส่งข้อความและรับคำตอบแบบทั้งก้อน (รอจนตอบเสร็จ)

#### TypeScript

```typescript
const response = await client.chat.create({
  message: "OpenRAG คืออะไร?",
});

console.log(response.response);
// → "OpenRAG เป็น open-source RAG platform ที่..."

console.log(response.chatId);
// → "chat_abc123" (ใช้ต่อบทสนทนา)

console.log(response.sources);
// → [{ filename: "intro.pdf", text: "...", score: 0.92, page: 1 }]
```

#### Python

```python
response = await client.chat.create(message="OpenRAG คืออะไร?")

print(response.response)
# → "OpenRAG เป็น open-source RAG platform ที่..."

print(response.chat_id)
# → "chat_abc123"

for source in response.sources:
    print(f"[{source.score:.2f}] {source.filename} (page {source.page})")
    print(f"  {source.text[:100]}...")
```

**ตัวอย่าง Response Object:**

```json
{
  "response": "OpenRAG เป็น open-source RAG (Retrieval-Augmented Generation) platform ที่สร้างบน Langflow ช่วยให้คุณสามารถอัปโหลดเอกสารและถามคำถามได้อย่างชาญฉลาด",
  "chat_id": "chat_7f3a9b2c1d4e",
  "sources": [
    {
      "filename": "openrag_intro.pdf",
      "text": "OpenRAG is an open-source RAG platform built on top of Langflow...",
      "score": 0.9234,
      "page": 1,
      "mimetype": "application/pdf"
    },
    {
      "filename": "readme.md",
      "text": "OpenRAG enables you to upload documents and ask intelligent questions...",
      "score": 0.8891,
      "page": null,
      "mimetype": "text/markdown"
    }
  ]
}
```

---

### Streaming Chat (วิธีที่ 1: ChatStream context manager)

วิธีนี้แนะนำสุด — รองรับทั้ง event iteration และ property access หลังจบ stream

#### TypeScript (ใช้ `using` syntax — ES2022+)

```typescript
// แบบ using (TypeScript ≥5.2, explicit resource management)
using stream = await client.chat.stream({
  message: "อธิบาย machine learning ให้หน่อย",
  limit: 5,
  scoreThreshold: 0.7,
});

for await (const event of stream) {
  if (event.type === "content") {
    process.stdout.write(event.delta);  // print ทีละ token
  } else if (event.type === "sources") {
    console.log("\n\nSources:", event.sources.length, "items");
  } else if (event.type === "done") {
    console.log("Chat ID:", event.chatId);
  }
}

// หลัง stream จบ — เข้าถึง properties ได้
console.log("Full text:", stream.text);
console.log("Chat ID:", stream.chatId);
console.log("Sources count:", stream.sources.length);
```

#### Python (ใช้ `async with`)

```python
async with client.chat.stream(
    message="อธิบาย machine learning ให้หน่อย",
    limit=5,
    score_threshold=0.7,
) as stream:
    async for event in stream:
        if event.type == "content":
            print(event.delta, end="", flush=True)
        elif event.type == "sources":
            print(f"\n\nSources: {len(event.sources)} items")
        elif event.type == "done":
            print(f"Chat ID: {event.chat_id}")

    # หลัง stream จบ
    print(f"\nFull text: {stream.text}")
    print(f"Chat ID: {stream.chat_id}")
    print(f"Sources: {len(stream.sources)}")
```

**ตัวอย่าง SSE Events ที่ได้จาก server:**

```
data: {"type": "content", "delta": "Machine"}
data: {"type": "content", "delta": " learning"}
data: {"type": "content", "delta": " คือ"}
data: {"type": "content", "delta": " กระบวนการ"}
...
data: {"type": "sources", "sources": [{"filename": "ml_intro.pdf", "text": "...", "score": 0.91, "page": 3}]}
data: {"type": "done", "chat_id": "chat_7f3a9b2c1d4e"}
```

---

### Streaming Chat (วิธีที่ 2: create() with stream=true)

#### TypeScript

```typescript
const events = await client.chat.create({
  message: "สรุปเอกสารนี้",
  stream: true,
});

for await (const event of events) {
  if (event.type === "content") {
    process.stdout.write(event.delta);
  }
}
```

#### Python

```python
async for event in await client.chat.create(
    message="สรุปเอกสารนี้",
    stream=True,
):
    if event.type == "content":
        print(event.delta, end="", flush=True)
```

---

### Text-only Stream (ง่ายสุด)

#### TypeScript

```typescript
using stream = await client.chat.stream({ message: "Hello" });

// วิธีที่ 1: textStream property
for await (const text of stream.textStream) {
  process.stdout.write(text);
}

// วิธีที่ 2: finalText() รับทั้งก้อน
const fullText = await stream.finalText();
console.log(fullText);
```

#### Python

```python
async with client.chat.stream(message="Hello") as stream:
    # วิธีที่ 1: text_stream property
    async for text in stream.text_stream:
        print(text, end="", flush=True)

    # วิธีที่ 2: final_text() รับทั้งก้อน
    # (ต้อง stream ใหม่เพราะ consumed แล้ว)

# ในทางปฏิบัติ เลือกใช้แบบใดแบบหนึ่ง
async with client.chat.stream(message="Hello") as stream:
    full_text = await stream.final_text()
    print(full_text)
```

---

### Multi-turn Conversation (ต่อบทสนทนา)

#### TypeScript

```typescript
// Turn 1
const r1 = await client.chat.create({ message: "สวัสดี ฉันชื่อ Alice" });
const chatId = r1.chatId!;
console.log(r1.response);
// → "สวัสดี Alice! มีอะไรให้ช่วยไหม?"

// Turn 2 — ส่ง chatId เดิม
const r2 = await client.chat.create({
  message: "ฉันชื่ออะไร?",
  chatId: chatId,
});
console.log(r2.response);
// → "คุณชื่อ Alice ครับ"
```

#### Python

```python
# Turn 1
r1 = await client.chat.create(message="สวัสดี ฉันชื่อ Alice")
chat_id = r1.chat_id
print(r1.response)
# → "สวัสดี Alice! มีอะไรให้ช่วยไหม?"

# Turn 2
r2 = await client.chat.create(
    message="ฉันชื่ออะไร?",
    chat_id=chat_id,
)
print(r2.response)
# → "คุณชื่อ Alice ครับ"
```

---

### Conversation Management

#### TypeScript

```typescript
// ดูรายการบทสนทนาทั้งหมด
const { conversations } = await client.chat.list();
console.log(conversations);
/*
[
  {
    chatId: "chat_7f3a9b2c",
    title: "OpenRAG คืออะไร?",
    createdAt: "2026-03-17T10:00:00Z",
    lastActivity: "2026-03-17T10:05:00Z",
    messageCount: 4
  },
  ...
]
*/

// ดูประวัติบทสนทนาเต็ม
const detail = await client.chat.get("chat_7f3a9b2c");
console.log(detail.messages);
/*
[
  { role: "user", content: "OpenRAG คืออะไร?", timestamp: "2026-03-17T10:00:00Z" },
  { role: "assistant", content: "OpenRAG เป็น...", timestamp: "2026-03-17T10:00:02Z" },
  ...
]
*/

// ลบบทสนทนา
const deleted = await client.chat.delete("chat_7f3a9b2c");
console.log(deleted);  // → true
```

#### Python

```python
# ดูรายการ
result = await client.chat.list()
for conv in result.conversations:
    print(f"{conv.chat_id}: {conv.title} ({conv.message_count} msgs)")

# ดูประวัติเต็ม
detail = await client.chat.get("chat_7f3a9b2c")
for msg in detail.messages:
    print(f"[{msg.role}] {msg.content}")

# ลบ
success = await client.chat.delete("chat_7f3a9b2c")
print(success)  # → True
```

---

### Chat Parameters ทั้งหมด

| Parameter | Type | Default | ความหมาย |
|-----------|------|---------|-----------|
| `message` | `string` | **required** | ข้อความที่ส่ง |
| `stream` | `boolean` | `false` | เปิด streaming |
| `chatId` / `chat_id` | `string \| null` | `null` | ID บทสนทนาที่ต้องการต่อ |
| `filters` | `SearchFilters \| null` | `null` | กรอง data sources |
| `limit` | `number` | `10` | จำนวน source chunks ที่ดึงมาใช้ |
| `scoreThreshold` / `score_threshold` | `number` | `0` | คะแนน relevance ขั้นต่ำ (0.0–1.0) |
| `filterId` / `filter_id` | `string \| null` | `null` | Knowledge filter ID |

---

## Search API

ค้นหาเนื้อหาใน knowledge base โดยตรง ไม่ผ่าน LLM

### Basic Search

#### TypeScript

```typescript
const results = await client.search.query("วิธีการ deploy บน Docker");

console.log(results.results);
/*
[
  {
    filename: "deployment_guide.pdf",
    text: "การ deploy ด้วย Docker ทำได้โดย...",
    score: 0.9341,
    page: 5,
    mimetype: "application/pdf"
  },
  {
    filename: "docker_compose.md",
    text: "ใช้คำสั่ง docker-compose up -d เพื่อ...",
    score: 0.8876,
    page: null,
    mimetype: "text/markdown"
  }
]
*/
```

#### Python

```python
results = await client.search.query("วิธีการ deploy บน Docker")

for r in results.results:
    print(f"Score: {r.score:.4f} | File: {r.filename} | Page: {r.page}")
    print(f"  {r.text[:150]}...")
    print()
```

---

### Search พร้อม Filters

#### TypeScript

```typescript
const results = await client.search.query("authentication flow", {
  filters: {
    data_sources: ["auth_docs.pdf", "api_reference.md"],  // จำกัดไฟล์
    document_types: ["application/pdf"],                   // จำกัดประเภทไฟล์
  },
  limit: 5,
  scoreThreshold: 0.75,  // เอาเฉพาะที่มีความเกี่ยวข้องสูง
});
```

#### Python

```python
from openrag_sdk.models import SearchFilters

results = await client.search.query(
    "authentication flow",
    filters=SearchFilters(
        data_sources=["auth_docs.pdf", "api_reference.md"],
        document_types=["application/pdf"],
    ),
    limit=5,
    score_threshold=0.75,
)

# หรือใช้ dict แทน
results = await client.search.query(
    "authentication flow",
    filters={
        "data_sources": ["auth_docs.pdf"],
    },
    limit=5,
)
```

---

### Search ด้วย Knowledge Filter

```python
# สร้าง filter ก่อน (ดูหัวข้อ Knowledge Filters)
results = await client.search.query(
    "API endpoints",
    filter_id="filter_abc123",  # ใช้ saved filter
)
```

**ตัวอย่าง Search Response:**

```json
{
  "results": [
    {
      "filename": "api_reference.pdf",
      "text": "Authentication is handled via JWT tokens. Each request must include the Authorization header with format: Bearer <token>...",
      "score": 0.9521,
      "page": 12,
      "mimetype": "application/pdf"
    },
    {
      "filename": "security_guide.md",
      "text": "The authentication flow consists of: 1. User submits credentials, 2. Server validates and issues JWT...",
      "score": 0.8934,
      "page": null,
      "mimetype": "text/markdown"
    }
  ]
}
```

### Search Parameters

| Parameter | Type | Default | ความหมาย |
|-----------|------|---------|-----------|
| `query` | `string` | **required** | คำค้นหา |
| `filters` | `SearchFilters \| null` | `null` | กรองแหล่งข้อมูล |
| `limit` | `number` | `10` | จำนวนผลลัพธ์สูงสุด |
| `scoreThreshold` / `score_threshold` | `number` | `0` | คะแนนขั้นต่ำ |
| `filterId` / `filter_id` | `string \| null` | `null` | Knowledge filter ID |

---

## Documents API

### Ingest Document

อัปโหลดและ index เอกสารเข้า knowledge base

#### TypeScript (Node.js — จาก file path)

```typescript
// อัปโหลดและรอจนเสร็จ (default: wait=true)
const status = await client.documents.ingest({
  filePath: "./docs/manual.pdf",
});

console.log(status.status);          // → "completed"
console.log(status.task_id);         // → "task_abc123"
console.log(status.successful_files); // → 1
console.log(status.failed_files);    // → 0
```

#### TypeScript (Browser — จาก File object)

```typescript
// รับ File จาก <input type="file">
const fileInput = document.getElementById("file") as HTMLInputElement;
const file = fileInput.files![0];

const status = await client.documents.ingest({
  file: file,
  filename: file.name,
});
```

#### Python (จาก file path)

```python
status = await client.documents.ingest("./docs/manual.pdf")

print(f"Status: {status.status}")
print(f"Task ID: {status.task_id}")
print(f"Successful: {status.successful_files}/{status.total_files}")
```

#### Python (จาก file object)

```python
with open("./docs/manual.pdf", "rb") as f:
    status = await client.documents.ingest(
        file=f,
        filename="manual.pdf",
    )
```

**ตัวอย่าง IngestTaskStatus Response:**

```json
{
  "task_id": "task_7f3a9b2c1d4e5f6g",
  "status": "completed",
  "total_files": 1,
  "processed_files": 1,
  "successful_files": 1,
  "failed_files": 0,
  "files": {
    "manual.pdf": {
      "status": "completed",
      "chunks_created": 23,
      "indexed_at": "2026-03-17T10:05:30Z"
    }
  }
}
```

---

### Ingest แบบ Non-blocking (background)

```typescript
// TypeScript: ไม่รอ ได้ task_id กลับมาทันที
const response = await client.documents.ingest({
  filePath: "./large_doc.pdf",
  wait: false,
});
console.log(response.task_id);  // → "task_abc123"

// ตรวจสอบ status ทีหลัง
const status = await client.documents.getTaskStatus(response.task_id);
console.log(status.status);  // → "running"

// รอจนเสร็จ
const finalStatus = await client.documents.waitForTask(response.task_id);
console.log(finalStatus.status);  // → "completed"
```

```python
# Python
response = await client.documents.ingest("./large_doc.pdf", wait=False)
task_id = response.task_id

# ตรวจสอบ status
status = await client.documents.get_task_status(task_id)
print(status.status)  # → "running"

# รอจนเสร็จ (พร้อม custom timeout)
final = await client.documents.wait_for_task(
    task_id,
    poll_interval=2.0,   # เช็คทุก 2 วินาที
    timeout=600.0,        # รอสูงสุด 10 นาที
)
print(final.status)  # → "completed"
```

---

### Task Status Values

| Status | ความหมาย |
|--------|-----------|
| `pending` | รออยู่ใน queue |
| `running` | กำลัง process |
| `completed` | เสร็จสมบูรณ์ |
| `failed` | เกิดข้อผิดพลาด |

---

### Delete Document

```typescript
// TypeScript
const result = await client.documents.delete("manual.pdf");
console.log(result.success);        // → true
console.log(result.deleted_chunks); // → 23 (จำนวน chunks ที่ลบ)
```

```python
# Python
result = await client.documents.delete("manual.pdf")
print(result.success)         # → True
print(result.deleted_chunks)  # → 23
```

**ตัวอย่าง Response:**
```json
{
  "success": true,
  "deleted_chunks": 23
}
```

> **หมายเหตุ**: ลบโดยใช้ `filename` ถ้ามีไฟล์ชื่อซ้ำกันหลายไฟล์จะลบทั้งหมด

---

### Ingest Options ทั้งหมด

| Parameter | Type | Default | ความหมาย |
|-----------|------|---------|-----------|
| `filePath` / `file_path` | `string \| Path` | - | Path ไฟล์ (Node.js/Python) |
| `file` | `File \| Blob \| BinaryIO` | - | File object (Browser/Python) |
| `filename` | `string` | - | ชื่อไฟล์ (จำเป็นเมื่อใช้ file object) |
| `wait` | `boolean` | `true` | รอจน ingest เสร็จหรือไม่ |
| `pollInterval` / `poll_interval` | `number` | `1` | วินาทีระหว่าง status check |
| `timeout` | `number` | `300` | วินาทีสูงสุดที่รอ |

---

## Settings API

ดูและแก้ไข configuration ของ OpenRAG

### ดู Settings ปัจจุบัน

#### TypeScript

```typescript
const settings = await client.settings.get();
console.log(settings);
/*
{
  agent: {
    llm_provider: "openai",
    llm_model: "gpt-4o",
    system_prompt: "You are a helpful assistant..."
  },
  knowledge: {
    embedding_provider: "openai",
    embedding_model: "text-embedding-3-small",
    chunk_size: 1000,
    chunk_overlap: 200,
    table_structure: true,
    ocr: false,
    picture_descriptions: false
  }
}
*/
```

#### Python

```python
settings = await client.settings.get()
print(f"LLM: {settings.agent.llm_provider}/{settings.agent.llm_model}")
print(f"Embedding: {settings.knowledge.embedding_provider}/{settings.knowledge.embedding_model}")
print(f"Chunk size: {settings.knowledge.chunk_size}")
```

---

### อัพเดต Settings

#### TypeScript

```typescript
await client.settings.update({
  llm_model: "gpt-4o-mini",
  chunk_size: 500,
  chunk_overlap: 50,
  table_structure: true,
  ocr: true,
});
```

#### Python

```python
await client.settings.update(
    llm_model="gpt-4o-mini",
    chunk_size=500,
    chunk_overlap=50,
    table_structure=True,
    ocr=True,
)
```

**Settings Parameters ทั้งหมด:**

| Parameter | Type | ความหมาย |
|-----------|------|-----------|
| `llm_model` | `string` | LLM model name |
| `llm_provider` | `string` | LLM provider (openai/anthropic/ollama) |
| `system_prompt` | `string` | System prompt สำหรับ chat |
| `embedding_model` | `string` | Embedding model name |
| `embedding_provider` | `string` | Embedding provider |
| `chunk_size` | `number` | ขนาด chunk (characters) |
| `chunk_overlap` | `number` | จำนวน characters ที่ overlap |
| `table_structure` | `boolean` | แยก table เป็น chunk ต่างหาก |
| `ocr` | `boolean` | เปิด OCR สำหรับ scanned documents |
| `picture_descriptions` | `boolean` | สกัดคำอธิบาย images |

---

## Knowledge Filters API

Knowledge Filters คือ saved search presets ที่ pre-configure query parameters ไว้ ช่วยกรองข้อมูลได้โดยไม่ต้องส่ง filters ทุกครั้ง

### สร้าง Filter

#### TypeScript

```typescript
const filter = await client.knowledgeFilters.create({
  name: "Technical Documentation Only",
  description: "กรองเฉพาะเอกสาร technical",
  queryData: {
    filters: {
      data_sources: ["api_docs.pdf", "architecture.md"],
      document_types: ["application/pdf"],
    },
    limit: 5,
    scoreThreshold: 0.8,
    color: "#3B82F6",
    icon: "BookOpen",
  },
});

console.log(filter.id);   // → "filter_abc123"
console.log(filter.name); // → "Technical Documentation Only"
```

#### Python

```python
from openrag_sdk.models import CreateKnowledgeFilterOptions, KnowledgeFilterQueryData

filter_result = await client.knowledge_filters.create(
    CreateKnowledgeFilterOptions(
        name="Technical Documentation Only",
        description="กรองเฉพาะเอกสาร technical",
        query_data=KnowledgeFilterQueryData(
            filters={
                "data_sources": ["api_docs.pdf", "architecture.md"],
                "document_types": ["application/pdf"],
            },
            limit=5,
            score_threshold=0.8,
            color="#3B82F6",
            icon="BookOpen",
        ),
    )
)

print(filter_result.id)  # → "filter_abc123"
```

**ตัวอย่าง Response:**
```json
{
  "success": true,
  "id": "filter_abc123def456",
  "error": null
}
```

---

### ค้นหา Filters

#### TypeScript

```typescript
// ค้นหาทั้งหมด
const filters = await client.knowledgeFilters.search();

// ค้นหาด้วย keyword
const techFilters = await client.knowledgeFilters.search("technical", 10);

for (const f of techFilters) {
  console.log(`${f.id}: ${f.name} - ${f.description}`);
}
```

#### Python

```python
filters = await client.knowledge_filters.search("technical", limit=10)
for f in filters:
    print(f"{f.id}: {f.name}")
    if f.query_data:
        print(f"  Sources: {f.query_data.filters}")
```

**ตัวอย่าง Response:**
```json
{
  "success": true,
  "filters": [
    {
      "id": "filter_abc123",
      "name": "Technical Documentation Only",
      "description": "กรองเฉพาะเอกสาร technical",
      "query_data": {
        "filters": {
          "data_sources": ["api_docs.pdf"],
          "document_types": ["application/pdf"]
        },
        "limit": 5,
        "scoreThreshold": 0.8,
        "color": "#3B82F6",
        "icon": "BookOpen"
      },
      "owner": "user_xyz",
      "created_at": "2026-03-17T09:00:00Z",
      "updated_at": "2026-03-17T09:00:00Z"
    }
  ]
}
```

---

### ดู / อัพเดต / ลบ Filter

#### TypeScript

```typescript
// ดู filter เดียว
const filter = await client.knowledgeFilters.get("filter_abc123");
console.log(filter?.queryData.limit);  // → 5

// อัพเดต
const updated = await client.knowledgeFilters.update("filter_abc123", {
  name: "Tech Docs (Updated)",
  queryData: {
    limit: 10,
  },
});
console.log(updated);  // → true

// ลบ
const deleted = await client.knowledgeFilters.delete("filter_abc123");
console.log(deleted);  // → true
```

#### Python

```python
# ดู
f = await client.knowledge_filters.get("filter_abc123")
print(f.query_data.limit if f else "Not found")

# อัพเดต
updated = await client.knowledge_filters.update(
    "filter_abc123",
    {"name": "Tech Docs (Updated)", "queryData": {"limit": 10}},
)
print(updated)  # → True

# ลบ
deleted = await client.knowledge_filters.delete("filter_abc123")
print(deleted)  # → True
```

---

### ใช้ Filter กับ Chat/Search

```typescript
// TypeScript
const response = await client.chat.create({
  message: "อธิบาย API authentication",
  filterId: "filter_abc123",  // ใช้ filter ที่สร้างไว้
});

const results = await client.search.query("API endpoints", {
  filterId: "filter_abc123",
});
```

```python
# Python
response = await client.chat.create(
    message="อธิบาย API authentication",
    filter_id="filter_abc123",
)

results = await client.search.query(
    "API endpoints",
    filter_id="filter_abc123",
)
```

---

## Models API

ดูรายการ models ที่ใช้งานได้ (Python เท่านั้น)

```python
# ดู models ของ OpenAI
models = await client.models.list("openai")
print("Language Models:", [m.value for m in models.language_models])
# → ["gpt-4o", "gpt-4o-mini", "gpt-3.5-turbo"]
print("Embedding Models:", [m.value for m in models.embedding_models])
# → ["text-embedding-3-small", "text-embedding-3-large"]

# providers ที่รองรับ: "openai", "anthropic", "ollama", "watsonx"
anthropic_models = await client.models.list("anthropic")
ollama_models = await client.models.list("ollama")
```

**ตัวอย่าง Response:**
```json
{
  "language_models": [
    { "value": "gpt-4o", "label": "GPT-4o", "default": true },
    { "value": "gpt-4o-mini", "label": "GPT-4o Mini", "default": false }
  ],
  "embedding_models": [
    { "value": "text-embedding-3-small", "label": "Text Embedding 3 Small", "default": true },
    { "value": "text-embedding-3-large", "label": "Text Embedding 3 Large", "default": false }
  ]
}
```

---

## Error Handling

### Error Class Hierarchy

```
OpenRAGError (base)
├── AuthenticationError  → 401: API key ไม่ถูกต้องหรือหมดอายุ
├── NotFoundError        → 404: ไม่พบ resource ที่ร้องขอ
├── ValidationError      → 422: ข้อมูลที่ส่งไม่ถูกต้อง
├── RateLimitError       → 429: เกิน rate limit
└── ServerError          → 5xx: Server error
```

### TypeScript

```typescript
import {
  OpenRAGError,
  AuthenticationError,
  NotFoundError,
  ValidationError,
  RateLimitError,
  ServerError,
} from "openrag-sdk";

try {
  const response = await client.chat.create({ message: "Hello" });
} catch (error) {
  if (error instanceof AuthenticationError) {
    console.error("Invalid API key:", error.message);
    // แนะนำ: ตรวจสอบ OPENRAG_API_KEY
  } else if (error instanceof NotFoundError) {
    console.error("Resource not found:", error.message);
  } else if (error instanceof ValidationError) {
    console.error("Invalid request:", error.message);
    // แนะนำ: ตรวจสอบ parameters ที่ส่งไป
  } else if (error instanceof RateLimitError) {
    console.error("Rate limit exceeded:", error.message);
    // แนะนำ: รอแล้วลองใหม่
  } else if (error instanceof ServerError) {
    console.error("Server error:", error.message, "Status:", error.statusCode);
  } else if (error instanceof OpenRAGError) {
    console.error("OpenRAG error:", error.message);
  }
}
```

### Python

```python
from openrag_sdk.exceptions import (
    OpenRAGError,
    AuthenticationError,
    NotFoundError,
    ValidationError,
    RateLimitError,
    ServerError,
)

try:
    response = await client.chat.create(message="Hello")
except AuthenticationError as e:
    print(f"Invalid API key: {e}")
except NotFoundError as e:
    print(f"Not found: {e}")
except ValidationError as e:
    print(f"Invalid request: {e}")
except RateLimitError as e:
    print(f"Rate limited: {e}")
    await asyncio.sleep(60)  # รอแล้วลองใหม่
except ServerError as e:
    print(f"Server error (HTTP {e.status_code}): {e}")
except OpenRAGError as e:
    print(f"OpenRAG error: {e}")
```

### Retry Pattern (Python)

```python
import asyncio

async def chat_with_retry(client, message, max_retries=3):
    for attempt in range(max_retries):
        try:
            return await client.chat.create(message=message)
        except RateLimitError:
            if attempt < max_retries - 1:
                wait_time = 2 ** attempt  # exponential backoff: 1s, 2s, 4s
                print(f"Rate limited, waiting {wait_time}s...")
                await asyncio.sleep(wait_time)
            else:
                raise
        except ServerError:
            if attempt < max_retries - 1:
                await asyncio.sleep(1)
            else:
                raise
```

---

## ตัวอย่าง Use Cases จริง

### Use Case 1: RAG Chatbot แบบ Production

```python
# Python — Full-featured chatbot
import asyncio
import os
from openrag_sdk import OpenRAGClient
from openrag_sdk.exceptions import OpenRAGError


async def chatbot():
    async with OpenRAGClient(
        api_key=os.environ["OPENRAG_API_KEY"],
        base_url=os.environ.get("OPENRAG_URL", "http://localhost:3000"),
    ) as client:
        chat_id = None
        print("RAG Chatbot พร้อมแล้ว (พิมพ์ 'quit' เพื่อออก)\n")

        while True:
            user_input = input("You: ").strip()
            if user_input.lower() == "quit":
                break
            if not user_input:
                continue

            try:
                print("Bot: ", end="", flush=True)
                async with client.chat.stream(
                    message=user_input,
                    chat_id=chat_id,
                    limit=5,
                    score_threshold=0.6,
                ) as stream:
                    async for text in stream.text_stream:
                        print(text, end="", flush=True)

                    chat_id = stream.chat_id  # เก็บ ID ต่อบทสนทนา
                    print()  # newline

                    if stream.sources:
                        print(f"\n[Sources: {', '.join(s.filename for s in stream.sources[:3])}]\n")

            except OpenRAGError as e:
                print(f"\nError: {e}\n")


asyncio.run(chatbot())
```

---

### Use Case 2: Document Ingestion Pipeline

```python
import asyncio
from pathlib import Path
from openrag_sdk import OpenRAGClient


async def ingest_directory(directory: str, pattern: str = "**/*.pdf"):
    async with OpenRAGClient() as client:
        files = list(Path(directory).glob(pattern))
        print(f"พบ {len(files)} ไฟล์")

        results = []
        for i, file_path in enumerate(files, 1):
            print(f"[{i}/{len(files)}] กำลัง ingest: {file_path.name}")
            try:
                status = await client.documents.ingest(
                    file_path,
                    poll_interval=2.0,
                    timeout=300.0,
                )
                if status.status == "completed":
                    print(f"  ✓ เสร็จ: {status.successful_files} files indexed")
                else:
                    print(f"  ✗ ล้มเหลว: {status.status}")
                results.append({"file": file_path.name, "status": status.status})
            except Exception as e:
                print(f"  ✗ Error: {e}")
                results.append({"file": file_path.name, "status": "error", "error": str(e)})

        successful = sum(1 for r in results if r["status"] == "completed")
        print(f"\nสรุป: {successful}/{len(files)} ไฟล์ สำเร็จ")
        return results


asyncio.run(ingest_directory("./documents", "**/*.pdf"))
```

---

### Use Case 3: Semantic Search + Re-ranking

```typescript
// TypeScript — Search หลาย queries และ merge ผลลัพธ์
async function searchAndRank(
  client: OpenRAGClient,
  queries: string[],
  topK: number = 5
) {
  const allResults = new Map<string, { score: number; result: SearchResult }>();

  for (const query of queries) {
    const { results } = await client.search.query(query, {
      limit: 10,
      scoreThreshold: 0.5,
    });

    for (const r of results) {
      const key = `${r.filename}:${r.page}`;
      const existing = allResults.get(key);
      // เอาคะแนนสูงสุดจากทุก query
      if (!existing || r.score > existing.score) {
        allResults.set(key, { score: r.score, result: r });
      }
    }
  }

  // Sort by score และเอา top K
  return Array.from(allResults.values())
    .sort((a, b) => b.score - a.score)
    .slice(0, topK)
    .map((v) => v.result);
}

// Usage
const results = await searchAndRank(client, [
  "Docker deployment steps",
  "container orchestration",
  "kubernetes vs docker-compose",
]);
console.log(`Top ${results.length} results:`);
results.forEach((r, i) => {
  console.log(`${i + 1}. [${r.score.toFixed(3)}] ${r.filename} p.${r.page}`);
});
```

---

### Use Case 4: Batch Ingestion (Parallel)

```typescript
// TypeScript — Ingest หลายไฟล์พร้อมกัน
import { readdir } from "fs/promises";
import { join } from "path";

async function batchIngest(client: OpenRAGClient, directory: string) {
  const files = (await readdir(directory))
    .filter((f) => f.endsWith(".pdf") || f.endsWith(".txt"));

  console.log(`Ingesting ${files.length} files...`);

  // รัน parallel แต่จำกัด concurrency ที่ 3
  const concurrency = 3;
  const chunks = [];
  for (let i = 0; i < files.length; i += concurrency) {
    chunks.push(files.slice(i, i + concurrency));
  }

  const results = [];
  for (const chunk of chunks) {
    const batch = await Promise.allSettled(
      chunk.map((file) =>
        client.documents.ingest({ filePath: join(directory, file) })
      )
    );
    results.push(...batch);
  }

  const succeeded = results.filter((r) => r.status === "fulfilled").length;
  console.log(`Done: ${succeeded}/${files.length} succeeded`);
}
```

---

### Use Case 5: Knowledge Filter + Scoped Chat

```python
# Python — สร้าง filter สำหรับแผนกต่างๆ
async def setup_department_filters(client):
    departments = {
        "HR": {
            "sources": ["hr_policy.pdf", "employee_handbook.pdf"],
            "color": "#10B981",
        },
        "Engineering": {
            "sources": ["api_docs.pdf", "architecture.md", "runbooks.pdf"],
            "color": "#3B82F6",
        },
        "Finance": {
            "sources": ["financial_report.pdf", "budget_guide.pdf"],
            "color": "#F59E0B",
        },
    }

    filter_ids = {}
    for dept, config in departments.items():
        result = await client.knowledge_filters.create({
            "name": f"{dept} Department",
            "queryData": {
                "filters": {"data_sources": config["sources"]},
                "limit": 8,
                "scoreThreshold": 0.7,
                "color": config["color"],
            },
        })
        filter_ids[dept] = result.id
        print(f"Created filter for {dept}: {result.id}")

    return filter_ids


# Usage
async with OpenRAGClient() as client:
    filter_ids = await setup_department_filters(client)

    # Chat กับข้อมูล HR เท่านั้น
    hr_response = await client.chat.create(
        message="วันลาพักร้อนมีกี่วัน?",
        filter_id=filter_ids["HR"],
    )
    print(hr_response.response)
```

---

## Type Reference

### Python Models (Pydantic)

```python
class Source(BaseModel):
    filename: str
    text: str
    score: float
    page: int | None = None
    mimetype: str | None = None

class ChatResponse(BaseModel):
    response: str
    chat_id: str | None = None
    sources: list[Source] = []

class ContentEvent(BaseModel):
    type: Literal["content"] = "content"
    delta: str

class SourcesEvent(BaseModel):
    type: Literal["sources"] = "sources"
    sources: list[Source]

class DoneEvent(BaseModel):
    type: Literal["done"] = "done"
    chat_id: str | None = None

StreamEvent = ContentEvent | SourcesEvent | DoneEvent

class SearchResult(BaseModel):
    filename: str
    text: str
    score: float
    page: int | None = None
    mimetype: str | None = None

class SearchFilters(BaseModel):
    data_sources: list[str] | None = None
    document_types: list[str] | None = None

class SearchResponse(BaseModel):
    results: list[SearchResult]

class IngestResponse(BaseModel):
    task_id: str
    status: str | None = None
    filename: str | None = None

class IngestTaskStatus(BaseModel):
    task_id: str
    status: str  # "pending" | "running" | "completed" | "failed"
    total_files: int = 0
    processed_files: int = 0
    successful_files: int = 0
    failed_files: int = 0
    files: dict = {}

class DeleteDocumentResponse(BaseModel):
    success: bool
    deleted_chunks: int = 0

class Conversation(BaseModel):
    chat_id: str
    title: str
    created_at: str | None = None
    last_activity: str | None = None
    message_count: int = 0

class Message(BaseModel):
    role: str  # "user" | "assistant"
    content: str
    timestamp: str | None = None

class ConversationDetail(Conversation):
    messages: list[Message] = []

class KnowledgeFilterQueryData(BaseModel):
    query: str | None = None
    filters: dict[str, list[str]] | None = None
    limit: int | None = None
    score_threshold: float | None = None
    color: str | None = None
    icon: str | None = None

class KnowledgeFilter(BaseModel):
    id: str
    name: str
    description: str | None = None
    query_data: KnowledgeFilterQueryData | None = None
    owner: str | None = None
    created_at: str | None = None
    updated_at: str | None = None
```

### TypeScript Types

```typescript
interface Source {
  filename: string;
  text: string;
  score: number;
  page?: number | null;
  mimetype?: string | null;
}

interface ChatResponse {
  response: string;
  chatId?: string | null;
  sources: Source[];
}

interface ContentEvent { type: "content"; delta: string; }
interface SourcesEvent { type: "sources"; sources: Source[]; }
interface DoneEvent { type: "done"; chatId?: string | null; }
type StreamEvent = ContentEvent | SourcesEvent | DoneEvent;

interface ChatCreateOptions {
  message: string;
  stream?: boolean;
  chatId?: string;
  filters?: SearchFilters;
  limit?: number;
  scoreThreshold?: number;
  filterId?: string;
}

interface SearchFilters {
  data_sources?: string[];
  document_types?: string[];
}

interface SearchResult {
  filename: string;
  text: string;
  score: number;
  page?: number | null;
  mimetype?: string | null;
}

interface IngestOptions {
  filePath?: string;
  file?: File | Blob;
  filename?: string;
  wait?: boolean;
  pollInterval?: number;
  timeout?: number;
}

interface IngestTaskStatus {
  task_id: string;
  status: string;
  total_files: number;
  processed_files: number;
  successful_files: number;
  failed_files: number;
  files: Record<string, unknown>;
}

interface DeleteDocumentResponse {
  success: boolean;
  deleted_chunks: number;
}
```

---

## Quick Reference

### TypeScript Cheatsheet

```typescript
import { OpenRAGClient } from "openrag-sdk";
const client = new OpenRAGClient({ apiKey: "orag_..." });

// Chat (non-stream)
const r = await client.chat.create({ message: "Hello" });
r.response; r.chatId; r.sources;

// Chat (stream)
using stream = await client.chat.stream({ message: "Hello" });
for await (const e of stream) { if (e.type === "content") process.stdout.write(e.delta); }
stream.text; stream.chatId; stream.sources;

// Search
const s = await client.search.query("query", { limit: 5, scoreThreshold: 0.7 });
s.results; // [{ filename, text, score, page, mimetype }]

// Ingest
const t = await client.documents.ingest({ filePath: "./doc.pdf" });
t.status; t.successful_files;

// Delete
await client.documents.delete("doc.pdf");

// Settings
await client.settings.update({ chunk_size: 500 });
```

### Python Cheatsheet

```python
from openrag_sdk import OpenRAGClient

async with OpenRAGClient(api_key="orag_...") as client:
    # Chat (non-stream)
    r = await client.chat.create(message="Hello")
    r.response; r.chat_id; r.sources

    # Chat (stream)
    async with client.chat.stream(message="Hello") as s:
        async for text in s.text_stream:
            print(text, end="")
        s.text; s.chat_id; s.sources

    # Search
    r = await client.search.query("query", limit=5, score_threshold=0.7)
    r.results  # [Source(filename, text, score, page, mimetype)]

    # Ingest
    t = await client.documents.ingest("./doc.pdf")
    t.status; t.successful_files

    # Delete
    await client.documents.delete("doc.pdf")

    # Settings
    await client.settings.update(chunk_size=500)
```

---

*Generated: 2026-03-17 | OpenRAG SDK Documentation v0.2.0*
