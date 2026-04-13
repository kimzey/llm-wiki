---
title: "เปรียบเทียบวิธีการทำ RAG: DIY vs Framework vs Off-the-shelf"
type: source
source_file: raw/notes/arona/RAG_COMPARISON.md
tags: [rag-comparison, diy, framework, off-the-shelf, cost-analysis]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุป

เปรียบเทียบ 3 วิธีในการสร้าง RAG system: DIY (แบบ Arona), LangChain, LlamaIndex และ Off-the-shelf solutions พร้อมคะแนน cost, flexibility, time-to-market และคำแนะนำ

## ประเด็นสำคัญ

### Executive Summary

| วิธี | Cost | Flexibility | Time-to-Market | เหมาะกับ | Score |
|------|------|-------------|----------------|------------|-------|
| **DIY (Arona)** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | Control, Scale, Custom | **4.2/5** |
| **LangChain** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Rapid Dev, POC | **3.5/5** |
| **LlamaIndex** | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | Data-heavy RAG | **3.8/5** |
| **Off-the-shelf** | ⭐ | ⭐⭐ | ⭐⭐⭐⭐⭐ | Quick MVP, Small Biz | **2.5/5** |

### Cost Comparison (ต่อเดือน)

```
Off-the-shelf:    $250 - $1,000+
LangChain (DIY):  $50 - $100
LlamaIndex (DIY): $50 - $100
DIY (Arona):      $20 - $30

Per 1,000 queries:
- Off-the-shelf:  $30 - $100
- Framework DIY:  $5 - $15
- Pure DIY:       $3 - $8
```

### Time to Market

```
Off-the-shelf:    1 วัน (config)
LangChain:        3-7 วัน (dev)
LlamaIndex:       5-10 วัน (dev)
DIY (Arona):      3-4 สัปดาห์ (scratch)
```

### Flexibility Score

```
DIY (Arona):      ████████████ 100%
LlamaIndex:       ████████░░░░ 80%
LangChain:        ███████░░░░░ 70%
Off-the-shelf:    ███░░░░░░░░░ 30%
```

### การวิเคราะห์ Arona (DIY Approach)

**แนวคิดหลัก:**
> "Cut cost เยอะๆ เลยเล่นหลายๆ ท่า เพื่อลด cost ให้เข้า AI น้อยสุด"

**หลักการ:**
1. ลด AI call ด้วยการ cache ทุกแบบ
2. ใช้ model ที่ถูกและเร็วที่สุด
3. ทำทุกอย่างไว้ที่เดียว (ลด latency)
4. Control write process ทั้งหมด (scale ง่าย)

**ทำไมไม่ใช้ Framework?**
> "แอบรู้สึกว่า overkill ไปนิดนึง แต่หลายๆ อย่างก็ทำเองไปแล้วชนกับของที่มีใน langchain"

สิ่งที่ทำเอง ≈ มีใน Framework:
- LLM normalized + semantic search
- Parent document retrieval
- Document processing (webhook + cron)
- OpenTelemetry tracing
- Tool calling / Agent loop

### DIY - ข้อดี/ข้อเสีย

**ข้อดี:**
```typescript
const pros = {
  cost: "ต่ำที่สุด (~$21/เดือน)",
  control: "คุมทุกอย่าง ปรับแต่งได้ทั้งหมด",
  performance: "ไม่มี overhead จาก framework",
  learning: "เข้าใจระบบลึกๆ",
  scale: "เขียนเอง scale แนวๆ เราได้เลย",
  vendor: "ไม่ติด lock-in"
}
```

**ข้อเสีย:**
```typescript
const cons = {
  time: "ใช้เวลานาน (3-4 สัปดาห์)",
  maintenance: "ต้องดูแลเองทั้งหมด",
  complexity: "ต้องเข้าใจหลายด้าน",
  debugging: "ต้องมี observability ดี",
  updates: "อัปเดต feature ใหม่ต้องทำเอง"
}
```

### LangChain - ข้อดี/ข้อเสีย

**ข้อดี:**
```typescript
const pros = {
  speed: "เริ่มต้นเร็วมาก (1-2 วัน)",
  features: "มี feature ครบ (chains, agents, tools)",
  community: "Community ใหญ่ มี template เยอะ",
  integration: "รองรับ LLM/VectorDB หลากหลาย",
  maintenance: "Framework ดูแล update"
}
```

**ข้อเสีย:**
```typescript
const cons = {
  overhead: "มี abstraction layer เยอะ",
  control: "คุมละเอียดไม่ได้เท่าทำเอง",
  learning: "ต้องเรียนรู้วิธีของ framework",
  performance: "มี overhead จาก abstraction",
  debug: "Debug ยากกว่า (หลาย layer)"
}
```

### LlamaIndex - จุดเด่น

**เหมาะกับ Data-heavy RAG:**
- Data connectors หลากหลาย
- Index หลายแบบ (Vector, Keyword, List, Tree)
- Query engines หลายแบบ
- Auto-routing ให้ query engine

### Off-the-shelf Options

| Service | Price/เดือน | Pros | Cons |
|---------|-------------|------|------|
| **Mintlify** | $250 | UI สวย, มี doc system | แพง, ต้องย้าย docs |
| **Kapa** | $1000+ | Custom AI, Enterprise | แพงมาก, contract ยาว |
| **Mendable** | $500+ | Easy integrate | แพง, ไม่บอกราคา |
| **DocsGPT** | $199 | Open source | Limit feature |

### เทคนิคที่ Arona ใช้

#### 1. Semantic Caching
ใช้ Vector Similarity หาคำถามที่ "คล้ายกัน"
- "ลาพักร้อนกี่วัน" ≈ "ลาพักผ่อนประจำปีกี่วัน" (90%+)

#### 2. Summarized Content
สรุปเนื้อหาตอน index เพื่อลด token
- Original: 2000 tokens
- Summarized: 200 tokens (90% reduction)

#### 3. Hybrid Search
รวม BM25 + Vector Search
```typescript
const score = 0.775 * bm25Score + 0.275 * weight
```

#### 4. Incremental Indexing
Update เฉพาะที่เปลี่ยน
```typescript
if (oldHash !== newHash) {
  const similarity = cosineSimilarity(oldEmbedding, newEmbedding)
  if (similarity > 0.98) updateMetadataOnly()
  else reEmbedAndUpdate()
}
```

### คำแนะนำตาม Scenario

**Scenario 1: Startup (2-3 devs)**
```
เลือก: DIY (Arona)
Timeline: 3-4 weeks
Cost: ~$20-30/month
```

**Scenario 2: Enterprise (เอกสารเยอะ)**
```
เลือก: LlamaIndex (หรือ DIY)
Timeline: 4-6 weeks
Cost: ~$50-100/month
```

**Scenario 3: POC**
```
เลือก: LangChain
Timeline: 1-2 weeks
Cost: ~$30-50/month
```

**Scenario 4: Non-tech**
```
เลือก: Off-the-shelf
Timeline: 1-3 days
Cost: ~$250-1000/month
```

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/rag]] - RAG Architecture
- [[wiki/sources/arona-vs-langchain]] - Detailed Comparison
- [[wiki/sources/rag-recommendation]] - Recommendation Guide
