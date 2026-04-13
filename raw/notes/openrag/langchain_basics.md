# LangChain พื้นฐาน — คู่มือฉบับสมบูรณ์

---

## LangChain คืออะไร?

LangChain คือ Python/JS library สำหรับสร้าง **แอปพลิเคชันที่ใช้ LLM (AI)** โดยมีเครื่องมือครบครัน ตั้งแต่คุยกับ AI, จำบทสนทนา, อ่านเอกสาร, ค้นหาข้อมูล, ไปจนถึงสร้าง Agent ที่คิดและทำงานเองได้

```
คุณ  →  LangChain  →  LLM (GPT, Claude, Gemini ฯลฯ)
                  ↕
             Tools, Memory, Documents, DB
```

---

## บทที่ 1 — LLM & ChatModel (คุยกับ AI)

### 1.1 ChatOpenAI — คุยกับ AI ตรงๆ

```python
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, SystemMessage

llm = ChatOpenAI(
    model="gpt-4o-mini",
    openai_api_key="sk-...",
    temperature=0.7,   # 0 = ตายตัว, 1 = สร้างสรรค์
    max_tokens=1024,
)

messages = [
    SystemMessage(content="คุณเป็นผู้ช่วยที่ตอบภาษาไทย"),
    HumanMessage(content="Python คืออะไร?"),
]

response = llm.invoke(messages)
print(response.content)
```

**OUTPUT:**
```
Python คือภาษาโปรแกรมระดับสูงที่อ่านง่าย เขียนง่าย นิยมใช้ใน
Data Science, Web Development, และ AI/ML ครับ
```

---

### 1.2 Message Types

| Class | ใช้เมื่อ |
|---|---|
| `SystemMessage` | กำหนด "บทบาท" ของ AI |
| `HumanMessage` | ข้อความจากผู้ใช้ |
| `AIMessage` | ข้อความที่ AI ตอบ (ใช้ตอนสร้าง history) |

---

## บทที่ 2 — PromptTemplate (แม่แบบ Prompt)

แทนที่จะเขียน string ตรงๆ ให้ใช้ Template ที่มี **ตัวแปร** แทน

### 2.1 ChatPromptTemplate

```python
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณเป็นผู้เชี่ยวชาญด้าน {subject}"),
    ("human", "อธิบาย {topic} ให้ {level} เข้าใจ"),
])

# ดู template
print(prompt.input_variables)
# OUTPUT: ['subject', 'topic', 'level']

# format เป็น messages จริงๆ
messages = prompt.format_messages(
    subject="Python",
    topic="Decorator",
    level="มือใหม่",
)
print(messages)
```

**OUTPUT:**
```
[
  SystemMessage(content='คุณเป็นผู้เชี่ยวชาญด้าน Python'),
  HumanMessage(content='อธิบาย Decorator ให้มือใหม่ เข้าใจ')
]
```

---

## บทที่ 3 — Chain (ต่อของหลายอย่างเข้าด้วยกัน)

LangChain ใช้ **LCEL (LangChain Expression Language)** สัญลักษณ์ `|` เหมือน pipe ใน Linux

### 3.1 Basic Chain: Prompt → LLM → Output

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

llm = ChatOpenAI(model="gpt-4o-mini", openai_api_key="sk-...")

prompt = ChatPromptTemplate.from_messages([
    ("system", "ตอบสั้นๆ ภาษาไทย"),
    ("human", "{question}"),
])

parser = StrOutputParser()  # แปลง AIMessage → str ธรรมดา

# สร้าง chain ด้วย |
chain = prompt | llm | parser

result = chain.invoke({"question": "ท้องฟ้าสีอะไร?"})
print(result)
```

**OUTPUT:**
```
ท้องฟ้ามีสีฟ้า เกิดจากการกระเจิงของแสงแดด (Rayleigh Scattering) ครับ
```

### 3.2 Data Flow ใน Chain

```
{"question": "ท้องฟ้าสีอะไร?"}
        ↓  prompt
[SystemMessage, HumanMessage]
        ↓  llm
AIMessage(content="ท้องฟ้ามีสีฟ้า...")
        ↓  StrOutputParser
"ท้องฟ้ามีสีฟ้า..."
```

---

## บทที่ 4 — Output Parser (แปลงผลลัพธ์)

### 4.1 StrOutputParser — ได้ string ธรรมดา

```python
from langchain_core.output_parsers import StrOutputParser
parser = StrOutputParser()
# AIMessage(content="hello") → "hello"
```

### 4.2 JsonOutputParser — ได้ dict/list

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.prompts import ChatPromptTemplate

prompt = ChatPromptTemplate.from_messages([
    ("system", "ตอบเป็น JSON เท่านั้น ไม่มี markdown"),
    ("human", "ให้ข้อมูลเกี่ยวกับ {country} ในรูปแบบ: name, capital, population"),
])

chain = prompt | llm | JsonOutputParser()
result = chain.invoke({"country": "Thailand"})
print(result)
print(type(result))
```

**OUTPUT:**
```python
{
  "name": "Thailand",
  "capital": "Bangkok",
  "population": "70 million"
}
<class 'dict'>
```

