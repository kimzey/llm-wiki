# Lang Ecosystem ทั้งหมด — ครบทุกตัว

> สรุปทุกสิ่งในตระกูล Lang ว่ามีอะไร ทำอะไร ฟรีไหม ใช้เมื่อไหร่

---

## ภาพรวมทั้งหมด

```
┌─────────────────────────────────────────────────────────────────┐
│                      LANG ECOSYSTEM                             │
│                                                                 │
│  ┌─── สร้าง ──────────────────────────────────────────────┐     │
│  │                                                        │     │
│  │  LangChain          LangGraph         LangChain.js     │     │
│  │  (Framework หลัก)    (Flow ซับซ้อน)    (JavaScript)     │     │
│  │                                                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌─── Deploy ─────────────────────────────────────────────┐     │
│  │                                                        │     │
│  │  LangServe          LangGraph Platform                 │     │
│  │  (REST API)          (Managed Cloud)                   │     │
│  │                                                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌─── ตรวจสอบ & ปรับปรุง ─────────────────────────────────┐     │
│  │                                                        │     │
│  │  LangSmith          LangGraph Studio                   │     │
│  │  (Trace/Eval/Monitor) (Debug GUI)                      │     │
│  │                                                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
│  ┌─── Community & Integrations ───────────────────────────┐     │
│  │                                                        │     │
│  │  LangChain Community    LangChain Hub                  │     │
│  │  (Third-party plugins)  (Prompt marketplace)           │     │
│  │                                                        │     │
│  └────────────────────────────────────────────────────────┘     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. LangChain — Framework หลัก

| | |
|---|---|
| **คือ** | Open Source Framework สำหรับสร้าง AI Application ด้วย LLM |
| **ภาษา** | Python |
| **ฟรี** | ✅ ฟรี 100% (MIT License) |
| **ติดตั้ง** | `pip install langchain` |

### ทำอะไรได้

```
📦 Models — เชื่อมต่อ LLM ทุกเจ้า
   ├── Google Gemini, OpenAI GPT, Anthropic Claude, Ollama (Local)
   ├── invoke(), stream(), batch(), ainvoke()
   └── เปลี่ยน model ได้ทันทีโดยไม่ต้องแก้ code

💬 Messages — ระบบสื่อสารกับ LLM
   ├── SystemMessage (กำหนดบทบาท)
   ├── HumanMessage (คำถาม/คำสั่ง + รูปภาพ + PDF)
   ├── AIMessage (คำตอบจาก AI)
   └── ToolMessage (ผลลัพธ์จาก Tool)

📝 Prompt Templates — สร้าง Prompt แบบมี Template
   ├── ChatPromptTemplate (ใส่ตัวแปร {topic})
   ├── MessagesPlaceholder (แทรก chat history)
   └── FewShotChatMessagePromptTemplate (สอน AI ด้วยตัวอย่าง)

🔧 Tools — สร้างเครื่องมือให้ AI ใช้
   ├── @tool decorator
   ├── bind_tools() ให้ model รู้จัก tools
   ├── ToolNode สำหรับรัน tools อัตโนมัติ
   └── Built-in tools (Google Search, Wikipedia, etc.)

🤖 Agents — สร้าง AI Agent ที่คิดเองได้
   ├── create_agent() สร้าง Agent สำเร็จรูป
   ├── ReAct Pattern (Reasoning + Acting loop)
   └── Tool calling อัตโนมัติ

📊 Structured Output — บังคับ Output เป็น JSON/Object
   ├── with_structured_output(PydanticModel)
   ├── Nested models, Enum, List
   └── Validation อัตโนมัติ

🔗 Chains & LCEL — เชื่อมขั้นตอนด้วย pipe |
   ├── prompt | model | parser
   ├── RunnableParallel (ทำงานขนาน)
   ├── RunnableLambda (แทรก function)
   └── stream(), batch() ทุก chain

📄 Document Loaders — โหลดข้อมูลจากทุกแหล่ง
   ├── PDF, Word, CSV, JSON, Text
   ├── Website (WebBaseLoader)
   ├── Database, API
   └── Directory (โหลดทั้ง folder)

✂️ Text Splitters — ตัดเอกสารเป็น Chunks
   ├── RecursiveCharacterTextSplitter
   ├── MarkdownHeaderTextSplitter
   ├── CodeSplitter (Python, JS, etc.)
   └── SemanticChunker (ตัดตามความหมาย)

