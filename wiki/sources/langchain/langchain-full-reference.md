---
title: "LangChain คู่มือสมบูรณ์ — ทุก Syntax ทุก Import"
type: source
source_file: raw/notes/openrag/langchain_full.md
url: ""
published: 2026-01-01
tags: [langchain, lcel, messages, prompt-template, output-parser, memory, document-loader, vector-store, agent, cache]
related: [wiki/concepts/langchain-framework.md, wiki/concepts/rag-retrieval-augmented-generation.md, wiki/concepts/semantic-caching.md]
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/frameworks/openrag/langchain_full.md|Original file]]


## สรุป

คู่มือ LangChain แบบสมบูรณ์ครอบคลุมทุก syntax และ import — ตั้งแต่ Messages, ChatModel, PromptTemplate, LCEL Chain ด้วย `|` operator, Output Parsers (Str/Json/Pydantic), Memory (InMemory/Redis/SQL), Document Loaders, Text Splitters, Vector Store, Agent, Callbacks, และ Caching ทั้งหมดพร้อม code examples จริง

## ประเด็นสำคัญ

- **Package Structure**: `langchain-core` (พื้นฐาน), `langchain` (Chains/Agents/Memory), `langchain-openai`, `langchain-community` (Loaders/VectorStores/Cache), `langchain-text-splitters`
- **Message Types**: `SystemMessage`, `HumanMessage`, `AIMessage`, `FunctionMessage`, `ToolMessage`
- **LLM Methods**: `.invoke()` (single), `.stream()` (real-time token), `.batch()` (หลายคำถามพร้อมกัน), `.ainvoke()` (async)
- **LCEL Syntax `|`**: chain = prompt | llm | parser — เหมือน Unix pipe, output ซ้าย → input ขวา
- **Runnable Types**: `RunnablePassthrough` (ส่งค่าผ่านไปตรงๆ), `RunnableLambda` (ใส่ Python function), `RunnableParallel` (รันพร้อมกัน), `.assign()` (เพิ่ม key), `.bind()` (กำหนด kwargs)
- **Output Parsers**: `StrOutputParser`, `JsonOutputParser`, `PydanticOutputParser`, `CommaSeparatedListOutputParser`, `RetryOutputParser`
- **Memory**: `InMemoryChatMessageHistory`, `RedisChatMessageHistory`, `SQLChatMessageHistory` — ทั้งหมดห่อด้วย `RunnableWithMessageHistory`
- **Document Loaders**: TextLoader, PyPDFLoader, CSVLoader, WebBaseLoader, DirectoryLoader — 100+ formats
- **Caching**: `InMemoryCache` (ชั่วคราว), `RedisCache` (ข้ามการรีสตาร์ต), set ผ่าน `set_llm_cache()`

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- LLM อื่นที่ใช้ได้: Anthropic Claude (`langchain_anthropic`), Google Gemini (`langchain_google_genai`), Ollama (`langchain_ollama`), OpenRouter (ผ่าน `ChatOpenAI` + `openai_api_base`)
- `PromptTemplate.partial()` ใช้กำหนดค่าบางตัวล่วงหน้า เช่น lock ภาษาไว้ก่อน
- `MessagesPlaceholder` ใช้ใส่ slot ใน prompt สำหรับ chat history
- `itemgetter` จาก `operator` ช่วยดึง key จาก dict ใน chain

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่มีข้อขัดแย้ง — เป็นเนื้อหา reference เชิง syntax ที่สอดคล้องกับ LangChain knowledge เดิม

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/langchain-framework|LangChain Framework]] — framework ที่ document นี้อธิบาย
- [[wiki/concepts/rag-retrieval-augmented-generation|RAG]] — Vector Store + Retrieval Chain ที่ครอบคลุมใน PART 7+
- [[wiki/concepts/semantic-caching|Semantic Caching]] — Caching section (PART 10)
