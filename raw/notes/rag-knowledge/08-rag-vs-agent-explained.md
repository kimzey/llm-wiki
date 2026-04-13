# RAG vs Agent — มันคืออะไร ต่างกันยังไง เชื่อมกันยังไง

---

## 1. ตอบคำถามตรงๆ ก่อน

### "RAG เป็น Agent ตัวนึงมั้ย?"

```
คำตอบสั้น: ไม่ — RAG ไม่ใช่ Agent

RAG = เทคนิค (technique) ในการดึงข้อมูลมาช่วย LLM ตอบ
Agent = ระบบ AI ที่ตัดสินใจเอง เลือกว่าจะทำอะไร

RAG เป็นแค่ "ท่อน้ำ" ที่ต่อ database เข้ากับ LLM
Agent เป็น "คนที่คิดเอง" ว่าจะเปิดก๊อกน้ำไหน เมื่อไหร่


เปรียบเทียบ:
  RAG   = สูตรอาหาร (ขั้นตอน 1-2-3 ตายตัว)
  Agent = พ่อครัว (ตัดสินใจเองว่าจะทำเมนูอะไร ใช้วัตถุดิบอะไร)

  สูตรอาหาร ≠ พ่อครัว
  แต่พ่อครัวอาจ "ใช้" สูตรอาหารเป็นส่วนหนึ่งของการทำงาน
  
  เช่นเดียวกัน:
  RAG ≠ Agent
  แต่ Agent อาจ "ใช้" RAG เป็นเครื่องมือหนึ่ง
```

### "AI + Vector Database ถือว่าเป็น RAG หรอ? หรือ Agent RAG?"

```
ขึ้นกับว่า "ทำงานยังไง":

ถ้า flow เป็น:
  คำถาม → search vector DB → ส่ง context ให้ LLM → ตอบ
  = RAG ✅ (เป็น RAG ไม่ใช่ Agent)

ถ้า flow เป็น:
  คำถาม → LLM ตัดสินใจ: "ต้อง search อะไร?" 
  → search vector DB → LLM ดูผล: "พอมั้ย?"
  → ถ้าไม่พอ → search อีก → ตอบ
  = Agentic RAG ✅ (เป็นทั้ง Agent และใช้ RAG)

สรุป:
  AI + Vector DB + fixed pipeline  = RAG
  AI + Vector DB + ตัดสินใจเอง     = Agentic RAG (Agent ที่ใช้ RAG)
```

### "ทำ RAG เสร็จถือว่าเป็น Agent มั้ย?"

```
ไม่ครับ — ทำ RAG เสร็จ ได้แค่ RAG

RAG ที่ทำเสร็จ:
  = ระบบที่ search + LLM ตอบ ตาม flow ตายตัว
  = ไม่มีการ "ตัดสินใจ"
  = ไม่ใช่ Agent

ถ้าอยากให้เป็น Agent ต้องเพิ่ม:
  1. ให้ LLM เลือกว่าจะ search หรือไม่ (decision making)
  2. ให้ LLM เลือกว่า search อะไร (tool selection)
  3. ให้ LLM ตรวจผลลัพธ์ว่าพอมั้ย (self-evaluation)
  4. ให้ LLM ทำซ้ำได้ถ้าไม่พอ (loop)
```

---

## 2. Spectrum ทั้งหมด — จาก LLM เปล่าๆ จนถึง Full Agent

```
ลองนึกว่ามัน "ระดับ" ไม่ใช่ "แยกขาด"

Level 0          Level 1          Level 2          Level 3          Level 4
LLM เปล่า ──────▶ RAG ───────────▶ Advanced RAG ──▶ Agentic RAG ──▶ Full Agent
                                                     
ไม่มี DB          มี DB             มี DB +          มี DB +          มี DB +
ไม่มี search      search ตายตัว     search ดีขึ้น     ตัดสินใจเอง       tools เยอะ
ตอบจากความรู้เดิม  flow ตายตัว       flow ตายตัว       เลือก tool        วางแผน
                                                     loop ได้          multi-step
```