🧮 Embeddings — แปลงข้อความเป็น Vector
   ├── Google text-embedding-004
   ├── OpenAI text-embedding-3
   └── HuggingFace (ฟรี, รันบนเครื่อง)

🗄️ Vector Stores — จัดเก็บ Embeddings
   ├── Chroma (เริ่มต้น)
   ├── FAISS (เร็วมาก)
   ├── Pinecone, Weaviate, Qdrant (Production)
   └── pgvector (PostgreSQL)

🔍 Retrievers — ค้นหาข้อมูล
   ├── Similarity Search
   ├── MMR (หลากหลาย)
   ├── Multi-Query Retriever
   ├── Self-Query Retriever (auto filter)
   ├── Ensemble Retriever (Hybrid Search)
   └── Contextual Compression

🧠 Memory — ระบบความจำ
   ├── Chat History (Short-term)
   ├── Summary Memory (สรุปเมื่อยาว)
   └── Sliding Window (จำกัดขนาด)

🛡️ Callbacks — ติดตามการทำงาน
   ├── on_llm_start / on_llm_end
   ├── on_tool_start / on_tool_end
   └── Custom callbacks

⚡ Error Handling
   ├── with_retry() (ลองใหม่อัตโนมัติ)
   ├── with_fallbacks() (model สำรอง)
   └── Rate Limiting
```

---

## 2. LangGraph — Graph-based Agent Orchestration

| | |
|---|---|
| **คือ** | Library สำหรับสร้าง Agent workflow ที่ซับซ้อนด้วย Graph |
| **ภาษา** | Python |
| **ฟรี** | ✅ ฟรี 100% (Open Source) |
| **ติดตั้ง** | `pip install langgraph` |
| **ความสัมพันธ์** | สร้างบน LangChain ใช้ร่วมกัน |

### ทำอะไรได้

```
📐 Core Concepts
   ├── State — ข้อมูลที่ไหลผ่าน Graph (TypedDict + Reducers)
   ├── Nodes — ขั้นตอนทำงาน (function ที่รับ/return state)
   ├── Edges — เส้นเชื่อมระหว่าง Nodes
   └── Conditional Edges — เส้นเชื่อมแบบมีเงื่อนไข (routing)

🔄 Flow Patterns
   ├── Linear: A → B → C → END
   ├── Branching: A → (เงื่อนไข?) → B หรือ C
   ├── Loop: Agent ↔ Tools (วนจนเสร็จ)
   ├── Fan-out/Fan-in: แยก → ทำขนาน → รวม
   └── Nested: Graph ซ้อน Graph (Subgraphs)

💾 Checkpointing — บันทึก State
   ├── MemorySaver (In-memory)
   ├── SqliteSaver (ลงไฟล์)
   ├── PostgresSaver (Production)
   ├── จำบทสนทนาข้าม invoke
   ├── Time-travel (ย้อนดู state เก่า)
   └── State History

🙋 Human-in-the-Loop — คนอนุมัติ
   ├── interrupt_before=["node"] (หยุดก่อน node)
   ├── interrupt_after=["node"] (หยุดหลัง node)
   ├── Dynamic interrupt (หยุดตามเงื่อนไข)
   └── update_state() (แก้ state แล้วทำต่อ)

👥 Multi-Agent Systems
   ├── Supervisor Pattern (มีหัวหน้า)
   ├── Handoff Pattern (ส่งต่อ)
   ├── Swarm Pattern (สื่อสารอิสระ)
   ├── Pipeline Pattern (สายพาน)
   └── Debate Pattern (ถกเถียง + Judge)

📡 Streaming
   ├── stream() — ทีละ Node
   ├── stream_mode="values" — ทุก state change
   └── astream_events() — ทีละ token

🔧 Prebuilt Components
   ├── ToolNode — รัน tools อัตโนมัติ
   ├── tools_condition — route tool calls
   └── create_react_agent — Agent สำเร็จรูป
```

### LangChain vs LangGraph ใช้เมื่อไหร่

```
ใช้ LangChain (create_agent):
  ✅ Chatbot ง่ายๆ
  ✅ Agent + Tools แบบพื้นฐาน
  ✅ RAG ไม่ซับซ้อน
  ✅ อยากเริ่มเร็ว

ใช้ LangGraph:
  ✅ ต้องการ Branching Logic
  ✅ Human-in-the-loop (คนอนุมัติ)
  ✅ Multi-Agent (หลาย Agent ทำงานร่วมกัน)
  ✅ Retry/Error handling ละเอียด
  ✅ State management ข้าม session
  ✅ Workflow ซับซ้อน
