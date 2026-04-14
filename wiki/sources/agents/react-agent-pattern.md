---
title: "ReAct Agent — ความรู้ฉบับสมบูรณ์"
type: source
source_file: raw/notes/ai/agents/react-agent.md
tags: [react, reasoning, acting, agent-pattern, loop]
related: []
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai/agents/react-agent.md|Original file]]

## สรุป

คู่มือครบถ้วนเกี่ยวกับ ReAct (Reasoning + Acting) pattern — หนึ่งใน patterns ที่สำคัญที่สุดของ LLM Agents  ครอบคลุมตั้งแต่ทฤษฎี การทำงาน ส่วนประกอบ ไปจนถึง implementation ใน frameworks ต่างๆ

## ประเด็นสำคัญ

### 1. ReAct คืออะไร

**ReAct = ReAsoning + Acting**

- LLM คิดออกมาเป็นข้อความ (Thought) แล้วทำ Action จริงกับโลกภายนอก
- ดู Observation แล้วคิดต่อ — วนซ้ำจนตอบได้
- **Paper:** "ReAct: Synergizing Reasoning and Acting in Language Models" (Yao et al., 2022)

### 2. Loop หลัก: Thought → Action → Observation

```
Input: "ทอง 1 บาทกี่กรัม และราคาวันนี้เท่าไหร่?"

Thought:  ต้องค้นหา 2 อย่าง น้ำหนักทอง 1 บาท และราคาตลาดวันนี้
Action:   search("ทอง 1 บาทกี่กรัม")
Observation: ทอง 1 บาท = 15.244 กรัม

Thought:  ได้น้ำหนักแล้ว ต้องหาราคาต่อไป
Action:   search("ราคาทองวันนี้")
Observation: ทอง 1 บาท = 40,250 บาท

Thought:  มีข้อมูลครบแล้ว ตอบได้
Final Answer: ทอง 1 บาท = 15.244 กรัม ราคาวันนี้ 40,250 บาท
```

- แต่ละ iteration = **1 LLM call**
- ตัวอย่างนี้ = 3 LLM calls
- Simple RAG = 1 LLM call → นี่คือ **overhead** ที่เห็นใน benchmark

### 3. ส่วนประกอบของ ReAct Agent

```
┌─────────────────────────────────────┐
│           ReAct Agent               │
│                                     │
│  ┌──────────┐  ┌──────────┐  ┌────┐│
│  │  Memory  │  │   LLM    │  │Tool││
│  │(context  │◄─►│(reasoning│  │Reg ││
│  │ window)  │  │ engine)  │  │istry││
│  └──────────┘  └────┬─────┘  └─┬──┘│
│                     │parse│action│
│                ┌────▼─────▼─────┐  │
│                │    Executor    │  │
│                │(call tool, get │  │
│                │   result)      │  │
│                └────────────────┘  │
└─────────────────────────────────────┘
```

**Memory (Context Window):**
- Short-term: ทั้ง conversation history รวม Thought/Action/Observation ทุก step
- ยิ่งวนหลายรอบ context ยิ่งยาว → cost สูงขึ้น

**LLM (Reasoning Engine):**
- Generate Thought (reasoning)
- Generate Action (tool selection + input)
- Generate Final Answer (when done)
- ตัว LLM ไม่ได้ "รู้" ว่าทำงาน agent อยู่ — มันแค่ generate text ตาม prompt

**Tool Registry:**
- function ที่ register ให้ agent เรียกได้
- แต่ละ tool มี: name, description, input schema
- LLM เลือก tool โดยอ่าน description → **description ต้องชัดมาก**

**Executor / Runtime:**
- Parse output ของ LLM หา `Action:` token
- Call tool function จริง
- ใส่ผลลัพธ์เป็น `Observation:` กลับใน context

### 4. เมื่อใช้ vs ไม่ใช้ ReAct

**ใช้ ReAct Agent เมื่อ:**
- คำถามต้องการข้อมูลจากหลายแหล่ง
- ต้องการคำนวณหรือ logic หลายขั้น
- ไม่รู้ล่วงหน้าว่าต้องการ step กี่ step
- ต้องการ interact กับ external system (API, database)
- งาน research / analysis ที่ต้องตัดสินใจระหว่างทาง

**ไม่ใช้ ReAct Agent เมื่อ:**
- Q&A จากเอกสารที่มีอยู่แล้ว (ใช้ simple RAG แทน)
- latency สำคัญกว่า accuracy
- คำถาม deterministic ที่รู้ล่วงหน้าแน่ชัด
- ต้องการ cost ต่ำ
- ระบบ production ที่ต้องการ predictable behavior

### 5. ข้อดีและข้อเสีย

**ข้อดี:**
- **Dynamic**: ปรับ plan ได้ real-time ตาม observation
- **Traceable**: Thought ทุก step มองเห็นได้ → debug ง่าย
- **Generalizable**: tools ใหม่แค่ add เข้า registry ไม่ต้อง retrain LLM
- **Multi-step**: แก้ปัญหาที่ต้องการหลาย action ได้

**ข้อเสีย:**
- **Latency สูง**: หลาย LLM calls → หลาย API round trips
- **Cost สูง**: tokens เพิ่มขึ้นทุก iteration
- **Unpredictable**: จำนวน iterations ไม่ fixed อาจ loop ไม่จบ
- **Hallucination ของ Action**: LLM อาจ generate tool name ที่ไม่มีจริง
- **Context window limit**: ประวัติยาวๆ อาจ overflow
- **Overkill สำหรับงานง่าย**: Simple Q&A ไม่ต้องการ agent

### 6. Patterns ที่เกี่ยวข้อง