### Level 0: LLM เปล่า (ไม่ใช่ RAG ไม่ใช่ Agent)

```python
# ไม่มี database ไม่มี search
# LLM ตอบจากความรู้ที่ train มาเท่านั้น

response = llm.chat("ลาพักร้อนกี่วัน?")
# → "ตามกฎหมายแรงงานไทย ลาพักร้อนได้ 6 วัน/ปี"
#    ← ตอบได้แค่ความรู้ทั่วไป ไม่รู้นโยบาย Sellsuki

สิ่งที่มี:
  ✅ LLM
  ❌ Vector Database
  ❌ Search
  ❌ การตัดสินใจ
  ❌ Tools
  ❌ Memory
```

### Level 1: RAG พื้นฐาน (เป็น RAG ไม่ใช่ Agent)

```python
# มี database + search + LLM
# แต่ flow ตายตัว: search → stuff → LLM

def basic_rag(question: str) -> str:
    # Step 1: ALWAYS search (ไม่ว่าคำถามอะไร)
    query_vector = embed(question)
    chunks = vector_db.search(query_vector, top_k=5)
    
    # Step 2: ALWAYS stuff ทุก chunk เข้า prompt
    context = "\n".join([c.content for c in chunks])
    
    # Step 3: ALWAYS ส่งให้ LLM
    answer = llm.chat(f"Context: {context}\nQuestion: {question}")
    
    return answer

# ใช้งาน:
basic_rag("ลาพักร้อนกี่วัน?")
# → search → ได้ chunks เรื่องลา → LLM ตอบ "10 วัน" ✅

basic_rag("สวัสดี")
# → search → ได้ chunks ที่ไม่เกี่ยว → LLM ตอบแปลกๆ ❌
#   (search ทุกครั้งแม้คำถามแค่ "สวัสดี" เพราะ flow ตายตัว)

basic_rag("สรุปนโยบายการลาทั้งหมด")
# → search ด้วยคำถามเดิม → ได้แค่ 5 chunks → อาจไม่ครบ ❌
#   (ไม่รู้ว่าต้อง search หลายรอบ)


สิ่งที่มี:
  ✅ LLM
  ✅ Vector Database
  ✅ Search (ตายตัว)
  ❌ การตัดสินใจ ← ทำเหมือนกันทุกครั้ง
  ❌ Tools (มีแค่ search เดียว)
  ❌ Loop
  ❌ Memory
```

### Level 2: Advanced RAG (ยังเป็น RAG ไม่ใช่ Agent)

```python
# ปรับปรุง search ให้ดีขึ้น แต่ยังเป็น fixed pipeline

def advanced_rag(question: str) -> str:
    # Step 1: Query rewriting (ALWAYS ทำ)
    rewritten = llm.rewrite_query(question)
    
    # Step 2: Hybrid search (ALWAYS ทำ BM25 + vector)
    chunks = hybrid_search(rewritten, top_k=5)
    
    # Step 3: Re-rank (ALWAYS ทำ)
    reranked = reranker.rank(question, chunks)[:3]
    
    # Step 4: Compress context (ALWAYS ทำ)
    compressed = llm.compress(question, reranked)
    
    # Step 5: Generate (ALWAYS ทำ)
    answer = llm.chat(compressed + question)
    
    return answer

# ดีขึ้น:
#   search แม่นขึ้น (hybrid + rerank)
#   context กระชับขึ้น (compress)
#
# แต่ยังเป็น fixed pipeline:
#   ทุกคำถามผ่านขั้นตอนเดียวกันหมด
#   ไม่ตัดสินใจว่า "คำถามนี้ง่ายไม่ต้อง rerank" หรือ "คำถามนี้ต้อง search 3 รอบ"


สิ่งที่มี:
  ✅ LLM
  ✅ Vector Database
  ✅ Search (ดีขึ้น — hybrid, rerank)
  ✅ Query rewriting
  ❌ การตัดสินใจ ← ยัง fixed pipeline
  ❌ Loop
  ❌ Multiple tools
```

