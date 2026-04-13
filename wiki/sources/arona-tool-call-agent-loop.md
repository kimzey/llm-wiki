---
title: "Arona RAG: Tool Call และ Agent Loop อธิบายเป็นภาษาไทย"
type: source
source_file: raw/notes/arona/ARONA_TOOL_CALL_AGENT_LOOP_GUIDE.md
tags: [arona, tool-calling, agent-loop, ai-orchestration]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุป

คำอธิบาย Tool Call และ Agent Loop ของ Arona อย่างละเอียด - วิธีการทำงานของ AI Tools, Agent Loop, Flex Mode และตำแหน่งโค้ดในระบบ

## ประเด็นสำคัญ

### Tool Call คืออะไร?

**Tool Call** คือกลไกที่ให้ AI Model สามารถ "เรียกใช้ฟังก์ชัน" เพื่อดึงข้อมูลเพิ่มเติมระหว่างการตอบคำถาม

เปรียบเหมือนการให้ผู้เชี่ยวชาญมี "คู่มือ" อยู่ข้างๆ
- เมื่อผู้เชี่ยวชาญไม่แน่ใจ → เปิดคู่มือ (เรียก Tool)
- เมื่อผู้เชี่ยวชาญต้องการข้อมูลเฉพาะ → ค้นหาในคู่มือ (เรียก Tool)
- เมื่อข้อมูลพอแล้ว → นำมาสรุปตอบ (สิ้นสุด)

### Tools ที่มีใน Arona

| Tool | ไว้ทำอะไร | Input | Output |
|------|------------|-------|--------|
| **search** | ค้นหาด้วยคำสำคัญ | `sentence: string` | รายการเอกสารที่เกี่ยวข้อง |
| **readPage** | อ่านเนื้อหาทั้งหน้า | `link: string` | เนื้อหาเต็มๆ ของหน้านั้น |
| **tableOfContents** | ดูเนื้อหาทั้งหมด | - | รายการเอกสารทั้งหมด |
| **readHistory** | อ่านประวัติการสนทนา | - | ข้อความเก่าๆ |

### Agent Loop ทำงานยังไง

**Agent Loop** คือกระบวนการที่ AI Model "คิดและกระทำ" ซ้ำๆ จนกว่าจะได้คำตอบที่น่าพอใจ

**Flow หลัก:**
```
User: "How do I create a route with validation?"

Step 1: AI ตัดสินใจ → เรียก search("route validation")
Step 2: search คืนผลลัพธ์ → AI อ่านแล้วยังไม่พอใจ
Step 3: AI ตัดสินใจ → เรียก readPage("essential/validation")
Step 4: readPage คืนเนื้อหา → AI พอใจแล้ว
Step 5: AI สรุปและตอบกลับ
```

**Stop Conditions:**
```typescript
stopWhen: [
  stepCountIs(think ? 12 : 8),  // สูงสุด 8 หรือ 12 steps
  () => references.length > 32   // หรือ references เกิน 32
]
```

### Flex Mode คืออะไร?

**Flex Mode** ไม่ใช่ชื่อ function ในโค้ด แต่เป็น **แนวคิด** ในการรับมือกับความผิดพลาด

> "ยอมรับว่า AI อาจจะผิดพลาด แต่จะ retry ด้วย exponential backoff จนกว่าจะสำเร็จ"

**ตำแหน่งในโค้ด:**
```typescript
maxRetries: 3  // ใน streamText
```

**Exponential Backoff Formula:**
```typescript
retry(..., 5, (n) => Math.pow(n, 2) * 1000 + 1000)
```

| Retry # | Delay | Formula |
|---------|-------|---------|
| 1 | 2s | (1² × 1000) + 1000 |
| 2 | 5s | (2² × 1000) + 1000 |
| 3 | 10s | (3² × 1000) + 1000 |
| 4 | 17s | (4² × 1000) + 1000 |
| 5 | 26s | (5² × 1000) + 1000 |

### โค้ดตัวอย่าง Tool Definition

**ตำแหน่ง:** `src/modules/ai/libs/tool.ts`

```typescript
// Search Tool
export const createSearchTool = (references: Reference[]) =>
  tool({
    description: "Search relevant information from Elysia documentation",
    inputSchema: z.object({
      sentence: z.string().meta({
        description: 'The keyword/sentence to search',
        examples: ['handler', 'OpenAPI type gen']
      })
    }),
    async execute({ sentence }) {
      let documents = await retry(
        () => cache(`search: ${sentence}`, () => search(sentence)),
        3
      )
      
      const newDocuments = documents.filter(
        (document) => !refs.includes(document.link)
      )
      
      references.push(...newDocuments)
      return newDocuments
    }
  })

// Read Page Tool
export const createPageTool = (references: Reference[]) =>
  tool({
    description: "Read a specific page with detail",
    inputSchema: z.object({
      link: z.string()
    }),
    async execute({ link }) {
      const documents = await retry(
        () => cache(`page:${link}`, () => readPage(link)),
        3
      )
      
      references.push(...documents)
      return documents
    }
  })
```

### โค้ด Agent Loop หลัก

**ตำแหน่ง:** `src/modules/ai/service.ts:46-133`

```typescript
export function ask({ message, history, references, think }: AskParams) {
  // 1. สร้าง Tools
  const searchTool = createSearchTool(references)
  const readPageTool = createPageTool(references)
  
  // 2. เรียก streamText (AI SDK จัดการ Agent Loop)
  return streamText({
    model,
    tools: {
      readPage: readPageTool,
      search: searchTool,
      tableOfContents: tableOfContentsTool
    },
    stopWhen: [
      stepCountIs(think ? 12 : 8),
      () => references.length > 32
    ],
    maxRetries: 3,  // Flex mode!
    system: instruction
  }).textStream
}
```

### Key Concepts Summary

| คำถาม | คำตอบ |
|--------|--------|
| Tool Call อยู่ตรงไหน | ใน `streamText()` จาก AI SDK |
| Tool ทำงานยังไง | AI ตัดสินใจเรียก → Execute → คืนผลลัพธ์ |
| Loop จนพอใจคืออะไร | Agent Loop เรียก Tool ซ้ำๆ จนพอใจ |
| Stop Condition | `stepCountIs(8)` หรือ `references > 32` |
| Flex Mode คืออะไร | Retry ด้วย exponential backoff |
| Flex Mode อยู่ตรงไหน | `maxRetries: 3` ใน streamText |

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/rag]] - RAG Architecture
- [[wiki/concepts/tool-calling]] - AI Tool Calling
- [[wiki/concepts/agent-loop]] - Agent Loop Pattern
