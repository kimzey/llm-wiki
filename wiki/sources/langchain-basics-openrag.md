---
title: "LangChain พื้นฐาน — คู่มือฉบับสมบูรณ์"
type: source
source_file: raw/notes/openrag/langchain_basics.md
url: ""
published: 2026-01-01
tags: [langchain, basics, tutorial, rag, agent, memory, caching, beginner]
related: [wiki/concepts/langchain-framework.md, wiki/concepts/lcel-langchain-expression-language.md, wiki/concepts/rag-retrieval-augmented-generation.md]
created: 2026-04-14
updated: 2026-04-14
---

**Full source**: [[../../raw/notes/openrag/langchain_basics.md|Original file]]

## สรุป

คู่มือ LangChain สำหรับมือใหม่ครอบคลุมตั้งแต่เริ่มต้นถึงระดับ production — ChatOpenAI, PromptTemplate, LCEL Chain ด้วย `|`, Output Parsers, Memory (InMemory), Document Loader, Text Splitter, Vector Store + RAG chain, Tools & Agent, Callbacks, และ Caching พร้อม expected output ของทุก code example

## ประเด็นสำคัญ

- **LangChain คืออะไร**: Python/JS library สำหรับสร้างแอปที่ใช้ LLM — เชื่อม Input, Processing, LLM, Output เข้าด้วยกัน
- **Architecture ทั้งหมด**: PromptTemplate → ChatModel → OutputParser (core), เสริมด้วย Memory, RAG, Tools, Agent, Cache, Callbacks
- **LCEL `|`**: Prompt | LLM | Parser — data flow ชัดเจน, รองรับ stream ได้ทันที
- **RAG Pattern**: `{"context": retriever | format_docs, "question": RunnablePassthrough()} | prompt | llm | StrOutputParser()`
- **Agent**: `create_tool_calling_agent()` + `AgentExecutor` — AI ตัดสินใจเองว่าจะใช้ tool ไหน
- **Callbacks**: `BaseCallbackHandler` — `on_llm_start`, `on_llm_end`, `on_llm_error` สำหรับ logging/monitoring
- **Caching**: `InMemoryCache` (ชั่วคราว) vs `RedisCache` (persistent) — ตั้งผ่าน `set_llm_cache()`

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- Tool ใน LangChain สร้างด้วย `@tool` decorator — docstring กลายเป็น description ที่ AI อ่าน
- `AgentExecutor` ใช้ `verbose=True` เพื่อดู thought process ของ agent ทุก step
- `MessagesPlaceholder(variable_name="agent_scratchpad")` จำเป็นใน agent prompt — agent ใช้พื้นที่นี้คิดก่อนตอบ
- RecursiveCharacterTextSplitter: chunk_size=500, chunk_overlap=50 — ตัวอย่างทั่วไปสำหรับ RAG

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — เนื้อหา tutorial เสริมความเข้าใจ LangChain framework เดิม

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/langchain-framework|LangChain Framework]] — framework ที่ document นี้สอน
- [[wiki/concepts/lcel-langchain-expression-language|LCEL]] — syntax `|` ที่ใช้ตลอดทั้ง tutorial
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — บทที่ 7 Vector Store & RAG
- [[wiki/concepts/ai-agent|AI Agent]] — บทที่ 8 Tools & Agent
