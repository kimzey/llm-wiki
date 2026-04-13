# วิเคราะห์ความเป็นไปได้: ทำ RAG ตอบคำถามเกี่ยวกับบริษัท Sellsuki

## Executive Summary

**คำตอบ:** ได้ ระบบ Arona สามารถนำไปปรับใช้เป็น RAG สำหรับบริษัท Sellsuki ได้ 100%

เนื่องจาก Arona ถูกออกแบบมาเพื่อ:
- เป็น self-hosted RAG system ที่ประหยัด
- รองรับการอัปเดตข้อมูลแบบ incremental
- มี Semantic Cache ช่วยลด cost
- ใช้ ParadeDB + DragonflyDB เหมือนที่ต้องการ

---

## 1. การวิเคราะห์ Use Case

### ปัญหาปัจจุบัน vs หลังใช้ RAG

| ประเด็น | ก่อนใช้ | หลังใช้ |
|---------|---------|---------|
| ความเร็ว | รอ HR/เพื่อน ช้า | ได้ทันที 24/7 |
| ความถูกต้อง | ตอบผิด/จำไม่ได้ | อ้างอิงจากเอกสารจริง |
| Source of Truth | ไม่มี/กระจัดกระจาย | เอกสารเดียว อัปเดตได้ |
| Workload | HR ตอบซ้ำๆ | AI ตอบแทน |

### ประเภทคำถามที่ระบบรองรับ

```
1. Policy Questions
   - "ลาพักร้อนกี่วัน?"
   - "โบนัสเมื่อไหร่จ่าย?"
   - "Dress code อะไร?"

2. How-to Questions
   - "สร้าง order ใน Sellsuki ยังไง?"
   - "ลาหยุดยังไง?"
   - "เคลมประกันยังไง?"

3. Technical/Business Rules
   - "Business rule ของการคำนวณ commission?"
   - "API endpoint สำหรับ..."

4. Quick Info
   - "WiFi password?"
   - "จองห้องประชุมยังไง?"
   - "เบอร์ IT support?"
```

---

## 2. สถาปัตยกรรมระบบ Sellsuki RAG

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Channels                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                 │
│  │ LINE    │  │ Slack   │  │ Web     │  │ Cursor  │                 │
│  │ Bot     │  │ Bot     │  │ Chat    │  │ (Dev)   │                 │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘                 │
└───────┼────────────┼────────────┼────────────┼──────────────────────┘
        │            │            │            │
        └────────────┴────────────┴────────────┘
                             │
                             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Sellsuki RAG API                                 │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Security & Access Control                                   │  │
│  │  - Authentication (LINE/Slack/Employee ID)                   │  │
│  │  - Role-based filtering (HR/Admin/Dev/Employee)              │  │
│  │  - Rate Limiting                                             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Query Processing                                            │  │
│  │  - Intent Classification                                     │  │
│  │  - Metadata Filtering (department, doc_type, access_level)   │  │
│  │  - Semantic Normalization                                    │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  Caching Layer                                               │  │
│  │  - Semantic Cache (DragonflyDB)                              │  │
│  │  - Exact Match Cache                                         │  │
│  │  - Embedding Cache                                           │  │
│  └──────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  AI Engine                                                   │  │
│  │  - GPT OSS 120B / GPT-4o-mini                                │  │
│  │  - Tools: search_docs, read_doc, list_categories             │  │
│  └──────────────────────────────────────────────────────────────┘  │
└─────────────────────────────┬───────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│  ParadeDB     │   │  DragonflyDB  │   │   OpenRouter  │
│  (Vector DB)  │   │  (Cache)      │   │   (AI Models) │
│               │   │               │   │               │
│ - Documents   │   │ - Semantic    │   │ - GPT-4o-mini │
│ - Embeddings  │   │   Search      │   │ - GPT OSS     │
│ - Metadata    │   │ - LRU Cache   │   │ - Embeddings  │
│ - BM25        │   │               │   │               │
└───────────────┘   └───────────────┘   └───────────────┘
        │
        ▼
