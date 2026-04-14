---
title: "Agent Patterns — คู่มืออ่านเข้าใจทุก Pattern"
type: source
source_file: raw/notes/ai/agents/agent-patterns.md
tags: [agent-patterns, reference, multi-agent, rag, reasoning]
related: []
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai/agents/agent-patterns.md|Original file]]

## สรุป

คู่มืออ้างอิงครบถ้วนที่ครอบคลุม **22 agent patterns** แบ่งเป็น 4 หมวดหลัก: Single-Agent Reasoning, RAG-Specific, Tool-Use Focused, และ Multi-Agent Systems  เหมาะสำหรับใช้เป็น reference เลือก pattern ที่เหมาะกับ use case แต่ละประเภท

## ประเด็นสำคัญ

### แผนที่ภาพรวมทั้งหมด

```
LLM Agent Patterns
│
├── Single-Agent Reasoning
│   ├── Chain-of-Thought (CoT)
│   ├── ReAct
│   ├── Reflexion
│   ├── Plan-and-Execute
│   ├── ReWOO
│   ├── Tree of Thoughts (ToT)
│   ├── Graph of Thoughts (GoT)
│   └── LATS
│
├── RAG-Specific Agents
│   ├── Self-RAG
│   ├── Corrective RAG (CRAG)
│   ├── Agentic RAG
│   └── Modular RAG
│
├── Tool-Use Focused
│   ├── Function Calling (native)
│   ├── Toolformer
│   └── Computer Use
│
└── Multi-Agent Systems
    ├── Orchestrator-Worker
    ├── Debate / Critic
    ├── Role-based (CrewAI)
    ├── Conversational (AutoGen)
    ├── Handoff (Swarm)
    └── Memory-Augmented (MemGPT)
```

### ส่วนที่ 1: Single-Agent Reasoning

**1. Chain-of-Thought (CoT)**
- Paper: Wei et al., 2022
- บังคับให้ LLM เขียน reasoning steps ออกมาก่อนตอบ
- ใช้เมื่อ: Math reasoning, logic puzzles, multi-step deduction
- **ไม่มี tool calls** — ทำงานใน LLM ล้วนๆ

**2. ReAct**
- Paper: Yao et al., 2022
- **Thought → Action → Observation** loop
- รวม CoT + tool use เข้าด้วยกัน
- ใช้เมื่อ: ต้องการ multi-step reasoning + tools

**3. Reflexion**
- Paper: Shinn et al., 2023
- ReAct + **self-critique หลังผิดพลาด**
- มี 3 ส่วน: Actor, Evaluator, Self-Reflector
- ใช้เมื่อ: งานที่มี verifiable outcome ชัดเจน, ต้องการ improve ด้วย trial-and-error

**4. Plan-and-Execute**
- Paper: Wang et al., 2023
- แยก "คนวางแผน" กับ "คนลงมือทำ" ออกจากกัน
- Phase 1: Planning (LLM ใหญ่), Phase 2: Execution (LLM เล็กกว่า)
- ใช้เมื่อ: Workflows ที่รู้ขั้นตอนล่วงหน้า, ต้องการ cost optimization

**5. ReWOO (Reasoning WithOut Observations)**
- Paper: Xu et al., 2023
- Plan ล่วงหน้าพร้อมระบุ dependency → รัน **parallel** ได้
- ใช้เมื่อ: Research tasks, tools ที่ call ได้ independent, ต้องการลด latency

**6. Tree of Thoughts (ToT)**
- Paper: Yao et al., 2023
- **Explore หลาย paths พร้อมกัน** แล้วเลือกที่ดีสุด
- 3 ขั้นตอน: Generation, Evaluation, Search (BFS/DFS)
- ใช้เมื่อ: ปัญหาที่ต้องการ exploration, creative writing, puzzle solving

**7. Graph of Thoughts (GoT)**
- Paper: Besta et al., 2023 (ETH Zürich)
- Thoughts สามารถ **split + merge + aggregate** ได้
- ใช้เมื่อ: ปัญหา divide-and-conquer, document synthesis

**8. LATS (Language Agent Tree Search)**
- Paper: Zhou et al., 2023
- ToT + **MCTS** (Monte Carlo Tree Search) + Reflexion
- ใช้ reward signals ย้อนกลับมา guide search
- ใช้เมื่อ: Code generation, optimization problems

### ส่วนที่ 2: RAG-Specific Agents