### Level 3: Agentic RAG (เป็นทั้ง RAG และ Agent)

```python
# LLM ตัดสินใจเอง → "Agent ที่ใช้ RAG เป็นเครื่องมือ"

def agentic_rag(question: str) -> str:
    messages = [
        {"role": "system", "content": "คุณเป็น Suki Bot มี tools ค้นหาข้อมูล"},
        {"role": "user", "content": question},
    ]
    
    for i in range(5):  # loop ได้สูงสุด 5 รอบ
        response = llm.chat(messages, tools=[
            search_hr_tool,
            search_product_tool,
            search_general_tool,
        ])
        
        # ────── จุดสำคัญ: LLM ตัดสินใจเอง ──────
        
        if response.has_tool_calls:
            # LLM เลือกว่าจะใช้ tool ไหน ด้วย query อะไร
            for tool_call in response.tool_calls:
                result = execute_tool(tool_call)
                messages.append(tool_result(result))
            # loop กลับไปให้ LLM ดูผลลัพธ์
        else:
            # LLM ตัดสินใจว่า "พอแล้ว ตอบได้"
            return response.content
    
    return "ขอโทษ หาข้อมูลไม่ได้"


# ตัวอย่างที่ 1: คำถามง่าย
agentic_rag("สวัสดี")
# LLM คิด: "คำถามนี้แค่ทักทาย ไม่ต้อง search"
# → ตอบเลย: "สวัสดีครับ! มีอะไรให้ช่วยมั้ยครับ?" ✅
# → ไม่ search เลย (ประหยัด!)

# ตัวอย่างที่ 2: คำถามเฉพาะเจาะจง
agentic_rag("ลาพักร้อนกี่วัน?")
# LLM คิด: "ต้อง search เรื่อง HR การลา"
# → tool_call: search_hr("ลาพักร้อน สิทธิ์การลา")
# → ได้ผลลัพธ์ → LLM: "ข้อมูลพอแล้ว"
# → ตอบ: "10 วัน/ปี ต้องแจ้ง 3 วัน" ✅

# ตัวอย่างที่ 3: คำถามซับซ้อน
agentic_rag("เปรียบเทียบนโยบายลาทุกประเภท")
# Iteration 1: LLM → search_hr("ลาพักร้อน")     → ได้ข้อมูล
# Iteration 2: LLM → search_hr("ลาป่วย")         → ได้ข้อมูล
# Iteration 3: LLM → search_hr("ลากิจ ลาคลอด")   → ได้ข้อมูล
# Iteration 4: LLM → "ข้อมูลครบแล้ว" → ตอบเปรียบเทียบทุกประเภท ✅
# → search 3 รอบ เอง ตัดสินใจเอง!


สิ่งที่มี:
  ✅ LLM
  ✅ Vector Database
  ✅ Search
  ✅ การตัดสินใจ ← LLM เลือกเอง
  ✅ Multiple tools ← เลือก tool ได้
  ✅ Loop ← ทำซ้ำได้
  ❌ วางแผนล่วงหน้า
  ❌ Multi-agent
```

### Level 4: Full Agent (Agent เต็มรูปแบบ ที่อาจใช้ RAG ด้วย)

