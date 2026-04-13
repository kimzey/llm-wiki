# Arona - เอกสารสอนการใช้งานระบบ

## สารบัญ

1. [ภาพรวมระบบ](#ภาพรวมระบบ)
2. [สถาปัตยกรรม](#สถาปัตยกรรม)
3. [โครงสร้างโปรเจกต์](#โครงสร้างโปรเจกต์)
4. [การทำงานของระบบ](#การทำงานของระบบ)
5. [การติดตั้งและตั้งค่า](#การติดตั้งและตั้งค่า)
6. [API Documentation](#api-documentation)
7. [ตัวอย่างการใช้งาน](#ตัวอย่างการใช้งาน)
8. [การแก้ไขและพัฒนาต่อยอด](#การแก้ไขและพัฒนาต่อยอด)

---

## ภาพรวมระบบ

**Arona** เป็นระบบค้นหาเอกสารด้วย AI โดยใช้เทคนิค RAG (Retrieval-Augmented Generation) สำหรับเอกสารของ Elysia Framework

### ทำไมต้องสร้างระบบเอง?

1. **ราคาแพง** - RaaS (RAG as a Service) providers อย่าง Kapa, Mendable มีราคาสูงถึง $1,000/เดือน
2. **ความยืดหยุ่น** - ต้องการควบคุมข้อมูลและฟีเจอร์เอง
3. **การลงทุนเดิม** - มีเอกสารบน VitePress แล้ว ไม่ต้องการย้าย

### สิ่งที่ระบบทำได้

- ค้นหาเอกสารด้วยภาษาธรรมชาติ
- ตอบคำถามเกี่ยวกับ Elysia Framework
- อ้างอิงแหล่งข้อมูลอัตโนมัติ
- Semantic Caching สำหรับประสิทธิภาพ
- ป้องกันการใช้งานโดยบอท

---

## สถาปัตยกรรม

```
┌─────────────────────────────────────────────────────────────────┐
│                         Client Application                       │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Arona API Server (Elysia)                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Security Layer                                            │  │
│  │  - Proof of Work (PoW)                                     │  │
│  │  - Cloudflare Turnstile                                    │  │
│  │  - Rate Limiting                                           │  │
│  │  - Checksum Verification                                   │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Cache Layer                                               │  │
│  │  - Redis (DragonflyDB)                                     │  │
│  │  - Semantic Cache (Vector Search)                          │  │
│  │  - LRU In-Memory Cache                                     │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  AI Engine                                                 │  │
│  │  - GPT OSS 120B (Main Model)                               │  │
│  │  - GPT OSS 20B (Query Normalization)                       │  │
│  │  - OpenAI Embeddings (text-embedding-3-small)              │  │
│  └───────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  Search Layer                                              │  │
│  │  - BM25 Full-Text Search                                   │  │
│  │  - Vector Semantic Search                                  │  │
│  │  - Hybrid Scoring                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  ParadeDB     │   │  DragonflyDB  │   │   OpenRouter  │
│  (PostgreSQL) │   │  (Redis)      │   │   (AI Models) │
│               │   │               │   │               │
│ - Documents   │   │ - Cache       │   │ - GPT OSS     │
│ - Embeddings  │   │ - Semantic    │   │ - Embeddings  │
│ - BM25 Index  │   │   Search      │   │               │
└───────────────┘   └───────────────┘   └───────────────┘
        │
        ▼
┌─────────────────────────────────┐
│  Background Processes           │
│  - Indexing (Every 12 hours)    │
│  - Webhook Re-index Trigger     │
│  - Cluster Workers              │
└─────────────────────────────────┘
```

---

## โครงสร้างโปรเจกต์

```
arona/
├── src/
│   ├── index.ts              # Entry point with clustering
│   ├── server.ts             # Elysia server setup
│   ├── libs/                 # Core libraries
│   │   ├── ai.ts            # AI providers & models
│   │   ├── cache.ts         # Generic caching wrapper
│   │   ├── database.ts      # Database connection
│   │   ├── embedding.ts     # Text embedding generation
│   │   ├── flags.ts         # Environment flags
│   │   ├── ip.ts            # IP extraction utilities
│   │   ├── log.ts           # Logging system
│   │   ├── rate-limit.ts    # Rate limiting logic
│   │   ├── redis.ts         # Redis client setup
│   │   ├── retry.ts         # Retry mechanism
│   │   ├── structure.ts     # Indexing & structure management
│   │   ├── turnstile.ts     # Cloudflare Turnstile
│   │   ├── burst-cache.ts   # LRU burst cache
│   │   ├── promise.ts       # Promise utilities
│   │   └── index.ts         # Barrel exports
│   └── modules/              # Feature modules
│       ├── ai/              # AI/RAG module
│       │   ├── index.ts     # AI routes & handlers
│       │   ├── service.ts   # Core AI logic
│       │   ├── model.ts     # Type definitions
│       │   ├── const.ts     # SQL queries
│       │   └── libs/
│       │       ├── tool.ts           # AI tools (search, read, etc.)
│       │       ├── utils.ts          # Utility functions
│       │       └── semantic-cache.ts # Semantic caching
│       └── pow/             # Proof of Work module
│           ├── index.ts     # PoW routes & macros
│           ├── model.ts     # Type definitions
│           └── const.ts     # Configuration
├── scripts/                 # Utility scripts
│   ├── setup.ts           # Initial database setup
│   ├── db.ts              # Database utilities
│   ├── search.ts          # Search utility
│   ├── read-page.ts       # Page reading utility
│   └── semantic-cache.ts  # Cache utilities
├── public/                 # Static assets
│   └── arona.webp         # Project logo
├── docker-compose.yml      # Docker services
├── Dockerfile              # Build configuration
├── package.json            # Dependencies
├── tsconfig.json           # TypeScript config
└── .env.example            # Environment variables template
```

---

## การทำงานของระบบ

### 1. Clustering & Entry Point (`src/index.ts`)

```typescript
// Production mode: Run in cluster mode
// - Spawn (CPU cores - 1) workers
// - Run indexing cron job every 12 hours
// - Auto-restart dead workers

// Development mode: Run single instance
```

**Key Features:**
- Multi-core processing ด้วย Node.js Cluster
- Cron job ทำ indexing ทุก 12 ชั่วโมง
- Auto-restart workers ที่ตาย

### 2. Elysia Server (`src/server.ts`)

```typescript
// Server setup includes:
- CORS configuration
- OpenAPI documentation (dev mode)
- OpenTelemetry tracing (Axiom)
- Cookie security settings
- Route registration
```

### 3. AI Module (`src/modules/ai/`)

#### Core Components

**`service.ts`** - หัวใจของระบบ:

```typescript
// ฟังก์ชันหลัก:
ask()              // ส่งคำถามไปยัง AI model
search()           // ค้นหาเอกสาร (BM25 + Vector)
readPage()         // อ่านหน้าเอกสารที่ระบุ
getCache()         // ดึงคำตอบจาก cache
Checksum           // สร้าง/ตรวจสอบ checksum
```

**การทำงานของ `ask()`:**

1. สร้าง AI tools (search, readPage, tableOfContents, readHistory)
2. กำหนด stop conditions (step count, reference limit)
3. เรียก `streamText()` จาก AI SDK
4. Stream ผลลัพธ์กลับไปยัง client

**การทำงานของ `search()`:**

```typescript
// 1. Clean and normalize query
value = value.toLowerCase()
  .replace(/elysia|framework|saltyaom|"|'/g, '')
  .trim()

// 2. BM25 Search (ParadeDB)
// - Full-text search on title, summary, file_name
// - Normalize scores
// - Filter by threshold (0.6)
// - Limit to top 10 results

// 3. Vector Search (if needed)
// - Get embedding for query
// - Cosine similarity search
// - Hybrid scoring with BM25

// 4. Chunk aggregation
// - Combine adjacent chunks
// - Return top 3 results
```

### 4. Proof of Work Module (`src/modules/pow/`)

**วัตถุประสงค์:** ป้องกันการโจมตีด้วยบอท

**การทำงาน:**

1. **Request Challenge (`GET /pow/request`)**
   ```typescript
   - Generate random nonce
   - Record timestamp
   - Store in cookie (signed)
   - Return: { nonce, bits, expires }
   ```

2. **Verify Solution (Macro)**
   ```typescript
   - Verify IP matches
   - Verify not expired (15 minutes)
   - Verify hash starts with required zeros
   - SHA256(nonce:suffix) must start with '0' * (bits/4)
   ```

**ความยาก:** 19 bits = 4.75 hex zeros

### 5. Caching Strategy

**3 Layers of Cache:**

| Layer | Storage | TTL | Purpose |
|-------|---------|-----|---------|
| In-Memory LRU | RAM | 30s - 9hr | Burst cache |
| Redis | DragonflyDB | 3hr - 10hr | Persistent cache |
| Semantic Cache | Redis (Vector) | 4hr | Similarity search |

**Semantic Cache:**
- ใช้ Vector Similarity Search (KNN)
- Normalizes queries with small model
- Matches similarity ≥ 90%
- Stores embeddings for similarity matching

### 6. Indexing System (`src/libs/structure.ts`)

**กระบวนการ Indexing:**

```
1. Clone/Update documentation repo
2. Parse markdown files
3. Split into chunks (by ## headers)
4. Generate embeddings (3x per chunk):
   - Content embedding
   - Title embedding
   - File name embedding
5. Generate summary (using AI)
6. Calculate weight (based on section)
7. Insert/Update in database
8. Remove deleted content
```

**Weight System:**
```typescript
essential: 1.0      // สำคัญที่สุด
blog: 0.8
eden: 0.7
patterns: 0.5
unknown: 0.4
migrate: 0.3
integrations: 0.3
tutorial: 0.3
```

### 7. Database Schema

```sql
CREATE TABLE documents (
    link VARCHAR(255) PRIMARY KEY,
    file VARCHAR(255) NOT NULL,
    file_name VARCHAR(255) GENERATED,
    title VARCHAR(255) NOT NULL,
    content TEXT NOT NULL,
    summary TEXT NOT NULL,
    weight FLOAT NOT NULL DEFAULT 0.5,
    embedding VECTOR(1536) NOT NULL,
    title_embedding VECTOR(1536) NOT NULL,
    file_name_embedding VECTOR(1536) NOT NULL,
    sequence smallint NOT NULL DEFAULT 0
);

-- BM25 Index
CREATE INDEX idx_documents_content_bm25
ON documents USING bm25 (link, title, summary, file_name);

-- Link Index
CREATE INDEX idx_documents_link ON documents (link);
```

---

## การติดตั้งและตั้งค่า

### 1. Environment Variables

```bash
# Database
REDIS_URL=redis://localhost:6379
DATABASE_URL=postgresql://user:pass@localhost:5432/arona

# AI Providers
OPENAI_API_KEY=sk-xxx
OPENROUTER_API_KEY=sk-or-xxx
# Optional: OPENROUTER_MAIN_PROVIDERS=Groq,Cerebras

# Security
TURNSTILE_SECRET=0x...
CHALLENGE_SECRET=your-secret-key
AI_CHECKSUM_SECRET=your-checksum-secret

# Optional: Monitoring
AXIOM_DATASET=arona
AXIOM_TOKEN=xaat-xxx

# Optional: Reindex Webhook
REINDEX_SECRET=webhook-secret
REINDEX_ENDPOINT=/webhook/reindex
```

### 2. Docker Services

```yaml
# docker-compose.yml
services:
  pg:
    image: paradedb/paradedb:latest
    ports:
      - "5432:5432"

  dragonfly:
    image: docker.dragonflydb.io/dragonflydb/dragonfly:latest
    ports:
      - "6379:6379"
```

### 3. Setup Commands

```bash
# Start services
bun run dev:up

# Setup database
bun run db:setup

# Run development server
bun run dev
```

---

## API Documentation

### POST /model/ask

ถามคำถามเกี่ยวกับ Elysia Framework

**Request Body:**

```typescript
{
  message: string,        // คำถาม (max 4096 chars)
  seed?: number,          // Random seed (optional)
  reference?: string,     // Specific page to reference
  think?: boolean,        // Enable deeper reasoning
  history?: Array<{       // Conversation history
    role: 'user' | 'assistant',
    content: string,      // max 8192 chars
    checksum: string      // For assistant messages only
  }>
}
```

**Response:**

```typescript
// Streaming response
"Here's how to create a route in Elysia..."

// + metadata at end
"\n\n---Elysia-Metadata---\nchecksum:abc123..."
```

**Example:**

```bash
curl -X POST http://localhost:3000/model/ask \
  -H "Content-Type: application/json" \
  -H "x-turnstile-token: token..." \
  -d '{
    "message": "How do I create a route?"
  }'
```

**Full Response Example:**

```
To create a route in Elysia:

```typescript
import { Elysia } from 'elysia'

const app = new Elysia()
  .get('/', () => 'Hello World')
  .listen(3000)
```

- [Route](https://elysiajs.com/essential/route)
- [Handler](https://elysiajs.com/essential/handler)

---Elysia-Metadata---
checksum:a1b2c3d4...
```

### GET /pow/request

ขอ Proof of Work challenge

**Response:**

```typescript
{
  nonce: "a1b2c3d4...",      // Random challenge
  bits: 19,                  // Difficulty level
  expires: 1699123456789     // Expiry timestamp
}
```

### GET /heath

Health check endpoint

**Response:** `"ok"`

---

## ตัวอย่างการใช้งาน

### Example 1: Basic Question

**Request:**

```json
POST /model/ask
{
  "message": "What is Elysia?"
}
```

**Response Stream:**

```
Elysia is a TypeScript web framework for building backend servers with
great performance and developer experience.

It's known for:
- Fast execution (2x faster than Express)
- Type-safe by default
- Simple, elegant syntax

---Elysia-Metadata---
checksum:abc123...
```

### Example 2: With Conversation History

**Request:**

```json
POST /model/ask
{
  "message": "How do I add validation?",
  "history": [
    {
      "role": "user",
      "content": "What is Elysia?"
    },
    {
      "role": "assistant",
      "content": "Elysia is...",
      "checksum": "abc123..."
    }
  ]
}
```

### Example 3: With Reference

**Request:**

```json
POST /model/ask
{
  "message": "Tell me more about lifecycle",
  "reference": "essential/life-cycle"
}
```

### Example 4: Proof of Work Flow

```typescript
// 1. Request challenge
const { nonce, bits, expires } = await fetch('/pow/request').then(r => r.json())

// 2. Solve (client-side)
let suffix = 0
while (Date.now() < expires) {
  const hash = sha256(`${nonce}:${suffix}`)
  if (hash.startsWith('0'.repeat(bits / 4))) {
    break // Found solution
  }
  suffix++
}

// 3. Send request
await fetch('/model/ask', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'x-turnstile-token': turnstileToken
  },
  body: JSON.stringify({
    pow: { suffix },
    message: 'Help me understand Elysia'
  })
})
```

---

## การแก้ไขและพัฒนาต่อยอด

### เปลี่ยน AI Provider

**File: `src/libs/ai.ts`**

```typescript
// Current: OpenRouter
export const model = router('openai/gpt-oss-120b:exacto')

// Change to Groq:
import { createGroq } from '@ai-sdk/groq'
export const groq = createGroq()
export const model = groq('openai/gpt-oss-120b')

// Change to Cerebras:
import { createCerebras } from '@ai-sdk/cerebras'
export const cerebras = createCerebras()
export const model = cerebras('gpt-oss-120b')
```

### ปรับแต่ง Indexing

**File: `src/libs/structure.ts`**

```typescript
// เปลี่ยน documentation source
await Bun.$`git clone --depth 1 https://your-repo/docs`

// ปรับ weight
const titleWeight = {
  yoursection: 1.0,
  another: 0.5
}
```

### ปรับแต่ง AI Personality

**File: `src/libs/ai.ts`**

```typescript
export const instruction = `You are YOUR assistant...

Purpose:
- Your purposes

Behavior:
- Your behavior guidelines

Constraints:
- Your constraints
`
```

### เพิ่ม Cache Layer

```typescript
// src/libs/my-cache.ts
export class CustomCache {
  async get(key: string) { }
  async set(key: string, value: any) { }
}

// Use in service.ts
const cached = await customCache.get(message)
if (cached) return cached
```

### เพิ่ม AI Tool

**File: `src/modules/ai/libs/tool.ts`**

```typescript
export const createMyTool = (references: Reference[]) =>
  tool({
    description: 'My custom tool',
    inputSchema: z.object({
      param: z.string()
    }),
    outputSchema: z.string(),
    async execute({ param }) {
      // Your logic
      return result
    }
  })

// Add in service.ts
tools: {
  ...existingTools,
  myTool: createMyTool(references)
}
```

### ปรับ Search Algorithm

**File: `src/modules/ai/service.ts`**

```typescript
// Adjust BM25 + Vector weighting
const score = 0.775 * r_norm + 0.275 * weight

// Adjust vector search threshold
WHERE score >= 0.4  // Change this
```

---

## Key Concepts

### Checksum Verification

ป้องกันการปลอมแปลงประวัติการสนทนา:

```typescript
// Generate (server)
checksum = sha256(content + secret)

// Verify (server)
isValid = timingSafeEqual(
  sha256(content + secret),
  receivedChecksum
)
```

### Vector Scoring

```typescript
// Hybrid scoring formula
score = 0.1 * title_similarity +      // 10% title
        0.675 * content_similarity +   // 67.5% content
        0.1 * filename_similarity +    // 10% filename
        0.125 * weight                 // 12.5% document weight
```

### Rate Limiting

```typescript
// Sliding window using Redis Sorted Set
- ZADD with timestamp as score
- ZREMRANGEBYSCORE to remove old entries
- ZCOUNT to check current count
```

---

## Monitoring & Debugging

### OpenTelemetry + Axiom

```typescript
// Automatic tracing for:
- AI requests
- Database queries
- External API calls
- Custom spans

// View in Axiom dashboard
```

### Logging

```typescript
import { log } from '@arona/libs'

log('Search:', query)
log('Cache hit for key')
log('Total', batches, 'to process')
```

---

## Best Practices

1. **Always use semantic cache** for similar queries
2. **Limit history** to prevent token explosion
3. **Use checksum** to verify AI responses
4. **Rate limit** to prevent abuse
5. **Monitor costs** via Axiom tracing
6. **Index incrementally** - only update changed content
7. **Use burst cache** for hot queries

---

## Troubleshooting

### Database connection fails

```bash
# Check ParadeDB is running
docker ps | grep paradedb

# Check connection string
echo $DATABASE_URL
```

### Redis connection fails

```bash
# Check DragonflyDB
docker ps | grep dragonfly

# Test connection
redis-cli -u $REDIS_URL PING
```

### AI errors

```bash
# Check API keys
echo $OPENAI_API_KEY
echo $OPENROUTER_API_KEY

# Test embedding
bun run scripts/semantic-cache.ts
```

### Indexing fails

```bash
# Run manually
bun run scripts/db.ts

# Check logs
log('Indexing error:', error)
```

---

## Summary

Arona เป็นระบบ RAG ที่ประหยัดและมีประสิทธิภาพ ประกอบด้วย:

- **Elysia Framework** - Web server
- **ParadeDB** - BM25 + Vector search
- **DragonflyDB** - Caching layer
- **OpenRouter** - AI models
- **Multi-layer caching** - Semantic + LRU + Redis
- **Security** - PoW + Turnstile + Rate limiting
- **Observability** - OpenTelemetry + Axiom

ระบบออกแบบมาให้ self-host ได้ง่าย และปรับแต่งได้ตามต้องการ