┌─────────────────────────────────┐
│  Document Sources               │
│  - HR Policies (PDF)            │
│  - Employee Handbook            │
│  - Technical Docs               │
│  - Confluence/Notion            │
│  - Google Drive                 │
└─────────────────────────────────┘
```

---

## 3. Database & Index Design สำหรับ RAG

### 3.1 Schema Design

```sql
-- Main documents table
CREATE TABLE documents (
    id VARCHAR(255) PRIMARY KEY,
    file_path VARCHAR(512) NOT NULL,
    file_name VARCHAR(255) NOT NULL,
    title VARCHAR(512) NOT NULL,
    content TEXT NOT NULL,

    -- Chunking info
    chunk_id VARCHAR(255) NOT NULL,
    chunk_index INTEGER NOT NULL,
    parent_id VARCHAR(255),

    -- Embeddings (1536 for OpenAI text-embedding-3-small)
    embedding VECTOR(1536) NOT NULL,
    title_embedding VECTOR(1536) NOT NULL,

    -- Metadata (crucial for filtering)
    metadata JSONB NOT NULL,

    -- Search optimization
    summary TEXT,
    weight FLOAT DEFAULT 0.5,

    -- Version control
    content_hash VARCHAR(64) NOT NULL,
    version INTEGER DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    -- Access control
    access_level VARCHAR(50) DEFAULT 'all',  -- all, hr_only, dev_only, admin_only
    department VARCHAR(100)[]
);

-- Metadata structure example:
-- {
--   "doc_type": "policy|handbook|technical|guide|faq",
--   "category": "leave|benefits|it|hr|finance|dev",
--   "tags": ["wifi", "password", "network"],
--   "author": "hr@sellsuki.com",
--   "last_reviewed": "2025-01-15",
--   "effective_date": "2025-01-01",
--   "status": "active|archived",
--   "language": "th|en",
--   "page_number": 15,
--   "section": "Chapter 3",
--   "source": "google_drive|confluence|notion|upload"
-- }

-- Indexes
CREATE INDEX idx_documents_embedding ON documents
USING hnsw (embedding vector_cosine_ops);

CREATE INDEX idx_documents_bm25 ON documents
USING bm25 (id, title, content, summary)
WITH (key_field='id');

CREATE INDEX idx_documents_metadata ON documents
USING gin (metadata jsonb_path_ops);

CREATE INDEX idx_documents_access ON documents(access_level);
CREATE INDEX idx_documents_content_hash ON documents(content_hash);
CREATE INDEX idx_documents_file_path ON documents(file_path);
```

### 3.2 Chunking Strategies

```typescript
// 1. Fixed-size Chunking (Simple)
interface FixedSizeChunk {
  content: string
  size: number
  overlap: number
}

// 2. Semantic Chunking (Recommended)
interface SemanticChunk {
  content: string
  boundary: 'paragraph' | 'section' | 'chapter'
  min_size: number
  max_size: number
}

// 3. Hierarchical Chunking (Best for large docs)
interface HierarchicalChunk {
  parent: {
    content: string      // Full document/chapter
    embedding: number[]
  }
  children: {
    content: string      // Sections
    embedding: number[]
  }[]
}

// 4. Metadata-Aware Chunking
interface MetadataAwareChunk {
  content: string
  metadata: {
    chunk_type: 'title' | 'section' | 'content' | 'table' | 'code'
    context_before?: string
    context_after?: string
    page_number?: number
    section_title?: string
  }
}
```

### 3.3 ตัวอย่างการ Chunk เอกสาร HR

```
Original Document: "Employee Handbook 2025.pdf"

Page 15:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Chapter 3: Leave Policy

3.1 Annual Leave
Employees are entitled to 10 days of annual leave per year.
Leave accrues monthly and must be used within the calendar year.

3.2 Sick Leave
Sick leave is provided at 30 days per year with medical certificate.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Chunks with Metadata:

Chunk 1:
{
  id: "doc123_chunk1",
  content: "Chapter 3: Leave Policy\n\n3.1 Annual Leave\nEmployees are entitled to 10 days of annual leave per year.",
  metadata: {
    doc_type: "handbook",
    category: "leave",
    section: "3.1",
    section_title: "Annual Leave",
    page: 15,
    chunk_type: "section",
    keywords: ["leave", "annual", "vacation"]
  },
  access_level: "all"
}

