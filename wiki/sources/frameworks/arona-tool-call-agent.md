---
title: "Arona — Tool Call และ Agent Loop"
type: source
source_file: raw/notes/arona/ARONA_TOOL_CALL_AGENT_LOOP_GUIDE.md
url: ""
published: 2026-04-13
tags: [arona, tool-calling, agent-loop, ai-sdk, streaming, flex-mode]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/stateless-rag-design.md, wiki/sources/arona-overview.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/frameworks/arona/ARONA_TOOL_CALL_AGENT_LOOP_GUIDE.md|Original file]]

## สรุป

อธิบายกลไก Tool Call, Agent Loop, และ Flex Mode ของ Arona ว่าทำงานอย่างไรภายใน ตั้งแต่ที่ AI ตัดสินใจเรียก tool จนถึงการ stream คำตอบกลับ

## ประเด็นสำคัญ

### Tools ที่มีใน Arona

| Tool | ไว้ทำอะไร | Input | Output |
|------|------------|-------|--------|
| `search` | ค้นหาด้วยคำสำคัญ (BM25 + Vector) | `sentence: string` | รายการเอกสารที่เกี่ยวข้อง |
| `readPage` | อ่านเนื้อหาทั้งหน้า | `link: string` | เนื้อหาเต็ม |
| `tableOfContents` | ดูเนื้อหาทั้งหมด | - | รายการเอกสารทั้งหมด |
| `readHistory` | อ่านประวัติสนทนา (ถ้ามี) | - | ข้อความก่อนหน้า |

### Agent Loop Flow

```
AI รับคำถาม
    ↓
ต้องการ Tool ไหม?
    ├── Yes → เรียก Tool → รับผลลัพธ์ → ตัดสินใจต่อ
    │            └── วนซ้ำจนพอใจหรือถึง limit
    └── No → ตอบเลย (stream)
```

### Stop Conditions (stopWhen)
```typescript
stopWhen: [
    stepCountIs(think ? 12 : 8),  // ปกติ 8 steps, think mode 12 steps
    () => references.length > 32   // ถ้า references เกิน 32 → หยุด
]
```

### ตัวอย่าง Agent Loop จริง (4 steps)
```
Step 1: AI → call search("route validation")
         ← [essential/route (0.95), essential/validation (0.88)]

Step 2: AI → call readPage("essential/route")
         ← Full content of route page

Step 3: AI → call readPage("essential/validation")
         ← Full content of validation page

Step 4: AI → พอใจแล้ว → stream คำตอบ
```

### Flex Mode (Retry with Exponential Backoff)

Flex Mode ไม่ใช่ function พิเศษ แต่เป็นแนวคิด retry ที่ใช้ทั่วระบบ:

```typescript
// ใน streamText
maxRetries: 3

// ใน retry.ts
retry(fn, retries=3, delay=(n) => Math.pow(n, 2) * 1000 + 1000)
```

| Retry | Delay |
|-------|-------|
| 1 | 2s |
| 2 | 5s |
| 3 | 10s |
| 4 | 17s |
| 5 | 26s |

ใช้ใน: Turnstile verification, search tool, semantic cache normalization

### Code Location Summary

| ส่วนประกอบ | ไฟล์ | บรรทัด |
|------------|------|--------|
| Tool Definition | `src/modules/ai/libs/tool.ts` | 10-86 |
| Agent Loop (ask) | `src/modules/ai/service.ts` | 46-133 |
| Stop Condition | `src/modules/ai/service.ts` | 85-88 |
| Retry Function | `src/libs/retry.ts` | 3-26 |
| Search Implementation | `src/modules/ai/service.ts` | 153-251 |

### Streaming Response
- ใช้ Vercel AI SDK `streamText()` — stream ทีละ chunk ไปยัง client
- ท้ายสุดต่อด้วย sources และ metadata (`---Elysia-Metadata---\nchecksum:...`)

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/stateless-rag-design|Stateless RAG Design]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG (Retrieval-Augmented Generation)]]
