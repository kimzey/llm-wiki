# LangChain & LlamaIndex — Deep Dive Complete Guide

> อธิบายตั้งแต่ศูนย์ จนถึงระดับที่ใช้งาน Production ได้จริง

---

## ส่วนที่ 1: ภาพรวมก่อนลงลึก

### ปัญหาที่ทั้งสองแก้

ลองนึกภาพว่าคุณอยากสร้าง AI ที่ตอบคำถามจากเอกสารบริษัท  
ถ้าทำเองตั้งแต่ต้น คุณต้องเขียนโค้ดทุกอย่างนี้ด้วยตัวเอง:

```
1. โหลดไฟล์ PDF, Word, Notion, Markdown
2. แบ่งข้อความเป็น chunks เล็กๆ
3. แปลง chunk เป็น vector (embedding)
4. เก็บ vector ลง database
5. รับคำถามจาก user
6. แปลงคำถามเป็น vector
7. หา chunk ที่ใกล้เคียงที่สุด
8. สร้าง prompt ส่งให้ GPT
9. รับคำตอบกลับมา
10. จัดการ conversation history
11. เชื่อม tool อื่นๆ เช่น ค้นเว็บ, ดู calendar
```

ทำเองทั้งหมดนี้ใช้เวลาเป็นเดือน  
**LangChain และ LlamaIndex คือ library ที่ทำทุกอย่างข้างบนให้แล้ว**

---

### เปรียบเทียบภาพรวม

```
LangChain   = "Swiss Army Knife"
             เครื่องมือครบทุกอย่าง
             เน้น: Chain, Agent, Tool, Memory
             ถามว่า "จะให้ AI ทำอะไร?"

LlamaIndex  = "Specialized Search Engine Builder"
             เครื่องมือเฉพาะทางสำหรับ data + retrieval
             เน้น: Index, Query, Retrieval, RAG
             ถามว่า "จะหาข้อมูลจาก doc ยังไง?"
```

---

## ส่วนที่ 2: LangChain — อธิบายละเอียด

### 2.1 LangChain คืออะไร?

LangChain เป็น framework ที่ช่วยให้คุณ **"เชื่อมต่อ" สิ่งต่างๆ เข้ากับ LLM** ได้ง่าย

คิดแบบง่ายๆ: LangChain คือ "สายพาน" ที่เชื่อม:
- Input (คำถาม, ไฟล์, ข้อมูล)
- Processing (แปลง, ค้นหา, คำนวณ)
- LLM (GPT, Claude, Gemini ฯลฯ)
- Output (คำตอบ, JSON, action)

### 2.2 Building Blocks ของ LangChain

#### Block 1: LLM / Chat Model — ตัวสมอง

```python
# ติดตั้ง
# pip install langchain langchain-openai

from langchain_openai import ChatOpenAI, OpenAI

# ChatOpenAI = รุ่นใหม่ (แนะนำ) — รับ messages format
llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0,          # 0 = deterministic, 1 = creative
    max_tokens=1000,
    api_key="sk-..."
)

# เรียกใช้แบบง่ายที่สุด
response = llm.invoke("สวัสดี คุณคือใคร?")
print(response.content)
# → "สวัสดีครับ ผมคือ AI assistant..."

# เรียกแบบ async
response = await llm.ainvoke("สวัสดี")

# Streaming — ตอบทีละ token
for chunk in llm.stream("เล่าเรื่องสั้นให้ฟัง"):
    print(chunk.content, end="", flush=True)
```

#### Block 2: Prompt Template — แม่พิมพ์คำถาม

```python
from langchain_core.prompts import (
    ChatPromptTemplate,
    SystemMessagePromptTemplate,
    HumanMessagePromptTemplate,
    MessagesPlaceholder,
)

# Template ธรรมดา
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือ HR bot ของบริษัท {company_name} ตอบเป็นภาษาไทย"),
    ("human", "{question}"),
])

# ใส่ค่าแล้ว invoke
chain = prompt | llm
result = chain.invoke({
    "company_name": "Sellsuki",
    "question": "ลาพักร้อนกี่วัน?",
})
print(result.content)

# Template ที่มี chat history
prompt_with_history = ChatPromptTemplate.from_messages([
    ("system", "คุณคือ HR bot ตอบสั้นๆ ตรงประเด็น"),
    MessagesPlaceholder("history"),   # ← slot สำหรับ history
    ("human", "{question}"),
])
```

#### Block 3: Output Parser — แปลง output

```python
from langchain_core.output_parsers import (
    StrOutputParser,        # แปลงเป็น string ธรรมดา
    JsonOutputParser,       # แปลงเป็น JSON
    PydanticOutputParser,   # แปลงเป็น Pydantic model (type-safe)
)
from pydantic import BaseModel

# StrOutputParser — เอาแค่ข้อความ
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"question": "สวัสดี"})
print(type(result))   # str ← ไม่ใช่ AIMessage object แล้ว
print(result)         # "สวัสดีครับ..."

# PydanticOutputParser — ได้ structured data
class LeaveInfo(BaseModel):
    days_per_year: int
    conditions: str
    requires_approval: bool

parser = PydanticOutputParser(pydantic_object=LeaveInfo)

prompt = ChatPromptTemplate.from_messages([
    ("system", "ตอบเป็น JSON ตาม format นี้:\n{format_instructions}"),
    ("human", "{question}"),
]).partial(format_instructions=parser.get_format_instructions())

chain = prompt | llm | parser
result = chain.invoke({"question": "ลาพักร้อนเป็นยังไง?"})
print(result.days_per_year)       # 10
print(result.requires_approval)   # True
```

#### Block 4: Chain — เชื่อมทุกอย่างเข้าด้วยกัน (LCEL)

LCEL = LangChain Expression Language คือการใช้ `|` เชื่อม components

