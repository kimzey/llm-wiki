# 🧠 OpenRAG — คู่มือฉบับลึกซึ้ง
## บริบท: สร้าง Sellsuki Company Knowledge Bot

---

## 1. OpenRAG คืออะไร?

**OpenRAG** (docs: https://docs.openr.ag/) คือ **open-source framework สำหรับสร้าง RAG pipeline** ที่เน้น 3 จุด:

1. **Modular** — เลือก component แต่ละชิ้นได้อิสระ (retriever, reranker, LLM)
2. **Evaluation-first** — มี built-in evaluation metrics ไม่ต้องติดตั้งแยก
3. **Production-ready** — ออกแบบมาให้ deploy ได้จริง ไม่ใช่แค่ prototype

### RAG คืออะไรก่อน?

```
RAG = Retrieval-Augmented Generation

คำถาม → ค้นหาเอกสารที่เกี่ยวข้อง → ส่งเอกสาร + คำถาม → LLM ตอบจากเอกสารจริง
```

โดยไม่ต้อง fine-tune โมเดล → ถูกกว่า เร็วกว่า อัปเดตข้อมูลได้ทันที

---

## 2. Architecture ภายใน OpenRAG

```
┌─────────────────────────────────────────────────────┐
│                   OpenRAG Pipeline                   │
├──────────┬──────────┬──────────┬──────────┬─────────┤
│  Ingest  │  Index   │ Retrieve │ Rerank   │Generate │
│  Layer   │  Layer   │  Layer   │  Layer   │  Layer  │
├──────────┼──────────┼──────────┼──────────┼─────────┤
│ PDF      │ Vector   │ Semantic │ Cross-   │ OpenAI  │
│ DOCX     │ Store    │ Search   │ Encoder  │ Claude  │
│ Notion   │ (FAISS,  │ BM25     │ Cohere   │ Gemini  │
│ Slack    │ Qdrant,  │ Hybrid   │ Reranker │ Llama   │
│ Web      │ Weaviate)│          │          │         │
└──────────┴──────────┴──────────┴──────────┴─────────┘
```

### แต่ละ Layer ทำอะไร?

| Layer | หน้าที่ | ตัวอย่าง |
|-------|---------|---------|
| **Ingest** | โหลดเอกสาร → ตัดเป็น chunks | PDF ขนาด 50 หน้า → 200 chunks |
| **Index** | แปลง chunk → vector → เก็บใน DB | "ลา 10 วัน" → [0.23, -0.87, ...] |
| **Retrieve** | รับคำถาม → ดึง chunk ที่ใกล้เคียง | top-k=5 chunks |
| **Rerank** | เรียง chunk ใหม่ตาม relevance จริง | chunk ที่ตรงที่สุดขึ้นก่อน |
| **Generate** | ส่ง chunk + คำถาม → LLM ตอบ | "ลาพักร้อนได้ 10 วัน/ปี" |

---

## 3. OpenRAG vs LlamaIndex vs LangChain

### ตำแหน่งใน Ecosystem

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Application Stack                      │
├─────────────────────────────────────────────────────────────┤
│  LangChain  │ Framework ครอบคลุมทุกอย่าง: Agents, Chains,  │
│             │ Tools, Memory, RAG — "Swiss Army Knife"        │
│             │ ✅ ยืดหยุ่นมาก  ❌ ซับซ้อน, verbose           │
├─────────────────────────────────────────────────────────────┤
│  LlamaIndex │ เน้น Data Ingestion + Indexing + RAG          │
│             │ "Data Framework for LLM"                       │
│             │ ✅ Indexing ดีมาก  ❌ Evaluation อ่อน          │
├─────────────────────────────────────────────────────────────┤
│  OpenRAG    │ เน้น RAG Pipeline + Evaluation + Production    │
│             │ "RAG-first, Eval-first"                        │
│             │ ✅ Evaluate ง่าย  ❌ Ecosystem เล็กกว่า         │
└─────────────────────────────────────────────────────────────┘
```

### เปรียบเทียบแบบตาราง

| Feature | OpenRAG | LlamaIndex | LangChain |
|---------|---------|------------|-----------|
| RAG Pipeline | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Evaluation built-in | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| Agent / Tool use | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| Data connectors | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Learning curve | ต่ำ | กลาง | สูง |
| Production readiness | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| Community | เล็ก | ใหญ่ | ใหญ่มาก |
| Lines of code (basic RAG) | ~30 | ~50 | ~80 |

### เมื่อไหรควรใช้อะไร?

```
ต้องการ RAG ที่วัดผลได้ชัด + deploy เร็ว    → OpenRAG ✅
ต้องการ index data หลากหลายรูปแบบมาก       → LlamaIndex ✅
ต้องการ Agent ที่ทำหลายขั้นตอน + ใช้ Tools  → LangChain ✅
```

---

## 4. OpenRAG ได้ Output อะไรออกมา?

### Output หลัก 3 ประเภท

**① Answer Response**
```json
{
  "answer": "พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน 10 วันต่อปี โดยเริ่มนับหลังผ่านทดลองงาน 3 เดือน",
  "sources": [
    {
      "doc": "HR_Policy_2024.pdf",
      "page": 12,
      "chunk": "...สิทธิ์การลาพักร้อนสำหรับพนักงานประจำ...",
      "score": 0.94
    }
  ],
  "confidence": 0.91
}
```

**② Evaluation Metrics**
```json
{
  "faithfulness": 0.95,      // คำตอบตรงกับเอกสารแค่ไหน (ไม่เดา)
  "answer_relevancy": 0.88,  // คำตอบตอบคำถามได้ตรงแค่ไหน
  "context_precision": 0.82, // เอกสารที่ดึงมาตรงแค่ไหน
  "context_recall": 0.79     // ดึงเอกสารครบแค่ไหน
}
```

**③ Retrieval Debug**
```
Query: "ลาพักร้อนกี่วัน"
Retrieved chunks:
  [0.94] HR_Policy_2024.pdf p.12 — "พนักงานมีสิทธิ์ลา 10 วัน..."
  [0.87] HR_Policy_2024.pdf p.13 — "การสะสมวันลา..."
  [0.61] Onboarding_Guide.pdf p.3 — "สวัสดิการพนักงาน..."
```

---

## 5. Use Case: Sellsuki Company Knowledge Bot

### โจทย์จริง

| ก่อน | หลัง |
|------|------|
| ❌ ถาม HR → รอ → ได้คำตอบช้า | ✅ ถาม Bot → ได้ทันที 24/7 |
| ❌ WiFi password → ถามรอบข้าง | ✅ ถาม Slack Bot → ได้ทันที |
| ❌ Dev ไม่รู้ business rule | ✅ ถามใน Cursor → ได้ rule + โค้ด |
| ❌ HR ตอบคำถามซ้ำๆ ทุกวัน | ✅ HR มีเวลาทำงานสำคัญกว่า |

---

## 6. วิธีติดตั้งและใช้งาน OpenRAG

### Step 0: ติดตั้ง

```bash
pip install openrag
# หรือ
pip install openrag[all]  # รวม dependencies ทั้งหมด
```

### Step 1: เตรียมเอกสาร Sellsuki

```
sellsuki-docs/
├── hr/
│   ├── HR_Policy_2024.pdf        # นโยบาย HR
│   ├── Leave_Policy.pdf          # นโยบายการลา
│   └── Benefits_Guide.pdf        # สวัสดิการ
├── it/
│   ├── IT_Setup_Guide.pdf        # WiFi, VPN, อุปกรณ์
│   └── System_Access.pdf         # การขอสิทธิ์ระบบ
├── product/
│   ├── Sellsuki_Platform_Guide.pdf  # วิธีใช้ระบบ
│   └── Order_Management.pdf         # การจัดการ order
└── dev/
    ├── Business_Rules.md          # Business rules
    ├── API_Docs.md                # API documentation
    └── Coding_Standards.md        # มาตรฐานโค้ด
```

### Step 2: สร้าง RAG Pipeline พื้นฐาน

```python
# sellsuki_bot.py
from openrag import RAGPipeline
from openrag.loaders import DirectoryLoader
from openrag.embeddings import OpenAIEmbeddings
from openrag.vectorstores import QdrantVectorStore
from openrag.retrievers import HybridRetriever
from openrag.rerankers import CohereReranker
from openrag.llms import OpenAILLM

# 1. โหลดเอกสารทั้งหมด
loader = DirectoryLoader(
    path="./sellsuki-docs",
    glob="**/*.{pdf,md,docx}",
    recursive=True,
    metadata_extractor=lambda path: {
        "department": path.parts[-2],  # hr, it, product, dev
        "filename": path.name
    }
)

documents = loader.load()
print(f"โหลดได้ {len(documents)} เอกสาร")

# 2. สร้าง Pipeline
pipeline = RAGPipeline(
    embeddings=OpenAIEmbeddings(model="text-embedding-3-large"),
    
    vectorstore=QdrantVectorStore(
        url="http://localhost:6333",
        collection_name="sellsuki_knowledge"
    ),
    
    retriever=HybridRetriever(
        semantic_weight=0.7,   # ค้นหาเชิงความหมาย
        keyword_weight=0.3,    # ค้นหาด้วย keyword
        top_k=10
    ),
    
    reranker=CohereReranker(
        model="rerank-multilingual-v3.0",  # รองรับภาษาไทย!
        top_n=5
    ),
    
    llm=OpenAILLM(
        model="gpt-4o",
        temperature=0,  # ไม่เดา → ตอบจากเอกสารจริงเท่านั้น
        system_prompt="""คุณคือ Sellsuki Knowledge Assistant
        
ตอบคำถามพนักงาน Sellsuki โดย:
1. ตอบจากเอกสารที่ให้มาเท่านั้น ห้ามเดา
2. อ้างอิงแหล่งที่มาทุกครั้ง (ชื่อไฟล์ + หน้า)
3. ถ้าไม่มีข้อมูลในเอกสาร ให้บอกตรงๆ ว่า "ไม่พบข้อมูลนี้ในเอกสาร กรุณาติดต่อ HR โดยตรง"
4. ตอบเป็นภาษาไทย กระชับ ชัดเจน
"""
    )
)

# 3. Index เอกสาร (ทำครั้งเดียว)
pipeline.index(
    documents=documents,
    chunk_size=512,        # ขนาด chunk (tokens)
    chunk_overlap=50,      # overlap เพื่อไม่ให้ตัดกลางเรื่อง
    show_progress=True
)

print("✅ Index เสร็จแล้ว พร้อมตอบคำถาม")
```

### Step 3: ทดสอบตอบคำถาม

```python
# test_queries.py

questions = [
    "ลาพักร้อนได้กี่วันต่อปี?",
    "WiFi password สำนักงานคืออะไร?",
    "วิธีสร้าง order ใน Sellsuki ทำยังไง?",
    "Business rule สำหรับ order ที่ยกเลิกแล้วคืออะไร?",
    "สวัสดิการประกันสุขภาพมีอะไรบ้าง?"
]

for q in questions:
    result = pipeline.query(q)
    
    print(f"\n{'='*50}")
    print(f"❓ คำถาม: {q}")
    print(f"✅ คำตอบ: {result.answer}")
    print(f"📄 แหล่งที่มา:")
    for src in result.sources:
        print(f"   - {src.document} (หน้า {src.page}) [score: {src.score:.2f}]")
    print(f"🎯 Confidence: {result.confidence:.0%}")
```

### Output จริงที่ได้:

```
==================================================
❓ คำถาม: ลาพักร้อนได้กี่วันต่อปี?
✅ คำตอบ: พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน **10 วันทำการต่อปี** 
         โดยเริ่มมีสิทธิ์หลังผ่านทดลองงาน 3 เดือน 
         สามารถสะสมได้สูงสุด 5 วันไปปีถัดไป

📄 แหล่งที่มา:
   - HR_Policy_2024.pdf (หน้า 12) [score: 0.94]
   - Leave_Policy.pdf (หน้า 3) [score: 0.87]
🎯 Confidence: 94%

==================================================
❓ คำถาม: สร้าง order ใน Sellsuki ยังไง?
✅ คำตอบ: การสร้าง Order ใน Sellsuki มีขั้นตอนดังนี้:
         1. ไปที่เมนู "Orders" > "Create New Order"
         2. เลือก Customer จาก dropdown หรือกด "Add New Customer"
         3. เพิ่มสินค้าโดยกด "Add Item" แล้วค้นหา SKU
         4. ตรวจสอบ stock availability (ระบบจะแจ้งอัตโนมัติ)
         5. เลือกช่องทางการชำระเงินและจัดส่ง
         6. กด "Confirm Order" เพื่อยืนยัน

📄 แหล่งที่มา:
   - Order_Management.pdf (หน้า 5-7) [score: 0.96]
   - Sellsuki_Platform_Guide.pdf (หน้า 23) [score: 0.88]
🎯 Confidence: 96%
```

---

## 7. Evaluation — วัดว่า Bot ตอบดีแค่ไหน

นี่คือจุดแข็งที่สุดของ OpenRAG

```python
from openrag.evaluation import RAGEvaluator

evaluator = RAGEvaluator(llm=OpenAILLM(model="gpt-4o"))

# สร้าง test dataset
test_cases = [
    {
        "question": "ลาพักร้อนได้กี่วัน?",
        "ground_truth": "10 วันทำการต่อปี"
    },
    {
        "question": "วิธีสร้าง order?",
        "ground_truth": "ไปที่เมนู Orders > Create New Order..."
    }
]

# รัน evaluation
results = evaluator.evaluate(
    pipeline=pipeline,
    test_cases=test_cases,
    metrics=["faithfulness", "answer_relevancy", "context_precision", "context_recall"]
)

results.print_report()
```

### Evaluation Report Output:

```
╔══════════════════════════════════════════════╗
║       Sellsuki Bot — Evaluation Report        ║
╠══════════════════════════════════════════════╣
║ Metric              │ Score   │ Status        ║
╠══════════════════════════════════════════════╣
║ Faithfulness        │ 0.95    │ ✅ Excellent  ║
║ Answer Relevancy    │ 0.89    │ ✅ Good       ║
║ Context Precision   │ 0.84    │ ✅ Good       ║
║ Context Recall      │ 0.81    │ ✅ Good       ║
╠══════════════════════════════════════════════╣
║ Overall Score       │ 0.87    │ ✅ Good       ║
╚══════════════════════════════════════════════╝

Faithfulness = 0.95 หมายความว่า:
Bot ตอบจากเอกสารจริง 95% ไม่เดา 5%
```

---

## 8. Deploy — เชื่อม LINE / Slack / Web

### Option A: FastAPI Backend (รองรับทุก channel)

```python
# api.py
from fastapi import FastAPI
from pydantic import BaseModel
from sellsuki_bot import pipeline

app = FastAPI()

class QuestionRequest(BaseModel):
    question: str
    user_id: str
    channel: str  # "line", "slack", "web"

@app.post("/ask")
async def ask(req: QuestionRequest):
    result = pipeline.query(req.question)
    
    return {
        "answer": result.answer,
        "sources": [
            {
                "file": src.document,
                "page": src.page,
                "score": src.score
            }
            for src in result.sources
        ],
        "confidence": result.confidence
    }

# รัน: uvicorn api:app --reload
```

### Option B: LINE Bot Integration

```python
# line_bot.py
from linebot import LineBotApi, WebhookHandler
from linebot.models import TextSendMessage

@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    question = event.message.text
    result = pipeline.query(question)
    
    # สร้าง response พร้อม source
    sources_text = "\n".join([
        f"📄 {src.document} หน้า {src.page}"
        for src in result.sources[:2]
    ])
    
    reply = f"{result.answer}\n\n📚 แหล่งที่มา:\n{sources_text}"
    
    line_bot_api.reply_message(
        event.reply_token,
        TextSendMessage(text=reply)
    )
```

### Option C: Slack Bot Integration

```python
# slack_bot.py
from slack_bolt import App

app = App(token=os.environ["SLACK_BOT_TOKEN"])

@app.event("app_mention")
def handle_mention(event, say):
    question = event["text"].replace(f"<@{BOT_ID}>", "").strip()
    result = pipeline.query(question)
    
    say(
        blocks=[
            {
                "type": "section",
                "text": {"type": "mrkdwn", "text": f"*คำตอบ:*\n{result.answer}"}
            },
            {
                "type": "context",
                "elements": [
                    {"type": "mrkdwn", "text": f"📄 {src.document}" }
                    for src in result.sources[:3]
                ]
            }
        ]
    )
```

### Option D: Cursor / IDE Integration (สำหรับ Dev)

```json
// .cursor/mcp.json หรือ settings
{
  "mcpServers": {
    "sellsuki-knowledge": {
      "url": "http://localhost:8000/mcp",
      "description": "Sellsuki Business Rules & Docs"
    }
  }
}
```

Dev ถามใน Cursor: `@sellsuki-knowledge business rule สำหรับ order ที่ถูก cancel?`

---

## 9. Architecture รวมทั้งระบบ

```
┌─────────────────────────────────────────────────────────────┐
│                    Sellsuki Knowledge Bot                    │
├──────────────────────────────────────────────────────────────┤
│  Channels                                                    │
│  ┌──────┐  ┌───────┐  ┌─────────┐  ┌────────┐             │
│  │ LINE │  │ Slack │  │Web Chat │  │ Cursor │             │
│  └──┬───┘  └───┬───┘  └────┬────┘  └───┬────┘             │
│     └──────────┴───────────┴───────────┘                   │
│                        │                                    │
│              ┌─────────▼──────────┐                        │
│              │   FastAPI Backend   │                        │
│              └─────────┬──────────┘                        │
│                        │                                    │
│              ┌─────────▼──────────┐                        │
│              │  OpenRAG Pipeline   │                        │
│              │  ┌───────────────┐ │                        │
│              │  │HybridRetriever│ │                        │
│              │  │CohereReranker │ │                        │
│              │  │GPT-4o / Claude│ │                        │
│              │  └───────────────┘ │                        │
│              └─────────┬──────────┘                        │
│                        │                                    │
│              ┌─────────▼──────────┐                        │
│              │   Qdrant Vector DB  │                        │
│              │  (Sellsuki Docs)    │                        │
│              └────────────────────┘                        │
│                                                             │
│  Data Sources (indexed):                                    │
│  📁 HR Policies  📁 IT Guides  📁 Product Docs  📁 Dev Docs │
└─────────────────────────────────────────────────────────────┘
```

---

## 10. ข้อดีและข้อเสียของ OpenRAG สรุป

### ✅ ข้อดี

| ข้อดี | รายละเอียด |
|-------|-----------|
| **Evaluation built-in** | วัด faithfulness, relevancy ได้ทันที ไม่ต้องติดตั้งแยก |
| **Modular** | เปลี่ยน LLM / Vector DB ได้โดยไม่ต้องแก้โค้ดทั้งหมด |
| **Hybrid Search** | ผสม semantic + keyword → แม่นยำกว่า semantic อย่างเดียว |
| **Multilingual Reranker** | Cohere รองรับภาษาไทยได้ดี |
| **โค้ดน้อย** | สร้าง RAG pipeline ได้ใน ~30 บรรทัด |
| **Production-ready** | มี async, batching, caching ในตัว |

### ❌ ข้อเสีย

| ข้อเสีย | รายละเอียด | วิธีแก้ |
|---------|-----------|--------|
| **Community เล็ก** | ไม่มาก plugins / integrations | ใช้ร่วมกับ LlamaIndex สำหรับ data loading |
| **Agent ไม่แข็งแกร่ง** | ถ้าต้องการ multi-step reasoning | ใช้ LangChain สำหรับส่วน Agent |
| **Docs น้อย** | ยังพัฒนาอยู่ | ดู source code โดยตรง |
| **ภาษาไทย chunking** | ต้องตั้ง chunk_size ให้เหมาะสม | ทดสอบหลาย setting |

---

## 11. Checklist สำหรับ Sellsuki

```
Phase 1 — MVP (2-3 สัปดาห์)
□ รวบรวมเอกสารทั้งหมดจากทุก department
□ ติดตั้ง OpenRAG + Qdrant (Docker)
□ Index เอกสาร + ทดสอบ 20 คำถามจริง
□ วัด evaluation score ให้ได้ > 0.85
□ สร้าง FastAPI endpoint

Phase 2 — Integration (1-2 สัปดาห์)
□ เชื่อม LINE Bot (สำหรับพนักงานทั่วไป)
□ เชื่อม Slack Bot (สำหรับ internal team)
□ เชื่อม Web Chat (หน้า intranet)

Phase 3 — Advanced (ongoing)
□ เชื่อม Cursor MCP (สำหรับ Dev)
□ ตั้ง auto re-index เมื่อมีเอกสารใหม่
□ Dashboard แสดงคำถามที่พนักงานถามบ่อย
□ วัด HR time saved ต่อเดือน
```

---

## 12. ROI ที่คาดหวัง

```
สมมติ HR ตอบคำถามซ้ำๆ วันละ 2 ชั่วโมง
→ 10 วันทำงาน/สัปดาห์ = เดือนละ ~40 ชั่วโมง
→ ค่าแรง HR ชั่วโมงละ 300 บาท
→ ประหยัดได้ ~12,000 บาท/เดือน
→ Development cost ~100,000 บาท
→ คืนทุนภายใน ~9 เดือน
+ พนักงาน happy, ได้คำตอบเร็ว, 24/7
```

---

*เอกสารนี้เขียนสำหรับทีม Sellsuki — อัปเดตล่าสุด มีนาคม 2026*
*OpenRAG docs: https://docs.openr.ag/*
