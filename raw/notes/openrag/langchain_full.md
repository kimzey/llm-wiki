# LangChain คู่มือสมบูรณ์ — ทุก Syntax ทุก Import

---

## ติดตั้งก่อนเลย

```bash
# Core
pip install langchain
pip install langchain-core
pip install langchain-community
pip install langchain-openai
pip install langchain-text-splitters

# Vector Store
pip install faiss-cpu       # ค้นหาแบบ local
pip install chromadb        # Vector DB แบบ persistent

# Document Loaders
pip install pypdf            # อ่าน PDF
pip install unstructured     # อ่านหลายรูปแบบ
pip install beautifulsoup4   # อ่าน HTML

# Cache
pip install redis

# Utility
pip install pydantic         # validate output
```

---

# ภาพรวม Package ทั้งหมด

```
langchain/
├── langchain-core          ← พื้นฐาน: Runnable, PromptTemplate, Messages
├── langchain               ← Chains, Agents, Memory
├── langchain-openai        ← ChatOpenAI, OpenAIEmbeddings
├── langchain-community     ← Loaders, VectorStores, Cache (community-built)
└── langchain-text-splitters ← ตัดข้อความเป็นชิ้น
```

---

# PART 1 — Messages (ข้อความ)

## 1.1 ชนิดของ Message

```python
from langchain_core.messages import (
    SystemMessage,    # บทบาทของ AI
    HumanMessage,     # ข้อความจากคน
    AIMessage,        # ข้อความจาก AI
    FunctionMessage,  # ผลลัพธ์จาก function/tool
    ToolMessage,      # ผลลัพธ์จาก tool (ใหม่กว่า)
)

# สร้าง message
sys  = SystemMessage(content="คุณเป็นนักแปลภาษา")
user = HumanMessage(content="แปล 'Hello' เป็นไทย")
ai   = AIMessage(content="สวัสดี")

# ส่ง list ไปให้ LLM
messages = [sys, user]
```

## 1.2 Attributes ของ Message

```python
msg = HumanMessage(
    content="สวัสดี",
    name="สมชาย",            # optional: ชื่อผู้ส่ง
    additional_kwargs={},     # ข้อมูลเพิ่มเติม
)

print(msg.content)           # สวัสดี
print(msg.type)              # human
print(msg.name)              # สมชาย
```

---

# PART 2 — ChatModel (LLM)

## 2.1 ChatOpenAI — Parameters ทั้งหมด

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    model="gpt-4o-mini",             # ชื่อ model
    openai_api_key="sk-...",         # API key
    openai_api_base="https://...",   # เปลี่ยน endpoint (OpenRouter ฯลฯ)
    temperature=0.7,                 # 0=ตายตัว  1=สร้างสรรค์
    max_tokens=1024,                 # จำนวน token สูงสุดที่ตอบ
    top_p=1.0,                       # nucleus sampling
    frequency_penalty=0.0,          # ลดการซ้ำคำ
    presence_penalty=0.0,           # ส่งเสริมหัวข้อใหม่
    n=1,                             # จำนวนคำตอบที่ต้องการ
    timeout=30,                      # วินาที
    max_retries=2,                   # retry เมื่อ error
    streaming=False,                 # True = stream ทีละ token
    cache=True,                      # ใช้ cache
    verbose=False,                   # True = แสดง log
)
```

## 2.2 วิธีเรียกใช้ LLM

```python
# 1. invoke — เรียกครั้งเดียว ได้ผลกลับมา
response = llm.invoke([HumanMessage(content="สวัสดี")])
print(response.content)          # สวัสดีครับ!
print(response.usage_metadata)   # {'input_tokens': 3, 'output_tokens': 8}

# 2. stream — ได้ทีละ token (real-time)
for chunk in llm.stream([HumanMessage(content="นับ 1-5")]):
    print(chunk.content, end="", flush=True)