Chunk 2:
{
  id: "doc123_chunk2",
  content: "Leave accrues monthly and must be used within the calendar year.\n\n3.2 Sick Leave\nSick leave is provided at 30 days per year.",
  metadata: {
    doc_type: "handbook",
    category: "leave",
    section: "3.1-3.2",
    page: 15,
    chunk_type: "content"
  },
  access_level: "all"
}
```

---

## 4. Incremental Update Strategy

### 4.1 การตรวจจับการเปลี่ยนแปลง

```typescript
// File Change Detection
interface DocumentChange {
  file_path: string
  old_hash: string | null
  new_hash: string
  action: 'created' | 'updated' | 'deleted' | 'unchanged'
}

// Hash-based comparison
async function detectChanges(
  files: DocumentFile[]
): Promise<DocumentChange[]> {
  const changes: DocumentChange[] = []

  for (const file of files) {
    const newHash = await hashFile(file.content)
    const existing = await db.getDocument(file.path)

    if (!existing) {
      changes.push({
        file_path: file.path,
        old_hash: null,
        new_hash,
        action: 'created'
      })
    } else if (existing.content_hash !== newHash) {
      changes.push({
        file_path: file.path,
        old_hash: existing.content_hash,
        new_hash,
        action: 'updated'
      })
    }
  }

  // Detect deletions
  const existingPaths = await db.getAllPaths()
  const currentPaths = new Set(files.map(f => f.path))

  for (const path of existingPaths) {
    if (!currentPaths.has(path)) {
      changes.push({
        file_path: path,
        old_hash: (await db.getDocument(path))!.content_hash,
        new_hash: '',
        action: 'deleted'
      })
    }
  }

  return changes
}
```

### 4.2 Smart Embedding Reuse

```typescript
// ตรวจสอบว่า chunk เปลี่ยนแปลงจริงหรือไม่
async function shouldReEmbed(oldChunk: Chunk, newChunk: Chunk): Promise<boolean> {
  // 1. Content comparison
  if (oldChunk.content === newChunk.content) {
    return false  // เหมือนเดิม ไม่ต้อง embed ซ้ำ
  }

  // 2. Semantic similarity (fast check)
  const similarity = cosineSimilarity(
    oldChunk.embedding,
    await getQuickEmbedding(newChunk.content)  // เอาแค่ draft embedding
  )

  if (similarity > 0.98) {
    return false  // แทบไม่เปลี่ยนแปลง
  }

  return true  // ต้อง embed ใหม่
}

