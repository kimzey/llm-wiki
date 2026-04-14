---
title: "Agent Fundamentals — ทุกอย่างที่ต้องรู้เกี่ยวกับ LLM Agents"
type: source
source_file: raw/notes/ai/agents/agent-fundamentals.md
tags: [agent, fundamentals, architecture, memory, planning, production]
related: []
created: 2026-04-14
updated: 2026-04-14
---

> **Full source**: [[../../../raw/notes/ai/agents/agent-fundamentals.md|Original file]]

## สรุป

คู่มือครบถ้วนเกี่ยวกับ LLM Agents ตั้งแต่นิยาม สถาปัตยกรรม สี่เสาหลัก (Memory, Planning, Tools, Action) จนถึงเรื่อง production เช่น observability, guardrails, cost control, และ failure modes

## ประเด็นสำคัญ

### 1. Agent คืออะไร — นิยามที่ใช้งานได้จริง

> **Agent คือระบบที่ LLM ทำหน้าที่ "ตัดสินใจ" ว่าจะทำอะไรต่อไป**
> โดยมี tools ให้ใช้ และ feedback loop ที่บอกผลของ action

**LLM ปกติ vs Agent:**
```
LLM ปกติ (Chatbot):
  Input → [LLM] → Output
  รับ → คิด → ตอบ (จบ)
  ไม่มี tools, ไม่มี memory, ไม่ทำอะไรกับโลกภายนอก

Agent:
  Input → [LLM] → Action → [World] → Observation → [LLM] → ... → Output
  รับ → คิด → ทำ → ดูผล → คิดต่อ → ทำต่อ → ... → ตอบ (loop)
  มี tools, มี memory, เปลี่ยนแปลงโลกได้
```

**สิ่งที่ต้องมีจึงจะเรียกว่า Agent:**

| องค์ประกอบ | คำอธิบาย | ถ้าไม่มี |
|---|---|---|
| **LLM Reasoning** | LLM ตัดสินใจว่าจะทำอะไร | เป็นแค่ script ทั่วไป |
| **Tools / Actions** | ทำสิ่งที่ LLM ตัดสินใจได้จริง | เป็นแค่ chatbot |
| **Observation / Feedback** | รู้ผลของ action | เป็นแค่ one-shot call |
| **Loop** | ทำซ้ำได้จนงานเสร็จ | เป็นแค่ pipeline |

### 2. สี่เสาหลักของ Agent

#### Memory — อะไรที่ agent "จำ" ได้

| ประเภท | ที่เก็บ | ขนาด | ความเร็ว | ตัวอย่าง |
|---|---|---|---|---|
| **In-Context** | Context window | จำกัด (~128K) | เร็วสุด | Conversation history, tool results |
| **External (Episodic)** | Vector DB / Database | ไม่จำกัด | กลาง | บันทึกการสนทนาเก่า |
| **External (Semantic)** | Vector DB | ไม่จำกัด | กลาง | Knowledge base, documents |
| **In-Weights** | Parameters ของ LLM | fixed | เร็วสุด | ความรู้ทั่วไป, skills |
| **In-Cache** | KV Cache | จำกัด | เร็วมาก | Cached system prompts |

**สิ่งที่ต้องรู้:**
- Agent ที่ไม่มี external memory จะ "ลืม" ทุกอย่างเมื่อ session จบ
- Context window เต็มได้ ถ้า session ยาวหรือ tool results ใหญ่
- การจัดการ memory ดีๆ คือความต่างระหว่าง agent ที่ใช้งานได้จริงกับไม่ได้

#### Planning — agent ตัดสินใจอย่างไร

Planning เกิดใน context window — LLM generate text ที่เป็น reasoning ก่อน action

**ระดับของ Planning:**

```
Low-level (Reactive):
  ดูสถานการณ์ → ตัดสินใจ 1 step → ทำ → ดูผล → ตัดสินใจต่อ
  ตัวอย่าง: ReAct agent

Mid-level (Plan-then-Execute):
  วิเคราะห์งานทั้งหมด → สร้าง plan → execute ทีละ step
  ตัวอย่าง: Plan-and-Execute agent

High-level (Hierarchical):
  แบ่งงานใหญ่เป็น sub-goals → assign ให้ sub-agents → synthesize
  ตัวอย่าง: Orchestrator-Worker multi-agent
```

#### Tools — agent ทำอะไรกับโลกได้