# OUTPUT: 1... 2... 3... 4... 5...

# 3. batch — ส่งหลายคำถามพร้อมกัน
questions = [
    [HumanMessage(content="ไทยเมืองหลวงคืออะไร?")],
    [HumanMessage(content="ญี่ปุ่นเมืองหลวงคืออะไร?")],
]
results = llm.batch(questions)
for r in results:
    print(r.content)
# กรุงเทพมหานครครับ
# โตเกียวครับ

# 4. ainvoke — async version
import asyncio
async def ask():
    r = await llm.ainvoke([HumanMessage(content="สวัสดี")])
    print(r.content)
asyncio.run(ask())
```

## 2.3 LLM อื่นๆ ที่ใช้ได้

```python
# Anthropic Claude
from langchain_anthropic import ChatAnthropic
llm = ChatAnthropic(model="claude-3-5-sonnet-20241022", api_key="...")

# Google Gemini
from langchain_google_genai import ChatGoogleGenerativeAI
llm = ChatGoogleGenerativeAI(model="gemini-pro", google_api_key="...")

# Ollama (Local AI)
from langchain_ollama import ChatOllama
llm = ChatOllama(model="llama3.2")  # ไม่ต้อง API key

# OpenRouter (หลาย model ใน 1 key)
llm = ChatOpenAI(
    model="anthropic/claude-3.5-sonnet",
    openai_api_key="sk-or-...",
    openai_api_base="https://openrouter.ai/api/v1",
)
```

---

# PART 3 — PromptTemplate

## 3.1 PromptTemplate พื้นฐาน (ไม่มี role)

```python
from langchain_core.prompts import PromptTemplate

# สร้างจาก string
p = PromptTemplate(
    input_variables=["name", "topic"],
    template="สวัสดี {name} อธิบาย {topic} ให้ฉันฟัง",
)

# หรือใช้ from_template (ง่ายกว่า)
p = PromptTemplate.from_template("สวัสดี {name} อธิบาย {topic} ให้ฉันฟัง")

result = p.format(name="สมชาย", topic="Python")
print(result)
# สวัสดี สมชาย อธิบาย Python ให้ฉันฟัง
```

## 3.2 ChatPromptTemplate (มี role)

```python
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# วิธีที่ 1: from_messages (แนะนำ)
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณเป็นผู้เชี่ยวชาญด้าน {domain}"),
    ("human", "อธิบาย {topic}"),
])

# วิธีที่ 2: ใช้ Message objects โดยตรง
from langchain_core.prompts import SystemMessagePromptTemplate, HumanMessagePromptTemplate
prompt = ChatPromptTemplate.from_messages([
    SystemMessagePromptTemplate.from_template("คุณเป็น {role}"),
    HumanMessagePromptTemplate.from_template("{question}"),
])

# MessagesPlaceholder — แทรก list ของ messages (ใช้กับ memory)
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณเป็นผู้ช่วย"),
    MessagesPlaceholder("history"),   # ← ใส่ history ที่นี่
    ("human", "{input}"),
])

# ดู variables
print(prompt.input_variables)
# ['domain', 'topic']

# format
messages = prompt.format_messages(domain="Python", topic="decorator")
print(messages)
# [SystemMessage(content='คุณเป็นผู้เชี่ยวชาญด้าน Python'),
#  HumanMessage(content='อธิบาย decorator')]
```

## 3.3 partial() — กำหนดค่าบางตัวล่วงหน้า

```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "ตอบเป็น {language} เสมอ"),
    ("human", "{question}"),
])

# กำหนด language ล่วงหน้า
thai_prompt = prompt.partial(language="ภาษาไทย")

# ตอนใช้ ส่งแค่ question
msgs = thai_prompt.format_messages(question="Python คืออะไร?")
```

---

# PART 4 — LCEL Chain (|)

## 4.1 Syntax พื้นฐาน

```python
# chain = A | B | C
# ผลลัพธ์ของ A → input ของ B → input ของ C