// Incremental embedding
async function updateDocument(
  change: DocumentChange,
  file: DocumentFile
): Promise<void> {
  const oldChunks = await db.getChunks(file.path)

  if (change.action === 'deleted') {
    await db.deleteChunks(file.path)
    return
  }

  const newChunks = await chunkDocument(file.content, file.metadata)
  const updates: ChunkUpdate[] = []

  // Compare chunks
  for (const newChunk of newChunks) {
    const oldChunk = oldChunks.find(c => c.chunk_id === newChunk.chunk_id)

    if (!oldChunk) {
      // New chunk - must embed
      updates.push({
        action: 'insert',
        chunk: newChunk,
        embed: true
      })
    } else if (await shouldReEmbed(oldChunk, newChunk)) {
      // Changed chunk - re-embed
      updates.push({
        action: 'update',
        chunk: { ...newChunk, id: oldChunk.id },
        embed: true
      })
    } else {
      // Unchanged - keep existing embedding
      updates.push({
        action: 'keep',
        chunk: { ...newChunk, embedding: oldChunk.embedding, id: oldChunk.id },
        embed: false
      })
    }
  }

  // Batch update
  await db.applyUpdates(file.path, updates)
}
```

### 4.3 Indexing Pipeline

```typescript
// Full pipeline with incremental support
async function indexDocuments(
  source: DocumentSource
): Promise<IndexResult> {
  // 1. Fetch files
  const files = await source.fetch()

  // 2. Detect changes
  const changes = await detectChanges(files)

  console.log(`Detected ${changes.length} changes:
    - ${changes.filter(c => c.action === 'created').length} created
    - ${changes.filter(c => c.action === 'updated').length} updated
    - ${changes.filter(c => c.action === 'deleted').length} deleted
    - ${changes.filter(c => c.action === 'unchanged').length} unchanged
  `)

  // 3. Process changes
  const results = {
    total: changes.length,
    created: 0,
    updated: 0,
    deleted: 0,
    embedded: 0,
    reused: 0
  }

  for (const change of changes) {
    if (change.action === 'unchanged') continue

    const file = files.find(f => f.path === change.file_path)!

    if (change.action === 'deleted') {
      await db.deleteDocument(change.file_path)
      results.deleted++
    } else {
      const stats = await updateDocument(change, file)
      results.created += stats.created
      results.updated += stats.updated
      results.embedded += stats.embedded
      results.reused += stats.reused
    }
  }

  return results
}
```

---

## 5. Caching Strategy

### 5.1 Semantic Cache ด้วย DragonflyDB

```typescript
// Redis setup with Vector Search
await redis.call('FT.CREATE', 'idx:sellsuki_cache',
  'ON', 'HASH',
  'PREFIX', '1', 'scache:',
  'SCHEMA',
  'embedding', 'VECTOR', 'HNSW', '6',
  'TYPE', 'FLOAT32',
  'DIM', '1536',
  'DISTANCE_METRIC', 'COSINE',
  'query', 'TEXT',
  'response', 'TEXT',
  'metadata', 'TEXT'
)

// Semantic Cache Class
class SellsukiSemanticCache {
  // Get cached response
  static async get(query: string, userContext: UserContext): Promise<string | null> {
    // 1. Normalize query
    const normalized = await this.normalize(query)

    // 2. Get embedding
    const embedding = await getEmbedding(normalized)

    // 3. Vector search
    const result = await redis.call(
      'FT.SEARCH',
      'idx:sellsuki_cache',
      '*=>[KNN 1 @embedding $vec AS score]',
      'PARAMS', '2', 'vec', Buffer.from(embedding),
      'DIALECT', '2',
      'RETURN', '3', 'score', 'response', 'metadata',
      'LIMIT', '0', '1'
    )

    if (!result || result.length < 3) return null

    const [, , [, score, , response, metadataStr]] = result
    const similarity = 1 - parseFloat(score)

    // 4. Check threshold
    if (similarity < 0.92) return null

    // 5. Check access level
    const metadata = JSON.parse(metadataStr)
    if (!this.checkAccess(metadata, userContext)) {
      return null
    }

    console.log(`[Semantic Cache HIT] similarity=${similarity.toFixed(4)}`)
    return response
  }

  // Store response
  static async set(
    query: string,
    response: string,
    metadata: CacheMetadata
  ): Promise<void> {
    const normalized = await this.normalize(query)
    const embedding = await getEmbedding(normalized)
    const key = `scache:${cyrb53(normalized)}`

    await redis.hset(key, {
      query: normalized,
      response,
      embedding: Buffer.from(embedding),
      metadata: JSON.stringify(metadata)
    })

    await redis.expire(key, 14400)  // 4 hours
  }

  // Normalize query (optional - for better cache hit)
  static async normalize(query: string): Promise<string> {
    // Use small model to normalize
    const { text } = await generateText({
      model: smallModel,
      system: 'Convert query to canonical form. Be concise.',
      prompt: query,
      maxOutputTokens: 64
    })

    return text.trim()
  }

  // Check access permission
  static checkAccess(metadata: CacheMetadata, user: UserContext): boolean {
    if (metadata.access_level === 'all') return true
    if (metadata.access_level === 'hr_only' && user.role === 'hr') return true
    if (metadata.access_level === 'dev_only' && user.role === 'dev') return true
    return false
  }
}
```

### 5.2 Embedding Cache

```typescript
// Cache embeddings to avoid duplicate API calls
class EmbeddingCache {
  private static lru = new LRUCache<string, number[]>({
    max: 1000,
    ttl: 9 * 60 * 60 * 1000  // 9 hours
  })

