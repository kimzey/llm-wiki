---
title: "OpenRAG SDK Reference"
type: source
source_file: raw/notes/openrag/docs-lean/sdk.md
published: 2026-03-17
tags: [openrag, sdk, typescript, python, api, chat, documents, knowledge-filters]
related: [wiki/concepts/openrag-platform.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/openrag/docs-lean/sdk.md|Original file]]

## สรุป

OpenRAG SDK สำหรับ TypeScript/JavaScript และ Python — ครอบคลุม Client setup, Chat API, Search, Documents, KnowledgeFilters API

## ประเด็นสำคัญ

### SDK Overview

| | TypeScript/JS | Python |
|---|---|---|
| Package | `openrag-sdk` | `openrag-sdk` |
| Version | 0.2.0 | 0.2.0 |
| Runtime | Node.js ≥18 | Python ≥3.10 |
| HTTP Client | Native `fetch` | `httpx` (async) |
| Validation | TypeScript types | Pydantic v2 |

### Sub-clients

```
OpenRAGClient
├── .chat              → Chat + Conversation management
├── .search            → Semantic search
├── .documents         → Ingest + Delete documents
├── .settings          → Configuration
├── .models            → List available LLM/embedding models
└── .knowledgeFilters  → Knowledge filter management
```

### API Key Format

```
orag_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

สร้างผ่าน REST API (ต้องมี JWT ก่อน): `POST /keys`

### Client Initialization

```typescript
// TypeScript
const client = new OpenRAGClient(); // อ่าน OPENRAG_API_KEY, OPENRAG_URL อัตโนมัติ
// หรือ
const client = new OpenRAGClient({
  apiKey: "orag_...",
  baseUrl: "http://localhost:3000",
});
```

```python
# Python — ต้องใช้ async with
async with OpenRAGClient() as client:
    response = await client.chat.create(message="Hello")
```

### Chat API

```typescript
// Non-streaming
const response = await client.chat.create({ message: "ลาพักร้อนได้กี่วัน?" });
console.log(response.response);    // คำตอบ
console.log(response.chatId);      // chat_abc123 (ใช้ต่อ conversation)
console.log(response.sources);     // [{ filename, text, score, page }]

// Streaming
const stream = await client.chat.stream({ message: "..." });
for await (const chunk of stream) {
  process.stdout.write(chunk.token ?? "");
}
```

```python
# Python
response = await client.chat.create(message="...")
print(response.chat_id)    # Python ใช้ snake_case
```

### Documents API

```python
# Ingest
status = await client.documents.ingest("report.pdf")
# status.status: "completed" | "unchanged" | "failed"
# status.successful_files: count

# Delete
result = await client.documents.delete("old_report.pdf")
# result.deleted_chunks: count

# List documents
docs = await client.documents.list()
# docs[0]: { document_id, filename, page_count, indexed_time }

# Search
results = await client.documents.search(query="ลาป่วย", top_k=5)
```

### Knowledge Filters API

```python
# List
filters = await client.knowledge_filters.list()

# Create
new_filter = await client.knowledge_filters.create(
    name="HR Policies",
    description="นโยบาย HR",
    allowed_groups=["all_employees"]
)

# Use in chat
response = await client.chat.create(
    message="ลาพักร้อนได้กี่วัน?",
    knowledge_filter_id=new_filter.id
)
```

### Weekly Update Pattern (Python SDK)

```python
async def weekly_update(client, directory):
    files = list(Path(directory).glob("*.pdf"))
    for file_path in files:
        status = await client.documents.ingest(file_path, poll_interval=2.0, timeout=300.0)
        if status.status == "unchanged":
            print(f"  ~ skip {file_path.name}")  # hash เหมือนเดิม ไม่เสีย embedding cost