```python
# Agent ที่ทำได้หลายอย่าง RAG เป็นแค่ 1 ใน tools

def full_agent(question: str) -> str:
    response = llm.chat(
        messages=[{"role": "user", "content": question}],
        tools=[
            # Knowledge tools (RAG)
            search_hr_tool,
            search_product_tool,
            
            # Action tools (ไม่ใช่ RAG)
            send_email_tool,         # ส่ง email
            create_ticket_tool,      # สร้าง ticket
            book_meeting_tool,       # จอง meeting
            calculate_leave_tool,    # คำนวณวันลา
            lookup_employee_tool,    # ค้นหาข้อมูลพนักงาน
            
            # External tools
            web_search_tool,         # search internet
            calendar_tool,           # ดูปฏิทิน
        ]
    )
    # LLM เลือกเองว่าจะใช้ tool ไหน กี่ตัว กี่รอบ


# ตัวอย่าง: "ลาพักร้อนกี่วัน?"
# → ใช้ search_hr (RAG) → ตอบ

# ตัวอย่าง: "ช่วยส่ง email ถึง HR เรื่องขอลา"
# → ใช้ send_email_tool → ไม่ใช้ RAG เลย

# ตัวอย่าง: "คำนวณวันลาเหลือของผม แล้วจอง meeting กับหัวหน้า"
# → ใช้ calculate_leave_tool → ดูว่าเหลือกี่วัน
# → ใช้ lookup_employee_tool → หาว่าหัวหน้าคือใคร
# → ใช้ calendar_tool → เช็คว่าหัวหน้าว่างเมื่อไหร่
# → ใช้ book_meeting_tool → จอง meeting
# → ไม่ได้ใช้ RAG เลย!


สิ่งที่มี:
  ✅ LLM
  ✅ Vector Database (อาจมีหรือไม่มี)
  ✅ Search (อาจใช้หรือไม่ใช้)
  ✅ การตัดสินใจ
  ✅ Multiple tools (เยอะมาก)
  ✅ Loop
  ✅ วางแผน
  ✅ Memory
  ✅ ลงมือทำ (send email, book meeting)
```

---

## 3. เปรียบเทียบแบบชัดเจน

### ตาราง

```
┌──────────────┬───────────┬─────────────┬─────────────┬──────────────┐
│ คุณสมบัติ     │ LLM เปล่า │ RAG          │ Agentic RAG │ Full Agent   │
├──────────────┼───────────┼─────────────┼─────────────┼──────────────┤
│ มี LLM       │ ✅        │ ✅           │ ✅           │ ✅           │
│ มี Database  │ ❌        │ ✅           │ ✅           │ ✅ (optional)│
│ Search       │ ❌        │ ✅ ตายตัว    │ ✅ เลือกเอง  │ ✅ เลือกเอง  │
│ ตัดสินใจ     │ ❌        │ ❌           │ ✅           │ ✅           │
│ Loop         │ ❌        │ ❌           │ ✅           │ ✅           │
│ หลาย Tools   │ ❌        │ ❌ (มีแค่    │ ✅ (search   │ ✅ (search + │
│              │           │  search 1)   │  หลายแบบ)   │  actions)    │
│ วางแผน       │ ❌        │ ❌           │ บางส่วน     │ ✅           │
│ ลงมือทำ      │ ❌        │ ❌           │ ❌           │ ✅ (email,   │
│ (Actions)    │           │              │              │  book, etc.) │
│ Memory       │ ❌        │ ❌/บางส่วน   │ ✅           │ ✅           │
├──────────────┼───────────┼─────────────┼─────────────┼──────────────┤
│ ถามตอบ       │ ❌ เดา    │ ✅ จาก data  │ ✅ จาก data  │ ✅ จาก data  │
│ ส่ง email    │ ❌        │ ❌           │ ❌           │ ✅           │
│ จอง meeting  │ ❌        │ ❌           │ ❌           │ ✅           │
│ สร้าง report │ ❌        │ ❌           │ ❌/บางส่วน  │ ✅           │
├──────────────┼───────────┼─────────────┼─────────────┼──────────────┤
│ ตัวอย่าง      │ ChatGPT   │ Docs chatbot│ Perplexity  │ Devin, Manus │
│              │ เปล่าๆ    │ FAQ bot     │ Suki Bot    │ Claude Agent │
└──────────────┴───────────┴─────────────┴─────────────┴──────────────┘
```

### อุปมาให้เข้าใจง่าย