**5 ประเภทหลักของ Tools:**

```
1. READ — ดึงข้อมูล (ไม่เปลี่ยนแปลงโลก)
   search_web, query_database, retrieve_docs, read_file

2. WRITE — เขียน/แก้ไขข้อมูล (เปลี่ยนแปลงโลก)
   create_file, update_database, send_email, post_to_api

3. EXECUTE — รัน code/commands
   python_repl, bash, run_tests

4. PERCEIVE — รับรู้สิ่งแวดล้อม
   screenshot, read_screen, listen_audio

5. COORDINATE — จัดการ agents อื่น
   spawn_agent, call_agent, handoff_to
```

**Blast Radius:**
- READ → ปลอดภัย ย้อนกลับได้เสมอ
- WRITE → ระวัง อาจแก้ไขยาก (email ส่งแล้วส่งเลย)
- EXECUTE → อันตราย อาจมีผลข้างเคียงกว้าง
- COORDINATE → ระวัง cascade failure ใน multi-agent

#### Action — วงจร perception-action

```python
class AgentLoop:
    def run(self, goal: str) -> str:
        self.memory.add(goal)

        while not self.is_done():
            state = self.memory.get_context()
            action = self.llm.decide(state)

            if action.type == "finish":
                return action.result

            result = self.tools.execute(action)
            self.memory.add(result)
            # → กลับไปดู state อีกครั้ง
```

### 3. Context Window — ทรัพยากรที่มีค่าที่สุด

**Token Budget:**
- claude-sonnet-4-6 = 200,000 tokens ≈ 150,000 words ≈ หนังสือ 600 หน้า
- แต่ในทางปฏิบัติ: system prompt, tool definitions, history, tool results, output buffer

**ปัญหาที่เกิดจาก Context เต็ม:**
1. **Lost in the middle** — LLM attention ลดลงสำหรับข้อมูลตรงกลาง
2. **Context Overflow** — token เกิน limit → API error
3. **Latency เพิ่ม** — token มากขึ้น → inference ช้าลง + ราคาสูงขึ้น

**Strategies จัดการ Context:**
- Context Compression — ย่อ history
- Selective Retrieval — ดึงเฉพาะที่ต้องการ
- Tool Result Truncation — ตัด output ของ tool
- Hierarchical Summarization — สรุปหลายชั้น

### 4. วิธีที่ Agent ล้มเหลว (Failure Modes)

**6 Failure Modes หลัก:**

1. **Hallucinated Tool Calls** — LLM generate tool ที่ไม่มีอยู่จริง
   - ป้องกัน: validate tool name, ใช้ native function calling

2. **Infinite Loop** — Agent วน loop ไม่จบ
   - ป้องกัน: max_iterations, detect repeated actions, timeout

3. **Context Poisoning** — Tool result ที่มี prompt injection
   - ป้องกัน: sandwich prompting, input validation, least privilege

4. **Action Cascade** — Agent A สั่ง B สั่ง C → action ลูกโซ่
   - ป้องกัน: human approval, dry-run mode, limit depth

5. **Reward Hacking** — Agent หา shortcut ที่ optimize metric แต่ไม่ได้ผลลัพธ์
   - ป้องกัน: ระบุ goal ชัดเจน + สิ่งที่ห้ามทำ, evaluator แยก

6. **Sycophancy** — Agent เห็นด้วยกับ user มากเกินไป
   - ป้องกัน: system prompt "ให้ข้อมูลที่ถูกต้อง ไม่ใช่ที่ user อยากได้ยิน"

### 5. System Prompt สำหรับ Agent

**โครงสร้างที่แนะนำ:**

```
[1. Identity & Role]
คุณเป็น HR Assistant ของบริษัท Acme Corp

[2. Capabilities]
คุณมีความสามารถ:
- ค้นหาเอกสาร HR ด้วย search_hr_docs tool
- คำนวณวันลาด้วย calculate_leave tool

[3. Constraints — สำคัญมาก]
คุณ MUST NOT:
- เปิดเผยข้อมูลส่วนตัวของพนักงานคนอื่น
- แก้ไขข้อมูลโดยไม่ได้รับอนุมัติ

[4. Behavior Guidelines]
- ถ้าไม่มั่นใจ ให้บอกว่าไม่ทราบ ไม่ต้องเดา
- อ้างอิง section/หน้า ของ document เสมอ

[5. Output Format]
ตอบเป็น markdown พร้อม source citation
```

