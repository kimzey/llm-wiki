---
title: "LangChain Framework"
type: concept
tags: [langchain, rag, llm, agent, chain, lcel, python, typescript]
sources: [langchain-llamaindex-deep-dive.md, langchain-rag-guide.md]
related: [wiki/concepts/llamaindex-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/rag-chunking-strategies.md, wiki/concepts/semantic-caching.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

LangChain คือ Framework สำหรับเชื่อม LLM เข้ากับ Data, Tools, และ Workflows — "Swiss Army Knife" ที่มี Chain, Agent, Memory, Retrieval, Tools ครบ รองรับทั้ง Python และ TypeScript เหมาะสำหรับ RAG, Chatbot, และ AI Agents ที่ซับซ้อน

## อธิบาย

LangChain แก้ปัญหาที่เรียก OpenAI API ตรง ๆ ทำได้แค่ Q&A ธรรมดา โดยให้ Building Blocks พร้อมใช้: Chain หลาย steps, Agents ที่ตัดสินใจเอง, Memory ที่จำ conversation, Retrievers สำหรับ RAG, Tools สำหรับเรียก external services, Callbacks สำหรับ logging/streaming/monitoring

### LangChain Ecosystem (5 ตัว)

| Framework | หน้าที่ | Output |
|-----------|--------|--------|
| **LangChain** | Core: Chains, Agents, Memory, Retrievers | LLM applications |
| **LangGraph** | Stateful workflows แบบ Graph/DAG พร้อม loops/conditions | Multi-step agents |
| **LangSmith** | Observability: Traces, Metrics, Evals | Debug dashboard |
| **LangServe** | Deploy chains เป็น REST API อัตโนมัติ | FastAPI endpoints |
| **LangChain Hub** | Prompt repository | Reusable prompts |

## ประเด็นสำคัญ

### Core Building Blocks

**LLM / Chat Model:**
```python
llm = ChatOpenAI(model="gpt-4o", temperature=0)
response = llm.invoke("สวัสดีครับ")
```

**Prompt Template:**
```python
prompt = ChatPromptTemplate.from_messages([
    ("system", "คุณคือ HR bot ของ {company}"),
    ("human", "{question}"),
])
```

**LCEL (LangChain Expression Language):** syntax `|` เหมือน Unix pipe
```python
chain = prompt | llm | StrOutputParser()
result = chain.invoke({"company": "Sellsuki", "question": "ลาพักร้อนกี่วัน?"})
```

**RunnableParallel:** ทำหลายอย่างพร้อมกัน
```python
chain = RunnableParallel(
    answer=(prompt | llm | StrOutputParser()),
    question=RunnablePassthrough(),
)
```

**Memory:** เก็บ chat history ใน Redis (Dragonfly)
```python
def get_session_history(session_id: str):
    return RedisChatMessageHistory(session_id, url="redis://dragonfly:6379", ttl=3600)

chain_with_history = RunnableWithMessageHistory(
    chain, get_session_history,
    input_messages_key="question",
    history_messages_key="history",
)
```

**Document Loaders:** รองรับ 100+ formats — PDF, Notion, Slack, Git, Web, etc.

**Text Splitters:**
- `RecursiveCharacterTextSplitter` — แนะนำ, ใช้บ่อยที่สุด
- `MarkdownHeaderTextSplitter` — แบ่งตาม `#`, `##`, `###`, เก็บเป็น metadata
- `SemanticChunker` — ตัดเมื่อความหมายเปลี่ยน

**Embeddings + VectorStore:**
```python
vectorstore = PGVector.from_documents(
    documents=chunks,
    embedding=OpenAIEmbeddings(model="text-embedding-3-small"),
    connection_string=CONNECTION_STRING,
    collection_name="sellsuki_docs",
)
```

**Retrieval Chain (RAG):**
```python
rag_chain = (
    {"context": vectorstore.as_retriever() | format_docs, "question": RunnablePassthrough()}
    | prompt | llm | StrOutputParser()
)
```

**Tools & Agents:**
```python
@tool
def get_leave_balance(employee_id: str) -> str:
    """ดูวันลาคงเหลือ"""
    return hr_api.get_balance(employee_id)

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```

### Agent vs Chain

```
Chain:  คำถาม → Step1 → Step2 → Step3 → คำตอบ  (กำหนดไว้ล่วงหน้า)
Agent:  คำถาม → AI คิด → เลือก Tool → ดูผล → คิดอีก → คำตอบ  (dynamic)
```

### LangGraph — Complex Workflows

เหมาะสำหรับ RAG ที่มี fallback logic:
```python
workflow.add_conditional_edges(
    "grade_docs",
    decide_next_step,
    {"relevant": "generate", "not_relevant": "web_search"}
)
```

### LangSmith — Observability

Enable tracing ง่ายมาก:
```python
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls_..."
```

ได้ Dashboard: Input → Retrieval → Prompt → LLM → Output, latency แต่ละ step, tokens used, cost

### LangServe — Deploy เป็น API

```python
from langserve import add_routes
app = FastAPI()
add_routes(app, sellsuki_rag_chain, path="/sellsuki-rag")
# ได้: POST /sellsuki-rag/invoke, /stream, GET /playground
```

### Semantic Cache ใน LangChain

```python
langchain.llm_cache = RedisSemanticCache(
    redis_url="redis://dragonfly:6379",
    embedding=OpenAIEmbeddings(),
    score_threshold=0.95
)
```

## ข้อดี-ข้อเสีย

**ข้อดี:**
- Ecosystem ใหญ่ที่สุด — มี integration กับ Notion, Slack, LINE, Jira ฯลฯ
- LangSmith (observability) ดีมาก
- เหมาะถ้าอนาคตจะทำ Agent ซับซ้อน
- Community ใหญ่ หา solution ได้ง่าย
- LangGraph: ทำ stateful workflows ซับซ้อนได้
- LangServe: Deploy เป็น API ง่ายมาก

**ข้อเสีย:**
- Abstraction ซ้อนกันหลายชั้น debug ยาก
- `langchain` → `langchain-community` → `langchain-core` split ทำให้ confusing
- ถ้า RAG เป็น core use case, มี feature น้อยกว่า LlamaIndex
- Chunking / Indexing ต้องประกอบ pieces เอง
- Package fragmentation ทำให้ import path สับสน

## ตัวอย่าง / กรณีศึกษา

**Sellsuki RAG ด้วย LangChain:**
```python
# Setup PGVector กับ ParadeDB
vectorstore = PGVector(
    connection_string=CONNECTION_STRING,
    embedding_function=OpenAIEmbeddings(),
    collection_name="sellsuki_docs",
)

# Chunking
splitter = RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=64)

# RAG Chain
qa_chain = RetrievalQAWithSourcesChain.from_chain_type(
    llm=ChatOpenAI(model="gpt-4o"),
    retriever=vectorstore.as_retriever(
        search_kwargs={"k": 5, "filter": {"department": "hr"}}
    ),
)

result = qa_chain({"question": "ลาพักร้อนได้กี่วัน?"})
```

## เปรียบเทียบกับ LlamaIndex

| เกณฑ์ | LangChain | LlamaIndex |
|-------|-----------|------------|
| **Focus** | General-purpose LLM apps | RAG/Data-focused |
| **RAG Features** | พอใช้ | ครบและดีกว่า |
| **Agent/Workflow** | ✅✅ | ✅ (LlamaIndex Workflows) |
| **Chunking** | หลายแบบ | หลากหลายกว่า |
| **Ingestion Pipeline** | Manual | ✅ Built-in |
| **Observability** | ✅ LangSmith | ✅ LlamaTrace |
| **Debug ง่าย** | ❌ | ✅ |
| **Time to Production** | 3-6 สัปดาห์ | 2-4 สัปดาห์ (สำหรับ RAG) |

## ควรใช้เมื่อไหร่

- ✅ ต้องการ Agent ที่ integrate หลาย tools (Slack, Calendar, Jira)
- ✅ ต้องการ LangSmith สำหรับ monitoring production
- ✅ ต้องการ stateful workflows ที่ซับซ้อน
- ✅ ทีมคุ้นเคยกับ LangChain อยู่แล้ว

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/llamaindex-framework|LlamaIndex Framework]] — คู่แข่งหลัก เน้น RAG โดยตรง
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — LangChain เป็นหนึ่งใน 3 approaches
- [[wiki/concepts/rag-chunking-strategies|RAG Chunking Strategies]] — LangChain มี splitters หลายแบบ
- [[wiki/concepts/semantic-caching|Semantic Caching]] — `RedisSemanticCache` built-in

## แหล่งที่มา

- [[wiki/sources/langchain-llamaindex-deep-dive|LangChain & LlamaIndex — Deep Dive]]
- [[wiki/sources/langchain-rag-guide|LangChain, RAG & Sellsuki Knowledge Base]]