```python
# "|" เหมือน pipe ใน Linux/Unix
# output ของซ้าย → input ของขวา

from langchain_core.runnables import RunnablePassthrough, RunnableLambda

# Chain พื้นฐาน
simple_chain = prompt | llm | StrOutputParser()

# Chain ซับซ้อน — ทำหลายอย่างพร้อมกัน
from langchain_core.runnables import RunnableParallel

parallel_chain = RunnableParallel(
    answer=(prompt | llm | StrOutputParser()),
    question=RunnablePassthrough(),   # ส่ง input ผ่านไปเลย
)

result = parallel_chain.invoke({"question": "ลาพักร้อนกี่วัน?"})
print(result["answer"])    # คำตอบจาก LLM
print(result["question"])  # คำถามเดิม

# Chain ที่มี logic — ถ้า/ไม่ถ้า
def route(info):
    if "ลา" in info["question"]:
        return hr_chain
    elif "โค้ด" in info["question"] or "api" in info["question"].lower():
        return tech_chain
    else:
        return general_chain

from langchain_core.runnables import RunnableBranch
# หรือใช้ RunnableLambda
routed_chain = RunnableLambda(route)
```

#### Block 5: Memory — จำการสนทนา

```python
from langchain_core.chat_history import BaseChatMessageHistory
from langchain_community.chat_message_histories import RedisChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# เก็บ history ใน Redis (Dragonfly)
def get_session_history(session_id: str) -> BaseChatMessageHistory:
    return RedisChatMessageHistory(
        session_id=session_id,
        url="redis://dragonfly:6379",
        ttl=3600,  # หมดอายุใน 1 ชั่วโมง
    )

# ห่อ chain ด้วย message history
chain_with_history = RunnableWithMessageHistory(
    chain,
    get_session_history,
    input_messages_key="question",
    history_messages_key="history",
)

# ใช้งาน — session_id แยก conversation ของแต่ละ user
result1 = chain_with_history.invoke(
    {"question": "ลาพักร้อนได้กี่วัน?"},
    config={"configurable": {"session_id": "user_123"}}
)

result2 = chain_with_history.invoke(
    {"question": "แล้วลาป่วยล่ะ?"},  # ← รู้ว่า context คือเรื่องลา
    config={"configurable": {"session_id": "user_123"}}
)
```

#### Block 6: Document Loaders — โหลดข้อมูลจากทุกที่

```python
# LangChain มี loader เยอะมาก
from langchain_community.document_loaders import (
    PyPDFLoader,          # PDF
    TextLoader,           # .txt
    CSVLoader,            # CSV
    UnstructuredMarkdownLoader,  # Markdown
    NotionDBLoader,       # Notion
    SlackDirectoryLoader, # Slack export
    GitLoader,            # Git repo
    WebBaseLoader,        # Web page
)

# โหลด PDF
loader = PyPDFLoader("hr_policy.pdf")
docs = loader.load()
# docs = list of Document objects
# Document มี: page_content (str), metadata (dict)
print(docs[0].page_content)   # เนื้อหาหน้าแรก
print(docs[0].metadata)       # {'source': 'hr_policy.pdf', 'page': 0}

# โหลด Web page
loader = WebBaseLoader("https://docs.sellsuki.com/hr")
docs = loader.load()

# โหลดหลายไฟล์พร้อมกัน
from langchain_community.document_loaders import DirectoryLoader
loader = DirectoryLoader("./docs/", glob="**/*.md")
docs = loader.load()  # โหลดทุก .md ใน folder
```

#### Block 7: Text Splitter — แบ่ง chunk

```python
from langchain_text_splitters import (
    RecursiveCharacterTextSplitter,  # แนะนำ — ใช้บ่อยที่สุด
    CharacterTextSplitter,
    MarkdownHeaderTextSplitter,      # แบ่งตาม # ## ###
    TokenTextSplitter,               # แบ่งตาม token จริงๆ
)

splitter = RecursiveCharacterTextSplitter(
    chunk_size=512,        # ขนาด chunk (characters)
    chunk_overlap=64,      # overlap ระหว่าง chunk (retain context)
    separators=["\n\n", "\n", ".", " ", ""],  # ลำดับการตัด
)

chunks = splitter.split_documents(docs)
print(len(chunks))          # จำนวน chunks
print(chunks[0].page_content)  # เนื้อหา chunk แรก

# MarkdownHeaderSplitter — เก็บ header เป็น metadata
splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=[
        ("#", "h1"),
        ("##", "h2"),
        ("###", "h3"),
    ]
)
chunks = splitter.split_text(markdown_content)
# chunks[0].metadata = {"h1": "HR Policy", "h2": "วันลา"}
```

#### Block 8: Embeddings & Vector Store

```python
from langchain_openai import OpenAIEmbeddings
from langchain_community.vectorstores import PGVector

embedding = OpenAIEmbeddings(model="text-embedding-3-small")

# ทดสอบ embedding
vector = embedding.embed_query("ลาพักร้อนกี่วัน?")
print(len(vector))   # 1536 — มิติของ vector

# เก็บลง PGVector (ParadeDB)
CONNECTION_STRING = "postgresql://user:pass@paradedb:5432/rag"
vectorstore = PGVector.from_documents(
    documents=chunks,
    embedding=embedding,
    connection_string=CONNECTION_STRING,
    collection_name="sellsuki_docs",
    pre_delete_collection=False,  # ไม่ลบ collection เดิม
)

# ค้นหา
results = vectorstore.similarity_search("ลาพักร้อน", k=5)
# results = list of Document (most similar)

# ค้นหาพร้อม score
results = vectorstore.similarity_search_with_score("ลาพักร้อน", k=5)
for doc, score in results:
    print(f"Score: {score:.3f} | {doc.page_content[:100]}")
```

#### Block 9: Retrieval Chain — RAG ครบวงจร

```python
from langchain.chains import RetrievalQA, ConversationalRetrievalChain
from langchain_core.prompts import ChatPromptTemplate

# แบบง่าย
qa_chain = RetrievalQA.from_chain_type(
    llm=ChatOpenAI(model="gpt-4o"),
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    return_source_documents=True,
)
result = qa_chain.invoke({"query": "ลาพักร้อนกี่วัน?"})
print(result["result"])             # คำตอบ
print(result["source_documents"])   # เอกสารที่ใช้อ้างอิง

# แบบ LCEL (ทันสมัยกว่า, แนะนำ)
from langchain_core.runnables import RunnablePassthrough
from langchain_core.output_parsers import StrOutputParser

prompt = ChatPromptTemplate.from_template("""
ตอบคำถามจาก context ที่ให้มาเท่านั้น ถ้าไม่มีข้อมูลให้บอกว่าไม่ทราบ

Context:
{context}

คำถาม: {question}

คำตอบ:
""")

def format_docs(docs):
    return "\n\n---\n\n".join([d.page_content for d in docs])

rag_chain = (
    {
        "context": vectorstore.as_retriever() | format_docs,
        "question": RunnablePassthrough(),
    }
    | prompt
    | llm
    | StrOutputParser()
)

answer = rag_chain.invoke("ลาพักร้อนกี่วัน?")
print(answer)
```