**9. Self-RAG**
- Paper: Asai et al., 2023
- LLM ตัดสินเอง: retrieve? relevant? grounded? useful?
- ใช้ **special tokens** → ต้อง **fine-tune LLM**
- ใช้เมื่อ: RAG ที่ต้องการ precision สูง, ต้องการ reduce hallucination

**10. Corrective RAG (CRAG)**
- Paper: Yan et al., 2024
- มี **evaluator** ตรวจ quality ของ retrieved docs
- Fallback → web search ถ้า docs ไม่ดีพอ
- ใช้เมื่อ: Corpus ที่ไม่ครบถ้วน, ไม่มี budget fine-tune

**11. Agentic RAG**
- ไม่มี paper เดียว — design pattern
- Multi-step retrieval + synthesis
- Features: Query decomposition, Multi-hop retrieval, Multi-source, Iterative refinement
- ใช้เมื่อ: คำถามซับซ้อน, knowledge base หลายแหล่ง

**12. Modular RAG**
- Paper: Gao et al., 2023
- ทำให้ทุก component เป็น **module ที่ swap-in/out ได้**
- ใช้เมื่อ: ต้องการ custom RAG pipeline, A/B test components

### ส่วนที่ 3: Tool-Use Focused

**13. Function Calling (Native Tool Use)**
- Feature ของ GPT-4, Claude, Gemini
- LLM generate **JSON ที่ structured** โดยตรง
- ใช้เมื่อ: ทุกครั้งที่ต้องการ tool use ที่ reliable, production systems

**14. Toolformer**
- Paper: Schick et al., 2023 (Meta AI)
- LLM **เรียนรู้การใช้ tools ด้วยตัวเอง** จาก self-supervised learning
- ใช้เมื่อ: ต้องการ LLM ที่เรียนรู้ tools ด้วยตัวเอง, research

**15. Computer Use Agent**
- Anthropic: Claude Computer Use (2024)
- LLM ควบคุม GUI/browser โดยตรงผ่าน screenshot + actions
- Tools: screenshot, click, type, key, scroll
- ใช้เมื่อ: Automation บน GUI ที่ไม่มี API, web scraping, RPA

### ส่วนที่ 4: Multi-Agent Systems

**16. Orchestrator-Worker**
- 1 Orchestrator → N specialist Workers
- Orchestrator: วางแผน, assign งาน, synthesize
- ใช้เมื่อ: งานซับซ้อนที่ต้องการ specialists

**17. Debate / Critic Pattern**
- Paper: Du et al., 2023
- หลาย agents "โต้กัน" → ลด hallucination, เพิ่ม factual accuracy
- ใช้เมื่อ: คำถามที่ต้องการ factual accuracy สูง

**18. CrewAI — Role-Based Agents**
- 4 concepts: Agent, Task, Crew, Process
- ใช้เมื่อ: Team workflows, ต้องการ clear roles

**19. AutoGen — Conversational Multi-Agent**
- Microsoft Research
- agents ทำงานร่วมกันผ่าน **chat interface**
- AssistantAgent + UserProxyAgent (รัน code จริง)
- ใช้เมื่อ: Data analysis + code generation, ต้องการ human-in-the-loop

**20. OpenAI Swarm / Handoff Pattern**
- **function ที่ return agent อื่น = handoff**
- ใช้เมื่อ: Routing, triage ที่ clear

**21. MemGPT / OS-like Memory Management**
- Paper: Packer et al., 2023
- Successor: **Letta** (ชื่อใหม่, production-ready)
- Agent เป็น **OS** — จัดการ memory เอง ย้าย data ระหว่าง levels
- ใช้เมื่อ: Chatbots ที่ต้องการ persistent memory, long-running agents

**22. Generative Agents (Stanford Simulacra)**
- Paper: Park et al., 2023
- Agents ที่มีพฤติกรรมสมจริง: จำประสบการณ์, มีความสัมพันธ์, วางแผน
- ใช้เมื่อ: Game NPC, social simulation, training data generation

### เปรียบเทียบทุก Pattern