chain = prompt | llm | parser

# invoke
result = chain.invoke({"topic": "Python"})

# stream
for chunk in chain.stream({"topic": "Python"}):
    print(chunk, end="")

# batch
results = chain.batch([
    {"topic": "Python"},
    {"topic": "JavaScript"},
])
```

## 4.2 RunnablePassthrough — ส่งค่าผ่านไปตรงๆ

```python
from langchain_core.runnables import RunnablePassthrough

# ส่ง input ผ่านไปโดยไม่เปลี่ยน
chain = RunnablePassthrough() | llm

# ใช้ใน RAG: ส่ง question ผ่านไปพร้อม context
chain = {
    "context": retriever | format_docs,
    "question": RunnablePassthrough(),   # ← ส่ง input เดิมผ่าน
} | prompt | llm
```

## 4.3 RunnableLambda — ใส่ฟังก์ชัน Python ใน Chain

```python
from langchain_core.runnables import RunnableLambda

def to_upper(text: str) -> str:
    return text.upper()

chain = prompt | llm | StrOutputParser() | RunnableLambda(to_upper)
result = chain.invoke({"question": "สวัสดี"})
# ผลลัพธ์จะถูก .upper() ก่อน return
```

## 4.4 RunnableParallel — รันพร้อมกัน

```python
from langchain_core.runnables import RunnableParallel

# รัน 2 chain พร้อมกัน
parallel = RunnableParallel({
    "summary": summary_chain,
    "keywords": keyword_chain,
})

result = parallel.invoke({"text": "..."})
print(result["summary"])   # ผลลัพธ์จาก summary_chain
print(result["keywords"])  # ผลลัพธ์จาก keyword_chain
```

## 4.5 .assign() — เพิ่ม key ใหม่เข้า dict

```python
from langchain_core.runnables import RunnablePassthrough

chain = (
    RunnablePassthrough.assign(
        context=lambda x: retriever.invoke(x["question"])
    )
    | prompt
    | llm
)
```

## 4.6 .bind() — กำหนด kwargs ให้ Runnable

```python
# bind stop words ให้ LLM
llm_with_stop = llm.bind(stop=["END", "STOP"])

# bind tools
llm_with_tools = llm.bind_tools([get_weather, calculate])
```

## 4.7 itemgetter — ดึง key จาก dict

```python
from operator import itemgetter

chain = (
    {
        "question": itemgetter("question"),
        "language": itemgetter("language"),
    }
    | prompt
    | llm
)

chain.invoke({"question": "สวัสดี", "language": "English"})
```

---

# PART 5 — Output Parsers

## 5.1 StrOutputParser

```python
from langchain_core.output_parsers import StrOutputParser

parser = StrOutputParser()
# AIMessage(content="hello") → "hello"
```

## 5.2 JsonOutputParser

```python
from langchain_core.output_parsers import JsonOutputParser

parser = JsonOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("system", "ตอบ JSON เท่านั้น ห้าม markdown"),
    ("human", "สร้าง user: name={name}, age={age}"),
])

chain = prompt | llm | parser
result = chain.invoke({"name": "สมชาย", "age": 30})

# OUTPUT:
# {"name": "สมชาย", "age": 30}
# type: dict
```

## 5.3 PydanticOutputParser

```python
from langchain_core.output_parsers import PydanticOutputParser
from pydantic import BaseModel, Field
from typing import List

class Person(BaseModel):
    name: str = Field(description="ชื่อ")
    age: int = Field(description="อายุ")
    skills: List[str] = Field(description="ทักษะ")

parser = PydanticOutputParser(pydantic_object=Person)

# ดู format instructions ที่ต้องแนบใน prompt
print(parser.get_format_instructions())

