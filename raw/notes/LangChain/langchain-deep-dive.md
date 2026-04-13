# LangChain Deep Dive — ทุกสิ่งที่ต้องรู้เพื่อใช้งานจริง

> เอกสารนี้สอนทุกองค์ประกอบของ LangChain อย่างละเอียด พร้อมตัวอย่าง code ที่รันได้จริง
> ใช้ Python + Gemini เป็นหลัก แต่สามารถเปลี่ยนไปใช้ OpenAI / Anthropic ได้ทันที

---

## สารบัญ

1. [โครงสร้าง Package และการติดตั้ง](#1-โครงสร้าง-package-และการติดตั้ง)
2. [Models — ทุกวิธีในการเรียกใช้ LLM](#2-models)
3. [Messages — ระบบสื่อสารกับ LLM อย่างละเอียด](#3-messages)
4. [Prompt Templates — สร้าง Prompt แบบมืออาชีพ](#4-prompt-templates)
5. [Output Parsers & Structured Output](#5-output-parsers--structured-output)
6. [Tools — สร้างเครื่องมือให้ LLM ใช้งาน](#6-tools)
7. [Agents — สร้าง AI Agent ที่คิดเองได้](#7-agents)
8. [Memory & State — ระบบความจำ](#8-memory--state)
9. [Chains & LCEL — เชื่อมต่อขั้นตอนเข้าด้วยกัน](#9-chains--lcel)
10. [Document Loaders — โหลดข้อมูลจากทุกแหล่ง](#10-document-loaders)
11. [Text Splitters — ตัดเอกสารอย่างชาญฉลาด](#11-text-splitters)
12. [Embeddings & Vector Stores](#12-embeddings--vector-stores)
13. [Callbacks & Middleware](#13-callbacks--middleware)
14. [Error Handling & Best Practices](#14-error-handling--best-practices)

---

## 1. โครงสร้าง Package และการติดตั้ง

### แผนผัง Package ของ LangChain (v1)

```
langchain (Namespace หลัก)
├── langchain-core        ← Interface พื้นฐาน (Messages, Prompts, Output Parsers)
├── langchain             ← Logic หลัก (Agents, Chains, Retrieval)
├── langchain-community   ← Third-party integrations (Community-maintained)
├── langchain-google-genai← Google Gemini integration
├── langchain-openai      ← OpenAI integration
├── langchain-anthropic   ← Anthropic Claude integration
├── langchain-chroma      ← Chroma Vector Store
├── langchain-text-splitters ← Text splitting utilities
└── langgraph             ← Graph-based agent orchestration (แยก package)
```

### การติดตั้งตาม Use Case

```bash
# พื้นฐาน: LangChain + Gemini
pip install -U "langchain[google-genai]"

# ถ้าใช้ OpenAI
pip install -U langchain-openai

# ถ้าใช้ Anthropic (Claude)
pip install -U langchain-anthropic

# สำหรับ RAG
pip install -U langchain-chroma langchain-text-splitters pypdf

# สำหรับ LangGraph
pip install -U langgraph

# สำหรับ LangSmith
pip install -U langsmith

# ติดตั้งทุกอย่างพร้อมกัน
pip install -U "langchain[google-genai]" langchain-chroma langchain-text-splitters langgraph langsmith pypdf python-dotenv
```

### โครงสร้าง Project แนะนำ

```
my-ai-project/
├── .env                    ← API Keys
├── requirements.txt
├── src/
│   ├── models.py           ← Model configuration
│   ├── tools.py            ← Custom tools
│   ├── agents.py           ← Agent setup
│   ├── prompts/
│   │   ├── system.txt      ← System prompts
│   │   └── templates.json  ← Prompt templates
│   ├── rag/
│   │   ├── loader.py       ← Document loading
│   │   ├── indexer.py       ← Embedding & indexing
│   │   └── retriever.py    ← Retrieval logic
│   └── graph/
│       └── workflow.py     ← LangGraph workflows
├── data/
│   └── documents/          ← Documents for RAG
├── chroma_db/              ← Vector store data
└── tests/
    └── test_agents.py
```

### ไฟล์ .env

```env
# Google Gemini
GOOGLE_API_KEY=AIzaSy...

# OpenAI (ถ้าใช้)
OPENAI_API_KEY=sk-...

# Anthropic (ถ้าใช้)
ANTHROPIC_API_KEY=sk-ant-...

# LangSmith (สำหรับ Tracing)
LANGSMITH_API_KEY=lsv2_pt_...
LANGSMITH_TRACING=true
LANGSMITH_PROJECT=my-project
```

---

## 2. Models

### 2.1 วิธีสร้าง Model — แบบ Provider-Specific

```python
import os
from dotenv import load_dotenv
load_dotenv()

# ---- Google Gemini ----
from langchain_google_genai import ChatGoogleGenerativeAI

gemini = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash-lite",
    temperature=0.7,       # 0 = ตอบตรงๆ, 1 = สร้างสรรค์
    max_output_tokens=1024,
    top_p=0.95,
    top_k=40,
)

# ---- OpenAI ----
from langchain_openai import ChatOpenAI

gpt = ChatOpenAI(
    model="gpt-4o",
    temperature=0.7,
    max_tokens=1024,
)

# ---- Anthropic ----
from langchain_anthropic import ChatAnthropic

claude = ChatAnthropic(
    model="claude-sonnet-4-20250514",
    temperature=0.7,
    max_tokens=1024,
)
```

### 2.2 วิธีสร้าง Model — แบบ Universal (init_chat_model)

ใช้ได้กับทุก provider โดยไม่ต้อง import class เฉพาะ:

```python
from langchain.chat_models import init_chat_model

# ระบุ provider:model_name
gemini = init_chat_model("google_genai:gemini-2.5-flash-lite")
gpt = init_chat_model("openai:gpt-4o")
claude = init_chat_model("anthropic:claude-sonnet-4-20250514")

# ทุกตัวใช้ .invoke() เหมือนกัน
response = gemini.invoke("Hello!")
```

### 2.3 Parameters สำคัญของ Model

```python
model = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash-lite",

    # --- Temperature ---
    # ควบคุมความ "สุ่ม" ในการตอบ
    # 0.0 = ตอบเหมือนเดิมทุกครั้ง (Deterministic) เหมาะกับงาน Factual
    # 0.3-0.5 = Balance ดี เหมาะกับงานทั่วไป
    # 0.7-1.0 = สร้างสรรค์ เหมาะกับงานเขียน
    temperature=0,

    # --- Max Output Tokens ---
    # จำนวน token สูงสุดที่โมเดลจะสร้าง
    # 1 token ≈ 0.75 คำภาษาอังกฤษ / 1-2 ตัวอักษรภาษาไทย
    max_output_tokens=2048,

    # --- Top-p (Nucleus Sampling) ---
    # พิจารณาเฉพาะ token ที่มีความน่าจะเป็นรวมกันถึง p
    # 0.1 = เลือกจาก top 10% → ตอบแคบ
    # 0.95 = เลือกจาก top 95% → ตอบหลากหลาย
    top_p=0.95,

    # --- Top-k ---
    # พิจารณาเฉพาะ k token ที่มีความน่าจะเป็นสูงสุด
    top_k=40,

    # --- Timeout ---
    timeout=30,  # วินาที

    # --- Retry ---
    max_retries=2,
)
```

### 2.4 วิธีเรียกใช้ Model

```python
# วิธีที่ 1: invoke() — ได้คำตอบทั้งหมดทีเดียว
response = model.invoke("สวัสดีครับ")
print(response.content)        # "สวัสดีครับ! มีอะไรให้ช่วยไหมครับ?"
print(response.response_metadata)  # {'token_usage': {...}, 'model_name': '...'}

# วิธีที่ 2: stream() — ได้คำตอบทีละ chunk (เหมาะกับ UI)
for chunk in model.stream("เล่านิทานให้ฟังหน่อย"):
    print(chunk.content, end="", flush=True)
# Output ทีละคำ: "กาล..." "ครั้ง..." "หนึ่ง..."

# วิธีที่ 3: batch() — ส่งหลาย prompt พร้อมกัน
responses = model.batch([
    "เมืองหลวงของไทย",
    "เมืองหลวงของญี่ปุ่น",
    "เมืองหลวงของฝรั่งเศส",
])
for r in responses:
    print(r.content)

# วิธีที่ 4: ainvoke() — Async version
import asyncio
async def ask():
    response = await model.ainvoke("สวัสดีครับ")
    print(response.content)
asyncio.run(ask())
```

### 2.5 ดู Token Usage

```python
response = model.invoke("อธิบาย AI ให้หน่อย")

# ดูจำนวน token ที่ใช้
metadata = response.response_metadata
print(f"Input tokens: {metadata.get('usage_metadata', {}).get('input_tokens')}")
print(f"Output tokens: {metadata.get('usage_metadata', {}).get('output_tokens')}")
print(f"Total tokens: {metadata.get('usage_metadata', {}).get('total_tokens')}")
```

**Output:**
```
Input tokens: 8
Output tokens: 142
Total tokens: 150
```

---

## 3. Messages

### 3.1 ประเภทของ Message ทั้งหมด

```python
from langchain.messages import (
    SystemMessage,    # คำสั่งระบบ / กำหนดพฤติกรรม
    HumanMessage,     # ข้อความจากผู้ใช้
    AIMessage,        # คำตอบจาก AI
    ToolMessage,      # ผลลัพธ์จาก Tool
    RemoveMessage,    # ลบ message ออกจาก state (ใช้ใน LangGraph)
)
```

### 3.2 SystemMessage — กำหนดพฤติกรรม AI

```python
from langchain.messages import SystemMessage, HumanMessage

# SystemMessage = "บุคลิกภาพ" ของ AI
# ใส่ตอนเริ่มบทสนทนาเพื่อบอก AI ว่า "คุณคือใคร ทำอะไร มีข้อจำกัดอะไร"

messages = [
    SystemMessage(content="""คุณคือ "เจ้าเตา" ผู้เชี่ยวชาญด้านอาหารไทย
กฎที่ต้องปฏิบัติ:
- ตอบเฉพาะเรื่องอาหารไทยเท่านั้น
- ถ้าถูกถามเรื่องอื่น ให้ปฏิเสธอย่างสุภาพ
- ใช้ภาษาเป็นกันเอง เรียกผู้ใช้ว่า "เชฟ"
- แนะนำเคล็ดลับการทำอาหารเสมอ"""),

    HumanMessage(content="สอนทำต้มยำกุ้งหน่อย")
]

response = model.invoke(messages)
print(response.content)
```

**Output:**
```
สวัสดีเชฟ! ต้มยำกุ้งเป็นเมนูที่ทำง่ายมากเลย

ส่วนผสมหลัก:
- กุ้งสด 10 ตัว
- น้ำ 3 ถ้วย
- ข่า ตะไคร้ ใบมะกรูด
- พริกขี้หนู น้ำปลา มะนาว
- เห็ดฟาง นมข้นจืด (ถ้าทำแบบน้ำข้น)

🔥 เคล็ดลับจากเจ้าเตา:
ใส่มะนาวตอนปิดไฟแล้วนะเชฟ ไม่งั้นจะขม!
แล้วก็ตะไคร้ต้องทุบให้แตกก่อน กลิ่นจะหอมกว่าหั่นเป็นแว่น
```

### 3.3 HumanMessage — ส่งได้ทั้ง Text และ Multimodal

```python
from langchain.messages import HumanMessage

# === Text ธรรมดา ===
msg_text = HumanMessage(content="สวัสดีครับ")

# === ส่งรูปภาพ (Base64) ===
import base64

def encode_image(path):
    with open(path, "rb") as f:
        return base64.b64encode(f.read()).decode("utf-8")

msg_image_b64 = HumanMessage(content=[
    {"type": "text", "text": "อธิบายภาพนี้หน่อย"},
    {
        "type": "image",
        "base64": encode_image("photo.jpg"),
        "mime_type": "image/jpeg",
    }
])

# === ส่งรูปภาพ (URL) ===
msg_image_url = HumanMessage(content=[
    {"type": "text", "text": "อธิบายภาพนี้"},
    {
        "type": "image_url",
        "image_url": {"url": "https://example.com/cat.jpg"}
    }
])

# === ส่งไฟล์ PDF (Gemini รองรับ) ===
msg_pdf = HumanMessage(content=[
    {"type": "text", "text": "สรุปเอกสารนี้ให้หน่อย"},
    {
        "type": "media",
        "mime_type": "application/pdf",
        "data": encode_image("document.pdf"),  # base64 encoded
    }
])
```

### 3.4 AIMessage — เข้าใจคำตอบจาก AI

```python
response = model.invoke("สวัสดี")

# response คือ AIMessage object
print(type(response))           # <class 'langchain_core.messages.ai.AIMessage'>
print(response.content)         # "สวัสดีครับ! ..."
print(response.response_metadata)  # {'model_name': '...', 'usage_metadata': {...}}
print(response.id)              # unique ID ของ message
print(response.tool_calls)      # [] ถ้าไม่มี tool call, [{'name': ..., 'args': ...}] ถ้ามี
```

### 3.5 สร้าง Conversation History (ประวัติบทสนทนา)

```python
from langchain.messages import SystemMessage, HumanMessage, AIMessage

# จำลองประวัติบทสนทนา เพื่อให้ AI มีบริบท
messages = [
    SystemMessage(content="คุณคือผู้ช่วย AI ที่ตอบสั้นกระชับ"),

    # รอบที่ 1
    HumanMessage(content="ผมชื่อ สมชาย ครับ"),
    AIMessage(content="สวัสดีครับ คุณสมชาย! มีอะไรให้ช่วยไหมครับ?"),

    # รอบที่ 2
    HumanMessage(content="ผมสนใจเรื่อง AI"),
    AIMessage(content="เรื่อง AI น่าสนใจมากครับ อยากรู้เรื่องอะไรเป็นพิเศษไหมครับ?"),

    # รอบที่ 3 (คำถามปัจจุบัน)
    HumanMessage(content="ผมชื่ออะไร และผมสนใจเรื่องอะไร?"),
]

response = model.invoke(messages)
print(response.content)
```

**Output:**
```
คุณชื่อ สมชาย ครับ และคุณสนใจเรื่อง AI ครับ
```

> AI สามารถตอบได้เพราะเราส่ง conversation history ไปด้วย
> ถ้าไม่ส่ง → AI จะไม่รู้ว่าเราชื่ออะไร

### 3.6 Dictionary Format (แบบย่อ)

```python
# แทนที่จะใช้ class สามารถใช้ dict ได้
messages = [
    {"role": "system", "content": "คุณคือผู้ช่วย AI"},
    {"role": "user", "content": "สวัสดี"},
    {"role": "assistant", "content": "สวัสดีครับ!"},
    {"role": "user", "content": "ขอบคุณ"},
]

# role mapping:
# "system"    = SystemMessage
# "user"      = HumanMessage
# "assistant" = AIMessage
# "tool"      = ToolMessage
```

---

## 4. Prompt Templates

### 4.1 ChatPromptTemplate — Template พื้นฐาน

```python
from langchain_core.prompts import ChatPromptTemplate

# สร้าง template ที่มีตัวแปร {language} และ {topic}
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือผู้เชี่ยวชาญด้าน {language}"),
    ("human", "อธิบาย {topic} แบบสั้นๆ ให้หน่อย"),
])

# ใส่ค่าตัวแปร
formatted = prompt.invoke({"language": "Python", "topic": "list comprehension"})

# ดูผลลัพธ์
print(formatted.messages)
# [SystemMessage(content='คุณคือผู้เชี่ยวชาญด้าน Python'),
#  HumanMessage(content='อธิบาย list comprehension แบบสั้นๆ ให้หน่อย')]

# ส่งให้โมเดลเลย
response = model.invoke(formatted)
print(response.content)
```

### 4.2 MessagesPlaceholder — แทรก Dynamic Messages

ใช้เมื่อต้องการแทรก message list ที่ไม่รู้จำนวนล่วงหน้า (เช่น conversation history):

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือผู้ช่วย AI ที่ตอบสั้นกระชับ"),

    # placeholder สำหรับ conversation history
    MessagesPlaceholder(variable_name="chat_history"),

    # คำถามปัจจุบัน
    ("human", "{question}"),
])

# ใช้งาน
from langchain.messages import HumanMessage, AIMessage

formatted = prompt.invoke({
    "chat_history": [
        HumanMessage(content="ผมชื่อ สมชาย"),
        AIMessage(content="สวัสดีครับ คุณสมชาย!"),
    ],
    "question": "ผมชื่ออะไร?"
})

response = model.invoke(formatted)
print(response.content)  # "คุณชื่อ สมชาย ครับ"
```

### 4.3 FewShotChatMessagePromptTemplate — สอน AI ด้วยตัวอย่าง

```python
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate

# ตัวอย่างที่จะสอน AI
examples = [
    {"input": "ดี", "output": "สวัสดีจ้า! 😊"},
    {"input": "บาย", "output": "ไว้เจอกันใหม่นะ! 👋"},
    {"input": "ขอบคุณ", "output": "ยินดีเสมอจ้า! 🙏"},
]

# template สำหรับแต่ละตัวอย่าง
example_prompt = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}"),
])

# รวมเป็น few-shot template
few_shot_prompt = FewShotChatMessagePromptTemplate(
    example_prompt=example_prompt,
    examples=examples,
)

# template สุดท้าย
final_prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือผู้ช่วยที่ตอบแบบน่ารักและใส่ emoji เสมอ"),
    few_shot_prompt,
    ("human", "{input}"),
])

response = model.invoke(final_prompt.invoke({"input": "สบายดีไหม"}))
print(response.content)  # "สบายดีจ้า! ขอบคุณที่ถาม 😄"
```

### 4.4 โหลด Prompt จากไฟล์

```python
import json

# prompts/customer_support.json
# {
#   "system": "คุณคือ Customer Support ของบริษัท {company}...",
#   "greeting": "สวัสดีครับ ผมคือ {bot_name} มีอะไรให้ช่วยครับ?"
# }

with open("prompts/customer_support.json", "r", encoding="utf-8") as f:
    prompts = json.load(f)

prompt = ChatPromptTemplate.from_messages([
    ("system", prompts["system"]),
    ("human", "{question}"),
])

formatted = prompt.invoke({
    "company": "ABC Corp",
    "question": "สินค้าของคุณราคาเท่าไหร่?"
})
```

---

## 5. Output Parsers & Structured Output

### 5.1 with_structured_output() — วิธีหลักที่แนะนำ

```python
from pydantic import BaseModel, Field
from typing import Optional

# กำหนด Schema ด้วย Pydantic
class MovieReview(BaseModel):
    """โครงสร้างข้อมูลรีวิวภาพยนตร์"""
    movie_name: str = Field(description="ชื่อภาพยนตร์")
    rating: float = Field(description="คะแนน 1-10", ge=1, le=10)
    genre: str = Field(description="ประเภทภาพยนตร์")
    pros: list[str] = Field(description="ข้อดี")
    cons: list[str] = Field(description="ข้อเสีย")
    recommend: bool = Field(description="แนะนำหรือไม่")
    summary: str = Field(description="สรุปรีวิวสั้นๆ 1 ประโยค")

# บังคับ output
structured_model = model.with_structured_output(MovieReview)

review = structured_model.invoke("รีวิวหนัง Inception ให้หน่อย")

# ได้ Pydantic object กลับมา → เข้าถึง field ได้เลย
print(f"ชื่อหนัง: {review.movie_name}")      # Inception
print(f"คะแนน: {review.rating}/10")           # 9.2/10
print(f"ประเภท: {review.genre}")              # Sci-Fi / Thriller
print(f"ข้อดี: {review.pros}")                # ['พล็อตซับซ้อนแต่เข้าใจได้', ...]
print(f"ข้อเสีย: {review.cons}")              # ['อาจต้องดูหลายรอบ', ...]
print(f"แนะนำ: {'✅' if review.recommend else '❌'}")  # ✅
print(f"สรุป: {review.summary}")

# แปลงเป็น dict / JSON ได้ทันที
print(review.model_dump())        # → dict
print(review.model_dump_json())   # → JSON string
```

### 5.2 Schema ซับซ้อน — Nested Models

```python
from pydantic import BaseModel, Field

class Address(BaseModel):
    street: str
    city: str
    province: str
    zipcode: str

class Employee(BaseModel):
    name: str = Field(description="ชื่อ-นามสกุล")
    age: int = Field(description="อายุ", ge=18, le=100)
    position: str = Field(description="ตำแหน่งงาน")
    address: Address = Field(description="ที่อยู่")
    skills: list[str] = Field(description="ทักษะ")

structured = model.with_structured_output(Employee)
result = structured.invoke("""
สร้างข้อมูลพนักงานจำลอง: นาย สมชาย มีความสุข อายุ 30 ปี
ตำแหน่ง Senior Developer อยู่ที่ถนนสุขุมวิท กรุงเทพ 10110
มีทักษะ Python, JavaScript, Docker
""")

print(result.model_dump_json(indent=2))
```

**Output:**
```json
{
  "name": "นาย สมชาย มีความสุข",
  "age": 30,
  "position": "Senior Developer",
  "address": {
    "street": "ถนนสุขุมวิท",
    "city": "กรุงเทพ",
    "province": "กรุงเทพมหานคร",
    "zipcode": "10110"
  },
  "skills": ["Python", "JavaScript", "Docker"]
}
```

### 5.3 Enum — บังคับให้เลือกจากตัวเลือก

```python
from enum import Enum
from pydantic import BaseModel, Field

class Sentiment(str, Enum):
    POSITIVE = "positive"
    NEGATIVE = "negative"
    NEUTRAL = "neutral"

class SentimentAnalysis(BaseModel):
    text: str = Field(description="ข้อความต้นฉบับ")
    sentiment: Sentiment = Field(description="ความรู้สึก")
    confidence: float = Field(description="ความมั่นใจ 0-1", ge=0, le=1)
    reason: str = Field(description="เหตุผลสั้นๆ")

structured = model.with_structured_output(SentimentAnalysis)
result = structured.invoke("วิเคราะห์: อาหารร้านนี้อร่อยมาก แต่รอนานไปหน่อย")

print(result.sentiment)    # Sentiment.NEUTRAL
print(result.confidence)   # 0.6
print(result.reason)       # "มีทั้งชมและติ"
```

### 5.4 List Output — ให้ตอบเป็น List

```python
class QuizQuestion(BaseModel):
    question: str = Field(description="คำถาม")
    choices: list[str] = Field(description="ตัวเลือก 4 ข้อ")
    correct_answer: str = Field(description="คำตอบที่ถูกต้อง")
    explanation: str = Field(description="คำอธิบาย")

class Quiz(BaseModel):
    topic: str = Field(description="หัวข้อ")
    questions: list[QuizQuestion] = Field(description="คำถาม 3 ข้อ")

structured = model.with_structured_output(Quiz)
quiz = structured.invoke("สร้างข้อสอบ Python เบื้องต้น 3 ข้อ")

for i, q in enumerate(quiz.questions, 1):
    print(f"\nข้อ {i}: {q.question}")
    for c in q.choices:
        marker = "✅" if c == q.correct_answer else "  "
        print(f"  {marker} {c}")
    print(f"  💡 {q.explanation}")
```

**Output:**
```
ข้อ 1: ข้อใดเป็น mutable data type ใน Python?
  ✅ list
     tuple
     string
     frozenset
  💡 list สามารถเพิ่ม ลบ แก้ไขสมาชิกได้หลังสร้าง

ข้อ 2: ผลลัพธ์ของ len("Hello") คืออะไร?
     4
  ✅ 5
     6
     Error
  💡 "Hello" มี 5 ตัวอักษร: H, e, l, l, o

ข้อ 3: keyword ใดใช้ประกาศฟังก์ชันใน Python?
     function
     func
  ✅ def
     define
  💡 ใน Python ใช้ def ตามด้วยชื่อฟังก์ชัน
```

---

## 6. Tools

### 6.1 สร้าง Tool ด้วย @tool Decorator

```python
from langchain.tools import tool

@tool
def add_numbers(a: float, b: float) -> float:
    """บวกเลข 2 จำนวนเข้าด้วยกัน ใช้เมื่อต้องการคำนวณผลบวก"""
    return a + b

# ดูข้อมูลของ tool
print(add_numbers.name)          # "add_numbers"
print(add_numbers.description)   # "บวกเลข 2 จำนวนเข้าด้วยกัน..."
print(add_numbers.args_schema.model_json_schema())
# {'properties': {'a': {'type': 'number'}, 'b': {'type': 'number'}}, ...}

# เรียกใช้ตรงๆ (สำหรับทดสอบ)
result = add_numbers.invoke({"a": 10, "b": 20})
print(result)  # 30
```

### 6.2 Tool ที่มี Schema ซับซ้อน (ใช้ Pydantic)

```python
from langchain.tools import tool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    """Schema สำหรับการค้นหาสินค้า"""
    query: str = Field(description="คำค้นหา")
    category: str = Field(description="หมวดหมู่: electronics, clothing, food")
    max_price: float = Field(description="ราคาสูงสุด (บาท)", default=10000)
    sort_by: str = Field(description="เรียงตาม: price, rating, newest", default="rating")

@tool(args_schema=SearchInput)
def search_products(query: str, category: str, max_price: float = 10000, sort_by: str = "rating") -> str:
    """ค้นหาสินค้าจากร้านค้าออนไลน์ตามเงื่อนไขที่กำหนด"""
    # Mock implementation
    return f"พบ 15 สินค้า '{query}' ในหมวด {category} ราคาไม่เกิน {max_price} บาท เรียงตาม {sort_by}"
```

### 6.3 Tool แบบ Async

```python
@tool
async def fetch_url(url: str) -> str:
    """ดึงเนื้อหาจาก URL ที่กำหนด"""
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            text = await response.text()
            return text[:500]  # ส่งกลับแค่ 500 ตัวอักษรแรก
```

### 6.4 Tool ที่ Return Artifact (ส่งข้อมูลให้ User ตรง)

```python
@tool(response_format="content_and_artifact")
def generate_chart_data(metric: str, days: int) -> tuple[str, dict]:
    """สร้างข้อมูลกราฟสำหรับ metric ที่ระบุ"""
    import random

    data = {
        "metric": metric,
        "labels": [f"Day {i+1}" for i in range(days)],
        "values": [random.randint(10, 100) for _ in range(days)]
    }

    # Return tuple: (ข้อความสำหรับ AI, artifact สำหรับ user)
    return f"สร้างข้อมูล {metric} จำนวน {days} วันแล้ว", data
```

### 6.5 bind_tools() — ให้ Model รู้จัก Tools

```python
from langchain.tools import tool

@tool
def get_weather(city: str) -> str:
    """ค้นหาสภาพอากาศปัจจุบัน"""
    return f"{city}: 35°C, แดดจัด"

@tool
def calculate(expression: str) -> str:
    """คำนวณนิพจน์ทางคณิตศาสตร์"""
    return str(eval(expression))

# bind tools เข้ากับ model
model_with_tools = model.bind_tools([get_weather, calculate])

# ทดสอบ — model จะตัดสินใจเองว่าจะเรียก tool หรือไม่
response = model_with_tools.invoke("อากาศที่เชียงใหม่เป็นยังไง?")

# ดู tool_calls
if response.tool_calls:
    for tc in response.tool_calls:
        print(f"Tool: {tc['name']}")
        print(f"Args: {tc['args']}")
        print(f"ID: {tc['id']}")
else:
    print(f"ตอบตรง: {response.content}")
```

**Output:**
```
Tool: get_weather
Args: {'city': 'เชียงใหม่'}
ID: call_abc123
```

### 6.6 เรียก Tool ด้วยตัวเอง (Manual Tool Execution)

```python
from langchain.messages import HumanMessage, AIMessage, ToolMessage

# Step 1: ส่งคำถามให้ model
response = model_with_tools.invoke([
    HumanMessage(content="100 * 25 เท่ากับเท่าไหร่?")
])

# Step 2: model ตอบกลับด้วย tool_calls
print(response.tool_calls)
# [{'name': 'calculate', 'args': {'expression': '100 * 25'}, 'id': 'call_xyz'}]

# Step 3: เรียก tool ด้วยตัวเอง
tool_call = response.tool_calls[0]
tool_result = calculate.invoke(tool_call["args"])

# Step 4: ส่ง tool result กลับไปให้ model
final = model_with_tools.invoke([
    HumanMessage(content="100 * 25 เท่ากับเท่าไหร่?"),
    response,  # AIMessage ที่มี tool_calls
    ToolMessage(content=str(tool_result), tool_call_id=tool_call["id"]),
])

print(final.content)  # "100 × 25 = 2,500 ครับ"
```

---

## 7. Agents

### 7.1 create_agent() — สร้าง Agent แบบง่าย

```python
from langchain.agents import create_agent
from langchain.tools import tool

@tool
def search_database(query: str) -> str:
    """ค้นหาข้อมูลจากฐานข้อมูลภายใน"""
    db = {
        "price": "แพ็คเกจ Basic 299 บาท/เดือน, Pro 599 บาท/เดือน, Enterprise ติดต่อฝ่ายขาย",
        "refund": "สามารถขอคืนเงินได้ภายใน 30 วัน ส่งอีเมลไปที่ refund@example.com",
        "support": "ติดต่อ support ได้ที่ Line: @company หรือโทร 02-xxx-xxxx เวลา 9:00-18:00",
    }
    for key, value in db.items():
        if key in query.lower():
            return value
    return "ไม่พบข้อมูล"

@tool
def send_email(to: str, subject: str, body: str) -> str:
    """ส่งอีเมลไปยังผู้รับที่ระบุ"""
    # Mock
    return f"ส่งอีเมลไปยัง {to} เรื่อง '{subject}' เรียบร้อยแล้ว"

agent = create_agent(
    model=model,
    tools=[search_database, send_email],
    system_prompt="""คุณคือ Customer Support Bot ของบริษัท
กฎ:
1. ค้นหาข้อมูลจากฐานข้อมูลก่อนตอบเสมอ
2. ถ้าลูกค้าต้องการติดต่อ ให้เสนอส่งอีเมลให้
3. ตอบสุภาพและเป็นมิตร""",
)

# ใช้งาน
result = agent.invoke({
    "messages": [{"role": "user", "content": "ราคาแพ็คเกจเท่าไหร่บ้างครับ?"}]
})

# ดูทุก message ที่เกิดขึ้น
for msg in result["messages"]:
    print(f"[{msg.type}] {msg.content[:100]}")
```

**Output:**
```
[human] ราคาแพ็คเกจเท่าไหร่บ้างครับ?
[ai]                                          ← (เรียก search_database)
[tool] แพ็คเกจ Basic 299 บาท/เดือน, Pro 599 บาท/เดือน, Enterprise ติดต่อฝ่ายขาย
[ai] เรามีแพ็คเกจให้เลือก 3 แบบครับ:
     - Basic: 299 บาท/เดือน
     - Pro: 599 บาท/เดือน
     - Enterprise: ติดต่อฝ่ายขายโดยตรง
     สนใจแพ็คเกจไหนเป็นพิเศษไหมครับ?
```

### 7.2 Agent Configuration ที่สำคัญ

```python
agent = create_agent(
    model=model,
    tools=[search_database, send_email],
    system_prompt="...",

    # จำกัดจำนวนรอบ tool calling (ป้องกัน infinite loop)
    max_iterations=10,

    # กำหนด structured output
    response_format=MySchema,
)
```

### 7.3 Parallel Tool Calls

LLM บางตัวสามารถเรียก tools หลายตัวพร้อมกันได้:

```python
result = agent.invoke({
    "messages": [{
        "role": "user",
        "content": "ช่วยหาราคา refund policy และช่องทาง support ให้หน่อย"
    }]
})

# AI อาจเรียก search_database 3 ครั้งพร้อมกัน:
# - search_database("price")
# - search_database("refund")
# - search_database("support")
# แล้วนำผลลัพธ์ทั้งหมดมาสรุปรวม
```

---

## 8. Memory & State

### 8.1 Short-term Memory — จำภายใน Session

วิธีที่ง่ายที่สุดคือเก็บ messages ไว้ใน list:

```python
from langchain.messages import HumanMessage, AIMessage

class ChatBot:
    def __init__(self, model, system_prompt=""):
        self.model = model
        self.system_prompt = system_prompt
        self.history = []  # ← เก็บประวัติบทสนทนา

    def chat(self, user_input: str) -> str:
        # สร้าง messages พร้อม history
        messages = []
        if self.system_prompt:
            messages.append({"role": "system", "content": self.system_prompt})
        messages.extend(self.history)
        messages.append({"role": "user", "content": user_input})

        # ส่งให้ model
        response = self.model.invoke(messages)

        # เก็บลง history
        self.history.append({"role": "user", "content": user_input})
        self.history.append({"role": "assistant", "content": response.content})

        return response.content

    def clear_history(self):
        self.history = []

# ใช้งาน
bot = ChatBot(model, system_prompt="คุณคือผู้ช่วย AI ตอบสั้นๆ")
print(bot.chat("ผมชื่อ สมชาย"))     # "สวัสดีครับ คุณสมชาย!"
print(bot.chat("ผมชอบ Python"))     # "Python เป็นภาษาที่ดีครับ..."
print(bot.chat("ผมชื่ออะไร?"))      # "คุณชื่อ สมชาย ครับ"
```

### 8.2 จำกัดขนาด History (Sliding Window)

```python
class ChatBotWithLimit:
    def __init__(self, model, max_messages=20):
        self.model = model
        self.history = []
        self.max_messages = max_messages

    def chat(self, user_input: str) -> str:
        self.history.append({"role": "user", "content": user_input})

        # ตัด history ให้เหลือแค่ N messages ล่าสุด
        recent_history = self.history[-self.max_messages:]

        response = self.model.invoke(recent_history)
        self.history.append({"role": "assistant", "content": response.content})

        return response.content
```

### 8.3 สรุป History เมื่อยาวเกินไป (Summary Memory)

```python
class SummaryMemoryBot:
    def __init__(self, model, max_messages=10):
        self.model = model
        self.history = []
        self.summary = ""
        self.max_messages = max_messages

    def _summarize_history(self):
        """สรุปประวัติเก่าๆ ด้วย LLM"""
        history_text = "\n".join([
            f"{m['role']}: {m['content']}" for m in self.history
        ])
        summary_response = self.model.invoke(
            f"สรุปบทสนทนานี้ให้สั้นที่สุด:\n{history_text}"
        )
        self.summary = summary_response.content
        self.history = []  # ล้าง history เก่า

    def chat(self, user_input: str) -> str:
        # ถ้า history ยาวเกินไป → สรุป
        if len(self.history) > self.max_messages:
            self._summarize_history()

        messages = []
        if self.summary:
            messages.append({
                "role": "system",
                "content": f"สรุปบทสนทนาก่อนหน้า: {self.summary}"
            })
        messages.extend(self.history)
        messages.append({"role": "user", "content": user_input})

        response = self.model.invoke(messages)

        self.history.append({"role": "user", "content": user_input})
        self.history.append({"role": "assistant", "content": response.content})

        return response.content
```

---

## 9. Chains & LCEL

### 9.1 LCEL คืออะไร ?

LCEL (LangChain Expression Language) คือ syntax พิเศษที่ใช้ **pipe operator `|`** เชื่อมต่อขั้นตอนเข้าด้วยกัน เหมือน Unix pipe:

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# สร้าง chain ด้วย | (pipe)
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือนักแปลมืออาชีพ"),
    ("human", "แปลเป็น {language}: {text}"),
])

chain = prompt | model | StrOutputParser()
#        ↑         ↑         ↑
#    สร้าง prompt → ส่งให้ model → ดึงแค่ text content

result = chain.invoke({"language": "English", "text": "สวัสดีครับ วันนี้อากาศดีมาก"})
print(result)  # "Hello, the weather is very nice today."
```

### 9.2 Chain หลายขั้นตอน

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough, RunnableLambda

# Chain 1: แปลภาษา
translate_prompt = ChatPromptTemplate.from_messages([
    ("human", "แปลข้อความนี้เป็นภาษาอังกฤษ ตอบแค่คำแปล:\n{text}"),
])
translate_chain = translate_prompt | model | StrOutputParser()

# Chain 2: สรุป
summarize_prompt = ChatPromptTemplate.from_messages([
    ("human", "สรุปข้อความนี้เป็น 1 ประโยค:\n{translated}"),
])
summarize_chain = summarize_prompt | model | StrOutputParser()

# รวม Chain
full_chain = (
    {"translated": translate_chain}  # Step 1: แปล
    | summarize_chain                 # Step 2: สรุป
)

result = full_chain.invoke({"text": "LangChain เป็น framework ที่ช่วยสร้าง AI application โดยใช้ LLM เป็นหลัก"})
print(result)
```

### 9.3 RunnableParallel — ทำงานขนานกัน

```python
from langchain_core.runnables import RunnableParallel

# สร้าง 3 chains ที่ทำงานพร้อมกัน
analyze_chain = RunnableParallel(
    translation=translate_chain,
    sentiment=ChatPromptTemplate.from_messages([
        ("human", "วิเคราะห์ sentiment ของ: {text}\nตอบแค่: positive/negative/neutral")
    ]) | model | StrOutputParser(),
    keywords=ChatPromptTemplate.from_messages([
        ("human", "ดึง keywords สำคัญจาก: {text}\nตอบเป็น comma-separated")
    ]) | model | StrOutputParser(),
)

result = analyze_chain.invoke({"text": "วันนี้อากาศดีมาก อยากไปเที่ยวทะเล"})
print(result)
# {
#   'translation': 'The weather is very nice today. I want to go to the beach.',
#   'sentiment': 'positive',
#   'keywords': 'อากาศดี, เที่ยวทะเล'
# }
```

### 9.4 RunnableLambda — แทรก Custom Function

```python
from langchain_core.runnables import RunnableLambda

def word_count(text: str) -> dict:
    """นับจำนวนคำ"""
    return {"text": text, "word_count": len(text.split())}

chain = (
    ChatPromptTemplate.from_messages([("human", "เขียนบทกวีสั้นๆ เกี่ยวกับ {topic}")])
    | model
    | StrOutputParser()
    | RunnableLambda(word_count)  # ← แทรก function
)

result = chain.invoke({"topic": "ฝน"})
print(result)
# {'text': 'ฝนโปรยปรายเบาๆ ...', 'word_count': 24}
```

### 9.5 Chain Methods

```python
# ทุก chain มี methods เหล่านี้
chain = prompt | model | StrOutputParser()

# invoke — รันครั้งเดียว
result = chain.invoke({"language": "EN", "text": "สวัสดี"})

# stream — ได้ผลลัพธ์ทีละ chunk
for chunk in chain.stream({"language": "EN", "text": "สวัสดี"}):
    print(chunk, end="")

# batch — รันหลาย input พร้อมกัน
results = chain.batch([
    {"language": "EN", "text": "สวัสดี"},
    {"language": "JP", "text": "สวัสดี"},
])
```

---

## 10. Document Loaders

### 10.1 โหลดจากข้อความตรง

```python
from langchain_core.documents import Document

docs = [
    Document(
        page_content="เนื้อหาเอกสาร...",
        metadata={"source": "manual", "author": "สมชาย", "date": "2025-01-01"}
    )
]
```

### 10.2 โหลดจาก Text File

```python
from langchain_community.document_loaders import TextLoader

loader = TextLoader("data/readme.txt", encoding="utf-8")
docs = loader.load()
print(f"จำนวนเอกสาร: {len(docs)}")
print(f"เนื้อหา: {docs[0].page_content[:200]}")
print(f"Metadata: {docs[0].metadata}")  # {'source': 'data/readme.txt'}
```

### 10.3 โหลดจาก PDF

```bash
pip install pypdf
```

```python
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("data/report.pdf")
docs = loader.load()  # แต่ละหน้าเป็น 1 Document

for doc in docs:
    print(f"หน้า {doc.metadata['page']}: {doc.page_content[:100]}...")
```

### 10.4 โหลดจาก CSV

```python
from langchain_community.document_loaders.csv_loader import CSVLoader

loader = CSVLoader("data/products.csv", encoding="utf-8")
docs = loader.load()

# แต่ละ row เป็น 1 Document
print(docs[0].page_content)
# "name: iPhone 15\nprice: 32900\ncategory: Electronics"
```

### 10.5 โหลดจากเว็บไซต์

```bash
pip install beautifulsoup4
```

```python
from langchain_community.document_loaders import WebBaseLoader

# โหลดหน้าเดียว
loader = WebBaseLoader("https://example.com/article")
docs = loader.load()

# โหลดหลายหน้า
loader = WebBaseLoader([
    "https://example.com/page1",
    "https://example.com/page2",
])
docs = loader.load()
```

### 10.6 โหลดจาก Directory (หลายไฟล์)

```python
from langchain_community.document_loaders import DirectoryLoader

# โหลดทุกไฟล์ .txt ใน folder
loader = DirectoryLoader(
    "data/documents/",
    glob="**/*.txt",       # pattern ของไฟล์
    show_progress=True,
)
docs = loader.load()
print(f"โหลดได้ {len(docs)} เอกสาร")
```

### 10.7 โหลดจาก JSON

```python
from langchain_community.document_loaders import JSONLoader

# สมมติ data.json:
# [{"title": "...", "content": "...", "author": "..."}]

loader = JSONLoader(
    file_path="data/data.json",
    jq_schema=".[]",                    # เลือกทุก item ใน array
    content_key="content",              # field ที่จะเป็น page_content
    metadata_func=lambda record, metadata: {
        **metadata,
        "title": record.get("title"),
        "author": record.get("author"),
    }
)
docs = loader.load()
```

---

## 11. Text Splitters

### 11.1 ทำไมต้อง Split ?

LLM มี context window จำกัด (เช่น 128K tokens) และ Embedding model ทำงานได้ดีกับ text สั้นๆ การตัดเอกสารเป็น chunks เล็กๆ ช่วยให้ retrieval แม่นยำขึ้น

### 11.2 RecursiveCharacterTextSplitter (แนะนำ)

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,          # ขนาดสูงสุดของ chunk (ตัวอักษร)
    chunk_overlap=100,       # ส่วนที่ซ้อนทับกัน (เพื่อรักษาบริบท)
    separators=[             # ลำดับการตัด (พยายามตัดที่ใหญ่ก่อน)
        "\n\n",              # 1. ย่อหน้า
        "\n",                # 2. บรรทัดใหม่
        "。",                # 3. จุดจบประโยค (ญี่ปุ่น/จีน)
        ".",                 # 4. จุด
        " ",                 # 5. เว้นวรรค
        "",                  # 6. ตัดทีละตัวอักษร (ทางเลือกสุดท้าย)
    ],
    length_function=len,
)

text = """LangChain คือ Framework สำหรับสร้าง AI Application

LangChain มีองค์ประกอบหลักดังนี้:
1. Models - เชื่อมต่อ LLM
2. Tools - เครื่องมือที่ AI ใช้ได้
3. Agents - ระบบที่นำ Models กับ Tools มาทำงานร่วมกัน

LangGraph เป็น library เสริมสำหรับสร้าง workflow ที่ซับซ้อน
ใช้แนวคิด Graph ที่มี Nodes และ Edges"""

chunks = splitter.split_text(text)
for i, chunk in enumerate(chunks):
    print(f"--- Chunk {i+1} ({len(chunk)} chars) ---")
    print(chunk)
    print()
```

### 11.3 Split Documents (ไม่ใช่แค่ text)

```python
# split_documents() จะรักษา metadata ไว้ให้
chunks = splitter.split_documents(docs)

for chunk in chunks[:3]:
    print(f"Source: {chunk.metadata['source']}")
    print(f"Content: {chunk.page_content[:100]}...")
    print()
```

### 11.4 เลือก Chunk Size อย่างไร ?

| Use Case | chunk_size | chunk_overlap | เหตุผล |
|----------|-----------|--------------|--------|
| Q&A สั้นๆ | 200-500 | 50-100 | Chunk เล็ก = retrieval แม่นยำ |
| สรุปเอกสาร | 1000-2000 | 200-400 | ต้องการบริบทมาก |
| Code | 500-1000 | 100-200 | ต้องรักษาโครงสร้าง |
| กฎหมาย/สัญญา | 300-800 | 100-200 | ต้องการความแม่นยำสูง |

---

## 12. Embeddings & Vector Stores

### 12.1 Embedding คืออะไร ?

Embedding คือการแปลงข้อความให้เป็น vector ตัวเลข (เช่น [0.12, -0.45, 0.78, ...]) ที่เก็บ "ความหมาย" ของข้อความไว้ ข้อความที่มีความหมายคล้ายกัน → vector ใกล้กัน

### 12.2 สร้าง Embeddings

```python
from langchain_google_genai import GoogleGenerativeAIEmbeddings

embeddings = GoogleGenerativeAIEmbeddings(model="models/text-embedding-004")

# Embed ข้อความเดียว
vector = embeddings.embed_query("LangChain คือ Framework สำหรับ AI")
print(f"Dimensions: {len(vector)}")   # 768
print(f"First 5 values: {vector[:5]}")  # [0.012, -0.045, ...]

# Embed หลายข้อความ
vectors = embeddings.embed_documents([
    "แมวชอบนอน",
    "สุนัขชอบวิ่ง",
    "Python เป็นภาษาโปรแกรม",
])
print(f"จำนวน vectors: {len(vectors)}")  # 3
```

### 12.3 Vector Store — Chroma

```python
from langchain_chroma import Chroma
from langchain_core.documents import Document

# สร้าง documents
docs = [
    Document(page_content="LangChain คือ Framework สำหรับ AI", metadata={"topic": "langchain"}),
    Document(page_content="Python เป็นภาษาโปรแกรมที่นิยม", metadata={"topic": "python"}),
    Document(page_content="RAG คือการดึงข้อมูลมาเสริม LLM", metadata={"topic": "rag"}),
]

# สร้าง Vector Store
vectorstore = Chroma.from_documents(
    documents=docs,
    embedding=embeddings,
    collection_name="my-collection",
    persist_directory="./chroma_db",  # บันทึกลงดิสก์
)

# ค้นหาด้วย similarity
results = vectorstore.similarity_search("framework สร้าง AI", k=2)
for doc in results:
    print(f"[{doc.metadata['topic']}] {doc.page_content}")

# ค้นหาพร้อม score
results_with_scores = vectorstore.similarity_search_with_score("framework สร้าง AI", k=2)
for doc, score in results_with_scores:
    print(f"Score: {score:.4f} | {doc.page_content}")
```

**Output:**
```
Score: 0.2341 | LangChain คือ Framework สำหรับ AI
Score: 0.5672 | RAG คือการดึงข้อมูลมาเสริม LLM
```

### 12.4 โหลด Vector Store ที่มีอยู่แล้ว

```python
# ครั้งถัดไป ไม่ต้องสร้างใหม่ แค่โหลด
vectorstore = Chroma(
    collection_name="my-collection",
    embedding_function=embeddings,
    persist_directory="./chroma_db",
)
```

### 12.5 Retriever — ตัวค้นหาอัตโนมัติ

```python
# สร้าง retriever จาก vector store
retriever = vectorstore.as_retriever(
    search_type="similarity",      # "similarity" | "mmr" | "similarity_score_threshold"
    search_kwargs={
        "k": 3,                    # จำนวนผลลัพธ์
        # "score_threshold": 0.5,  # (ถ้าใช้ similarity_score_threshold)
        # "fetch_k": 10,           # (ถ้าใช้ mmr) จำนวนที่ดึงมาก่อน rerank
    }
)

# ใช้ retriever
docs = retriever.invoke("อธิบาย LangChain")
for doc in docs:
    print(doc.page_content)
```

### 12.6 Filter ด้วย Metadata

```python
# ค้นหาเฉพาะ topic ที่ต้องการ
results = vectorstore.similarity_search(
    "framework",
    k=3,
    filter={"topic": "langchain"}  # กรองเฉพาะ topic = langchain
)
```

---

## 13. Callbacks & Middleware

### 13.1 Callbacks — ติดตามการทำงาน

```python
from langchain_core.callbacks import BaseCallbackHandler

class MyCallback(BaseCallbackHandler):
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"🚀 LLM เริ่มทำงาน...")

    def on_llm_end(self, response, **kwargs):
        print(f"✅ LLM ทำงานเสร็จ")
        # ดู token usage
        if hasattr(response, 'llm_output') and response.llm_output:
            print(f"   Token usage: {response.llm_output.get('usage', 'N/A')}")

    def on_tool_start(self, serialized, input_str, **kwargs):
        print(f"🔧 เรียก Tool: {serialized.get('name', 'unknown')}")

    def on_tool_end(self, output, **kwargs):
        print(f"📦 Tool result: {output[:100]}")

    def on_llm_error(self, error, **kwargs):
        print(f"❌ Error: {error}")

# ใช้งาน
callback = MyCallback()
response = model.invoke("สวัสดี", config={"callbacks": [callback]})
```

**Output:**
```
🚀 LLM เริ่มทำงาน...
✅ LLM ทำงานเสร็จ
```

### 13.2 Streaming Callback

```python
from langchain_core.callbacks import BaseCallbackHandler

class StreamCallback(BaseCallbackHandler):
    def on_llm_new_token(self, token: str, **kwargs):
        print(token, end="", flush=True)

# ใช้กับ streaming
model.invoke("เล่านิทานสั้นๆ", config={"callbacks": [StreamCallback()]})
```

---

## 14. Error Handling & Best Practices

### 14.1 with_retry — Retry อัตโนมัติ

```python
from langchain_core.runnables import RunnableConfig

# model ที่มี retry
model_with_retry = model.with_retry(
    stop_after_attempt=3,    # ลองใหม่สูงสุด 3 ครั้ง
    wait_exponential_multiplier=1,
    wait_exponential_max=10,
)

response = model_with_retry.invoke("สวัสดี")
```

### 14.2 with_fallbacks — ใช้ Model สำรอง

```python
from langchain_google_genai import ChatGoogleGenerativeAI
from langchain_openai import ChatOpenAI

primary = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite")
backup = ChatOpenAI(model="gpt-4o-mini")

# ถ้า primary ล้มเหลว → ใช้ backup
model_safe = primary.with_fallbacks([backup])
response = model_safe.invoke("สวัสดี")
```

### 14.3 Rate Limiting

```python
from langchain_core.rate_limiters import InMemoryRateLimiter

rate_limiter = InMemoryRateLimiter(
    requests_per_second=1,    # จำกัด 1 request/วินาที
    check_every_n_seconds=0.1,
    max_bucket_size=10,
)

model = ChatGoogleGenerativeAI(
    model="gemini-2.5-flash-lite",
    rate_limiter=rate_limiter,
)
```

### 14.4 Best Practices รวม

```python
# 1. ใช้ temperature=0 สำหรับ Agent (ลด hallucination)
agent_model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", temperature=0)

# 2. ตั้ง timeout
model = ChatGoogleGenerativeAI(model="gemini-2.5-flash-lite", timeout=30)

# 3. เก็บ API Key ใน .env เสมอ (ห้าม hardcode)
# ❌ model = ChatGoogleGenerativeAI(api_key="AIza...")
# ✅ load_dotenv() → อ่านจาก .env

# 4. ใช้ structured output เมื่อต้องการ output ที่คาดเดาได้
# ❌ response = model.invoke("สร้าง JSON ให้หน่อย")  → อาจได้ format ผิด
# ✅ response = structured_model.invoke("สร้างข้อมูล")  → ได้ Pydantic object

# 5. เขียน Docstring ใน Tool ให้ชัดเจน
# ❌ @tool
#    def search(q): ...
# ✅ @tool
#    def search(q: str) -> str:
#        """ค้นหาข้อมูลสินค้าจากฐานข้อมูล ใช้เมื่อต้องการหาราคาหรือรายละเอียดสินค้า"""

# 6. จำกัด max_iterations ของ Agent
# agent = create_agent(model=model, tools=tools, max_iterations=10)
```

---

## สรุป Quick Reference

| ต้องการทำอะไร | ใช้อะไร |
|---|---|
| เรียก LLM | `model.invoke("...")` |
| Stream ผลลัพธ์ | `model.stream("...")` |
| ส่งรูปภาพ | `HumanMessage(content=[{"type": "image", ...}])` |
| บังคับ Output format | `model.with_structured_output(Schema)` |
| สร้าง Prompt template | `ChatPromptTemplate.from_messages(...)` |
| สร้าง Tool | `@tool` decorator |
| สร้าง Agent | `create_agent(model, tools, system_prompt)` |
| เชื่อม Chain | `prompt \| model \| parser` |
| โหลดเอกสาร | `PyPDFLoader`, `WebBaseLoader`, etc. |
| ตัดเอกสาร | `RecursiveCharacterTextSplitter` |
| สร้าง Vector Store | `Chroma.from_documents(docs, embedding)` |
| ค้นหา | `vectorstore.similarity_search("...")` |
| Retry | `model.with_retry(stop_after_attempt=3)` |
| Fallback | `model.with_fallbacks([backup_model])` |