```
LLM เปล่า = คนฉลาดที่ถูกปิดตา
  ฉลาดมาก แต่ไม่เห็นข้อมูลบริษัท
  ถามอะไรก็เดาจากความรู้ทั่วไป
  "ลาพักร้อนกี่วัน?" → "ตามกฎหมาย 6 วัน" (ไม่รู้นโยบาย Sellsuki)

RAG = คนฉลาดที่มีหนังสือวางข้างๆ
  ถาม → เปิดหนังสือหน้าที่เกี่ยวข้อง → อ่าน → ตอบ
  ทำเหมือนกันทุกครั้ง: เปิดหนังสือ → อ่าน → ตอบ
  "ลาพักร้อนกี่วัน?" → เปิดคู่มือ HR → "10 วัน ตามนโยบาย Sellsuki" ✅
  "สวัสดี" → เปิดหนังสือ → หาไม่เจอ → ตอบแปลกๆ ❌

Agentic RAG = คนฉลาดที่มีห้องสมุด
  ถาม → คิดก่อนว่า "ต้องหาอะไร?"
  → เลือกหนังสือที่เหมาะ → อ่าน → ยังไม่พอ? → เลือกเล่มอื่นอีก → ตอบ
  "สวัสดี" → "ไม่ต้องเปิดหนังสือ สวัสดีครับ!" ✅
  "เปรียบเทียบการลาทุกประเภท" → เปิดหนังสือ 3 เล่ม → สรุป → ตอบ ✅

Full Agent = พนักงานเต็มตัว
  ถาม → คิด → ค้นข้อมูล → ส่ง email → จอง meeting → ทำรายงาน
  "ช่วยลาพักร้อนให้ด้วย"
  → ค้นนโยบาย → คำนวณวันลาเหลือ → กรอกฟอร์ม → ส่งหัวหน้าอนุมัติ ✅
  ← ลงมือทำให้จริงๆ!
```

---

## 4. ความสัมพันธ์ระหว่าง RAG กับ Agent

```
RAG และ Agent ไม่ได้ตรงข้ามกัน — มันอยู่คนละแกน:

RAG = เทคนิคการดึงข้อมูล (Retrieval technique)
Agent = สถาปัตยกรรมการทำงาน (Architecture)


แกน X: เทคนิคการดึงข้อมูล
  ไม่มี ──────── RAG ──────── Advanced RAG

แกน Y: ระดับ Agency (ตัดสินใจเอง)
  Fixed pipeline ──────── Agentic ──────── Full Agent


              Agent (ตัดสินใจเอง)
                ▲
                │
                │   ┌─────────────┐    ┌─────────────┐
                │   │ Agent       │    │ Agent       │
                │   │ (ไม่มี RAG) │    │ + RAG       │
                │   │             │    │ (Agentic    │
                │   │ เช่น coding │    │  RAG)       │
                │   │ agent       │    │             │
                │   └─────────────┘    └─────────────┘
                │
                │   ┌─────────────┐    ┌─────────────┐
                │   │ LLM เปล่า   │    │ RAG         │
                │   │             │    │ (ไม่มี      │
                │   │             │    │  agency)    │
                │   └─────────────┘    └─────────────┘
                │
                └──────────────────────────────────────▶
                  ไม่มี knowledge        มี knowledge (RAG)


สรุป:
  RAG ที่ไม่มี agency         = RAG ธรรมดา (Level 1-2)
  RAG ที่มี agency            = Agentic RAG (Level 3)
  Agent ที่ไม่ใช้ RAG          = Agent (เช่น coding agent, math agent)
  Agent ที่ใช้ RAG ด้วย       = Agent + RAG (Level 4)

ดังนั้น:
  ❌ "RAG เป็น Agent" → ไม่จริง
  ❌ "Agent เป็น RAG" → ไม่จริง
  ✅ "RAG อาจเป็นส่วนหนึ่งของ Agent" → จริง
  ✅ "Agent อาจใช้ RAG เป็น tool" → จริง
```

---