prompt = ChatPromptTemplate.from_messages([
    ("system", "ตอบตามรูปแบบ:\n{format_instructions}"),
    ("human", "สร้างข้อมูล developer ชื่อ {name}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser
result = chain.invoke({"name": "สมชาย"})

print(result.name)    # สมชาย
print(result.age)     # 28
print(result.skills)  # ['Python', 'FastAPI', 'Docker']
print(type(result))   # <class 'Person'>
```

## 5.4 CommaSeparatedListOutputParser

```python
from langchain.output_parsers import CommaSeparatedListOutputParser

parser = CommaSeparatedListOutputParser()

prompt = ChatPromptTemplate.from_messages([
    ("human", "รายชื่อ framework Python 5 อัน คั่นด้วย comma"),
])

chain = prompt | llm | parser
result = chain.invoke({})
# OUTPUT: ['Django', 'FastAPI', 'Flask', 'Tornado', 'Sanic']
# type: list
```

## 5.5 XMLOutputParser

```python
from langchain.output_parsers import XMLOutputParser

parser = XMLOutputParser(tags=["name", "age", "role"])
# parse XML string → dict
```

## 5.6 RetryOutputParser (retry เมื่อ parse ผิดรูปแบบ)

```python
from langchain.output_parsers import RetryOutputParser

retry_parser = RetryOutputParser.from_llm(
    parser=pydantic_parser,
    llm=llm,
    max_retries=3,
)
```

---

# PART 6 — Memory

## 6.1 InMemoryChatMessageHistory

```python
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# เก็บ history หลาย session
store = {}

def get_session_history(session_id: str):
    if session_id not in store:
        store[session_id] = InMemoryChatMessageHistory()
    return store[session_id]

prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณเป็นผู้ช่วย"),
    MessagesPlaceholder("history"),
    ("human", "{input}"),
])

chain = prompt | llm | StrOutputParser()

chain_mem = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="input",
    history_messages_key="history",
)

# config ต้องส่ง session_id ทุกครั้ง
cfg = {"configurable": {"session_id": "user_001"}}

r1 = chain_mem.invoke({"input": "ผมชื่อ สมชาย"}, config=cfg)
r2 = chain_mem.invoke({"input": "ผมชื่ออะไร?"}, config=cfg)

print(r2)
# OUTPUT: คุณชื่อ สมชาย ครับ
```

## 6.2 ดู/จัดการ History โดยตรง

```python
history = get_session_history("user_001")

# ดูทุก message
for msg in history.messages:
    print(f"{msg.type}: {msg.content}")

# เพิ่ม message เอง
history.add_user_message("สวัสดี")
history.add_ai_message("สวัสดีครับ")

# ล้าง history
history.clear()
```

## 6.3 SQLChatMessageHistory (เก็บใน DB)

```python
from langchain_community.chat_message_histories import SQLChatMessageHistory

def get_session_history(session_id: str):
    return SQLChatMessageHistory(
        session_id=session_id,
        connection_string="sqlite:///chat_history.db",
    )
```

## 6.4 RedisChatMessageHistory

```python
from langchain_community.chat_message_histories import RedisChatMessageHistory

def get_session_history(session_id: str):
    return RedisChatMessageHistory(
        session_id=session_id,
        url="redis://localhost:6379",
        ttl=3600,
    )
```

---

# PART 7 — Document Loaders

## 7.1 TextLoader

```python
from langchain_community.document_loaders import TextLoader

loader = TextLoader("file.txt", encoding="utf-8")
docs = loader.load()

# docs คือ List[Document]
print(docs[0].page_content)   # เนื้อหา
print(docs[0].metadata)       # {'source': 'file.txt'}
```

## 7.2 PyPDFLoader

```python
from langchain_community.document_loaders import PyPDFLoader

loader = PyPDFLoader("document.pdf")
docs = loader.load()           # 1 doc ต่อ 1 หน้า
pages = loader.load_and_split() # โหลดแล้วตัดด้วย