### 4.3 PydanticOutputParser — Validate ด้วย Pydantic

```python
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field

class Country(BaseModel):
    name: str = Field(description="ชื่อประเทศ")
    capital: str = Field(description="เมืองหลวง")
    population: int = Field(description="จำนวนประชากร")

parser = PydanticOutputParser(pydantic_object=Country)

prompt = ChatPromptTemplate.from_messages([
    ("system", "ตอบตามรูปแบบนี้: {format_instructions}"),
    ("human", "ข้อมูลประเทศ {country}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser
result = chain.invoke({"country": "Japan"})

print(result.name)        # Japan
print(result.capital)     # Tokyo
print(result.population)  # 125000000
print(type(result))       # <class 'Country'>
```

---

## บทที่ 5 — Memory (จำบทสนทนา)

### 5.1 ChatMessageHistory — เก็บ history เอง

```python
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# store เก็บ history แยกตาม session_id
store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณเป็นผู้ช่วยที่เป็นมิตร"),
    MessagesPlaceholder(variable_name="history"),  # ← ใส่ history ตรงนี้
    ("human", "{input}"),
])

chain = prompt | llm | StrOutputParser()

# ห่อด้วย RunnableWithMessageHistory เพื่อให้จำได้
chain_with_memory = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history",
)

# Turn 1
r1 = chain_with_memory.invoke(
    {"input": "สวัสดี ผมชื่อ สมชาย"},
    config={"configurable": {"session_id": "user_001"}}
)
print(r1)

# Turn 2
r2 = chain_with_memory.invoke(
    {"input": "ผมชื่ออะไร?"},
    config={"configurable": {"session_id": "user_001"}}
)
print(r2)
```

**OUTPUT:**
```
สวัสดีครับ คุณสมชาย ยินดีที่ได้รู้จักครับ มีอะไรให้ช่วยไหมครับ?

คุณชื่อ สมชาย ครับ ที่คุณบอกผมไปเมื่อกี้นี้เองครับ 😊
```

---

## บทที่ 6 — Document Loader & Text Splitter (อ่านเอกสาร)

### 6.1 Load เอกสาร

```python
from langchain_community.document_loaders import (
    TextLoader,       # .txt
    PyPDFLoader,      # .pdf
    CSVLoader,        # .csv
    WebBaseLoader,    # URL
)

# โหลด PDF
loader = PyPDFLoader("document.pdf")
docs = loader.load()

print(len(docs))          # จำนวนหน้า
print(docs[0].page_content[:100])  # ข้อความหน้าแรก
print(docs[0].metadata)   # metadata
```

**OUTPUT:**
```
15
รายงานประจำปี 2567 บริษัท ABC จำกัด...
{'source': 'document.pdf', 'page': 0}
```

### 6.2 Text Splitter — ตัดข้อความยาวๆ ออกเป็นชิ้น

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # ตัวอักษรต่อชิ้น
    chunk_overlap=50,    # ทับซ้อน 50 ตัว (เพื่อไม่ให้ context ขาด)
)

chunks = splitter.split_documents(docs)
print(f"จาก {len(docs)} หน้า → {len(chunks)} chunks")
```

**OUTPUT:**
```
จาก 15 หน้า → 87 chunks
```

---

## บทที่ 7 — Vector Store & RAG (ค้นหาในเอกสาร)

RAG = Retrieval-Augmented Generation — ให้ AI ตอบโดยอ้างอิงจากเอกสารของเรา

### 7.1 สร้าง Vector Store

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import FAISS

# Embedding = แปลงข้อความเป็นตัวเลข (vector)
embeddings = OpenAIEmbeddings(openai_api_key="sk-...")

# สร้าง vector store จาก chunks
vectorstore = FAISS.from_documents(chunks, embeddings)

# ค้นหาข้อความที่ใกล้เคียง
results = vectorstore.similarity_search("รายได้ปี 2567", k=3)
for r in results:
    print(r.page_content[:80])
    print("---")
```

**OUTPUT:**
```
รายได้รวมของบริษัท ABC ในปี 2567 อยู่ที่ 1,250 ล้านบาท เพิ่มขึ้น 15%
---
ผลประกอบการไตรมาส 4 ปี 2567 มีกำไรสุทธิ 180 ล้านบาท
---
โครงสร้างรายได้แบ่งเป็น Product 60% และ Service 40%
---
```

### 7.2 RAG Chain สมบูรณ์

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough

retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

prompt = ChatPromptTemplate.from_messages([
    ("system", """ตอบคำถามโดยใช้ข้อมูลที่ให้มาเท่านั้น
ถ้าไม่มีข้อมูล ให้บอกว่าไม่ทราบ

Context:
{context}"""),
    ("human", "{question}"),
])