**Chain-of-Thought (CoT) — ancestor ของ ReAct:**
- CoT คิดทีละ step แต่ **ไม่ call tool ภายนอก**
- ทำงานในหัว LLM อย่างเดียว

**Self-Ask — ReAct แบบถามตัวเอง:**
- คล้าย ReAct แต่ decompose คำถามเป็น sub-questions ก่อน

**Plan-and-Execute (แยก planning กับ execution):**
- ไม่ reactive — plan ถูก lock ตั้งแต่ต้น แก้แผนระหว่างทางไม่ได้

**Reflexion — ReAct + self-reflection:**
- เพิ่ม memory layer บันทึก failure/insight ระหว่าง attempts

**Tree of Thoughts (ToT):**
- ไม่ใช่ linear — explore หลาย reasoning paths พร้อมกัน

**Self-RAG — ReAct สำหรับ RAG โดยเฉพาะ:**
- LLM generate special tokens ตัดสินใจเองว่าต้อง retrieve มั้ย

**Agentic RAG:**
- Multi-step retrieval + synthesis แทนที่จะ retrieve ครั้งเดียว

### 7. Implementation ใน Framework ต่างๆ

**LangChain (ที่ใช้ใน benchmark นี้):**
```python
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent

@tool
def search(query: str) -> str:
    """Search for information."""
    return ...

agent = create_react_agent(llm, [search])
result = agent.invoke({"messages": [{"role": "user", "content": "..."}]})
```

`create_react_agent` สร้าง LangGraph graph:
```
START → llm_node → tool_node → llm_node → ... → END
              (ถ้า LLM เลือก tool)
```

**LangGraph (control มากขึ้น):**
```python
from langgraph.graph import StateGraph

graph = StateGraph(AgentState)
graph.add_node("reason", reason_node)
graph.add_node("act", action_node)
graph.add_edge("reason", "act")
graph.add_conditional_edges("act", should_continue,
    {"continue": "reason", "end": END})
```

**LlamaIndex:**
```python
from llama_index.core.agent import ReActAgent
from llama_index.core.tools import FunctionTool

tool = FunctionTool.from_defaults(fn=search_fn)
agent = ReActAgent.from_tools([tool], llm=llm, verbose=True)
response = agent.chat("คำถาม")
```

**ทำเอง (ไม่ใช้ framework):**
```python
def react_loop(question, tools, llm, max_iter=5):
    history = [{"role": "user", "content": question}]

    for _ in range(max_iter):
        response = llm.complete(build_prompt(history))
        thought, action = parse_react(response)

        if action.name == "finish":
            return action.input

        observation = tools[action.name](action.input)
        history.append({"thought": thought,
                       "action": action,
                       "observation": observation})

    raise MaxIterationsError
```

### 8. ความสัมพันธ์กับ Multi-Agent Systems

ReAct เป็น **single agent** pattern แต่เป็น building block ของ multi-agent:

```
Orchestrator Agent (ReAct)
├── Action: call_agent("researcher", "ค้นหาข้อมูล X")
│   └── Researcher Agent (ReAct) → search tools
├── Action: call_agent("analyst", "วิเคราะห์ข้อมูล")
│   └── Analyst Agent (ReAct) → calculator, code tools
└── Final Answer: สรุปจากทั้ง 2 agent
```

Framework ที่รองรับ multi-agent:
- **LangGraph**: stateful, persistent agents
- **AutoGen** (Microsoft): agent conversations
- **CrewAI**: role-based agents
- **OpenAI Swarm**: lightweight handoffs

## ข้อมูล / หลักฐาน ที่น่าสนใจ

### Benchmark Comparison

| | bare_metal | llamaindex | haystack | langchain |
|---|---|---|---|---|
| Pattern | retrieve-then-read | retrieve-then-read | retrieve-then-read | **ReAct agent** |
| LLM calls/query | 1 | 1 | 1 | **2–3** |
| Retrieval rounds | 1 (fixed) | 1 (fixed) | 1 (fixed) | 1–2 (dynamic) |
| Latency | ~2.7s | ~3.0s | ~2.7s | **~6.4s** |
| เหมาะกับงานนี้ | ✅ | ✅ | ✅ | ❌ (overkill) |

**LangChain ไม่ได้แย่กว่า** — ใช้ tool ผิดงาน  
ถ้าเปลี่ยนเป็น LCEL chain (ไม่ใช้ agent) latency จะใกล้เคียงคนอื่น

### Tool Types ที่ Agent ใช้ได้

| ประเภท | ตัวอย่าง | use case |
|---|---|---|
| **Search / Retrieval** | web search, vector DB query | หาข้อมูลที่ไม่อยู่ใน training data |
| **Code Execution** | Python REPL, bash | คำนวณ, data analysis |
| **API Calls** | REST API, database query | ดึงข้อมูล real-time |
| **File I/O** | read/write file, parse PDF | จัดการเอกสาร |
| **Browser** | Playwright, Selenium | เปิด URL, กรอก form |
| **Memory** | store/retrieve to vector DB | long-term memory |
| **Sub-agents** | call another LLM agent | multi-agent orchestration |

## Concepts ที่เกี่ยวข้อง

- [[ReAct]] - Reasoning + Acting pattern
- [[Chain-of-Thought]] - CoT prompting technique
- [[Agent]] - LLM Agent architecture
- [[Tools]] - Agent tools and actions
- [[Function Calling]] - Tool use mechanism
- [[Memory]] - Agent memory systems
- [[LangChain]] - Framework for building agents
- [[LangGraph]] - Stateful agent framework
- [[LlamaIndex]] - Data framework for LLM apps
- [[Multi-Agent Systems]] - Multiple agents working together