print(docs[0].metadata)
# {'source': 'document.pdf', 'page': 0}
```

## 7.3 CSVLoader

```python
from langchain_community.document_loaders import CSVLoader

loader = CSVLoader(
    "data.csv",
    csv_args={"delimiter": ","},
    source_column="url",    # ใช้ column นี้เป็น source
)
docs = loader.load()
# แต่ละแถว CSV = 1 Document
```

## 7.4 WebBaseLoader

```python
from langchain_community.document_loaders import WebBaseLoader

loader = WebBaseLoader(["https://example.com", "https://example2.com"])
docs = loader.load()
```

## 7.5 DirectoryLoader — โหลดทั้ง folder

```python
from langchain_community.document_loaders import DirectoryLoader

loader = DirectoryLoader(
    "./documents/",
    glob="**/*.pdf",          # pattern ไฟล์
    loader_cls=PyPDFLoader,   # ใช้ loader ไหน
    show_progress=True,
)
docs = loader.load()
```

## 7.6 Document Object

```python
from langchain_core.documents import Document

# สร้างเอง
doc = Document(
    page_content="เนื้อหาของเอกสาร",
    metadata={
        "source": "manual",
        "author": "สมชาย",
        "date": "2024-01-01",
    }
)

print(doc.page_content)
print(doc.metadata)
```

---

# PART 8 — Text Splitters

## 8.1 RecursiveCharacterTextSplitter (แนะนำ)

```python
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,          # ขนาดสูงสุดต่อชิ้น (ตัวอักษร)
    chunk_overlap=50,        # ทับซ้อน (ป้องกัน context ขาด)
    separators=["\n\n", "\n", " ", ""],  # ลำดับการตัด
    length_function=len,     # ฟังก์ชันวัดขนาด
    add_start_index=True,    # เพิ่ม start_index ใน metadata
)

# ตัด string
texts = splitter.split_text("ข้อความยาวๆ ...")

# ตัด Document
chunks = splitter.split_documents(docs)
print(f"{len(docs)} docs → {len(chunks)} chunks")
```

## 8.2 CharacterTextSplitter

```python
from langchain_text_splitters import CharacterTextSplitter

splitter = CharacterTextSplitter(
    separator="\n",     # ตัดที่ newline
    chunk_size=500,
    chunk_overlap=50,
)
```

## 8.3 TokenTextSplitter (ตัดตาม Token)

```python
from langchain_text_splitters import TokenTextSplitter

splitter = TokenTextSplitter(
    chunk_size=100,    # token (ไม่ใช่ตัวอักษร)
    chunk_overlap=10,
)
```

---

# PART 9 — Embeddings & Vector Store

## 9.1 OpenAIEmbeddings

```python
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",   # หรือ text-embedding-3-large
    openai_api_key="sk-...",
)

# แปลง text → vector
vector = embeddings.embed_query("Python คืออะไร?")
print(len(vector))    # 1536 (dimensions)

# แปลงหลาย text
vectors = embeddings.embed_documents(["text1", "text2"])
```

## 9.2 FAISS (In-Memory)

```python
from langchain_community.vectorstores import FAISS

# สร้างจาก documents
vectorstore = FAISS.from_documents(chunks, embeddings)

# สร้างจาก texts
vectorstore = FAISS.from_texts(["text1", "text2"], embeddings)

# บันทึก / โหลด
vectorstore.save_local("faiss_index")
vectorstore = FAISS.load_local("faiss_index", embeddings,
                               allow_dangerous_deserialization=True)

# ค้นหา
results = vectorstore.similarity_search("query", k=3)
results_with_score = vectorstore.similarity_search_with_score("query", k=3)

for doc, score in results_with_score:
    print(f"Score: {score:.3f} | {doc.page_content[:50]}")
# Score: 0.123 | Python คือภาษาโปรแกรม...
# Score: 0.456 | ใช้ใน Data Science...
```

## 9.3 Chroma (Persistent)

```python
from langchain_community.vectorstores import Chroma