#### Block 10: Tools & Agent — AI ที่ทำงานได้เอง

```python
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain.tools import tool
from langchain_community.tools import DuckDuckGoSearchRun, WikipediaQueryRun

# สร้าง custom tool
@tool
def get_leave_balance(employee_id: str) -> str:
    """ดูวันลาคงเหลือของพนักงาน"""
    # เรียก HR system API
    balance = hr_api.get_balance(employee_id)
    return f"พนักงาน {employee_id} มีวันลาคงเหลือ {balance} วัน"

@tool
def search_company_docs(query: str) -> str:
    """ค้นหาข้อมูลจากเอกสารบริษัท"""
    results = vectorstore.similarity_search(query, k=3)
    return "\n".join([r.page_content for r in results])

@tool
def create_leave_request(
    employee_id: str, 
    start_date: str, 
    end_date: str, 
    reason: str
) -> str:
    """สร้างคำขอลาใหม่"""
    result = hr_api.create_leave(employee_id, start_date, end_date, reason)
    return f"สร้างคำขอลาสำเร็จ ID: {result['id']}"

tools = [get_leave_balance, search_company_docs, create_leave_request]

# Agent — ตัดสินใจเองว่าจะใช้ tool ไหน เมื่อไหร่
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือ HR assistant ของ Sellsuki ช่วยพนักงานเรื่องการลา"),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),  # ← agent ใช้คิดเอง
])

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(
    agent=agent, 
    tools=tools, 
    verbose=True,     # แสดง thought process
    max_iterations=5, # กันวนลูปไม่สิ้นสุด
)

# Agent ทำงาน
result = agent_executor.invoke({
    "input": "ฉัน ID E001 อยากลาพักร้อน 3 วัน วันที่ 20-22 มีนา ได้ไหม?",
    "chat_history": [],
})

# Agent จะ:
# 1. คิดว่าต้องดูวันลาคงเหลือก่อน
# 2. เรียก get_leave_balance("E001") → "คงเหลือ 8 วัน"
# 3. คิดว่า 3 < 8 ลาได้
# 4. เรียก create_leave_request(...)
# 5. ตอบ user ว่า "สร้างคำขอลาเรียบร้อยแล้วครับ"
```

---

## ส่วนที่ 3: LlamaIndex — อธิบายละเอียด

### 3.1 LlamaIndex คืออะไร?

LlamaIndex คือ framework ที่เชี่ยวชาญเรื่อง **"เชื่อมข้อมูลของคุณกับ LLM"**  
ถ้า LangChain คือ "สิ่งที่ AI ทำได้", LlamaIndex คือ "ข้อมูลที่ AI รู้"

ปัญหาที่ LlamaIndex แก้:
- เอกสารมีหลายรูปแบบ ทำยังไงให้ LLM อ่านได้ทั้งหมด?
- ข้อมูลเยอะมาก ทำยังไงให้ค้นหาได้แม่นยำ?
- เอกสารอัพเดตบ่อย ทำยังไงไม่ให้ต้อง re-index ทั้งหมด?
- ต้องการ filter ตาม permission ทำยังไง?

### 3.2 Mental Model ของ LlamaIndex

```
ขั้นตอนที่ 1: INGESTION (กินข้อมูลเข้า)
  Documents → Nodes → Embeddings → Index (Vector Store)
  
ขั้นตอนที่ 2: QUERYING (ตอบคำถาม)
  Question → Retrieve Nodes → Synthesize → Answer
```

### 3.3 Building Blocks ของ LlamaIndex

#### Block 1: Document & Node — หน่วยข้อมูล

```python
from llama_index.core import Document
from llama_index.core.schema import TextNode, ImageNode

# Document = เอกสารทั้งชิ้น (ก่อน chunk)
doc = Document(
    text="พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน 10 วัน ลาป่วย 30 วัน...",
    metadata={
        "source": "hr_policy_2025.md",
        "department": "hr",
        "doc_type": "policy",
        "access_level": "all",
        "version": "2025.1",
    },
    id_="hr-policy-2025",           # unique ID สำหรับ update/delete
    
    # ควบคุม metadata ที่ส่งให้ LLM
    excluded_llm_metadata_keys=["internal_id", "raw_path"],
    # ควบคุม metadata ที่ใช้ embed
    excluded_embed_metadata_keys=["version"],
    
    # format ที่ LLM จะเห็น
    metadata_template="[{key}]: {value}",
    text_template="Metadata:\n{metadata_str}\n\nContent:\n{content}",
)

# Node = chunk ที่แตกจาก Document
node = TextNode(
    text="พนักงานมีสิทธิ์ลาพักร้อน 10 วันต่อปี",
    metadata={"source": "hr_policy.md", "section": "วันลา"},
    id_="node-001",
)
print(node.get_content())         # เนื้อหา node
print(node.get_metadata_str())    # metadata เป็น string
```

#### Block 2: SimpleDirectoryReader — โหลดไฟล์

```python
from llama_index.core import SimpleDirectoryReader

# โหลดทุกไฟล์ใน folder
reader = SimpleDirectoryReader(
    input_dir="./docs/",
    recursive=True,                  # รวม subfolder
    required_exts=[".pdf", ".md", ".txt"],  # filter นามสกุล
    filename_as_id=True,             # ใช้ชื่อไฟล์เป็น doc ID
)
docs = reader.load_data()

# โหลดไฟล์เดียว
docs = SimpleDirectoryReader(input_files=["hr.pdf"]).load_data()

# ทุก doc จะมี metadata อัตโนมัติ
print(docs[0].metadata)
# {'file_path': '/docs/hr.pdf', 'file_name': 'hr.pdf', 
#  'file_type': 'application/pdf', 'creation_date': '2025-01-01'}
```

