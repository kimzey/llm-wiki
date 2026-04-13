---
title: "วิเคราะห์ความเป็นไปได้ RAG สำหรับ Sellsuki"
type: source
source_file: raw/notes/arona/SELLSUKI_RAG_ANALYSIS.md
tags: [sellsuki, rag-analysis, enterprise, use-case]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุป

วิเคราะห์ความเป็นไปได้ในการนำระบบ Arona ไปปรับใช้เป็น RAG สำหรับบริษัท Sellsuki พร้อม architecture, database schema, การทำ incremental update, caching strategy, metadata filtering และ integration channels

## ประเด็นสำคัญ

### Executive Summary

**คำตอบ:** ได้ ระบบ Arona สามารถนำไปปรับใช้เป็น RAG สำหรับบริษัท Sellsuki ได้ 100%

เนื่องจาก Arona ถูกออกแบบมาเพื่อ:
- เป็น self-hosted RAG system ที่ประหยัด
- รองรับการอัปเดตข้อมูลแบบ incremental
- มี Semantic Cache ช่วยลด cost
- ใช้ ParadeDB + DragonflyDB เหมือนที่ต้องการ

### Use Case

**ปัญหาปัจจุบัน vs หลังใช้ RAG:**
| ประเด็น | ก่อนใช้ | หลังใช้ |
|---------|---------|---------|
| ความเร็ว | รอ HR/เพื่อน ช้า | ได้ทันที 24/7 |
| ความถูกต้อง | ตอบผิด/จำไม่ได้ | อ้างอิงจากเอกสารจริง |
| Source of Truth | ไม่มี/กระจัดกระจาย | เอกสารเดียว อัปเดตได้ |
| Workload | HR ตอบซ้ำๆ | AI ตอบแทน |

### Architecture สำหรับ Sellsuki

```
Channels (LINE/Slack/Web/Cursor)
    ↓
Sellsuki RAG API
├── Security & Access Control
│   ├── Authentication
│   ├── Role-based filtering
│   └── Rate Limiting
├── Query Processing
│   ├── Intent Classification
│   ├── Metadata Filtering
│   └── Semantic Normalization
├── Caching Layer
│   ├── Semantic Cache (DragonflyDB)
│   ├── Exact Match Cache
│   └── Embedding Cache
└── AI Engine
    ├── GPT OSS 120B / GPT-4o-mini
    └── Tools: search_docs, read_doc, list_categories
    ↓
ParadeDB + DragonflyDB + OpenRouter
```

### Database Schema Design

```sql
CREATE TABLE documents (
    id VARCHAR(255) PRIMARY KEY,
    file_path VARCHAR(512) NOT NULL,
    title VARCHAR(512) NOT NULL,
    content TEXT NOT NULL,
    
    chunk_id VARCHAR(255) NOT NULL,
    chunk_index INTEGER NOT NULL,
    parent_id VARCHAR(255),
    
    embedding VECTOR(1536) NOT NULL,
    title_embedding VECTOR(1536) NOT NULL,
    
    metadata JSONB NOT NULL,  -- Access control สำคัญ
    summary TEXT,
    weight FLOAT DEFAULT 0.5,
    
    content_hash VARCHAR(64) NOT NULL,
    version INTEGER DEFAULT 1,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    access_level VARCHAR(50) DEFAULT 'all',  -- all, hr_only, dev_only
    department VARCHAR(100)[]
);
```

### Metadata Structure

```json
{
  "doc_type": "policy|handbook|technical|guide|faq",
  "category": "leave|benefits|it|hr|finance|dev",
  "tags": ["wifi", "password", "network"],
  "access_level": "all|hr_only|dev_only|admin_only",
  "departments": ["hr", "engineering"],
  "language": "th|en",
  "status": "active|archived|draft",
  "source": "google_drive|confluence|notion|upload"
}
```

### Incremental Update Strategy

**1. File Change Detection:**
```typescript
interface DocumentChange {
  file_path: string
  old_hash: string | null
  new_hash: string
  action: 'created' | 'updated' | 'deleted' | 'unchanged'
}
```

