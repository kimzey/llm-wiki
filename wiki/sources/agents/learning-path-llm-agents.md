---
title: "Learning Path: LLM Agents"
type: source
source_file: raw/notes/ai/agents/README.md
tags: [llm, agents, learning-path, rag]
related: []
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai/agents/README.md|Original file]]

## สรุป

เอกสารแนะนำเส้นทางการเรียนรู้ LLM Agents ตั้งแต่พื้นฐานจนถึงระดับ production-ready  เป็นคู่มือที่ออกแบบมาเพื่อให้ผู้อ่านเข้าใจ Agent อย่างลึกซึ้งผ่านเอกสาร 6 ไฟล์ที่เรียงลำดับจากพื้นฐานสู่ advanced patterns

## ประเด็นสำคัญ

### ลำดับการเรียนที่แนะนำ

เอกสารแบ่งเป็น 5 ระดับตามลำดับความยากและความซับซ้อน:

1. **00-llm-basics.md** (~30 นาที) - เริ่มต้นที่นี่
   - LLM คืออะไร, tokens, prompting, RAG 101

2. **tool-calls.md** (~20 นาที)
   - Function Calling — กลไกที่ทุก agent ใช้

3. **agent-fundamentals.md** (~40 นาที)
   - Agent = LLM + Loop + Tools + Memory
   - 4 เสาหลัก, failure modes, production

4. **react-agent.md** (~30 นาที)
   - Pattern หลัก: Thought → Action → Observation

5. **agent-patterns.md** (อ้างอิง)
   - 22 patterns ครบทุกอย่าง: CoT, Reflexion, RAG, Multi-agent

### สิ่งที่จะได้เรียนรู้

จากแต่ละไฟล์:

**LLM Basics (00-llm-basics.md)**
- LLM ทำงานยังไง (transformer, autoregressive, tokens)
- Context window คืออะไร ทำไมสำคัญ
- วิธีสื่อสารกับ LLM (system prompt, user, few-shot)
- Chain-of-Thought — เพิ่ม reasoning แบบไม่ต้องเขียน code
- RAG คืออะไร ทำไมต้องมี embeddings
- ทำไม RAG ไม่พอ → ทำไมต้องมี Agent

**Tool Calls (tool-calls.md)**
- LLM ไม่ได้รัน code เอง — generate JSON แล้วเราเรียก
- โครงสร้าง tool definition ที่ถูกต้อง
- Loop request → tool use → tool result → final answer
- Parallel tool calls
- Error handling ใน tools
- ความต่าง Anthropic vs OpenAI API

**Agent Fundamentals (agent-fundamentals.md)**
- นิยามที่ใช้งานได้จริงของ Agent
- 4 เสาหลัก: Memory, Planning, Tools, Action loop
- Memory หลายชั้น: in-context, episodic, semantic, in-weights
- 6 failure modes ที่พบบ่อย + วิธีป้องกัน
- System prompt ที่ดีสร้างอย่างไร
- Production: observability, guardrails, cost control, HITL

**ReAct Agent (react-agent.md)**
- Thought → Action → Observation loop
- ทำไม ReAct ดีกว่า CoT (มี tools) และ Action-only (มี reasoning)
- ส่วนประกอบ: Memory, LLM, Tool Registry, Executor
- Implementation ใน LangChain, LangGraph, LlamaIndex, raw API
- เมื่อใช้ vs ไม่ใช้ ReAct

**Agent Patterns (agent-patterns.md)**
- 22 patterns แบ่งเป็น 4 กลุ่ม:
  - **Single-agent reasoning**: CoT, ReAct, Reflexion, Plan-and-Execute, ReWOO, ToT, GoT, LATS
  - **RAG-specific**: Self-RAG, CRAG, Agentic RAG, Modular RAG
  - **Tool-use**: Function Calling, Toolformer, Computer Use
  - **Multi-agent**: Orchestrator-Worker, Debate, CrewAI, AutoGen, Swarm, MemGPT, Generative Agents

### แนวทางการเรียนตามเวลาที่มี

- **30 นาที**: อ่านแค่ 00-llm-basics.md + tool-calls.md
  → รู้ว่า agent ทำงานยังไงในระดับ API

- **1 ชั่วโมง**: + agent-fundamentals.md
  → เข้าใจ architecture และสิ่งที่ผิดพลาดได้

- **2 ชั่วโมง**: + react-agent.md
  → implement agent เองได้

- **ครบทุกอย่าง**: + agent-patterns.md
  → เลือก pattern ได้ถูกต้องตาม use case

## ข้อมูล / หลักฐาน ที่น่าสนใจ

### ความต้องการพื้นฐาน

- Python พื้นฐาน (function, dict, list, async)
- เคยใช้ ChatGPT หรือ Claude มาบ้าง
- **ไม่ต้องรู้ Machine Learning หรือ Math**

### Learning Outcomes

หลังจากอ่านครบแล้ว ควรสามารถ:
- [ ] อธิบายได้ว่า Agent ต่างจาก Chatbot อย่างไร
- [ ] เขียน tool definition ที่ถูกต้องได้
- [ ] implement ReAct loop แบบ raw API ได้
- [ ] เลือก pattern ที่เหมาะกับงานแต่ละประเภทได้
- [ ] รู้ว่า agent จะล้มเหลวอย่างไร และป้องกันได้

## Concepts ที่เกี่ยวข้อง

- [[LLM]] - Large Language Models
- [[RAG]] - Retrieval-Augmented Generation
- [[ReAct]] - Reasoning + Acting pattern
- [[Agent]] - AI Agent architecture
- [[Function Calling]] - Tool use mechanism
- [[Context Window]] - LLM memory limit
- [[Chain-of-Thought]] - CoT prompting technique
- [[Embeddings]] - Vector representations of text
- [[Multi-Agent Systems]] - Multiple AI agents working together