#### Block 3: NodeParser — Chunking Strategy

```python
from llama_index.core.node_parser import (
    SentenceSplitter,
    SemanticSplitterNodeParser,
    HierarchicalNodeParser,
    MarkdownNodeParser,
    CodeSplitter,
)
from llama_index.embeddings.openai import OpenAIEmbedding

# 1. SentenceSplitter — แบ่งตาม sentence (พื้นฐาน ดี)
splitter = SentenceSplitter(
    chunk_size=512,
    chunk_overlap=64,
)
nodes = splitter.get_nodes_from_documents(docs)

# 2. SemanticSplitter — แบ่งตาม semantic (ฉลาดกว่า แต่ช้ากว่า)
#    วัด embedding similarity ระหว่างประโยค
#    ถ้า similarity ลดฮวบ = topic เปลี่ยน = ตัด chunk
embed_model = OpenAIEmbedding()
semantic_splitter = SemanticSplitterNodeParser(
    buffer_size=1,           # มองข้างๆ กี่ประโยค
    breakpoint_percentile_threshold=95,  # จุดตัด
    embed_model=embed_model,
)
nodes = semantic_splitter.get_nodes_from_documents(docs)

# 3. HierarchicalNodeParser — สร้าง parent-child (แนะนำมาก)
#    ใช้หา chunk เล็กๆ (ละเอียด) แต่ส่ง parent (context ใหญ่) ให้ LLM
hierarchical = HierarchicalNodeParser.from_defaults(
    chunk_sizes=[2048, 512, 128],
)
all_nodes = hierarchical.get_nodes_from_documents(docs)

# แยก leaf (เล็กสุด) ออกมา → ใช้ index
from llama_index.core.node_parser import get_leaf_nodes, get_root_nodes
leaf_nodes = get_leaf_nodes(all_nodes)    # 128 tokens — ค้นหา
root_nodes = get_root_nodes(all_nodes)   # 2048 tokens — ส่งให้ LLM

# 4. MarkdownNodeParser — เข้าใจ Markdown structure
md_parser = MarkdownNodeParser()
nodes = md_parser.get_nodes_from_documents(docs)
# nodes จะมี metadata: {'header_path': '# HR Policy > ## วันลา > ### ลาพักร้อน'}
```

#### Block 4: IngestionPipeline — Pipeline ครบวงจร

IngestionPipeline คือหัวใจของ LlamaIndex มันรวมทุก step เข้าด้วยกัน + **cache อัตโนมัติ**

```python
from llama_index.core.ingestion import IngestionPipeline, IngestionCache
from llama_index.core.node_parser import SentenceSplitter
from llama_index.core.extractors import (
    TitleExtractor,              # ดึงชื่อเรื่องอัตโนมัติ
    QuestionsAnsweredExtractor,  # สร้าง Q&A จาก chunk อัตโนมัติ
    SummaryExtractor,            # สรุป chunk อัตโนมัติ
    KeywordExtractor,            # ดึง keyword
)
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.storage.kvstore.redis import RedisKVStore

# Setup stores
vector_store = PGVectorStore.from_params(
    host="paradedb", port=5432,
    database="sellsuki_rag",
    table_name="documents",
    embed_dim=1536,
)

# Cache ใน Dragonfly (Redis)
cache = IngestionCache(
    cache=RedisKVStore.from_host_and_port("dragonfly", 6379),
    collection="ingestion_cache",
)

# Pipeline
pipeline = IngestionPipeline(
    transformations=[
        # Step 1: Chunk
        SentenceSplitter(chunk_size=512, chunk_overlap=64),
        
        # Step 2: Extract metadata เพิ่ม (LLM-based — ช้าแต่ดี)
        TitleExtractor(nodes=5),    # ดูรอบๆ 5 nodes หาชื่อเรื่อง
        QuestionsAnsweredExtractor(questions=3),  # สร้าง 3 คำถาม/chunk
        
        # Step 3: Embed
        OpenAIEmbedding(model="text-embedding-3-small"),
    ],
    vector_store=vector_store,
    cache=cache,  # ← ถ้า chunk เดิม ไม่ embed ซ้ำ
)

# รัน pipeline
nodes = pipeline.run(documents=docs)
# ครั้งที่ 1: embed ทุก chunk = เสียเงิน
# ครั้งที่ 2 (ไม่เปลี่ยน): ไม่ embed = ประหยัด

# async version (แนะนำสำหรับ production)
nodes = await pipeline.arun(documents=docs, num_workers=4)
```

#### Block 5: Index — สร้าง Search Engine

```python
from llama_index.core import VectorStoreIndex, StorageContext

# VectorStoreIndex — index หลัก
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# สร้างจาก documents ใหม่
index = VectorStoreIndex.from_documents(
    docs,
    storage_context=storage_context,
    show_progress=True,
)

# หรือ load จาก vector store ที่มีข้อมูลแล้ว (production)
index = VectorStoreIndex.from_vector_store(
    vector_store,
    storage_context=storage_context,
)

# เพิ่ม document ใหม่ทีหลัง
index.insert(new_doc)
index.insert_nodes(new_nodes)

# ลบ document
index.delete_ref_doc("doc-id-to-delete", delete_from_docstore=True)
```

#### Block 6: Retriever — ดึงข้อมูล

