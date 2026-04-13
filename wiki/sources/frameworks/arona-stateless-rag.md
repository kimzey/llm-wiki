---
title: "Stateless RAG — ทำไม Arona ไม่ต้องการ Agent ที่ซับซ้อน"
type: source
source_file: raw/notes/arona/STATELESS_RAG_EXPLAINED.md
url: ""
published: 2026-04-13
tags: [stateless, rag, stateful, scaling, documentation-search, design-philosophy]
related: [wiki/concepts/stateless-rag-design.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../../raw/notes/frameworks/arona/STATELESS_RAG_EXPLAINED.md|Original file]]

## สรุป

อธิบายปรัชญาการออกแบบ Stateless RAG ของ Arona ทำไมระบบค้นหาเอกสารจึงไม่ต้องการ stateful agent loop, memory management หรือ session state ที่ซับซ้อน

## ประเด็นสำคัญ

### Stateless vs Stateful RAG

**Stateful** (เช่น LangChain ConversationalRetrievalChain):
- เก็บ conversation history ที่ server
- เข้าใจคำแทน ("มัน", "ตัวนั้น")
- ต้องมี Sticky sessions เมื่อ scale
- Infrastructure ซับซ้อนกว่า

**Stateless** (Arona):
- ไม่เก็บ state ที่ server
- แต่ละ request อิสระจากกัน
- ถ้าต้องการ history → client ส่งมาเอง (optional, จำกัด 3 ข้อความล่าสุด)
- Scale แบบ Horizontal ได้ง่าย

### ทำไม Documentation Search เหมาะกับ Stateless

```
❌ ไม่ดี: "มันคืออะไร?" (ต้องการ context)
✅ ดี:    "Elysia คืออะไร?" (คำถามสมบูรณ์ในตัวเอง)

Documentation users มักถามคำถามที่ชัดเจนในตัวเอง
ไม่ใช่สนทนาแบบ chatbot
```

### Scaling Advantage
```
STATEFUL: ต้อง Sticky sessions (request ต้องไปหา server เดิม)
STATELESS: server ไหนก็รับ request ได้หมด → Horizontal scale ง่าย
```

### Hybrid Approach (ได้ทุกอย่าง)
```typescript
// Client จัดการ history เอง
class HybridChat {
  private history: Message[] = []
  async ask(question: string) {
    // ส่ง history 3 ล่าสุดให้ server
    const response = await fetch('/ask', {
      body: JSON.stringify({
        message: question,
        history: this.history.slice(-3)
      })
    })
    this.history.push({ role: 'user', content: question })
    this.history.push({ role: 'assistant', content: response })
    return response
  }
}
// Server ยังเป็น stateless, client ดูแล context
```

### ประโยชน์ของ Stateless (สรุป)
- ไม่ต้อง Memory storage ที่ server
- ดีบักง่าย (1 request = 1 หน่วยอิสระ)
- Reliability สูง (ไม่มีปัญหา session หาย)
- ค่าใช้จ่าย infrastructure ต่ำกว่า

### Use Cases เหมาะกับ Stateless RAG
- ค้นหาเอกสาร / Documentation Search
- FAQ Systems
- Knowledge Base queries
- API Documentation
- Technical Support

### ไม่เหมาะกับ
- Chatbot สนทนายาว
- Personal Assistants
- Long-form tutoring
- Role-playing

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

โดยทั่วไปมักคิดว่า RAG ต้องการ stateful conversation history แต่ Arona แสดงให้เห็นว่าสำหรับ documentation search คำถามสมบูรณ์ในตัวเองพอแล้ว ไม่จำเป็นต้องมี server-side memory

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/stateless-rag-design|Stateless RAG Design]]
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG (Retrieval-Augmented Generation)]]