rag_chain = (
    {"context": retriever | format_docs, "question": RunnablePassthrough()}
    | prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("รายได้ปี 2567 เป็นเท่าไร?")
print(answer)
```

**OUTPUT:**
```
จากรายงาน รายได้รวมของบริษัท ABC ในปี 2567 อยู่ที่ 1,250 ล้านบาท
เพิ่มขึ้น 15% จากปีก่อนหน้าครับ
```

---

## บทที่ 8 — Tools & Agent (ให้ AI ทำงานเอง)

### 8.1 สร้าง Tool

```python
from langchain_core.tools import tool

@tool
def get_weather(city: str) -> str:
    """ดึงข้อมูลอากาศของเมืองที่ระบุ"""
    # จริงๆ เรียก API แต่ตัวอย่างนี้ hardcode
    data = {
        "Bangkok": "อากาศร้อน 35°C ความชื้น 80%",
        "Chiang Mai": "อากาศเย็น 22°C ความชื้น 65%",
    }
    return data.get(city, f"ไม่มีข้อมูลของ {city}")

@tool
def calculate(expression: str) -> str:
    """คำนวณสูตรคณิตศาสตร์ เช่น '2 + 2' หรือ '100 * 0.07'"""
    return str(eval(expression))

print(get_weather.name)         # get_weather
print(get_weather.description)  # ดึงข้อมูลอากาศของเมืองที่ระบุ
```

### 8.2 สร้าง Agent

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

tools = [get_weather, calculate]

prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณเป็นผู้ช่วยที่มีเครื่องมือช่วยหลายอย่าง"),
    ("human", "{input}"),
    MessagesPlaceholder(variable_name="agent_scratchpad"),
])

agent = create_tool_calling_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

result = executor.invoke({
    "input": "อากาศกรุงเทพวันนี้เป็นยังไง และ 35 * 1.07 เท่ากับเท่าไร?"
})
print(result["output"])
```

**OUTPUT (verbose=True):**
```
> Entering new AgentExecutor chain...
Invoking: `get_weather` with {'city': 'Bangkok'}
อากาศร้อน 35°C ความชื้น 80%

Invoking: `calculate` with {'expression': '35 * 1.07'}
37.45

> Finished chain.

อากาศกรุงเทพวันนี้ร้อน 35°C ความชื้น 80% ครับ
และ 35 × 1.07 = 37.45 ครับ
```

---

## บทที่ 9 — Callbacks (ดู process ข้างใน)

```python
from langchain_core.callbacks import BaseCallbackHandler

class MyLogger(BaseCallbackHandler):
    def on_llm_start(self, serialized, prompts, **kwargs):
        print(f"🚀 LLM เริ่มทำงาน | prompt ยาว {len(prompts[0])} ตัวอักษร")

    def on_llm_end(self, response, **kwargs):
        tokens = response.llm_output.get("token_usage", {})
        print(f"✅ เสร็จ | ใช้ {tokens.get('total_tokens', '?')} tokens")

    def on_llm_error(self, error, **kwargs):
        print(f"❌ Error: {error}")

llm_with_log = ChatOpenAI(
    model="gpt-4o-mini",
    openai_api_key="sk-...",
    callbacks=[MyLogger()],
)

llm_with_log.invoke([HumanMessage(content="สวัสดี")])
```

**OUTPUT:**
```
🚀 LLM เริ่มทำงาน | prompt ยาว 8 ตัวอักษร
✅ เสร็จ | ใช้ 23 tokens
```

---

## บทที่ 10 — Caching (ประหยัด Token)

### In-Memory Cache

```python
from langchain_core.globals import set_llm_cache
from langchain_community.cache import InMemoryCache

set_llm_cache(InMemoryCache())

# ครั้งแรก → เรียก API
r1 = llm.invoke([HumanMessage(content="2+2 เท่ากับเท่าไร?")])

# ครั้งที่สอง → ได้จาก cache ทันที ไม่เสีย token
r2 = llm.invoke([HumanMessage(content="2+2 เท่ากับเท่าไร?")])
```

### Redis Cache (เก็บข้ามการรีสตาร์ต)

```python
import redis
from langchain_community.cache import RedisCache
from langchain_core.globals import set_llm_cache

r = redis.from_url("redis://localhost:6379")
set_llm_cache(RedisCache(redis_=r, ttl=3600))
```

---

## สรุป Architecture ทั้งหมด

```
Input
  ↓
PromptTemplate  ← ใส่ตัวแปรลงใน template
  ↓
ChatModel (LLM) ← ส่งไปให้ AI ตอบ
  ↓
OutputParser    ← แปลงผลลัพธ์ (str / json / pydantic)
  ↓
Output

--- เสริมพลัง ---
Memory     → จำบทสนทนา
RAG        → อ้างอิงเอกสาร
Tools      → ให้ AI ใช้เครื่องมือ
Agent      → ให้ AI ตัดสินใจเองว่าจะทำอะไร
Cache      → ประหยัด token
Callbacks  → ดู/log ทุก step
```

---

## Package ที่ต้อง Install

```bash
pip install langchain
pip install langchain-openai          # สำหรับ OpenAI / OpenRouter
pip install langchain-community       # loaders, vectorstores, cache
pip install langchain-text-splitters  # text splitter
pip install faiss-cpu                 # vector store
pip install redis                     # redis cache
pip install pypdf                     # อ่าน PDF
```