```

---

## 3. LangSmith — Observability & Evaluation Platform

| | |
|---|---|
| **คือ** | Platform สำหรับ Tracing, Debugging, Evaluation, Monitoring |
| **ฟรี** | ⚠️ Free Tier 5,000 traces/เดือน → จ่ายเริ่ม $39/เดือน |
| **ติดตั้ง** | `pip install langsmith` |
| **ทางเลือกฟรี** | LangFuse (self-host), Phoenix (local), เขียน Callback เอง |

### ทำอะไรได้

```
🔍 Tracing — ดูทุก step ที่เกิดขึ้น
   ├── Auto tracing (แค่ตั้ง env ไม่ต้องแก้ code)
   ├── @traceable (trace custom function)
   ├── Nested traces (ดู step ซ้อน step)
   ├── ดู input/output/latency/tokens ของทุก step
   └── Tags & Metadata สำหรับ filter

🐛 Debugging — หาจุดที่ผิดพลาด
   ├── คลิกดูแต่ละ step ว่าส่ง prompt อะไร ได้คำตอบอะไร
   ├── ดู token usage
   ├── ดู error traces
   └── Compare traces

📊 Evaluation — วัดคุณภาพ
   ├── สร้าง Datasets (ชุดทดสอบ)
   ├── Rule-based Evaluators (exact match, keyword)
   ├── LLM-as-Judge (ใช้ AI ตรวจ AI)
   ├── Pairwise Comparison (เปรียบเทียบ 2 versions)
   ├── Multi-dimension Grading (หลายมิติ)
   └── RAG-specific Evaluators (Faithfulness, Relevance)

📈 Monitoring — ดู Production
   ├── Dashboard (latency, error rate, token usage)
   ├── Alerts (แจ้งเตือนเมื่อผิดปกติ)
   ├── Online Evaluation (ตรวจ realtime)
   └── Export data สำหรับวิเคราะห์

📝 Prompt Hub — จัดการ Prompts
   ├── Version control
   ├── Push/Pull prompts
   └── แชร์ภายในทีม

✍️ Annotation — ให้คนมา Review
   ├── Human labeling
   ├── Feedback collection
   └── สร้าง dataset จาก production traces
```

---

## 4. LangServe — สร้าง REST API อัตโนมัติ

| | |
|---|---|
| **คือ** | Library ที่แปลง LangChain Chain/Agent เป็น REST API ทันที |
| **ฟรี** | ✅ ฟรี 100% (Open Source) |
| **ติดตั้ง** | `pip install langserve[all]` |

### ทำอะไรได้

```
🌐 Auto REST API
   ├── POST /invoke — เรียก chain
   ├── POST /batch — เรียกหลาย input
   ├── POST /stream — Stream ผลลัพธ์
   └── เขียน 3 บรรทัดได้ API

📖 Auto Documentation
   ├── Swagger UI (/docs)
   ├── Input/Output Schema อัตโนมัติ
   └── ดูได้เลยไม่ต้องเขียนเอง

🎮 Playground
   ├── Interactive UI สำหรับทดสอบ
   └── ทดสอบได้เลยบน browser

🔌 Client SDK
   ├── RemoteRunnable (Python)
   └── ใช้เหมือน chain ปกติ
```

---

## 5. LangGraph Platform — Managed Deployment

| | |
|---|---|
| **คือ** | Managed infrastructure สำหรับ deploy LangGraph agents |
| **ฟรี** | ⚠️ มี Free Tier จำกัด → จ่ายตาม usage |
| **ติดตั้ง** | `pip install langgraph-cli langgraph-sdk` |

### ทำอะไรได้

```
☁️ Managed Infrastructure
   ├── langgraph deploy → ได้ API ทันที
   ├── Auto-scaling
   ├── State persistence (built-in)
   └── ไม่ต้องจัดการ server

📡 API Server
   ├── REST API + WebSocket
   ├── Streaming support
   ├── Thread management (session)
   └── Async task queue

🛠️ Developer Tools
   ├── langgraph dev (local testing)
   ├── LangGraph SDK (Python client)
   ├── Cron jobs
   └── Double-texting handling

🔗 Integrations
   ├── LangSmith (tracing อัตโนมัติ)
   └── Webhooks