# สร้างและบันทึก
vectorstore = Chroma.from_documents(
    documents=chunks,
    embedding=embeddings,
    persist_directory="./chroma_db",
)

# โหลดที่มีอยู่
vectorstore = Chroma(
    persist_directory="./chroma_db",
    embedding_function=embeddings,
)
```

## 9.4 Retriever Methods

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",          # similarity / mmr / similarity_score_threshold
    search_kwargs={
        "k": 4,                        # จำนวนผลลัพธ์
        "score_threshold": 0.7,        # เฉพาะ similarity_score_threshold
        "fetch_k": 20,                 # เฉพาะ mmr
        "lambda_mult": 0.5,            # เฉพาะ mmr (diversity)
        "filter": {"source": "a.pdf"}, # filter metadata
    }
)

# ใช้ retriever
docs = retriever.invoke("Python คืออะไร?")
```

---

# PART 10 — Tools & Agent

## 10.1 สร้าง Tool ด้วย @tool

```python
from langchain_core.tools import tool
from typing import Optional

@tool
def search_web(query: str) -> str:
    """ค้นหาข้อมูลบนอินเทอร์เน็ต ใช้เมื่อต้องการข้อมูลล่าสุด"""
    # implementation จริง
    return f"ผลการค้นหา: {query}"

@tool
def get_weather(
    city: str,
    unit: Optional[str] = "celsius"
) -> dict:
    """
    ดึงข้อมูลอากาศ

    Args:
        city: ชื่อเมือง
        unit: หน่วยอุณหภูมิ celsius หรือ fahrenheit
    """
    return {"city": city, "temp": 35, "unit": unit}

# ดู metadata ของ tool
print(search_web.name)          # search_web
print(search_web.description)   # ค้นหาข้อมูล...
print(search_web.args)          # {'query': {'type': 'string'}}
```

## 10.2 StructuredTool (input หลายตัว)

```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel

class CalculatorInput(BaseModel):
    a: float
    b: float
    operation: str  # add, subtract, multiply, divide

def calculator(a: float, b: float, operation: str) -> float:
    ops = {"add": a+b, "subtract": a-b, "multiply": a*b, "divide": a/b}
    return ops[operation]

calc_tool = StructuredTool.from_function(
    func=calculator,
    name="calculator",
    description="คำนวณ +/-/×/÷",
    args_schema=CalculatorInput,
)
```

## 10.3 สร้าง Agent

```python
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

tools = [search_web, get_weather, calc_tool]

prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณเป็นผู้ช่วยที่มีประโยชน์ ใช้ tools เมื่อจำเป็น"),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),  # ← จำเป็น สำหรับ agent
])

agent = create_tool_calling_agent(llm, tools, prompt)

executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,         # แสดง thought process
    max_iterations=5,     # กันวนลูปไม่สิ้นสุด
    handle_parsing_errors=True,
    return_intermediate_steps=True,   # เก็บ steps ไว้ด้วย
)

result = executor.invoke({"input": "อากาศกรุงเทพวันนี้เป็นยังไง?"})

print(result["output"])
print(result["intermediate_steps"])  # [(tool_call, result), ...]
```

**OUTPUT (verbose=True):**
```
> Entering new AgentExecutor chain...
Invoking: `get_weather` with {'city': 'Bangkok'}
{'city': 'Bangkok', 'temp': 35, 'unit': 'celsius'}
อากาศกรุงเทพวันนี้อุณหภูมิ 35°C ครับ ค่อนข้างร้อน

> Finished chain.
```

## 10.4 Tool Error Handling

```python
@tool
def divide(a: float, b: float) -> str:
    """หารตัวเลข a ด้วย b"""
    if b == 0:
        return "Error: หารด้วย 0 ไม่ได้"
    return str(a / b)
```

---