## 5. Flow จริงเทียบกัน

### RAG Flow (ไม่มี agency)

```
User: "ลาพักร้อนกี่วัน?"
│
▼ ── ขั้นตอนที่ 1 ตายตัว ──
[Embed คำถาม]
│
▼ ── ขั้นตอนที่ 2 ตายตัว ──
[Search vector DB → top 5 chunks]       ← ALWAYS search
│
▼ ── ขั้นตอนที่ 3 ตายตัว ──
[Stuff chunks เข้า prompt]              ← ALWAYS stuff ทั้ง 5 chunks
│
▼ ── ขั้นตอนที่ 4 ตายตัว ──
[LLM Generate answer]                   ← ALWAYS ส่งให้ LLM
│
▼
"ลาพักร้อนได้ 10 วัน/ปี"

ไม่มีจุดไหนที่ "ตัดสินใจ" เลย
ทำเหมือนกันทุกครั้ง ไม่ว่าคำถามจะเป็นอะไร

ข้อดี:  เร็ว, คาดเดาได้, debug ง่าย
ข้อเสีย: ไม่ยืดหยุ่น, เสียเงิน search ทุก request (แม้แค่ "สวัสดี")
```

### Agentic RAG Flow (มี agency)

```
User: "เปรียบเทียบนโยบายลาทุกประเภท"
│
▼
[LLM THINK] ← จุดตัดสินใจ #1
"คำถามนี้กว้าง ต้อง search หลายรอบ
 เริ่มจากลาพักร้อนก่อน"
│
▼ ── ตัดสินใจ: search_hr("ลาพักร้อน") ──
[Search] → ได้ข้อมูลลาพักร้อน
│
▼
[LLM THINK] ← จุดตัดสินใจ #2
"ได้ลาพักร้อนแล้ว ยังไม่ครบ ต้องหาลาป่วย"
│
▼ ── ตัดสินใจ: search_hr("ลาป่วย") ──
[Search] → ได้ข้อมูลลาป่วย
│
▼
[LLM THINK] ← จุดตัดสินใจ #3
"ยังขาดลากิจ ลาคลอด ลาบวช"
│
▼ ── ตัดสินใจ: search_hr("ลากิจ ลาคลอด ลาบวช") ──
[Search] → ได้ข้อมูล
│
▼
[LLM THINK] ← จุดตัดสินใจ #4
"ข้อมูลครบแล้ว สรุปได้เลย"
│
▼ ── ตัดสินใจ: ไม่ search แล้ว ตอบเลย ──
[LLM Generate] → ตารางเปรียบเทียบครบทุกประเภท ✅

มีจุดตัดสินใจ 4 จุด:
  #1: ตัดสินใจว่าต้อง search + เลือกคำค้นเอง
  #2: ตัดสินใจว่ายังไม่ครบ + search เพิ่ม
  #3: ตัดสินใจว่ายังไม่ครบ + search เพิ่ม
  #4: ตัดสินใจว่าครบแล้ว + ตอบ

นี่คือ "agency" — ความสามารถในการตัดสินใจ
```

### Full Agent Flow