```

---

## 6. LangGraph Studio — Visual Debugger

| | |
|---|---|
| **คือ** | Desktop application สำหรับ visualize และ debug LangGraph |
| **ฟรี** | ✅ ฟรี |
| **ดาวน์โหลด** | smith.langchain.com |

### ทำอะไรได้

```
🎨 Visual Graph
   ├── เห็น Nodes และ Edges แบบ visual
   ├── ดู data flow ระหว่าง Nodes
   └── เห็น current state ณ แต่ละจุด

🐛 Interactive Debugging
   ├── Step-by-step execution
   ├── ดู state ที่แต่ละ Node
   ├── แก้ state แล้วรันต่อ
   └── Time-travel (ย้อนกลับ)

🔄 Live Testing
   ├── ส่ง input ทดสอบผ่าน UI
   ├── ดู streaming output
   └── Interrupt & resume
```

---

## 7. LangChain.js — JavaScript/TypeScript Version

| | |
|---|---|
| **คือ** | LangChain สำหรับ JavaScript/TypeScript |
| **ฟรี** | ✅ ฟรี 100% (Open Source) |
| **ติดตั้ง** | `npm install langchain` |

### ทำอะไรได้

```
เหมือน LangChain Python แต่สำหรับ JS/TS:
   ├── ใช้กับ Node.js, Deno, Bun
   ├── ใช้กับ Next.js, Vercel
   ├── ใช้กับ Cloudflare Workers
   ├── Models, Tools, Agents, RAG
   └── LangGraph.js (Graph สำหรับ JS)
```

---

## 8. LangChain Community — Third-party Integrations

| | |
|---|---|
| **คือ** | Collection ของ integrations จาก community |
| **ฟรี** | ✅ ฟรี 100% (Open Source) |
| **ติดตั้ง** | `pip install langchain-community` |

### ทำอะไรได้

```
🔌 Integrations (100+ ตัว)
   ├── LLM Providers: Ollama, Cohere, HuggingFace, Together, Groq
   ├── Vector Stores: FAISS, Weaviate, Qdrant, Milvus, pgvector
   ├── Document Loaders: Notion, Confluence, Slack, GitHub, S3
   ├── Tools: Wikipedia, Arxiv, DuckDuckGo, Tavily Search
   ├── Retrievers: BM25, TF-IDF, Elastic Search
   └── Callbacks: Weights & Biases, MLflow
```

---

## 9. LangChain Hub — Prompt Marketplace

| | |
|---|---|
| **คือ** | ที่เก็บและแชร์ Prompts |
| **ฟรี** | ✅ ฟรี |
| **เว็บ** | smith.langchain.com/hub |

### ทำอะไรได้

```
📝 Prompt Management
   ├── Push/Pull prompts (version control)
   ├── แชร์ prompts ภายในทีม/สาธารณะ
   ├── ดู prompts จากคนอื่น
   └── ใช้เป็น template ได้เลย

# ใช้งาน
from langchain import hub

prompt = hub.pull("rlm/rag-prompt")  # ดึง prompt จาก hub
hub.push("my-org/my-prompt", prompt) # push ขึ้นไป
```

---

## 10. LangChain Core — Package พื้นฐาน

| | |
|---|---|
| **คือ** | Base interfaces และ abstractions ที่ทุกตัวใช้ร่วมกัน |
| **ฟรี** | ✅ ฟรี 100% |
| **ติดตั้ง** | ติดตั้งมาอัตโนมัติเมื่อลง langchain |

### ประกอบด้วย

```
🧱 Base Classes
   ├── BaseLanguageModel
   ├── BaseChatModel
   ├── BaseRetriever
   ├── BaseTool
   ├── BaseOutputParser
   └── Runnable interface (LCEL)

📨 Messages
   ├── SystemMessage, HumanMessage, AIMessage, ToolMessage
   └── Message utilities

📋 Prompts
   ├── ChatPromptTemplate
   ├── MessagesPlaceholder
   └── FewShotChatMessagePromptTemplate

📄 Documents
   ├── Document class
   └── Document transformers
```

---

## 11. Provider-specific Packages

แต่ละ LLM provider มี package แยก:

| Package | Provider | ติดตั้ง |
|---------|----------|--------|
| `langchain-google-genai` | Google Gemini | `pip install langchain-google-genai` |
| `langchain-openai` | OpenAI (GPT) | `pip install langchain-openai` |
| `langchain-anthropic` | Anthropic (Claude) | `pip install langchain-anthropic` |
| `langchain-chroma` | Chroma Vector Store | `pip install langchain-chroma` |
| `langchain-text-splitters` | Text Splitting | `pip install langchain-text-splitters` |
| `langchain-huggingface` | HuggingFace | `pip install langchain-huggingface` |

---

## แผนผังความสัมพันธ์

```
             ┌──────────────────┐
             │   LangSmith      │  ← ตรวจสอบ & ประเมินผล
             │  (Trace/Eval)    │
             └────────┬─────────┘
                      │ ส่ง traces
                      │