**2. Smart Embedding Reuse:**
```typescript
async function shouldReEmbed(oldChunk: Chunk, newChunk: Chunk): Promise<boolean> {
  // 1. Content comparison
  if (oldChunk.content === newChunk.content) return false
  
  // 2. Semantic similarity
  const similarity = cosineSimilarity(oldChunk.embedding, newChunk.embedding)
  if (similarity > 0.98) return false  // แทบไม่เปลี่ยนแปลง
  
  return true  // ต้อง embed ใหม่
}
```

### Caching Strategy

**Multi-level Cache:**
```
Query → L1: In-Memory LRU (5min) 
       → L2: Semantic Cache (4hr) 
       → L3: Exact Match Cache (10hr)
```

**Semantic Cache Implementation:**
```typescript
class SellsukiSemanticCache {
  static async get(query: string, userContext: UserContext): Promise<string | null> {
    const embedding = await getEmbedding(query)
    const result = await redis.call('FT.SEARCH', 'idx:sellsuki_cache', ...)
    
    const similarity = 1 - parseFloat(score)
    if (similarity < 0.92) return null  // Threshold 92%
    
    if (!this.checkAccess(metadata, userContext)) return null
    
    return response
  }
}
```

### Metadata Filtering & Access Control

```typescript
async function searchWithFilter(query: string, user: UserContext) {
  const accessFilter = buildAccessFilter(user)
  // ['all'] for employee
  // ['all', 'hr_only'] for HR
  // ['all', 'dev_only', 'hr_only'] for Dev
  // ['all', 'hr_only', 'dev_only', 'admin_only'] for Admin
  
  const results = await sql`
    SELECT * FROM documents
    WHERE access_level = ANY(${accessFilter})
      AND metadata->>'status' = 'active'
      AND metadata->>'language' = ${user.language}
  `
}
```

### Integration Channels

**1. LINE Bot:**
```typescript
const lineClient = new LINE({ channelAccessToken: token })
const query = event.message.text
const userContext = await getUserContext(profile.userId)
const answer = await askQuestion(query, userContext)
```

**2. Slack Bot:**
```typescript
const app = new App({ token, signingSecret })
const userContext = await getUserContext(userInfo.user.email)
const answer = await askQuestion(query, userContext)
```

**3. Web Chat:**
```typescript
const session = await getSession(sessionId)
const stream = await askQuestionStream(message, session.user)
```

**4. Cursor/IDE (MCP Server):**
```typescript
mcp.addTool({
  name: 'sellsuki_search',
  async execute({ query }) {
    const user = await getUserFromToken(context.token)
    return await searchWithFilter(query, user)
  }
})
```

### Chunk Strategy สำหรับ Content ต่างๆ

| Content Type | Recommended Chunk | Size | Metadata |
|--------------|------------------|------|----------|
| Policies | Section-based | 500-1000 chars | section_title, category |
| Handbooks | Chapter + subsection | 800-1500 chars | chapter, section |
| Technical Docs | Code/function | 300-800 chars | language, framework |
| FAQs | Per question | 200-500 chars | question, keywords |
| Announcements | Per item | 200-600 chars | date, category |

### Cost Estimation

**Infrastructure (Monthly):**
- ParadeDB: $0-20
- DragonflyDB: $0-30
- VM/Hosting: $20-50
- **Total: $20-100**

**AI Cost (Per 1,000 queries):**
- Embedding: ~$0.00002
- Main AI: ~$0.30
- Query Norm: ~$0.03
- **Total without cache: ~$0.33**
- **Total with 70% cache: ~$0.10**

**Annual (10K queries/month):**
- Infrastructure: ~$600/year
- AI Cost: ~$1,200/year
- **Total: ~$1,800/year** vs RaaS ~$12,000/year

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/rag]] - RAG Architecture
- [[wiki/concepts/semantic-caching]] - Semantic Similarity Cache
- [[wiki/concepts/stateless-rag]] - Stateless Design