```python
from llama_index.core.retrievers import (
    VectorIndexRetriever,
    BM25Retriever,
)
from llama_index.core.postprocessor import (
    SimilarityPostprocessor,   # กรอง score ต่ำออก
    MetadataReplacementPostProcessor,  # แทน leaf ด้วย parent
    SentenceTransformerRerank, # rerank ด้วย cross-encoder
)

# Vector Retriever
retriever = VectorIndexRetriever(
    index=index,
    similarity_top_k=10,  # ดึงมา 10 candidates
)

# Retrieval พร้อม metadata filter
from llama_index.core.vector_stores import (
    MetadataFilters, MetadataFilter, FilterOperator, FilterCondition
)

filters = MetadataFilters(
    filters=[
        MetadataFilter(key="department", value="hr"),
        MetadataFilter(key="access_level", value="all"),
    ],
    condition=FilterCondition.AND,  # ต้องผ่านทั้งคู่
)

retriever = VectorIndexRetriever(
    index=index,
    similarity_top_k=10,
    filters=filters,
)

nodes = retriever.retrieve("ลาพักร้อนกี่วัน?")
for node in nodes:
    print(f"Score: {node.score:.3f}")
    print(f"Text: {node.text[:100]}")
    print(f"Source: {node.metadata.get('source')}")

# Reranker — เรียงลำดับใหม่ให้แม่นยำขึ้น
reranker = SentenceTransformerRerank(
    model="cross-encoder/ms-marco-MiniLM-L-6-v2",
    top_n=3,  # เหลือ 3 ที่ดีที่สุด
)

reranked_nodes = reranker.postprocess_nodes(
    nodes, query_str="ลาพักร้อนกี่วัน?"
)
```

#### Block 7: QueryEngine — ตอบคำถาม

```python
from llama_index.core.query_engine import (
    RetrieverQueryEngine,
    SubQuestionQueryEngine,
    RouterQueryEngine,
    TransformQueryEngine,
)
from llama_index.core.response_synthesizers import (
    get_response_synthesizer,
    ResponseMode,
)

# Query Engine พื้นฐาน
query_engine = index.as_query_engine(
    similarity_top_k=5,
    response_mode="compact",       # compact / tree_summarize / refine
    streaming=True,
)

response = query_engine.query("ลาพักร้อนได้กี่วัน?")
print(response.response)
print(response.source_nodes)       # nodes ที่ใช้ตอบ
print(response.metadata)

# Streaming
for token in response.response_gen:
    print(token, end="", flush=True)

# Custom QueryEngine ละเอียด
synthesizer = get_response_synthesizer(
    response_mode=ResponseMode.TREE_SUMMARIZE,
    streaming=False,
)

query_engine = RetrieverQueryEngine(
    retriever=retriever,
    response_synthesizer=synthesizer,
    node_postprocessors=[reranker],
)

# SubQuestion Engine — แตกคำถามซับซ้อน
from llama_index.core.tools import QueryEngineTool

hr_tool = QueryEngineTool.from_defaults(
    query_engine=hr_query_engine,
    name="hr_docs",
    description="ข้อมูล HR: วันลา, สวัสดิการ, กฎระเบียบ",
)
tech_tool = QueryEngineTool.from_defaults(
    query_engine=tech_query_engine,
    name="tech_docs",
    description="เอกสาร Technical: API, วิธีใช้ระบบ, code",
)

sub_engine = SubQuestionQueryEngine.from_defaults(
    query_engine_tools=[hr_tool, tech_tool],
    verbose=True,
)

# คำถามซับซ้อน → แตกออกเป็น sub-questions อัตโนมัติ
response = sub_engine.query(
    "ลาพักร้อนได้กี่วัน และ API สร้าง order ต้องเรียก endpoint ไหน?"
)
# LlamaIndex จะแตกเป็น:
# Sub Q1: ลาพักร้อนได้กี่วัน? → ส่งไป hr_docs
# Sub Q2: API สร้าง order ต้องเรียก endpoint ไหน? → ส่งไป tech_docs
# รวม answer ทั้งสองกลับมา
```

#### Block 8: Chat Engine — สนทนาต่อเนื่อง

```python
from llama_index.core.chat_engine import (
    SimpleChatEngine,           # แค่ memory, ไม่มี RAG
    CondenseQuestionChatEngine, # ทำ RAG + memory (แนะนำ)
    ContextChatEngine,          # ทำ RAG ทุก turn
)

# CondenseQuestion — ฉลาดที่สุด
# ก่อน query จะ "condense" คำถามให้ standalone
# เช่น "แล้วลาป่วยล่ะ?" → condense เป็น "พนักงาน Sellsuki ลาป่วยได้กี่วัน?"
chat_engine = index.as_chat_engine(
    chat_mode="condense_question",
    verbose=True,
    streaming=True,
)

# สนทนา
r1 = chat_engine.chat("ลาพักร้อนได้กี่วัน?")
print(r1.response)  # "10 วันต่อปีครับ"

r2 = chat_engine.chat("แล้วต้องแจ้งล่วงหน้ากี่วัน?")
# รู้ว่า context = เรื่องลาพักร้อน
print(r2.response)  # "ต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการครับ"

# reset history
chat_engine.reset()
```

#### Block 9: LlamaIndex Workflows — ทันสมัยที่สุด

```python
# Workflows = วิธีใหม่ (>= 0.10.30) ที่แนะนำมากที่สุด
# Event-driven, async, ง่าย debug, type-safe

from llama_index.core.workflow import (
    Workflow, StartEvent, StopEvent, step, Event, Context
)

# Define events (ข้อมูลที่ส่งระหว่าง step)
class QueryReceivedEvent(Event):
    query: str
    user_id: str
    role: str

class NodesRetrievedEvent(Event):
    query: str
    nodes: list

class AnswerGeneratedEvent(Event):
    answer: str
    sources: list

# Workflow
class SellsukiRAGWorkflow(Workflow):
    
    def __init__(self, index, cache, **kwargs):
        super().__init__(**kwargs)
        self.index = index
        self.cache = cache
    
    @step
    async def check_semantic_cache(
        self, ctx: Context, ev: StartEvent
    ) -> QueryReceivedEvent | StopEvent:
        """ดู cache ก่อน ถ้ามีตอบจาก cache เลย"""
        cached = await self.cache.get(ev.query)
        if cached:
            return StopEvent(result=cached)  # ← จบที่นี่
        return QueryReceivedEvent(
            query=ev.query, 
            user_id=ev.user_id,
            role=ev.role
        )
    
    @step
    async def retrieve_nodes(
        self, ctx: Context, ev: QueryReceivedEvent
    ) -> NodesRetrievedEvent:
        """ดึง nodes จาก index ตาม permission"""
        # สร้าง filter ตาม role
        filters = build_filters_for_role(ev.role)
        retriever = self.index.as_retriever(
            similarity_top_k=10,
            filters=filters,
        )
        nodes = await retriever.aretrieve(ev.query)
        # save query ไว้ใช้ step ถัดไป
        await ctx.set("query", ev.query)
        return NodesRetrievedEvent(query=ev.query, nodes=nodes)
    
    @step
    async def rerank_nodes(
        self, ctx: Context, ev: NodesRetrievedEvent
    ) -> NodesRetrievedEvent:
        """rerank ให้แม่นขึ้น"""
        reranked = reranker.postprocess_nodes(
            ev.nodes, query_str=ev.query
        )
        return NodesRetrievedEvent(query=ev.query, nodes=reranked[:3])
    
    @step
    async def generate_answer(
        self, ctx: Context, ev: NodesRetrievedEvent
    ) -> StopEvent:
        """สร้างคำตอบ"""
        context = "\n\n".join([n.text for n in ev.nodes])
        sources = [n.metadata.get("source") for n in ev.nodes]
        
        response = await llm.acomplete(
            f"Context:\n{context}\n\nQuestion: {ev.query}\nAnswer:"
        )
        
        result = {
            "answer": str(response),
            "sources": sources,
        }
        
        # เก็บ cache
        query = await ctx.get("query")
        await self.cache.set(query, result)
        
        return StopEvent(result=result)

# ใช้งาน
workflow = SellsukiRAGWorkflow(
    index=index, 
    cache=semantic_cache,
    timeout=30,
    verbose=True,
)

result = await workflow.run(
    query="ลาพักร้อนกี่วัน?",
    user_id="E001",
    role="employee",
)
print(result["answer"])
print(result["sources"])
```