# PART 11 — RAG (Retrieval-Augmented Generation)

## Full RAG Pipeline

```python
from langchain_community.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

# Step 1: Load
loader = PyPDFLoader("knowledge.pdf")
docs = loader.load()

# Step 2: Split
splitter = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=50)
chunks = splitter.split_documents(docs)

# Step 3: Embed & Store
embeddings = OpenAIEmbeddings(openai_api_key="sk-...")
vectorstore = FAISS.from_documents(chunks, embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# Step 4: Prompt
prompt = ChatPromptTemplate.from_messages([
    ("system", """ตอบคำถามโดยใช้ข้อมูลที่ให้มาเท่านั้น
ถ้าไม่รู้ ให้บอกว่า 'ไม่มีข้อมูลในเอกสาร'

Context:
{context}"""),
    ("human", "{question}"),
])

# Step 5: Chain
def format_docs(docs):
    return "\n\n".join(d.page_content for d in docs)

rag_chain = (
    {
        "context": retriever | format_docs,
        "question": RunnablePassthrough(),
    }
    | prompt
    | ChatOpenAI(model="gpt-4o-mini", openai_api_key="sk-...")
    | StrOutputParser()
)

# ใช้งาน
answer = rag_chain.invoke("บริษัทก่อตั้งเมื่อไร?")
print(answer)
# OUTPUT: บริษัทก่อตั้งเมื่อปี 2550 ครับ ตามที่ระบุในรายงาน...
```

---

# PART 12 — Callbacks

```python
from langchain_core.callbacks import BaseCallbackHandler
from langchain_core.outputs import LLMResult
from typing import Any, Dict, List

class DetailedLogger(BaseCallbackHandler):

    def on_llm_start(self, serialized: Dict, prompts: List[str], **kwargs):
        print(f"\n🚀 [LLM START] model: {serialized.get('name')}")
        print(f"   prompt: {prompts[0][:80]}...")

    def on_llm_end(self, response: LLMResult, **kwargs):
        text = response.generations[0][0].text
        usage = response.llm_output.get("token_usage", {})
        print(f"✅ [LLM END] tokens: {usage}")
        print(f"   reply: {text[:80]}...")

    def on_llm_error(self, error: Exception, **kwargs):
        print(f"❌ [LLM ERROR] {error}")

    def on_chain_start(self, serialized: Dict, inputs: Dict, **kwargs):
        print(f"\n⛓️  [CHAIN START] {serialized.get('name')}")

    def on_chain_end(self, outputs: Dict, **kwargs):
        print(f"⛓️  [CHAIN END]")

    def on_tool_start(self, serialized: Dict, input_str: str, **kwargs):
        print(f"\n🔧 [TOOL START] {serialized.get('name')} | input: {input_str}")

    def on_tool_end(self, output: str, **kwargs):
        print(f"🔧 [TOOL END] output: {output[:80]}")

# ใช้ใน LLM
llm = ChatOpenAI(
    model="gpt-4o-mini",
    openai_api_key="sk-...",
    callbacks=[DetailedLogger()],
)

# ใช้ใน invoke เฉพาะครั้ง
result = chain.invoke(
    {"question": "สวัสดี"},
    config={"callbacks": [DetailedLogger()]},
)
```

---

# PART 13 — Caching

## 13.1 InMemoryCache

```python
from langchain_core.globals import set_llm_cache
from langchain_community.cache import InMemoryCache

set_llm_cache(InMemoryCache())

r1 = llm.invoke([HumanMessage(content="2+2?")])  # เรียก API
r2 = llm.invoke([HumanMessage(content="2+2?")])  # จาก cache ทันที
```

## 13.2 RedisCache

```python
import redis
from langchain_community.cache import RedisCache
from langchain_core.globals import set_llm_cache

r = redis.from_url("redis://localhost:6379")

set_llm_cache(RedisCache(
    redis_=r,
    ttl=3600,     # หมดอายุใน 1 ชั่วโมง (None = ไม่หมด)
))
```