| Pattern | LLM calls | Complexity | Cost | เหมาะกับ |
|---|---|---|---|---|
| **CoT** | 1 | ต่ำ | ต่ำ | Math, logic |
| **ReAct** | N (dynamic) | กลาง | กลาง | Multi-step Q&A |
| **Plan-and-Execute** | 1+N | กลาง | กลาง-ต่ำ | Structured workflows |
| **ReWOO** | 1+parallel | กลาง | ต่ำกว่า ReAct | Research |
| **Reflexion** | N×attempts | สูง | สูง | Code gen |
| **ToT** | k×depth | สูงมาก | สูงมาก | Creative, planning |
| **GoT** | k×depth | สูงมาก | สูงมาก | Divide-and-conquer |
| **Self-RAG** | 1 (fine-tuned) | ต่ำ | ต่ำ | RAG precision สูง |
| **CRAG** | 1–2 | ต่ำ | ต่ำ | RAG fallback |
| **Agentic RAG** | N | สูง | สูง | Multi-hop Q&A |
| **Function Calling** | N | กลาง | กลาง | Tool use ทั่วไป |
| **Computer Use** | N | สูง | สูง | GUI automation |
| **Orchestrator-Worker** | N×agents | สูง | สูง | Complex multi-step |
| **Debate** | N×agents×rounds | สูงมาก | สูงมาก | Fact-critical |
| **Swarm/Handoff** | N | กลาง | กลาง | Routing, triage |

### วิธีเลือก Pattern

```
คำถาม/งานที่ได้รับ
    ↓
ต้องการข้อมูลจาก external tools/sources มั้ย?
  NO → CoT หรือ direct prompting
  YES ↓

รู้ steps ล่วงหน้าและ predictable มั้ย?
  YES → Plan-and-Execute หรือ ReWOO
  NO ↓ (exploratory / dynamic)

เป็น RAG task (ค้นข้อมูลจาก corpus)?
  YES → Simple RAG ก่อน
        → ถ้า corpus ไม่ครบ → CRAG
        → ถ้าต้องการ precision สูง + fine-tune → Self-RAG
        → ถ้าคำถามซับซ้อน multi-hop → Agentic RAG
  NO ↓

ต้องการ multi-step reasoning แบบ reactive?
  YES → ReAct (หรือ Function Calling)
  NO ↓

ต้องการ improve ด้วย self-critique หลังผิด?
  YES → Reflexion
  NO ↓

ต้องการ explore หลาย solution paths?
  YES, ไม่ต้อง merge → Tree of Thoughts (ToT)
  YES, ต้อง merge → Graph of Thoughts (GoT)
  NO ↓

ต้องการ multi-agent?
  Specialist roles → CrewAI
  Coding + iteration → AutoGen
  Routing/triage → Swarm
  Complex stateful → Orchestrator-Worker (LangGraph)
  Fact-critical → Debate

ต้องการ long-horizon memory?
  YES → MemGPT / Letta
```

### ความสัมพันธ์กับ Benchmark ของเรา

งาน Phase 2 (RAG frameworks) เป็น **Simple/Agentic RAG ระดับเริ่มต้น**:

| Framework | Pattern จริงๆ |
|---|---|
| **bare_metal** | Simple RAG |
| **LlamaIndex** | Modular RAG |
| **Haystack** | Modular RAG |
| **LangChain** | **ReAct Agent** |

**ทำไม LangChain ช้ากว่า?**  
ไม่ใช่เพราะ LangChain แย่ — แต่เพราะใช้ ReAct agent (2-3 LLM calls) แทน simple retrieve-then-read (1 LLM call)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

### Use Cases จริงที่เหมาะกับแต่ละ pattern

| Use Case | Pattern แนะนำ |
|---|---|
| FAQ จาก documents | Simple RAG |
| Multi-doc synthesis | Agentic RAG |
| Tool + retrieval | ReAct + RAG tools |
| Accuracy สูง, corpus อาจไม่ครบ | CRAG + fallback |
| Code generation | ReAct หรือ Reflexion |
| Multi-team workflows | Orchestrator-Worker |
| Customer service routing | Swarm/Handoff |

## Concepts ที่เกี่ยวข้อง

- [[Agent Patterns]] - All agent patterns
- [[Chain-of-Thought]] - CoT prompting
- [[ReAct]] - Reasoning + Acting
- [[Reflexion]] - Self-critique pattern
- [[Plan-and-Execute]] - Separate planning and execution
- [[Tree of Thoughts]] - Multi-path exploration
- [[Self-RAG]] - Retrieval-aware generation
- [[CRAG]] - Corrective RAG with fallback
- [[Agentic RAG]] - Multi-step retrieval
- [[Modular RAG]] - Composable RAG pipeline
- [[Function Calling]] - Native tool use
- [[Orchestrator-Worker]] - Multi-agent pattern
- [[CrewAI]] - Role-based agents
- [[AutoGen]] - Conversational agents
- [[Swarm]] - Handoff pattern
- [[MemGPT]] - OS-like memory management