---

## ส่วนที่ 4: LangChain vs LlamaIndex เปรียบเทียบ Concept ต่อ Concept

```
สิ่งที่ทำ              LangChain                    LlamaIndex
──────────────────────────────────────────────────────────────
โหลด document         DocumentLoader               SimpleDirectoryReader
                                                    Custom Reader
แบ่ง chunk            TextSplitter                 NodeParser
หน่วยข้อมูล          Document                     Document → Node
สร้าง embedding       Embeddings class             EmbedModel
เก็บ vector           VectorStore                  VectorStore (ใช้ร่วมกันได้)
ค้นหา                 Retriever                    Retriever
ประมวลผล             Chain (LCEL)                  QueryEngine / Workflow
สนทนา               ConversationChain             ChatEngine
จำประวัติ            Memory / MessageHistory       ChatMemoryBuffer
ทำงานอัตโนมัติ        Agent + Tools                Agent + Tools
cache pipeline        ไม่มี built-in               IngestionCache ✅
metadata filter       ✅                            ✅✅ (ละเอียดกว่า)
evaluate quality      LangSmith (แยก)              LlamaIndex Evaluators
```

---

## ส่วนที่ 5: ใช้ร่วมกัน (LlamaIndex + LangChain)

### ทำไมถึงใช้ร่วมกัน?

```
LlamaIndex เก่ง: Index, Retrieval, RAG pipeline, Metadata
LangChain เก่ง:  Agent, Tools, Memory, ทำงานหลายขั้นตอน

ใช้คู่กัน = ดึงจุดแข็งทั้งสอง
```

### วิธีที่ 1: LlamaIndex เป็น Tool ของ LangChain Agent

```python
# LlamaIndex สร้าง RAG ที่ดี
# LangChain Agent ใช้ RAG นั้นเป็น tool

from llama_index.core import VectorStoreIndex
from llama_index.core.query_engine import RetrieverQueryEngine
from langchain.tools import Tool
from langchain.agents import AgentExecutor, create_tool_calling_agent

# --- LlamaIndex Part ---
# สร้าง query engine คุณภาพสูง
query_engine = index.as_query_engine(
    similarity_top_k=5,
    node_postprocessors=[reranker],
    response_mode="tree_summarize",
)

# --- Bridge: แปลง LlamaIndex engine เป็น LangChain Tool ---
def query_company_docs(question: str) -> str:
    """ฟังก์ชันที่เชื่อม LlamaIndex กับ LangChain"""
    response = query_engine.query(question)
    sources = [n.metadata.get("source", "") for n in response.source_nodes]
    return f"{response.response}\n\nแหล่งที่มา: {', '.join(set(sources))}"

company_docs_tool = Tool(
    name="company_docs_search",
    func=query_company_docs,
    description="""ใช้ค้นหาข้อมูลจากเอกสารบริษัท Sellsuki
    ใช้เมื่อถามเกี่ยวกับ: HR policy, วันลา, สวัสดิการ, วิธีใช้ระบบ, business rules
    Input: คำถามเป็น string
    Output: คำตอบพร้อมแหล่งที่มา"""
)

# --- LangChain Part ---
# สร้าง tools อื่นๆ เพิ่ม
@tool
def check_hr_system(employee_id: str) -> str:
    """ดูข้อมูล realtime จาก HR system เช่น วันลาคงเหลือ"""
    return hr_api.get_employee_info(employee_id)

@tool
def send_slack_notification(user_id: str, message: str) -> str:
    """ส่ง notification ไปหาพนักงานผ่าน Slack"""
    slack_api.send_message(user_id, message)
    return "ส่ง notification สำเร็จ"

# Agent รวม tools ทั้งหมด
all_tools = [company_docs_tool, check_hr_system, send_slack_notification]

prompt = ChatPromptTemplate.from_messages([
    ("system", """คุณคือ HR assistant อัจฉริยะของ Sellsuki
    มีความสามารถ:
    1. ค้นหาข้อมูลจากเอกสาร company (company_docs_search)
    2. ดูข้อมูล realtime จาก HR system (check_hr_system)
    3. ส่ง notification (send_slack_notification)
    
    ตอบเป็นภาษาไทย สุภาพ กระชับ"""),
    MessagesPlaceholder("chat_history"),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

agent = create_tool_calling_agent(
    llm=ChatOpenAI(model="gpt-4o"),
    tools=all_tools,
    prompt=prompt,
)

agent_executor = AgentExecutor(
    agent=agent,
    tools=all_tools,
    verbose=True,
    max_iterations=5,
    return_intermediate_steps=True,  # ดู reasoning
)

# ทดสอบ
result = agent_executor.invoke({
    "input": "ผม ID E001 มีวันลาเหลือกี่วัน และนโยบายลาพักร้อนเป็นยังไง?",
    "chat_history": []
})

print(result["output"])
# Agent จะ:
# 1. เรียก check_hr_system("E001") → "วันลาคงเหลือ 7 วัน"
# 2. เรียก company_docs_search("นโยบายลาพักร้อน") → "10 วัน/ปี ต้องแจ้งล่วงหน้า 3 วัน"
# 3. สรุปคำตอบรวม
```