## 13.3 SQLiteCache

```python
from langchain_community.cache import SQLiteCache
from langchain_core.globals import set_llm_cache

set_llm_cache(SQLiteCache(database_path="llm_cache.db"))
```

---

# PART 14 — Streaming

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

chain = ChatPromptTemplate.from_messages([
    ("human", "{input}")
]) | llm | StrOutputParser()

# Stream ทีละ token
print("AI: ", end="")
for chunk in chain.stream({"input": "เล่าเรื่องสั้นๆ ให้ฟัง"}):
    print(chunk, end="", flush=True)
print()

# Async Stream
import asyncio
async def astream():
    async for chunk in chain.astream({"input": "สวัสดี"}):
        print(chunk, end="", flush=True)

asyncio.run(astream())
```

---

# PART 15 — Runnable Config

```python
# ส่ง config พิเศษเข้าไปทุก invoke
result = chain.invoke(
    {"input": "สวัสดี"},
    config={
        "callbacks": [MyLogger()],         # callbacks เฉพาะครั้ง
        "tags": ["production", "user_a"],  # tag สำหรับ tracing
        "metadata": {"user_id": "123"},    # metadata
        "run_name": "my_run",              # ชื่อ run
        "max_concurrency": 5,             # สำหรับ batch
        "configurable": {                  # ค่า configurable
            "session_id": "abc",
        }
    }
)
```

---

# PART 16 — LangSmith (Tracing & Debug)

```python
import os

# เปิด tracing
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls-..."
os.environ["LANGCHAIN_PROJECT"] = "my-project"

# หลังจากนี้ทุก chain/agent run จะถูก log ที่ smith.langchain.com
result = chain.invoke({"input": "test"})
```

---

# สรุป Import ทั้งหมด (Cheat Sheet)

```python
# === MESSAGES ===
from langchain_core.messages import SystemMessage, HumanMessage, AIMessage, ToolMessage

# === LLM ===
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_anthropic import ChatAnthropic
from langchain_ollama import ChatOllama

# === PROMPTS ===
from langchain_core.prompts import (
    ChatPromptTemplate,
    PromptTemplate,
    MessagesPlaceholder,
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
)

# === OUTPUT PARSERS ===
from langchain_core.output_parsers import StrOutputParser, JsonOutputParser
from langchain_core.output_parsers import PydanticOutputParser
from langchain.output_parsers import CommaSeparatedListOutputParser

# === RUNNABLES (LCEL) ===
from langchain_core.runnables import (
    RunnablePassthrough,
    RunnableLambda,
    RunnableParallel,
)

# === MEMORY ===
from langchain_core.chat_history import InMemoryChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory
from langchain_community.chat_message_histories import SQLChatMessageHistory, RedisChatMessageHistory

# === DOCUMENT LOADERS ===
from langchain_community.document_loaders import (
    TextLoader, PyPDFLoader, CSVLoader, WebBaseLoader, DirectoryLoader
)
from langchain_core.documents import Document

# === TEXT SPLITTERS ===
from langchain_text_splitters import RecursiveCharacterTextSplitter, CharacterTextSplitter

# === VECTOR STORES ===
from langchain_community.vectorstores import FAISS, Chroma

# === TOOLS & AGENT ===
from langchain_core.tools import tool, StructuredTool
from langchain.agents import create_tool_calling_agent, AgentExecutor

# === CALLBACKS ===
from langchain_core.callbacks import BaseCallbackHandler

# === CACHE ===
from langchain_core.globals import set_llm_cache
from langchain_community.cache import InMemoryCache, RedisCache, SQLiteCache

# === GLOBALS ===
from langchain_core.globals import set_verbose, set_debug
set_verbose(True)   # แสดงรายละเอียด chain
set_debug(True)     # แสดงทุกอย่าง
```
