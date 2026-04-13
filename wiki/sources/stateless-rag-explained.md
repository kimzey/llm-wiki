---
title: "Stateless RAG: ทำไม Arona ไม่ต้องการ Agent ซับซ้อน"
type: source
source_file: raw/notes/arona/STATELESS_RAG_EXPLAINED.md
tags: [arona, stateless-rag, rag, architecture]
related: []
created: 2026-04-13
updated: 2026-04-13
---

## สรุป

คำอธิบายทำไม Arona เลือกใช้ Stateless RAG architecture แทน Stateful แบบ Chatbot ทั่วไป พร้อมเปรียบเทียบข้อดีข้อเสียและ use case ที่เหมาะสม

## ประเด็นสำคัญ

### Stateless RAG คืออะไร?

**Stateless RAG** หมายถึงการประมวลผลแต่ละคำขอแยกจากกันโดยไม่ต้องเก็บ "สถานะการสนทนา" ไว้ที่ Server ระบบจะไม่ "จำ" บทสนทนาที่ผ่านมา ยกเว้นผู้ใช้ส่งมาให้ชัดเจน

### เปรียบเทียบ Stateful vs Stateless

**Stateful RAG (Chatbot สนทนา):**
```
Request 1: "Elysia คืออะไร?"
→ AI ตอบ, เก็บ context ไว้ใน memory

Request 2: "ติดตั้งยังไง?"
→ AI ใช้ context จาก Request 1 ("มัน" = Elysia)
```

**Stateless RAG (Documentation Search):**
```
Request 1: "Elysia คืออะไร?"
→ AI ตอบ

Request 2: "ติดตั้งยังไง?"
→ AI มองเป็นคำถามใหม่ (ไม่รู้ว่าคุยเรื่อง Elysia)
```

### ทำไม Arona เลือก Stateless

#### 1. Use Case: ค้นหาเอกสาร
คำถามเกี่ยวกับเอกสารมักเป็น "คำถามที่สมบูรณ์":
- ❌ ไม่ดี: "มันคืออะไร?"
- ✅ ดี: "Elysia คืออะไร?" (ชัดเจน)

#### 2. ประหยัดค่าใช้จ่าย
```typescript
// Stateful ต้องมี Memory storage
const statefulCost = {
  memory: "Redis/Database สำหรับเก็บ session",
  complexity: "Logic สำหรับจัดการ context",
  retrieval: "ดึง history ทุกครั้ง",
  tokens: "ต้องใส่ history ทุกครั้ง"
}

// Stateless ง่ายกว่า
const statelessCost = {
  memory: "ไม่มี (client จัดการเอง)",
  complexity: "ไม่มี",
  retrieval: "ไม่มี",
  tokens: "เฉพาะคำถามปัจจุบัน"
}
```

#### 3. ขยายง่ายกว่า (Scaling)
```
STATEFUL:
├── Server 1 (เก็บ sessions A, B, C)
├── Server 2 (เก็บ sessions D, E, F)
❌ ต้อง Sticky sessions
❌ ต้องมี Session management

STATELESS:
├── Server 1 (รับ request ได้ทั้งหมด)
├── Server 2 (รับ request ได้ทั้งหมด)
✅ server ไหนก็ได้ จัดการได้หมด
✅ ไม่ต้องมี Session management
```

### ข้อดี-ข้อเสีย

**Stateless Pros:**
- ขยายแบบ Horizontal ง่าย
- ไม่ต้องจัดการ Session
- ลดค่าใช้จ่าย Memory/Infrastructure
- ไม่มีปัญหา Session หาย
- ดีบักง่าย (request = 1 หน่วย)
- ไม่ต้องมี Sticky sessions

**Stateless Cons:**
- สนทนายาวไม่ได้
- คำแทนต้องระบุชัดเจน
- ไม่มีความจำแต่ละ user
- คำถามต่อยอดไม่เป็นธรรมชาติ

### ทำไมข้อเสียไม่สำคัญสำหรับ Documentation?

Documentation search คือ:
- ใช้ความจริงเป็นหลัก (ไม่ใช่สนทนา)
- คำถามสมบูรณ์ในตัวเอง
- ตอบครั้งเดียว
- เน้นการอ้างอิงแหล่งที่มา

### Hybrid Approach (ได้ทุกอย่าง)

Stateless RAG + Client-side Conversation History

```typescript
class HybridChat {
  private history: Message[] = []

  async ask(question: string) {
    // 1. เพิ่มลงใน client-side history
    this.history.push({ role: 'user', content: question })

    // 2. เรียก Stateless RAG พร้อม history ล่าสุด
    const response = await fetch('/ask', {
      body: JSON.stringify({
        message: question,
        history: this.history.slice(-3)  // เฉพาะ 3 ล่าสุด
      })
    })

    // 3. อัปเดต client-side history
    this.history.push({ role: 'assistant', content: response })

    return response
  }

  // ประโยชน์:
  // - Server ยังเป็น Stateless (ขยายง่าย)
  // - Client ดูแล context การสนทนา
  // - User ได้ประสบการณ์สนทนา
  // - ไม่ต้องมี Session management ที่ Server
}
```

### เหมาะกับ Use Case อะไร?

**เหมาะอย่างยิ่ง:**
- ค้นหาเอกสาร (Documentation Search)
- ระบบ FAQ
- ค้นหา Knowledge Base
- ปัญหาเทคนิค (Technical Support)
- ค้นหาอ้างอิง (Reference Lookups)
- API Documentation
- ตัวอย่างโค้ด (Code Examples)

**ไม่เหมาะ:**
- Chatbot สนทนา
- Personal Assistants
- สอนยาวๆ (Long-form Tutoring)
- การปรึกษา/โค้ชชิ่ง (Therapy/Coaching)
- การใช้เหตุผลหลายขั้น (Multi-step Reasoning)
- การรับบทบาท (Role-playing Characters)

## Concepts ที่เกี่ยวข้อง
- [[wiki/concepts/rag]] - RAG Architecture
- [[wiki/concepts/stateless-rag]] - Stateless Design Pattern