### วิธีที่ 2: LlamaIndex query engine ใน LangChain LCEL Chain

```python
# ใช้ LlamaIndex retriever ใน LangChain chain โดยตรง
from langchain_core.runnables import RunnableLambda

# แปลง LlamaIndex retriever เป็น LangChain-compatible
def llamaindex_retrieve(query: str) -> str:
    nodes = index.as_retriever(similarity_top_k=5).retrieve(query)
    return "\n\n".join([
        f"[{n.metadata.get('source')}]\n{n.text}" 
        for n in nodes
    ])

retriever_runnable = RunnableLambda(llamaindex_retrieve)

# ใส่ใน LangChain LCEL chain
rag_chain = (
    {
        "context": retriever_runnable,    # ← LlamaIndex ทำงาน
        "question": RunnablePassthrough(),
    }
    | ChatPromptTemplate.from_template(
        "Context:\n{context}\n\nคำถาม: {question}\nตอบ:"
    )
    | ChatOpenAI(model="gpt-4o")          # ← LangChain ทำงาน
    | StrOutputParser()
)

answer = rag_chain.invoke("ลาพักร้อนกี่วัน?")
```

### วิธีที่ 3: ใช้ Vector Store ร่วมกัน

```python
# ทั้งสองใช้ PGVector ร่วมกันได้ (แชร์ database เดียวกัน)

# LlamaIndex เขียนข้อมูล (Ingestion)
from llama_index.vector_stores.postgres import PGVectorStore
llama_vector_store = PGVectorStore.from_params(
    host="paradedb", port=5432,
    database="sellsuki_rag", table_name="documents",
    embed_dim=1536,
)
pipeline.run(documents=docs)  # เขียนเข้า ParadeDB

# LangChain อ่านข้อมูล (Query)
from langchain_community.vectorstores import PGVector
lc_vectorstore = PGVector(
    connection_string="postgresql://user:pass@paradedb:5432/sellsuki_rag",
    embedding_function=OpenAIEmbeddings(),
    collection_name="documents",  # ← ชื่อเดียวกัน!
)
results = lc_vectorstore.similarity_search("ลาพักร้อน")
```

---

## ส่วนที่ 6: ตัวอย่าง Real Use Case ครบวงจร

### Use Case: HR Bot สำหรับ Sellsuki

```python
# === FULL IMPLEMENTATION ===
# file: sellsuki_hr_bot.py

import asyncio
from fastapi import FastAPI, WebSocket
from fastapi.responses import StreamingResponse
from pydantic import BaseModel

# --- LlamaIndex: Data Layer ---
from llama_index.core import VectorStoreIndex, Settings
from llama_index.core.ingestion import IngestionPipeline, IngestionCache
from llama_index.core.node_parser import SentenceSplitter
from llama_index.core.postprocessor import SentenceTransformerRerank
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.embeddings.openai import OpenAIEmbedding
from llama_index.llms.openai import OpenAI as LlamaOpenAI
from llama_index.storage.kvstore.redis import RedisKVStore

# --- LangChain: Application Layer ---
from langchain_openai import ChatOpenAI
from langchain.agents import AgentExecutor, create_tool_calling_agent
from langchain.tools import tool
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder
from langchain_community.chat_message_histories import RedisChatMessageHistory
from langchain_core.runnables.history import RunnableWithMessageHistory

# === SETUP ===
Settings.embed_model = OpenAIEmbedding(model="text-embedding-3-small")
Settings.llm = LlamaOpenAI(model="gpt-4o")

vector_store = PGVectorStore.from_params(
    host="paradedb", port=5432, database="rag",
    table_name="docs", embed_dim=1536,
)

index = VectorStoreIndex.from_vector_store(vector_store)
reranker = SentenceTransformerRerank(model="cross-encoder/ms-marco-MiniLM-L-6-v2", top_n=3)

# === LLAMAINDEX: สร้าง Query Engine คุณภาพสูง ===
def get_query_engine(role: str):
    from llama_index.core.vector_stores import MetadataFilters, MetadataFilter
    
    access_map = {
        "employee": ["all"],
        "hr": ["all", "hr", "confidential_hr"],
        "engineer": ["all", "engineering"],
    }
    allowed = access_map.get(role, ["all"])
    
    return index.as_query_engine(
        similarity_top_k=10,
        node_postprocessors=[reranker],
        response_mode="tree_summarize",
        filters=MetadataFilters(filters=[
            MetadataFilter(key="access_level", value=allowed[0]),
        ]),
    )

# === LANGCHAIN: สร้าง Tools ===
@tool
def search_company_policy(question: str) -> str:
    """ค้นหานโยบาย กฎ และขั้นตอนต่างๆ ของบริษัท"""
    engine = get_query_engine("employee")
    response = engine.query(question)
    sources = list({n.metadata.get("source", "") for n in response.source_nodes})
    return f"คำตอบ: {response.response}\n\nอ้างอิงจาก: {', '.join(sources)}"

@tool
def get_employee_leave_balance(employee_id: str) -> str:
    """ดูจำนวนวันลาคงเหลือของพนักงาน input: employee_id"""
    # Mock HR API
    return f"พนักงาน {employee_id}: ลาพักร้อนคงเหลือ 7 วัน, ลาป่วยคงเหลือ 25 วัน"

@tool 
def submit_leave_request(
    employee_id: str,
    leave_type: str, 
    start_date: str,
    end_date: str
) -> str:
    """ยื่นคำขอลา input: employee_id, leave_type (ลาพักร้อน/ลาป่วย), start_date, end_date"""
    return f"ยื่นคำขอลาสำเร็จ! รหัส LV-{hash(employee_id)%9999:04d} สถานะ: รอ HR อนุมัติ"

# === LANGCHAIN: Agent ===
agent_prompt = ChatPromptTemplate.from_messages([
    ("system", """คุณคือ HR Bot ของ Sellsuki ชื่อ "เซล"
    
ความสามารถ:
- ตอบคำถามเกี่ยวกับนโยบาย HR จากเอกสารจริง (search_company_policy)
- ดูวันลาคงเหลือ (get_employee_leave_balance) 
- ยื่นขอลา (submit_leave_request)

กฎการตอบ:
- ตอบภาษาไทยเสมอ
- ถ้าไม่มีข้อมูลในเอกสาร บอกตรงๆ ว่าไม่ทราบ อย่าเดา
- ระบุแหล่งที่มาเสมอเมื่อตอบจาก policy
- ถ้าต้องการ employee_id ให้ถามก่อน"""),
    MessagesPlaceholder("history"),
    ("human", "{input}"),
    MessagesPlaceholder("agent_scratchpad"),
])

lc_llm = ChatOpenAI(model="gpt-4o", temperature=0)
tools = [search_company_policy, get_employee_leave_balance, submit_leave_request]
agent = create_tool_calling_agent(lc_llm, tools, agent_prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# ใส่ memory
agent_with_history = RunnableWithMessageHistory(
    agent_executor,
    lambda sid: RedisChatMessageHistory(sid, url="redis://dragonfly:6379"),
    input_messages_key="input",
    history_messages_key="history",
)

# === FastAPI ===
app = FastAPI()

class QueryRequest(BaseModel):
    question: str
    session_id: str
    employee_id: str = ""

@app.post("/chat")
async def chat(req: QueryRequest):
    result = await agent_with_history.ainvoke(
        {"input": req.question},
        config={"configurable": {"session_id": req.session_id}}
    )
    return {"answer": result["output"]}

# === ทดสอบ ===
async def test():
    session = "user_E001_session_1"
    questions = [
        "ลาพักร้อนได้กี่วันต่อปี?",
        "ผม ID E001 มีวันลาเหลือกี่วัน?",
        "อยากลาพักร้อน 20-22 มีนา 68 ช่วยยื่นให้หน่อยได้ไหม?",
        "แล้วต้องแจ้งล่วงหน้ากี่วัน?"   # ← ยังจำ context เรื่องลาอยู่
    ]
    
    for q in questions:
        print(f"\n🙋 User: {q}")
        result = await agent_with_history.ainvoke(
            {"input": q},
            config={"configurable": {"session_id": session}}
        )
        print(f"🤖 เซล: {result['output']}")

asyncio.run(test())
```

