---
title: "Stateless RAG Design"
type: concept
tags: [stateless, rag, design-philosophy, scaling, documentation-search, architecture]
sources: [arona/STATELESS_RAG_EXPLAINED.md, arona/RAG_COMPARISON.md]
related: [wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Stateless RAG คือการออกแบบระบบ RAG ที่ไม่เก็บ conversation state ที่ server — แต่ละ request ประมวลผลอิสระจากกัน เหมาะอย่างยิ่งสำหรับ documentation search ที่คำถามสมบูรณ์ในตัวเอง ให้ข้อดีด้าน scalability และ simplicity สูงกว่า stateful approach

## อธิบาย

### Stateful vs Stateless

**Stateful RAG:**
- Server เก็บ conversation history ของแต่ละ user
- AI เข้าใจคำแทน ("มัน", "ตัวนั้น") จาก context ก่อนหน้า
- ต้องมี Sticky sessions เมื่อ scale horizontal (request ต้องไป server เดิม)
- Infrastructure ซับซ้อน: session storage, memory management, replication

**Stateless RAG (Arona):**
- ไม่เก็บ state ที่ server
- ถ้าต้องการ history → client ส่งมาเอง (optional, จำกัด 3 ข้อความล่าสุด)
- server ไหนก็รับ request ได้ทั้งหมด → horizontal scale ง่าย
- infrastructure เรียบง่าย

### ทำไม Documentation Search เหมาะกับ Stateless

ผู้ใช้ที่ค้นหาเอกสารมักถามคำถามที่สมบูรณ์ในตัวเอง:

```
❌ Style สนทนา (ต้องการ context):
   Q: "Elysia คืออะไร?"
   Q: "ติดตั้งยังไง?" ← "มัน" คืออะไร?

✅ Style ค้นหาเอกสาร (ชัดเจนในตัวเอง):
   Q: "Elysia คืออะไร?"
   Q: "ติดตั้ง Elysia ยังไง?" ← ไม่ต้องการ context
```

### Scaling Advantage

```
STATEFUL:
├── Server 1: sessions A, B, C (ต้อง sticky)
├── Server 2: sessions D, E, F
└── ปัญหา: request ต้องไปหา server ที่เก็บ session

STATELESS:
├── Server 1, 2, 3: รับ request ใดก็ได้
└── เพิ่ม server ใหม่ → รับ load ทันที ไม่ต้อง migrate state
```

## ประเด็นสำคัญ

### Hybrid Approach (ได้ทุกอย่าง)

Arona รองรับ history แต่ client จัดการเอง:

```typescript
// Client-side
class DocumentationChat {
  private history: Message[] = []

  async ask(question: string) {
    const response = await fetch('/ask', {
      body: JSON.stringify({
        message: question,
        history: this.history.slice(-3)  // ส่งเฉพาะ 3 ล่าสุด
      })
    })
    this.history.push({ role: 'user', content: question })
    this.history.push({ role: 'assistant', content: response })
    return response
  }
}
// Server ยัง stateless, client ดูแล context
```

### Client-side History Compression

Arona มี `compressHistory()` ที่ทำงาน server-side:
- รับ history จาก client
- compress ให้สั้นลง (เอาเฉพาะ essence)
- ใช้เพียง 3 ข้อความล่าสุด เพื่อไม่ให้ token บานปลาย

### Checksum ป้องกัน History Tampering

ทุก assistant message มี checksum:
```typescript
checksum = sha256(content + secret)
// Server verify ด้วย timingSafeEqual ก่อนใช้ history
```
ป้องกัน client ส่ง history ปลอมมา

### Tool Calling (ไม่ใช่ Agent State Machine)

แม้ AI จะใช้ tools หลายครั้งใน request เดียว แต่ไม่มี "agent state" ที่ persist:
```
Request 1: search → readPage → ตอบ (loop ใน request เดียว)
Request 2: เริ่มใหม่ทั้งหมด ไม่จำว่า Request 1 ใช้ tool อะไร
```

### เหมาะกับ Use Cases ใด

**เหมาะ:**
- Documentation Search
- FAQ Systems
- Knowledge Base queries
- API Reference

**ไม่เหมาะ:**
- Conversational Chatbot (ต้องการ multi-turn context)
- Personal Assistants
- Long-form tutoring

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — stateless เป็นหนึ่งใน RAG architectures
- [[wiki/concepts/semantic-caching|Semantic Caching]] — cache ช่วยรักษา UX แม้ไม่มี state

## แหล่งที่มา

- [[wiki/sources/arona-stateless-rag|Stateless RAG Explained]]
- [[wiki/sources/arona-tool-call-agent|Tool Call & Agent Loop]]