**Anti-patterns:**
- แย่: "You are a helpful assistant." (ไม่บอก constraints ชัด)
- แย่: ยาวเกินไป 5,000 tokens (LLM อาจลืม instructions ตรงกลาง)
- ดี: สั้น ชัด มี constraints

### 6. Production Concerns

**Observability:**
- ต้อง trace ทุก step ของ agent ใน production
- Tools: LangSmith, Langfuse, Arize Phoenix, OpenTelemetry

**Guardrails:**
- Input: ตรวจ prompt injection, filter toxic content, validate permissions
- Output: ตรวจ PII, hallucination, harmful content
- Tool: validate arguments, check permissions, confirm irreversible actions

**Human-in-the-Loop:**
```python
REQUIRE_HUMAN_APPROVAL = [
    "send_email", "delete_files", "update_production_db", "post_social_media"
]
```

**Cost Control:**
- Token ราคาไม่ถูก ต้องควบคุม
- Strategies: Cache, ใช้ LLM เล็กสำหรับ simple tasks, Prompt caching, จำกัด max_iterations

**Retry & Fallback:**
- ใช้ exponential backoff สำหรับ rate limits
- Fallback to smaller model ถ้า primary unavailable

### 7. เมื่อไหร่ควร / ไม่ควรใช้ Agent

**Decision Framework:**

```
ต้องการข้อมูลจาก external source มั้ย?
  NO → ใช้แค่ LLM call ธรรมดา
  YES ↓

steps ที่ต้องทำ รู้ล่วงหน้าหมดมั้ย?
  YES + simple → ใช้ RAG pipeline ธรรมดา
  YES + complex → Plan-and-Execute
  NO ↓

ต้องตัดสินใจระหว่างทางว่าจะ retrieve อะไร?
  YES → ReAct Agent หรือ Agentic RAG
  NO ↓

ต้องการหลาย specialists ทำงานพร้อมกัน?
  YES → Multi-Agent System
  NO → Single Agent ก็พอ
```

**ตารางเปรียบเทียบ:**

| Use Case | แนะนำ | หลีกเลี่ยง |
|---|---|---|
| FAQ / simple Q&A | Simple RAG | Agent (overkill) |
| Multi-document synthesis | Agentic RAG | Simple RAG |
| Coding assistant | ReAct + code tool | CoT only |
| Customer service routing | Swarm handoffs | Monolithic agent |
| Research report | Orchestrator-Worker | Single agent |
| Latency-critical | Direct LLM / pipeline | Agent (ช้าเกิน) |
| Fully deterministic | Simple pipeline | Agent (unpredictable) |

## ข้อมูล / หลักฐาน ที่น่าสนใจ

### Agent Loop Diagram

```
START → Build Context → LLM Call → stop_reason?
                              ↓
                    ┌─────────┴─────────┐
               "end_turn"          "tool_use"
                    ↓                    ↓
               RETURN           Execute Tool(s)
                                    ↓
                            Add Observation
                                    ↓
                          iteration limit?
                          yes → ERROR  no → LLM Call
```

### Evaluation Metrics

| Metric | วัดอะไร | วิธีวัด |
|---|---|---|
| Task Success Rate | งานสำเร็จกี่ % | binary: success/fail |
| Step Efficiency | ใช้ tool calls กี่ครั้ง | count steps |
| Accuracy | คำตอบถูกต้องแค่ไหน | human eval |
| Hallucination Rate | พูดสิ่งที่ไม่จริงบ่อยแค่ไหน | factual verification |
| Latency | ใช้เวลาต่องานกี่วิ | timer |
| Cost | ราคาต่องาน | token count × price |
| Safety | ทำสิ่งที่ไม่ควรทำมั้ย | red-team testing |

## Concepts ที่เกี่ยวข้อง

- [[Agent]] - LLM Agent architecture
- [[ReAct]] - Reasoning + Acting pattern
- [[Memory]] - Agent memory systems
- [[Planning]] - Agent planning approaches
- [[Tools]] - Agent tools and actions
- [[Context Window]] - LLM memory limit
- [[System Prompt]] - Agent instructions
- [[Observability]] - Tracing agent behavior
- [[Guardrails]] - Safety measures for agents
- [[Multi-Agent Systems]] - Multiple agents working together
