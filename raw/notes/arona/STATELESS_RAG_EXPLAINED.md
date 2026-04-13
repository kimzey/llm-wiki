# Stateless RAG: ทำไม Arona ไม่ต้องการ Agent ที่ซับซ้อน

## สารบัญ

1. [Stateless RAG คืออะไร?](#stateless-rag-คืออะไร)
2. [Stateful vs Stateless RAG](#stateful-vs-stateless-rag)
3. [ทำไม Arona เลือกแบบ Stateless](#ทำไม-arona-เลือกแบบ-stateless)
4. [Stateless RAG ทำงานยังไง](#stateless-rag-ทำงานยังไง)
5. [ข้อดี-ข้อเสีย](#ข้อดี-ข้อเสีย)
6. [การนำไปใช้](#การนำไปใช้)

---

## Stateless RAG คืออะไร?

**Stateless RAG** หมายถึงการประมวลผลแต่ละคำขอแยกจากกันโดยไม่ต้องเก็บ "สถานะการสนทนา" ไว้ที่ Server ระบบจะไม่ "จำ" บทสนทนาที่ผ่านมา ยกเว้นผู้ใช้ส่งมาให้ชัดเจน

### เปรียบเทียบแบบเห็นภาพ:

```
┌─────────────────────────────────────────────────────────────────┐
│                  STATEFUL RAG (แบบมีสถานะ)                  │
│                     ใช้ใน Chatbot สนทนา                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request 1: "Elysia คืออะไร?"                                 │
│    → AI ตอบ, เก็บ context ไว้ใน memory                         │
│                                                                 │
│  Request 2: "ติดตั้งยังไง?"                                   │
│    → AI ใช้ context จาก Request 1 ("มัน" = Elysia)             │
│                                                                 │
│  ต้องการ: Memory storage, Context management                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  STATELESS RAG (แบบไร้ยสถานะ)                │
│                     ใช้ใน Documentation Search                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Request 1: "Elysia คืออะไร?"                                 │
│    → AI ตอบ                                                   │
│                                                                 │
│  Request 2: "ติดตั้งยังไง?"                                   │
│    → AI มองเป็นคำถามใหม่ (ไม่รู้ว่าคุยเรื่อง Elysia)           │
│                                                                 │
│  ไม่ต้องการ: Memory, Context carryover                       │
└─────────────────────────────────────────────────────────────────┘
```

---

## Stateful vs Stateless RAG

### Stateful RAG (แบบ Chatbot ทั่วไป)

ใช้ใน LangChain เป็นต้น:

```python
# LangChain style - Stateful
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationalRetrievalChain

memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True
)

chain = ConversationalRetrievalChain.from_llm(
    llm=llm,
    retriever=retriever,
    memory=memory,  # ← เก็บ state ไว้
    verbose=True
)

# สนทนาครั้งที่ 1
response1 = chain({
    "question": "Elysia คืออะไร?"
})
# เก็บใน memory: { HumanMessage: "...", AIMessage: "..." }

# สนทนาครั้งที่ 2
response2 = chain({
    "question": "สร้าง route ยังไง?"
})
# AI มีประวัติการสนทนาให้ใช้
```

**คุณสมบัติ:**
- ✅ เก็บประวัติการสนทนา
- ✅ เข้าใจคำแทน (เช่น "มัน", "ตัวนั้น")
- ✅ เหมาะกับ Chatbot สนทนา
- ❌ ต้องมีระบบจัดการ Memory
- ❌ สถาปัตยกรรมซับซ้อน

### Stateless RAG (สไตล์ Arona)

```typescript
// Arona style - Stateless
export async function ask(params: {
  message: string
  history?: History[]  // ถ้ามี ผู้ใช้ต้องส่งมาเอง
  references: Reference[]
}) {
  const { message, history, references } = params

  // ประมวลผลเฉพาะที่ส่งมาให้เท่านั้น
  const stream = streamText({
    model,
    tools: { search, readPage, tableOfContents },
    messages: [
      { role: 'user', content: message },
      // ใช้เฉพาะ 3 ข้อความล่าสุด ถ้ามี history
      ...(history ? compressHistory(history).slice(0, 3) : [])
    ]
  })

  return stream.textStream
}
```

**คุณสมบัติ:**
- ✅ ไม่ต้องเก็บ Memory ที่ Server
- ✅ แต่ละ Request อิสระจากกัน
- ✅ Client จัดการ history เองถ้าต้องการ
- ✅ สถาปัตยกรรมง่ายกว่า
- ✅ ขยายง่ายกว่า (Scale)

---

## ทำไม Arona เลือกแบบ Stateless

จากคำพูดของผู้สร้าง:

> "RAG ไม่ต้องพึ่ง memory, stateful, agent loop มาก"

### 1. Use Case: ค้นหาเอกสาร

คำถามเกี่ยวกับเอกสารมักเป็น "คำถามที่สมบูรณ์":

```
❌ ไม่ดี: "มันคืออะไร?" (มันคืออะไร?)
✅ ดี: "Elysia คืออะไร?" (ชัดเจน)

❌ ไม่ดี: "เพิ่มยังไง?" (เพิ่มอะไรยังไง?)
✅ ดี: "เพิ่ม plugin ยังไง?" (ชัดเจน)

ผู้ใช้ส่วนใหญ่ถามคำถามที่สมบูรณ์ ไม่ใช่แบบสนทนา
```

### 2. ประหยัดค่าใช้จ่าย

```typescript
// Stateful ต้องมี Memory storage
const statefulCost = {
  memory: "Redis/Database สำหรับเก็บ session",
  complexity: "Logic สำหรับจัดการ context",
  retrieval: "ดึง history ทุกครั้งที่รับ request",
  tokens: "ต้องใส่ history ทุกครั้งใน prompt"
}

// Stateless ง่ายกว่า
const statelessCost = {
  memory: "ไม่มี (client จัดการเอง)",
  complexity: "ไม่มี",
  retrieval: "ไม่มี",
  tokens: "เฉพาะคำถามปัจจุบัน"
}
```

### 3. ขยายง่ายกว่า (Scaling)

```
STATEFUL:
┌─────────────┐
│ Load Balancer │
└──────┬──────────┘
       │
       ├─→ Server 1 (เก็บ sessions A, B, C)
       ├─→ Server 2 (เก็บ sessions D, E, F)
       └─→ Server 3 (เก็บ sessions G, H, I)

❌ ต้อง Sticky sessions (request ต้องไป server เดิม)
❌ ต้องมี Session management
❌ ต้องมี Memory replication

STATELESS:
┌─────────────┐
│ Load Balancer │
└──────┬──────────┘
       │
       ├─→ Server 1 (รับ request ได้ทั้งหมด)
       ├─→ Server 2 (รับ request ได้ทั้งหมด)
       └─→ Server 3 (รับ request ได้ทั้งหมด)

✅  server ไหนก็ได้ จัดการ request ได้ทั้งหมด
✅ ไม่ต้องมี Session management
✅ ขยายแบบ Horizontal ง่ายๆ
```

### 4. ใช้ Tool Calling แทน Agent ที่ซับซ้อน

```typescript
// Arona ใช้ Tool Calling แบบง่าย
const tools = {
  search: "ค้นหาเอกสารจาก keyword",
  readPage: "อ่านเอกสารทั้งหน้า",
  tableOfContents: "ดูเนื้อหาทั้งหมด"
}

// AI จะตัดสินใจเองว่าทำอะไร (สูงสุด 8 steps)
// ไม่ต้องมี Agent state machine ที่ซับซ้อน
// ไม่ต้องจำว่าครั้งก่อนใช้ Tool อะไร
// แค่: คำถาม → เรียก Tool → ตอบ
```

---

## Stateless RAG ทำงานยังไง

### Tool-Calling Loop

```
User: "สร้าง route พร้อม validation ยังไง?"

Step 1: AI รับคำถาม
├─ ไม่มีความจำ
├─ มีแค่คำถาม
└─ มี Tools ให้ใช้

Step 2: AI ตัดสินใจค้นหา
├─ เรียก: search("สร้าง route")
└─ ได้: [เอกสารพื้นฐาน route]

Step 3: AI ต้องการข้อมูลเพิ่ม
├─ เรียก: search("validation")
└─ ได้: [เอกสาร validation]

Step 4: AI อ่านรายละเอียด
├─ เรียก: readPage("essential/validation")
└─ ได้: เนื้อหาทั้งหน้า validation

Step 5: AI มีข้อมูลพอแล้ว
├─ สรุปคำตอบ
├─ Stream ส่งคำตอบ
└─ เสร็จ (ไม่เก็บ state)

Request ถัดไป: เริ่มใหม่ทั้งหมด
```

### เมื่อต้องการ History

Arona รองรับ History แต่:

```typescript
// Client ส่ง history มาเอง (optional)
const response = await fetch('/ask', {
  body: JSON.stringify({
    message: "แล้วเพิ่ม validation ยังไง?",
    history: [
      { role: "user", content: "สร้าง route ยังไง?" },
      { role: "assistant", content: "สร้าง route ทำ...", checksum: "..." }
    ]
  })
})

// แต่:
// 1. History จำกัด 3 ข้อความล่าสุด
// 2. History ถูกบีบอัด (compress)
// 3. ไม่เก็บที่ Server
// 4. Checksum ป้องกันการปลอมแปลง
```

---

## ข้อดี-ข้อเสีย

### Stateless Pros (ข้อดี)

```typescript
const statelessPros = {
  scaling: "ขยายแบบ Horizontal ง่าย",
  simplicity: "ไม่ต้องจัดการ Session",
  cost: "ลดค่าใช้จ่าย Memory/Infrastructure",
  reliability: "ไม่มีปัญหา Session หาย",
  debugging: "ดีบักง่าย (request = 1 หน่วย)",
  deployment: "ไม่ต้อง Sticky sessions"
}
```

### Stateless Cons (ข้อเสีย)

```typescript
const statelessCons = {
  conversation: "สนทนายาวไม่ได้",
  context: "คำแทนต้องระบุชัดเจน",
  personalization: "ไม่มีความจำแต่ละ user",
  followup: "คำถามต่อยอดไม่เป็นธรรมชาติ"
}
```

### ทำไมข้อเสียไม่สำคัญสำหรับ Documentation?

```
❌ ไม่สำคัญ: "แล้วเพิ่ม validation ยังไง?"
✅ ผู้ใช้ถาม: "เพิ่ม validation ให้ route ยังไง?"

❌ ไม่สำคัญ: "ผมสามารถทำอะไรได้บ้าง"
✅ ผู้ใช้ถาม: "สร้าง plugin ยังไง?"

Documentation search คือ:
- ใช้ความจริงเป็นหลัก (ไม่ใช่สนทนา)
- คำถามสมบูรณ์ในตัวเอง
- ตอบครั้งเดียว
- เน้นการอ้างอิงแหล่งที่มา
```

---

## การนำไปใช้

### Pattern ของ Stateless RAG

```typescript
// 1. กำหนด Interface (stateless)
interface AskRequest {
  message: string
  history?: History[]  // Optional, client จัดการ
  reference?: string   // Context เพิ่ม
  think?: boolean      // ระดับการใช้คิด
}

// 2. ประมวลผลแยกจากกัน
export async function ask(req: AskRequest) {
  const { message, history, reference, think } = req

  // ตรวจสอบ Cache (ไม่มี state, เฉพาะ similarity)
  const cached = await getCache(message)
  if (cached) return cached

  // ดึงข้อมูลเฉพาะจาก request นี้เท่านั้น
  const references = reference
    ? await readPage(reference)
    : []

  // สร้างคำตอบ (ไม่มี external state)
  const response = streamText({
    model,
    tools: createTools(references),
    messages: buildMessages(message, history),
    stopWhen: [
      stepCountIs(think ? 12 : 8),
      () => references.length > 32
    ]
  })

  // เก็บใน Cache (query เดียวกัน = คำตอบเดียวกัน)
  await setCache(message, response)

  return response
}

// 3. Client จัดการ History (ถ้าต้องการ)
// ใน Frontend:
const chatHistory = []

async function sendMessage(message) {
  const response = await fetch('/ask', {
    body: JSON.stringify({
      message,
      history: chatHistory.slice(-3)  // ส่งเฉพาะ 3 ล่าสุด
    })
  })

  chatHistory.push(
    { role: 'user', content: message },
    { role: 'assistant', content: response }
  )
}
```

### Tool-Calling โดยไม่มี Agent State

```typescript
// แนวทาง Arona: Tools ง่าย, Loop ง่าย

type Tool = {
  description: string
  input: z.Schema
  execute: (input) => Promise<Result>
}

const tools: Record<string, Tool> = {
  search: {
    description: "ค้นหาเอกสาร",
    input: z.object({ sentence: z.string() }),
    execute: async ({ sentence }) => {
      const results = await hybridSearch(sentence)
      references.push(...results)
      return results
    }
  },
  readPage: {
    description: "อ่านเนื้อหาทั้งหน้า",
    input: z.object({ link: z.string() }),
    execute: async ({ link }) => {
      const page = await database.getPage(link)
      references.push(page)
      return page
    }
  }
}

// AI ใช้ Tools แต่ไม่มี "agent state"
// แค่: คำถาม → เรียก Tool → ตอบ
// ไม่จำว่าก่อนหน้านี้ใช้ Tool อะไร
// ไม่เรียนรู้จากการโต้ตอบก่อนหน้า
```

---

## เหมาะกับ Use Case อะไร?

### เหมาะอย่างยิ่ง:

```
✅ ค้นหาเอกสาร (Documentation Search)
✅ ระบบ FAQ
✅ ค้นหา Knowledge Base
✅ ปัญหาเทคนิค (Technical Support)
✅ ค้นหาอ้างอิง (Reference Lookups)
✅ API Documentation
✅ ตัวอย่างโค้ด (Code Examples)
```

### ไม่เหมาะ:

```
❌ Chatbot สนทนา
❌ Personal Assistants
❌ สอนยาวๆ (Long-form Tutoring)
❌ การปรึกษา/โค้ชชิ่ง (Therapy/Coaching)
❌ การใช้เหตุผลหลายขั้น (Multi-step Reasoning)
❌ การรับบทบาท (Role-playing Characters)
```

---

## Hybrid Approach (ได้ทุกอย่าง)

```typescript
// Stateless RAG + Client-side Conversation History

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

---

## สรุป

### ปรัชญาของการออกแบบ Arona

```
RAG สำหรับ Documentation ≠ Conversational AI

Documentation RAG:
- คำถามสมบูรณ์ในตัวเอง
- คำตอบอ้างอิงความจริง
- Context อยู่ในคำถาม
- การอ้างอิงสำคัญกว่าการสนทนา

ดังนั้น:
- ไม่ต้องการ Agent loop ที่ซับซ้อน
- ไม่ต้องการ Memory ที่ Server
- ไม่ต้องการ Conversation state
- Tool calling แบบง่ายเพียงพอ

ผลลัพธ์:
- สถาปัตยกรรมง่ายกว่า
- ค่าใช้จ่ายต่ำกว่า
- ขยายง่ายกว่า
- เชื่อถือได้มากกว่า
```

### ข้อควรจำ

1. **Stateless RAG ใช้ได้กับการค้นหาเอกสาร** เพราะคำถามสมบูรณ์ในตัวเอง
2. **Tool calling แทนการมี Agent ที่ซับซ้อน** - AI ตัดสินใจเองทุก request
3. **Client จัดการ History** ถ้าต้องการประสบการณ์สนทนา
4. **ขยายง่าย** - server ไหนก็รับ request ได้หมด
5. **ค่าใช้จ่ายต่ำกว่า** - ไม่ต้องมี Infrastructure สำหรับ Memory

---

**หมายเหตุ:** นี่คือการเลือกออกแบบที่ตั้งใจไว้สำหรับ **Documentation Search** (ไม่ใช่ Chatbot ทั่วไป) เพื่อ:
- ประหยัดต้นทุน (Infrastructure น้อย)
- ความเรียบง่าย (ดูแลง่าย)
- ความเชื่อถือได้ (ปัญหาน้อย)