```
User: "ช่วยขอลาพักร้อน 3 วัน วันที่ 15-17 ม.ค. ให้ด้วย"
│
▼
[LLM PLAN] ← วางแผนก่อนทำ
"ต้องทำ 5 ขั้นตอน:
 1. ค้นนโยบายลาพักร้อน (RAG)
 2. ค้นหาข้อมูลพนักงาน
 3. คำนวณวันลาคงเหลือ
 4. ตรวจสอบว่าตรงตามนโยบายมั้ย
 5. สร้างใบลาแล้วส่งหัวหน้า"
│
▼ ── Step 1: RAG ──
[search_hr("นโยบายลาพักร้อน")]
→ "ลาได้ 10 วัน/ปี ต้องแจ้ง 3 วัน"
│
▼ ── Step 2: Database lookup ──
[lookup_employee("current_user")]
→ "สมชาย แผนก Engineering หัวหน้า: คุณวิชัย"
│
▼ ── Step 3: Calculation ──
[calculate_leave("current_user", "vacation")]
→ "ใช้ไปแล้ว 3 วัน เหลือ 7 วัน"
│
▼ ── Step 4: Policy check ──  ← ตัดสินใจ
[LLM THINK] "ขอลา 3 วัน เหลือ 7 วัน ✅ OK
             แจ้งล่วงหน้า... วันนี้ 10 ม.ค. ลา 15 ม.ค. = 5 วัน ✅ OK"
│
▼ ── Step 5: Action ──
[create_leave_request({
    employee: "สมชาย",
    type: "vacation",
    dates: "15-17 Jan",
    approver: "คุณวิชัย"
})]
[send_notification("คุณวิชัย", "สมชายขอลาพักร้อน 15-17 ม.ค.")]
│
▼
"เรียบร้อยครับ! ส่งใบลาพักร้อน 3 วัน (15-17 ม.ค.) 
 ให้คุณวิชัยอนุมัติแล้ว วันลาคงเหลือ: 4 วัน"

← Agent ทำทุกอย่างเอง: ค้นข้อมูล + คำนวณ + ตรวจสอบ + ลงมือทำ
← RAG เป็นแค่ Step 1 จาก 5 steps
```

---

## 6. สำหรับ Sellsuki — ทำแค่ไหนดี

```
คำถาม: ควรทำ "RAG" หรือ "Agent"?

ตอบ: เริ่มจาก RAG แล้วค่อยขยับเป็น Agentic RAG

เหตุผล:

Phase 1: RAG (สัปดาห์ 1-4)
  ✅ ตอบคำถามเกี่ยวกับบริษัทได้
  ✅ ง่าย debug ง่าย
  ✅ Cost ต่ำ (fixed pipeline ไม่มี loop)
  ✅ พอสำหรับ 80% ของ use cases
  
  ตัวอย่าง:
    "ลาพักร้อนกี่วัน?" → search → ตอบ ✅
    "WiFi password อะไร?" → search → ตอบ ✅

Phase 2: Agentic RAG (สัปดาห์ 5-6)
  เพิ่ม agency:
    - ให้ LLM เลือกว่าจะ search หรือไม่
    - ให้เลือก category (HR, Product, IT)
    - ให้ loop ได้ถ้าข้อมูลไม่พอ
    
  ตัวอย่าง:
    "สวัสดี" → ไม่ search ตอบเลย ✅
    "เปรียบเทียบการลาทุกประเภท" → search 3 รอบ → ตอบครบ ✅

Phase 3: Full Agent (อนาคต)
  เพิ่ม action tools:
    - ส่ง email
    - สร้าง ticket
    - จอง meeting
    - คำนวณ
    
  ตัวอย่าง:
    "ช่วยลาพักร้อนให้ด้วย" → ค้น + คำนวณ + สร้างใบลา + แจ้งหัวหน้า ✅

  
ไม่ต้องกระโดดไป Full Agent ทันที
เริ่มจาก RAG → ใช้งานได้แล้ว → ค่อยเพิ่ม agency
```

### สรุปสุดท้าย

```
                    ┌─────────────────────────────┐
                    │                             │
                    │   RAG ≠ Agent               │
                    │   RAG = เทคนิคการดึงข้อมูล    │
                    │   Agent = ระบบที่คิดเอง       │
                    │                             │
                    │   RAG อาจเป็นส่วนหนึ่ง       │
                    │   ของ Agent ได้              │
                    │                             │
                    │   ทำ RAG เสร็จ ≠ ได้ Agent   │
                    │   ทำ RAG เสร็จ = ได้ RAG     │
                    │                             │
                    │   AI + VectorDB             │
                    │   + fixed flow = RAG         │
                    │   + ตัดสินใจเอง = Agentic RAG│
                    │                             │
                    └─────────────────────────────┘
```