**output ที่ได้:**
```
🙋 User: ลาพักร้อนได้กี่วันต่อปี?
🤖 เซล: พนักงาน Sellsuki มีสิทธิ์ลาพักร้อน 10 วันทำการต่อปี 
        หลังผ่านทดลองงาน 3 เดือนแล้วครับ 
        (อ้างอิงจาก: hr_policy_2025.md)

🙋 User: ผม ID E001 มีวันลาเหลือกี่วัน?
🤖 เซล: พนักงาน E001 มีวันลาคงเหลือดังนี้ครับ:
        - ลาพักร้อน: 7 วัน
        - ลาป่วย: 25 วัน

🙋 User: อยากลาพักร้อน 20-22 มีนา 68 ช่วยยื่นให้หน่อยได้ไหม?
🤖 เซล: ยื่นคำขอลาเรียบร้อยแล้วครับ!
        รหัสคำขอ: LV-3847
        ประเภท: ลาพักร้อน (3 วัน)
        วันที่: 20-22 มีนาคม 2568
        สถานะ: รอ HR อนุมัติ

🙋 User: แล้วต้องแจ้งล่วงหน้ากี่วัน?
🤖 เซล: ตามนโยบายบริษัท การลาพักร้อนต้องแจ้งล่วงหน้าอย่างน้อย 3 วันทำการครับ
        (อ้างอิงจาก: hr_policy_2025.md)
```

---

## ส่วนที่ 7: สรุปแบบ Visual

```
เมื่อไหร่ใช้อะไร?

ต้องการแค่ "ตอบคำถามจากเอกสาร"
→ LlamaIndex อย่างเดียว ✅

ต้องการ "AI ทำงานได้เอง, เรียก API, ทำหลายขั้นตอน"
→ LangChain อย่างเดียว ✅

ต้องการทั้งคู่ = "ตอบคำถามจากเอกสาร + ทำงานอัตโนมัติ"
→ LlamaIndex (Retrieval) + LangChain (Agent) ✅✅

ตัวอย่างจริง:
┌──────────────────────────────────────────────────┐
│  "ลาพักร้อนกี่วัน?"                              │
│   → LlamaIndex ค้นเอกสาร → ตอบ ✅               │
│                                                  │
│  "ช่วยนัด meeting ให้หน่อย"                      │
│   → LangChain Agent → เรียก Calendar API ✅      │
│                                                  │
│  "มีวันลาเหลือกี่วัน แล้วช่วยจองวันหยุดเลย"      │
│   → LlamaIndex หา policy                         │
│   + LangChain Agent เรียก HR API + จอง ✅✅      │
└──────────────────────────────────────────────────┘
```

---

## ส่วนที่ 8: Package & Version Reference

```bash
# LangChain (แยก package ตาม integration)
pip install langchain                # core
pip install langchain-openai         # OpenAI integration
pip install langchain-community      # community integrations (PGVector ฯลฯ)
pip install langchain-core           # base classes
pip install langgraph                # Agent graphs (advanced)

# LlamaIndex (แยก package เช่นกัน)
pip install llama-index-core                        # core
pip install llama-index-embeddings-openai           # OpenAI embedding
pip install llama-index-llms-openai                 # OpenAI LLM
pip install llama-index-vector-stores-postgres       # ParadeDB/PGVector
pip install llama-index-storage-kvstore-redis        # Redis/Dragonfly cache
pip install llama-index-postprocessor-cohere-rerank  # Cohere reranker

# Shared utilities
pip install openai redis psycopg2-binary fastapi uvicorn
```

---

*จัดทำสำหรับ Sellsuki Engineering | March 2026*