  private static pending = new Map<string, Promise<number[]>>()

  static async get(text: string): Promise<number[]> {
    const key = this.key(text)

    // Check cache
    if (this.lru.has(key)) {
      return this.lru.get(key)!
    }

    // Check pending
    if (this.pending.has(key)) {
      return this.pending.get(key)!
    }

    // Fetch new
    const promise = this.fetch(text)
    this.pending.set(key, promise)

    try {
      const embedding = await promise
      this.lru.set(key, embedding)
      return embedding
    } finally {
      this.pending.delete(key)
    }
  }

  private static key(text: string): string {
    // Normalize for cache key
    return text.toLowerCase().slice(0, 200)
  }

  private static async fetch(text: string): Promise<number[]> {
    return embed({
      model: openai.embeddingModel('text-embedding-3-small'),
      value: text
    }).then(r => r.embedding)
  }
}
```

### 5.3 Multi-level Cache Strategy

```
Query → L1: In-Memory LRU (5min) → L2: Semantic Cache (4hr) → L3: Exact Match Cache (10hr)
       ↓ HIT                    ↓ HIT                        ↓ HIT
    Return immediately      Return immediately          Return immediately
       ↓ MISS                   ↓ MISS                      ↓ MISS
    Check L2               Check L3                    Generate AI Response
```

```typescript
async function getCachedResponse(
  query: string,
  user: UserContext
): Promise<string | null> {
  // L1: In-memory exact match (fastest)
  const l1Key = `${user.role}:${query.slice(0, 100)}`
  if (lruCache.has(l1Key)) {
    return lruCache.get(l1Key)!
  }

  // L2: Semantic cache (smart)
  const semantic = await SellsukiSemanticCache.get(query, user)
  if (semantic) {
    lruCache.set(l1Key, semantic)
    return semantic
  }

  // L3: Exact match in Redis (fallback)
  const exact = await redis.get(`cache:${hash(query)}`)
  if (exact) {
    lruCache.set(l1Key, exact)
    return exact
  }

  return null  // Cache miss - need AI
}
```

---

## 6. Metadata Filtering & Access Control

### 6.1 Query with Metadata Filter

```typescript
// Search with metadata filtering
async function searchWithFilter(
  query: string,
  user: UserContext
): Promise<SearchResult[]> {
  const embedding = await getEmbedding(query)

  // Build filter based on user role
  const accessFilter = buildAccessFilter(user)

  // Vector search with filter
  const results = await sql`
    WITH q AS (
      SELECT ${embedding}::vector AS embedding
    ),
    filtered AS (
      SELECT DISTINCT ON (d.file_path)
        d.*,
        (d.embedding <#> q.embedding) AS similarity
      FROM documents d, q
      WHERE
        d.access_level = ANY(${accessFilter})
        AND d.metadata->>'status' = 'active'
        AND d.metadata->>'language' = ${user.language}
      ORDER BY d.file_path, similarity DESC
    )
    SELECT
      id,
      title,
      content,
      summary,
      file_path,
      metadata,
      similarity,
      (1 - similarity) AS score
    FROM filtered
    WHERE similarity < 0.2  -- cosine distance < 0.2 = score > 0.8
    ORDER BY similarity DESC
    LIMIT 10
  `

  return results
}

// Build access filter based on role
function buildAccessFilter(user: UserContext): string[] {
  const baseAccess = ['all']

  switch (user.role) {
    case 'hr':
      return [...baseAccess, 'hr_only']
    case 'dev':
      return [...baseAccess, 'dev_only', 'hr_only']  // Dev can see HR docs
    case 'admin':
      return ['all', 'hr_only', 'dev_only', 'admin_only']
    default:
      return baseAccess
  }
}
```

### 6.2 Metadata Schema

```typescript
interface DocumentMetadata {
  // Classification
  doc_type: 'policy' | 'handbook' | 'technical' | 'guide' | 'faq' | 'announcement'
  category: string
  tags: string[]

