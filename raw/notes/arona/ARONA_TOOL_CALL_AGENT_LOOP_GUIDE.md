# Arona RAG - Tool Call และ Agent Loop อธิบายเป็นภาษาไทย

## สารบัญ
1. [ภาพรวม Request Flow](#ภาพรวม-request-flow)
2. [Tool Call คืออะไร](#tool-call-คืออะไร)
3. [Agent Loop ทำงานยังไง](#agent-loop-ทำงานยังไง)
4. [Flex Mode คืออะไร](#flex-mode-คืออะไร)
5. [โค้ดตัวอย่างและตำแหน่ง](#โค้ดตัวอย่างและตำแหน่ง)

---

## ภาพรวม Request Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              USER REQUEST                                    │
│                    "How do I create a route in Elysia?"                     │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STEP 1: Security Layer                              │
│  ┌──────────────────┐           ┌──────────────────┐                        │
│  │  Proof of Work  │           │   Turnstile      │                        │
│  │  19 bits SHA256  │──────────▶│   Bot Detect     │                        │
│  └──────────────────┘           └──────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STEP 2: Cache Layer                                 │
│  Redis Cache ─────▶ Semantic Cache ─────▶ LRU Cache                         │
│       (No)                  (No)                  (No)                       │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STEP 3: AI with Tools                                │
│                    streamText() from AI SDK                                 │
│                                                                              │
│   Model จะตัดสินใจเองว่าจะเรียก Tool ไหน เมื่อไหร่                         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STEP 4: Agent Loop                                   │
│                   Loop จนกว่าจะได้คำตอบที่พอใจ                               │
│                                                                              │
│   Step 1: AI ตัดสินใจ → เรียก search("route")                              │
│   Step 2: search คืนผลลัพธ์ → AI อ่านแล้วยังไม่พอใจ                          │
│   Step 3: AI ตัดสินใจ → เรียก readPage("essential/route")                   │
│   Step 4: readPage คืนเนื้อหา → AI พอใจแล้ว                                 │
│   Step 5: AI สรุปและตอบกลับ                                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STEP 5: Stream Response                              │
│                     ส่งคำตอบทีละส่วน (Streaming)                            │
│                     แล้วต่อท้ายด้วย Sources                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Tool Call คืออะไร

**Tool Call** คือกลไกที่ให้ AI Model สามารถ "เรียกใช้ฟังก์ชัน" เพื่อดึงข้อมูลเพิ่มเติมระหว่างการตอบคำถาม

### เปรียบเปรย
เหมือนการให้ผู้เชี่ยวชาญมี "คู่มือ" อยู่ข้างๆ เวลาตอบคำถาม
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

### โค้ด Tool Definition

**ตำแหน่ง**: `src/modules/ai/libs/tool.ts`

```typescript
// Tool สำหรับค้นหา (Search Tool)
export const createSearchTool = (references: Reference[]) =>
    tool({
        description:
            "Search relevant information from Elysia documentation. " +
            "Content is only some part of a page, use 'read_page' tool to read full content. " +
            "This tool is deterministic, don't call with the same parameters twice",
        inputSchema: z.object({
            sentence: z.string().meta({
                description: 'The keyword/sentence to search',
                examples: ['handler', 'OpenAPI type gen', 'Eden Treaty']
            })
        }),
        outputSchema: Models.references,
        async execute({ sentence }) {
            log('Search:', sentence)

            // ค้นหาด้วย BM25 + Vector Search
            let documents = await retry(
                () => cache(`search: ${sentence}`, () => search(sentence)),
                3
            )

            if (!documents) return null

            // กรองเอกสารซ้ำ
            const refs = references.map((ref) => ref.link)
            const newDocuments = documents.filter(
                (document) => !refs.includes(document.link)
            )

            if (!newDocuments.length) return null

            // เพิ่มเข้าไปใน references
            references.push(...newDocuments)

            return newDocuments
        }
    })

// Tool สำหรับอ่านหน้า (Read Page Tool)
export const createPageTool = (references: Reference[]) =>
    tool({
        description: "Read a specific page from Elysia documentation with detail. " +
                     "This tool is deterministic, don't call with the same parameters twice",
        inputSchema: z.object({
            link: z.string().meta({
                description: 'The link of the page to read',
                examples: ['patterns/openapi', 'essential/life-cycle#transform']
            })
        }),
        async execute({ link }) {
            link = normalizePage(link)
            log('Read:', link)

            // ดึงข้อมูลจาก Database
            const documents = await retry(
                () => cache(`page:${link}`, () => readPage(link)),
                3
            )

            if (!documents) return null

            // เพิ่มเข้า references
            if (Array.isArray(documents)) references.push(...documents)
            else references.push(documents)

            return documents
        }
    })
```

---

## Agent Loop ทำงานยังไง

**Agent Loop** คือกระบวนการที่ AI Model "คิดและกระทำ" ซ้ำๆ จนกว่าจะได้คำตอบที่น่าพอใจ

### สิ่งที่ AI SDK จัดการให้อัตโนมัติ (อยู่ใน `streamText`)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AI SDK Agent Loop                                    │
│                          (streamText)                                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                    ┌─────────────────┴─────────────────┐
                    │         เริ่มต้น Loop              │
                    └─────────────────┬─────────────────┘
                                      │
                                      ▼
                    ┌─────────────────────────────────┐
                    │  Step 1: AI ตัดสินใจ               │
                    │  - อ่าน User Question             │
                    │  - อ่าน System Prompt              │
                    │  - ตัดสินใจว่าต้องการ Tool ไหม?  │
                    └─────────────────────────────────┘
                                      │
                        ┌─────────────┴─────────────┐
                        │  ต้องการ Tool ไหม?         │
                        └─────────────┬─────────────┘
                           Yes        │         No
                            │          │          │
                            ▼          │          ▼
              ┌─────────────────┐      │    ┌─────────────────┐
              │ เรียก Tool      │      │    │ ตอบเลย!        │
              │ (search/readPage)│     │    │ Stream Response │
              └─────────────────┘      │    └─────────────────┘
                        │               │              │
                        ▼               │              │
              ┌─────────────────┐      │              │
              │ Tool Execute    │      │              │
              │ - ค้นหา DB       │      │              │
              │ - คืนผลลัพธ์     │      │              │
              └─────────────────┘      │              │
                        │               │              │
                        ▼               │              │
              ┌─────────────────┐      │              │
              │ AI อ่านผลลัพธ์  │      │              │
              │ - พอใจไหม?      │      │              │
              │ - ต้องการข้อมูล│      │              │
              │   เพิ่มไหม?      │      │              │
              └─────────────────┘      │              │
                        │               │              │
            ┌───────────┴───────────┐   │              │
            │  ถึง Limit หรือยัง?    │   │              │
            │  - Max 8-12 steps     │   │              │
            │  - Max 32 references  │   │              │
            └───────────┬───────────┘   │              │
               No        │      Yes      │              │
                │        │       │        │              │
                ▼        │       ▼        ▼              ▼
          กลับไปทำซ้ำ  │   ┌─────────────────────────────┐
                      │   │         จบ Loop!               │
                      │   │    Stream Response + Sources   │
                      │   └─────────────────────────────────┘
                      │
                      └──► ไปยัง Step 1 อีกครั้ง
```

### โค้ดที่ควบคุม Agent Loop

**ตำแหน่ง**: `src/modules/ai/service.ts:46-133`

```typescript
export function ask({
    abortSignal,
    seed,
    message,
    history,
    references,
    ip,
    onFinish,
    think
}: AskParams) {
    // 1. สร้าง Tools
    const searchTool = createSearchTool(references)
    const readPageTool = createPageTool(references)
    const readHistoryTool = history?.length
        ? createHistoryTool(() => compressHistory(history).slice(3).slice(-5))
        : undefined

    // 2. เรียก streamText (AI SDK จะจัดการ Agent Loop)
    return record('Gather Resources', () =>
        streamText({
            model,
            abortSignal,

            // 👇 นี่คือจุดที่บอก AI ว่ามี Tools อะไรให้ใช้
            tools: Object.assign(
                {
                    readPage: readPageTool,
                    search: searchTool,
                    tableOfContents: tableOfContentsTool
                },
                (history?.length ?? 0) > 3
                    ? { readHistory: readHistoryTool! }
                    : {}
            ) as any,

            // 👇 นี่คือเงื่อนไขที่จะหยุด Loop
            stopWhen: [
                stepCountIs(think ? 12 : 8),  // สูงสุด 8 หรือ 12 steps
                () => references.length > 32   // หรือ references เกิน 32
            ],

            seed,
            topP: 0.75,
            presencePenalty: 0.4,
            maxOutputTokens: 2560,
            maxRetries: 3,  // 👈 นี่คือ Flex mode!

            system: instruction,  // 👈 คำสั่งให้ Elysia chan

            messages: [
                {
                    role: 'user',
                    content: history?.length
                        ? message
                        : `Hi Elysia chan! Would you kindly help me? ${message}`
                },
                ...(initialReference ? [initialReference] : []),
                ...(history?.length ? compressHistory(history).slice(0, 3) : [])
            ],

            providerOptions: {
                openrouter: {
                    user: ip,
                    reasoning: {
                        effort: think ? 'medium' : 'low'
                    }
                }
            },

            onFinish(metadata) {
                onFinish?.(metadata as any)
            }
        })
    ).textStream  // 👈 ส่งกลับเป็น stream
}
```

### อธิบาย stopWhen

```typescript
stopWhen: [
    stepCountIs(think ? 12 : 8),  // ถ้า think mode = 12 steps, ปกติ = 8 steps
    () => references.length > 32   // หรือ references เกิน 32 ก็หยุด
]
```

**หมายความว่า**:
- ปกติ AI จะเรียก Tool ได้สูงสุด **8 ครั้ง**
- ถ้าเปิด think mode (โค้ดยาวๆ หรือมี ```) จะเรียกได้ **12 ครั้ง**
- ถ้าเก็บ references ครบ **32 รายการ** จะหยุดทันที

### ตัวอย่าง Agent Loop จริง

```
User: "How do I create a route with validation?"

─────────────────────────────────────────────────────────────────
Step 1: AI คิด → ต้องการข้อมูลเกี่ยวกับ route
─────────────────────────────────────────────────────────────────
AI Decision: เรียก search("route validation")

Tool Output:
  - essential/route (score: 0.95)
  - essential/validation (score: 0.88)

─────────────────────────────────────────────────────────────────
Step 2: AI อ่านผลลัพธ์ → ต้องการข้อมูลเพิ่ม
─────────────────────────────────────────────────────────────────
AI Decision: เรียก readPage("essential/route")

Tool Output:
  Full content of essential/route page with examples

─────────────────────────────────────────────────────────────────
Step 3: AI อ่านเนื้อหา → ยังไม่พอใจ ต้องการ validation
─────────────────────────────────────────────────────────────────
AI Decision: เรียก readPage("essential/validation")

Tool Output:
  Full content of essential/validation page

─────────────────────────────────────────────────────────────────
Step 4: AI อ่านเนื้อหา → พอใจแล้ว!
─────────────────────────────────────────────────────────────────
AI Decision: ตอบคำถามพร้อมตัวอย่างโค้ด

Response:
  "To create a route with validation in Elysia..."
  [Examples...]
  [Sources]
```

---

## Flex Mode คืออะไร

**Flex Mode** ไม่ใช่ชื่อ function ในโค้ด แต่เป็น **แนวคิด** ในการรับมือกับความผิดพลาด

### แนวคิด Flex Mode

> "ยอมรับว่า AI อาจจะผิดพลาด แต่จะ retry ด้วย exponential backoff จนกว่าจะสำเร็จ"

### ตำแหน่งในโค้ด

**ตำแหน่ง**: `src/modules/ai/service.ts:93` และ `src/libs/retry.ts`

```typescript
// ใน streamText
maxRetries: 3,  // 👈 นี่คือ Flex mode!
```

```typescript
// ใน src/libs/retry.ts
export const retry = <T>(
    fn: () => MaybePromise<T>,
    retries = 3,           // 👈 ลอง 3 ครั้ง
    delay: number | ((n: number) => number) = 1000  // 👈 เริ่มที่ 1 วินาที
) =>
    new Promise<Awaited<T>>((resolve, reject) => {
        async function attempt(n: number) {
            try {
                let temp = fn()
                if (temp instanceof Promise) temp = await temp
                resolve(temp as Awaited<T>)
            } catch (err) {
                // 👇 ถ้ายังไม่หมด retries ให้ retry
                if (n - 1 > 0)
                    setTimeout(
                        () => attempt(n - 1),
                        typeof delay === 'function' ? delay(n - 1) : n
                    )
                else
                    reject(err)  // 👈 หมด retries แล้วจึง reject
            }
        }
        attempt(retries)
    })
```

### ตัวอย่างการใช้งาน

```typescript
// ใน src/libs/turnstile.ts - retry Turnstile verification
const data = await retry(() =>
    fetch('https://challenges.cloudflare.com/turnstile/v0/siteverify', {
        method: 'POST',
        body: formData
    }).then(r => r.json())
)  // 👈 ถ้า Turnstile ล้มเหลว จะ retry 3 ครั้ง

// ใน src/modules/ai/libs/tool.ts - retry search
let documents = await retry(
    () => cache(`search: ${sentence}`, () => search(sentence)),
    3
)  // 👈 ถ้า search ล้มเหลว จะ retry 3 ครั้ง

// ใน src/modules/ai/libs/semantic-cache.ts - retry normalization
let { text } = await retry(
    () => generateText({
        model: smallModel,
        system: normalizePromptInstruction,
        prompt,
        temperature: 0,
        maxOutputTokens: 192
    }),
    3,           // 👈 3 retries
    500          // 👈 exponential: 500ms, 1000ms, 2000ms
)
```

### Exponential Backoff Formula

จาก `src/libs/structure.ts:334`:
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

### Flex Mode ทำให้ระบบ Resilient ขึ้น

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ไม่มี Flex Mode                                    │
│                                                                             │
│  Request → AI API Call → ERROR 💥 → Fail → Return Error to User             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         มี Flex Mode                                        │
│                                                                             │
│  Request → AI API Call → ERROR 💥 → Retry 1 (1s)                             │
│                                  → Retry 2 (2s)                             │
│                                  → Retry 3 (3s)                             │
│                                  → Success! ✅                              │
│                                                                             │
│  User ไม่รู้เลยว่าเกิด Error ขณะหลัง                                                │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## โค้ดตัวอย่างและตำแหน่ง

### Summary Table

| ส่วนประกอบ | ตำแหน่งไฟล์ | บรรทัด | คำอธิบาย |
|-------------|---------------|--------|----------|
| **Tool Definition** | `src/modules/ai/libs/tool.ts` | 10-86 | สร้าง tools: search, readPage, tableOfContents, readHistory |
| **Agent Loop Main** | `src/modules/ai/service.ts` | 46-133 | `ask()` function ที่เรียก `streamText` |
| **Tools Parameter** | `src/modules/ai/service.ts` | 75-84 | ส่ง tools เข้าไปให้ AI |
| **Stop Condition** | `src/modules/ai/service.ts` | 85-88 | `stopWhen` กำหนด max steps |
| **Flex Mode (maxRetries)** | `src/modules/ai/service.ts` | 93 | `maxRetries: 3` |
| **Retry Function** | `src/libs/retry.ts` | 3-26 | retry logic พื้นฐาน |
| **Search Implementation** | `src/modules/ai/service.ts` | 153-251 | BM25 + Vector search |
| **ReadPage Implementation** | `src/modules/ai/service.ts` | 136-151 | Database query |

### Flow Chart แบบละเอียด

```
USER: "How do I create a route?"
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  POST /model/ai/ask                                                          │
│  - PoW verification                                                          │
│  - Turnstile verification                                                    │
│  - Rate limit check                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  src/modules/ai/index.ts                                                    │
│  - Check cache (Redis, Semantic, LRU)                                       │
│  - If cache miss → call AI.ask()                                            │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  src/modules/ai/service.ts → ask()                                         │
│  - Create tools (search, readPage, tableOfContents, readHistory)            │
│  - Call streamText() from AI SDK                                           │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│  AI SDK Agent Loop (streamText)                                            │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │ Step 1: AI Model ตัดสินใจ                                           │        │
│  │   - อ่าน user message: "How do I create a route?"                   │        │
│  │   - อ่าน system prompt: "You are Elysia chan..."                     │        │
│  │   - Decision: ไม่พอข้อมูล → เรียก search("route")                   │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │ Step 2: Tool Execute                                              │        │
│  │   - Function: createSearchTool().execute({ sentence: "route" })   │        │
│  │   - Location: src/modules/ai/libs/tool.ts:21-41                    │        │
│  │   - Action: cache() → search() → BM25 + Vector                    │        │
│  │   - Return: [{ title: "essential/route", link: "...", ... }]      │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │ Step 3: AI อ่านผลลัพธ์                                                 │        │
│  │   - AI Model ได้รับ search results                                   │        │
│  │   - Decision: เนื้อหาไม่เพองาน → เรียก readPage("essential/route")    │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │ Step 4: Tool Execute Again                                        │        │
│  │   - Function: createPageTool().execute({ link: "essential/route" })│        │
│  │   - Location: src/modules/ai/libs/tool.ts:43-69                    │        │
│  │   - Action: cache() → readPage() → SQL query                      │        │
│  │   - Return: { title, summary, link, content }                      │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │ Step 5: AI อ่านผลลัพธ์                                                  │        │
│  │   - AI Model ได้รับ full page content                               │        │
│  │   - Decision: พอใจแล้ว! → ตอบคำถาม                                 │        │
│  │   - Check stopWhen: stepCount < 8 ✅, references < 32 ✅             │        │
│  └─────────────────────────────────────────────────────────────────┘        │
│                              │                                               │
│                              ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────┐        │
│  │ Step 6: Stream Response                                           │        │
│  │   - AI สร้างคำตอบจากข้อมูลที่ได้                                       │        │
│  │   - Stream ทีละ chunk ไปยัง client                                   │        │
│  │   - Append sources ท้ายสุด                                          │        │
│  └─────────────────────────────────────────────────────────────────┘        │
└─────────────────────────────────────────────────────────────────────────────┘
   │
   ▼
Cache Response (Async, non-blocking)
   │
   ▼
USER: "To create a route in Elysia..."
```

---

## สรุป

| คำถาม | คำตอบ |
|--------|--------|
| **Tool Call อยู่ตรงไหน** | ใน `streamText()` จาก AI SDK, parameter `tools` ที่ `src/modules/ai/service.ts:75-84` |
| **Tool ทำงานยังไง** | AI ตัดสินใจเรียก Tool → Tool Execute → คืนผลลัพธ์ → AI อ่านแล้วตัดสินใจต่อ |
| **Loop จนพอใจคืออะไร** | Agent Loop ของ AI SDK ที่เรียก Tool ซ้ำๆ จนกว่าจะได้คำตอบ |
| **Stop Condition อะไร** | `stopWhen: [stepCountIs(8), () => references > 32]` |
| **Flex Mode คืออะไร** | แนวคิด retry ด้วย exponential backoff (`maxRetries: 3` + `retry()` function) |
| **Flex Mode อยู่ตรงไหน** | `maxRetries: 3` ใน `streamText` และ `retry()` ในทุก external API call |

---

*สร้างจากการวิเคราะห์ Arona codebase*
*ตำแหน่งโค้ด: `src/modules/ai/`*