┌─────────────────────┼──────────────────────┐
│                     │                      │
│  ┌──────────────────▼──────────────────┐   │
│  │          LangGraph                  │   │
│  │    (Complex Flows / Multi-Agent)    │   │
│  │                                     │   │
│  │  ┌─────────────────────────────┐    │   │
│  │  │        LangChain            │    │   │
│  │  │  (Models, Tools, RAG,       │    │   │
│  │  │   Messages, Chains)         │    │   │
│  │  │                             │    │   │
│  │  │  ┌───────────────────────┐  │    │   │
│  │  │  │   LangChain Core     │  │    │   │
│  │  │  │ (Base interfaces)    │  │    │   │
│  │  │  └───────────────────────┘  │    │   │
│  │  └─────────────────────────────┘    │   │
│  └─────────────────────────────────────┘   │
│                     │                      │
│         ┌───────────┼───────────┐          │
│         ▼           ▼           ▼          │
│   ┌──────────┐ ┌──────────┐ ┌────────┐    │
│   │LangServe │ │LangGraph │ │ Docker │    │
│   │(REST API)│ │ Platform │ │FastAPI │    │
│   └──────────┘ │ (Cloud)  │ │(Self)  │    │
│                └──────────┘ └────────┘    │
│                                            │
│         ┌──────────────────────┐           │
│         │  LangChain Community │           │
│         │  (100+ integrations) │           │
│         └──────────────────────┘           │
│                                            │
│         ┌──────────────────────┐           │
│         │   Provider Packages  │           │
│         │ (Gemini, OpenAI, ..) │           │
│         └──────────────────────┘           │
│                                            │
└────────────────────────────────────────────┘
```

---

## สรุปราคา

| ตัว | ฟรี? | หมายเหตุ |
|-----|------|----------|
| LangChain | ✅ 100% ฟรี | Open Source |
| LangGraph | ✅ 100% ฟรี | Open Source |
| LangServe | ✅ 100% ฟรี | Open Source |
| LangChain.js | ✅ 100% ฟรี | Open Source |
| LangChain Core | ✅ 100% ฟรี | Open Source |
| LangChain Community | ✅ 100% ฟรี | Open Source |
| LangGraph Studio | ✅ ฟรี | Desktop App |
| LangChain Hub | ✅ ฟรี | Web |
| LangSmith | ⚠️ Free 5K traces/เดือน | $39/เดือน ขึ้นไป |
| LangGraph Platform | ⚠️ Free tier จำกัด | จ่ายตาม usage |

**ค่าใช้จ่ายจริงๆ มาจาก LLM API:**

| LLM | ราคา |
|-----|------|
| Google Gemini (Flash) | ✅ Free tier ใจกว้าง |
| Ollama (Local) | ✅ ฟรี 100% (ต้องมี GPU) |
| OpenAI GPT-4o-mini | $0.15 / 1M input tokens |
| Anthropic Claude Sonnet | $3 / 1M input tokens |

---

## เส้นทางการเรียนรู้แนะนำ

```
Step 1: LangChain พื้นฐาน
   → Model, Messages, Prompt Templates
   → เรียก LLM ได้ ส่งรูปได้ กำหนด output ได้

Step 2: Tools & Agents
   → สร้าง Tool, สร้าง Agent
   → AI เรียกใช้เครื่องมือเองได้

Step 3: RAG
   → Document Loaders → Splitters → Embeddings → Vector Store → Retriever
   → AI ตอบจากเอกสารของเราได้

Step 4: LangGraph
   → State, Nodes, Edges
   → Conditional routing, Loops
   → Human-in-the-loop

Step 5: Advanced RAG
   → Hybrid Search, Re-ranking, Multi-query
   → CRAG, Self-RAG, Adaptive RAG

Step 6: Multi-Agent
   → Supervisor, Handoff, Pipeline
   → หลาย Agent ทำงานร่วมกัน

Step 7: Evaluation
   → LangSmith / LangFuse / Phoenix
   → LLM-as-Judge, RAG Triad

Step 8: Deployment
   → LangServe / FastAPI / Docker
   → LangGraph Platform
   → Production monitoring
```