```

### Streaming — ChatStream Context Manager (แนะนำ)

```typescript
// TypeScript — ES2022 using syntax
using stream = await client.chat.stream({ message: "อธิบาย ML", limit: 5, scoreThreshold: 0.7 });
for await (const event of stream) {
  if (event.type === "content") process.stdout.write(event.delta);
  else if (event.type === "sources") console.log("Sources:", event.sources.length);
}
console.log("Full text:", stream.text);   // เข้าถึงหลัง stream จบ
```

```python
# Python — async with
async with client.chat.stream(message="อธิบาย ML", limit=5, score_threshold=0.7) as stream:
    async for event in stream:
        if event.type == "content": print(event.delta, end="", flush=True)
    print(f"\nFull: {stream.text}")
```

**Text-only (ง่ายสุด):**
```typescript
using stream = await client.chat.stream({ message: "Hello" });
for await (const text of stream.textStream) process.stdout.write(text);
const full = await stream.finalText();  // รับทั้งก้อน
```

### Multi-turn Conversation

```typescript
const r1 = await client.chat.create({ message: "ฉันชื่อ Alice" });
const r2 = await client.chat.create({ message: "ฉันชื่ออะไร?", chatId: r1.chatId });
// → "คุณชื่อ Alice"
```

### Conversation Management

```typescript
const { conversations } = await client.chat.list();
const detail = await client.chat.get("chat_7f3a9b2c");  // ประวัติ messages เต็ม
await client.chat.delete("chat_7f3a9b2c");
```

### Chat Parameters ครบ

| Parameter | Type | Default | ความหมาย |
|-----------|------|---------|-----------|
| `message` | string | **required** | ข้อความ |
| `stream` | boolean | false | เปิด streaming |
| `chatId` / `chat_id` | string | null | ต่อ conversation |
| `filters` | SearchFilters | null | กรอง data sources |
| `limit` | number | 10 | จำนวน chunks |
| `scoreThreshold` / `score_threshold` | number | 0 | relevance ขั้นต่ำ |
| `filterId` / `filter_id` | string | null | Knowledge filter ID |

### Documents API — Non-blocking

```python
# อัปโหลด + ไม่รอ (background)
response = await client.documents.ingest("large.pdf", wait=False)
task_id = response.task_id

# รอทีหลัง
final = await client.documents.wait_for_task(task_id, poll_interval=2.0, timeout=600.0)
print(final.status)  # → "completed"
```

**Task Status:** `pending` → `running` → `completed` / `failed`

### Settings API

```python
settings = await client.settings.get()
# { agent: { llm_provider, llm_model, system_prompt }, knowledge: { chunk_size, chunk_overlap, ... } }

await client.settings.update(llm_model="gpt-4o-mini", chunk_size=500, chunk_overlap=50, ocr=True)
```

### Models API (Python only)

```python
models = await client.models.list("openai")
# models.language_models: ["gpt-4o", "gpt-4o-mini"]
# models.embedding_models: ["text-embedding-3-small", "text-embedding-3-large"]
# providers: "openai", "anthropic", "ollama", "watsonx"
```

### Error Handling

```
OpenRAGError (base)
├── AuthenticationError  → 401
├── NotFoundError        → 404
├── ValidationError      → 422
├── RateLimitError       → 429
└── ServerError          → 5xx
```

```python
try:
    r = await client.chat.create(message="Hello")
except RateLimitError:
    await asyncio.sleep(60)  # retry
except AuthenticationError:
    print("ตรวจสอบ OPENRAG_API_KEY")
```

**Retry Pattern (exponential backoff):**
```python
for attempt in range(3):
    try:
        return await client.chat.create(message=msg)
    except RateLimitError:
        await asyncio.sleep(2 ** attempt)  # 1s, 2s, 4s
```

### TypeScript Type Reference ย่อ

```typescript
interface Source { filename, text, score, page?, mimetype? }
interface ChatResponse { response, chatId?, sources: Source[] }
type StreamEvent = ContentEvent | SourcesEvent | DoneEvent
interface SearchFilters { data_sources?, document_types? }
interface IngestTaskStatus { task_id, status, successful_files, failed_files, files }
```

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/openrag-platform|OpenRAG Platform]] — platform overview