  // Access control
  access_level: 'all' | 'hr_only' | 'dev_only' | 'admin_only' | 'management_only'
  departments?: string[]  // ['hr', 'engineering', 'sales']
  teams?: string[]        // ['backend', 'frontend']

  // Content info
  language: 'th' | 'en'
  page_number?: number
  section?: string
  chapter?: string

  // Source info
  source: 'google_drive' | 'confluence' | 'notion' | 'upload' | 'github'
  source_url?: string
  author?: string
  owner?: string

  // Lifecycle
  status: 'active' | 'archived' | 'draft'
  effective_date?: string
  expiry_date?: string
  last_reviewed?: string
  version: number

  // Search optimization
  keywords: string[]
  synonyms?: string[]
  related_docs?: string[]  // IDs of related documents
}
```

---

## 7. Integration with Channels

### 7.1 LINE Bot Integration

```typescript
// LINE Bot handler
import { LINE } from '@line/bot-sdk'

const lineClient = new LINE({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN!
})

app.post('/line/webhook', async (req) => {
  const events = req.body.events

  for (const event of events) {
    if (event.type === 'message' && event.message.type === 'text') {
      const userId = event.source.userId
      const query = event.message.text

      // Get user profile for access control
      const profile = await lineClient.getProfile(userId)
      const userContext = await getUserContext(profile.userId)

      // Get answer
      const answer = await askQuestion(query, userContext)

      // Reply
      await lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: answer
      })
    }
  }
})
```

### 7.2 Slack Bot Integration

```typescript
// Slack Bot handler
import { App } from '@slack/bolt'

const app = new App({
  token: process.env.SLACK_BOT_TOKEN,
  signingSecret: process.env.SLACK_SIGNING_SECRET
})

app.message(async ({ message, say, client }) => {
  const userId = message.user
  const query = message.text

  // Get user info
  const userInfo = await client.users.info({ user: userId })
  const userContext = await getUserContext(userInfo.user.email)

  // Get answer
  const answer = await askQuestion(query, userContext)

  await say(answer)
})
```

### 7.3 Web Chat Widget

```typescript
// Web chat API endpoint
app.post('/api/chat', async (req) => {
  const { message, sessionId } = req.body

  // Get user from session
  const session = await getSession(sessionId)
  const userContext = session.user

  // Stream response
  const stream = await askQuestionStream(message, userContext)

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache'
    }
  })
})
```

### 7.4 Cursor / IDE Integration (for Devs)

```typescript
// MCP Server for Cursor/IDE
import { MCPServer } from 'mcp-server'

const mcp = new MCPServer()

