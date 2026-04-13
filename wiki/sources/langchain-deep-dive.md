---
title: "LangChain Deep Dive — ทุกสิ่งที่ต้องรู้เพื่อใช้งานจริง"
type: source
source_file: raw/notes/LangChain/langchain-deep-dive.md
tags: [langchain, deep-dive, tutorial, comprehensive]
related: []
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/LangChain/langchain-deep-dive.md|Original file]]

## สรุป

เอกสารสอนทุกองค์ประกอบของ LangChain อย่างละเอียด พร้อมตัวอย่าง code ที่รันได้จริง ใช้ Python + Gemini เป็นหลัก แต่สามารถเปลี่ยนไปใช้ OpenAI / Anthropic ได้ทันที

## ประเด็นสำคัญ

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

### หัวข้อที่ครอบคลุม

1. **โครงสร้าง Package และการติดตั้ง**
2. **Models** — ทุกวิธีในการเรียกใช้ LLM
3. **Messages** — ระบบสื่อสารกับ LLM อย่างละเอียด
4. **Prompt Templates** — สร้าง Prompt แบบมืออาชีพ
5. **Output Parsers & Structured Output**
6. **Tools** — สร้างเครื่องมือให้ LLM ใช้งาน
7. **Agents** — สร้าง AI Agent ที่คิดเองได้
8. **Memory & State** — ระบบความจำ
9. **Chains & LCEL** — เชื่อมต่อขั้นตอนเข้าด้วยกัน
10. **Document Loaders** — โหลดข้อมูลจากทุกแหล่ง
11. **Text Splitters** — ตัดเอกสารอย่างชาญฉลาด
12. **Embeddings & Vector Stores**
13. **Callbacks & Middleware**
14. **Error Handling & Best Practices**

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- LangChain แยก package ออกเป็นส่วนๆ เพื่อให้ติดตั้งได้ตามที่ใช้จริง
- `langchain-core` = Interface พื้นฐา� (เบา, จำเป็นเสมอ)
- `langchain-community` = Third-party integrations (โดย community)
- Integration เฉพาะ provider (เช่น `langchain-google-genai`) = โดยทีม LangChain

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อมูลที่ขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/ai-agent.md|AI Agent]]
- [[wiki/concepts/agentic-rag.md|Agentic RAG]]