mcp.addTool({
  name: 'sellsuki_search',
  description: 'Search Sellsuki documentation',
  parameters: {
    query: { type: 'string', description: 'Search query' }
  },
  async execute({ query }, context) {
    const user = await getUserFromToken(context.token)
    const results = await searchWithFilter(query, user)

    return results.map(r => ({
      title: r.title,
      content: r.content,
      source: r.file_path,
      metadata: r.metadata
    }))
  }
})
```

---

## 8. Implementation Roadmap

### Phase 1: Core RAG (Week 1-2)
- [ ] Set up ParadeDB + DragonflyDB
- [ ] Create document schema
- [ ] Implement chunking (start with semantic chunking)
- [ ] Build indexing pipeline
- [ ] Test with sample HR documents

### Phase 2: AI & Search (Week 3-4)
- [ ] Integrate AI model (GPT-4o-mini / GPT OSS)
- [ ] Implement hybrid search (BM25 + Vector)
- [ ] Add metadata filtering
- [ ] Build semantic cache

### Phase 3: Channels (Week 5-6)
- [ ] LINE Bot integration
- [ ] Slack Bot integration
- [ ] Web chat widget
- [ ] Access control implementation

### Phase 4: Advanced (Week 7-8)
- [ ] Incremental indexing
- [ ] Embedding reuse
- [ ] Multi-language support (TH/EN)
- [ ] Analytics & monitoring

---

## 9. Cost Estimation

### Infrastructure Cost (Monthly)

| Service | Cost | Notes |
|---------|------|-------|
| ParadeDB | $0-20 | Self-host on small VM |
| DragonflyDB | $0-30 | Free tier or small instance |
| VM/Hosting | $20-50 | 2 vCPU, 4GB RAM |
| **Total Infrastructure** | **$20-100** | |

### AI Cost (Per 1,000 queries)

| Operation | Model | Tokens (est) | Cost/1K |
|-----------|-------|--------------|---------|
| Embedding | text-embedding-3-small | ~100 tokens | $0.00002 |
| Main AI | GPT-4o-mini | ~2,000 tokens | $0.30 |
| Query Norm | GPT-4o-mini | ~200 tokens | $0.03 |
| **Total without cache** | | | **~$0.33** |
| **Total with 70% cache** | | | **~$0.10** |

### Annual Estimate (10K queries/month)
- Infrastructure: ~$600/year
- AI Cost: ~$1,200/year (with cache)
- **Total: ~$1,800/year** vs RaaS providers ~$12,000/year

---

## 10. Key Recommendations

### Do's
1. **Start with semantic chunking** - Better context preservation
2. **Implement incremental indexing from day 1** - Will save lots of cost later
3. **Use metadata filtering** - Crucial for access control
4. **Cache aggressively** - Semantic cache gives ~70% hit rate
5. **Monitor embeddings** - Track cache hit rates and similarity scores

### Don'ts
1. Don't use fixed-size chunking for policy documents
2. Don't skip access control implementation
3. Don't embed everything on every update
4. Don't ignore multi-language support (TH/EN)
5. Don't forget to prune old cache entries

### Chunk Strategy for Different Content

| Content Type | Recommended Chunk | Size | Metadata |
|--------------|------------------|------|----------|
| Policies | Section-based | 500-1000 chars | section_title, category |
| Handbooks | Chapter + subsection | 800-1500 chars | chapter, section |
| Technical Docs | Code/function | 300-800 chars | language, framework |
| FAQs | Per question | 200-500 chars | question, keywords |
| Announcements | Per item | 200-600 chars | date, category |

---

## 11. Example Queries & Responses

### Example 1: Leave Policy
```
Q: "ลาพักร้อนกี่วัน?"
A: "พนักงานมีสิทธิ์ลาพักร้อน 10 วัน/ปี
   - ละสะสมได้สูงสุด 10 วัน
   - ต้องใช้ภายในปีปฏิทิน
   - แจ้งล่วงหน้าอย่างน้อย 3 วัน

   ที่มา: Employee Handbook 2025, หน้า 15, ตอนที่ 3.1"
```

### Example 2: Technical (Dev Only)
```
Q: "API สำหรับสร้าง order ยังไง?"
A: "POST /api/v1/orders
   Authentication: Bearer token

   Request body:
   {
     "customer_id": "string",
     "items": [{
       "product_id": "string",
       "quantity": number
     }],
     "currency": "THB"
   }

   ที่มา: API Documentation, Order Service"
```

### Example 3: WiFi Password
```
Q: "WiFi password อะไร?"
A: "Sellsuki@2025!
   Network: Sellsuki-Guest

   ที่มา: IT Guide, Quick Info"
```

---

## Summary

ระบบ Arona สามารถนำไปปรับใช้เป็น RAG สำหรับ Sellsuki ได้ **100%** เนื่องจาก:

1. ✅ Architecture เหมือนกัน (RAG + Hybrid Search)
2. ✅ Database เหมือนกัน (ParadeDB + Vector + BM25)
3. ✅ Cache เหมือนกัน (DragonflyDB + Semantic Cache)
4. ✅ รองรับ incremental update
5. ✅ รองรับ metadata filtering
6. ✅ Cost-effective (self-hosted)

**สิ่งที่ต้องเพิ่ม/ปรับ:**
- Access control ตาม role (HR/Dev/Admin)
- Multi-language support (TH/EN)
- Integration กับ LINE/Slack/Web
- Document source connectors (Google Drive, Confluence, etc.)
