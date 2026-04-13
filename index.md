# Index

วันที่อัปเดตล่าสุด: 2026-04-13

---

## Sources (`wiki/sources/`)

| หน้า | สรุปสั้น | Source file | วันที่ |
|------|----------|-------------|--------|
| [[wiki/sources/distributed-tracing-context-propagation\|Distributed Tracing & Context Propagation]] | ภาพรวมและ implementation ของ Distributed Tracing + Context Propagation ใน Go พร้อม OTel components ครบ | [[wiki/sources/distributed-tracing-context-propagation]] | 2026-04-13 |
| [[wiki/sources/otel-trace-propagation-migration-guide\|OTel Trace Propagation — Migration Guide]] | คู่มือ migration แก้ trace ขาดใน Go services (gRPC, HTTP) พร้อม diff-style code ทุกกรณี | [[wiki/sources/otel-trace-propagation-migration-guide]] | 2026-04-13 |
| [[wiki/sources/opentelemetry-deep-dive\|OpenTelemetry — Deep Dive]] | คู่มือ Deep Dive ครบทุก layer ของ OTel: observability, 3 pillars, architecture, sampling, Jaeger | [[wiki/sources/opentelemetry-deep-dive]] | 2026-04-13 |
| [[wiki/sources/opentelemetry-deep-dive-part2\|OpenTelemetry — Deep Dive Part 2]] | เจาะลึก TextMapCarrier, TextMapPropagator internals, BatchSpanProcessor, Jaeger assembly | [[wiki/sources/opentelemetry-deep-dive-part2]] | 2026-04-13 |
| [[wiki/sources/fiber-context-otel-bug\|Fiber Context OTel Bug]] | bug analysis: c.UserContext() vs getSpanContext() — เหตุที่ trace ไม่ propagate ใน Fiber | [[wiki/sources/fiber-context-otel-bug]] | 2026-04-13 |
| [[wiki/sources/settextmappropagator-explained\|SetTextMapPropagator Explained]] | อธิบายทุกชิ้นของ SetTextMapPropagator: global propagator, TraceContext, Baggage และ end-to-end flow | [[wiki/sources/settextmappropagator-explained]] | 2026-04-13 |
| [[wiki/sources/ai-context-phase1\|AI Context Phase 1: โครงสร้าง .claude และ .opencode]] | ภาพรวมโครงสร้าง .claude/, CLAUDE.md, rules/, settings, hooks — หัวใจของ AI-assisted development | [[wiki/sources/ai-context-phase1]] | 2026-04-13 |
| [[wiki/sources/ai-context-phase2\|AI Context Phase 2: Agents, Commands, Skills]] | 4 Project Agents, 9 Commands, 9 Skills — ระบบ AI Workflow อัตโนมัติ + Jira/Outline integration | [[wiki/sources/ai-context-phase2]] | 2026-04-13 |
| [[wiki/sources/ai-context-phase3\|AI Context Phase 3: .opencode, MCP, Security]] | OpenCode CLI, MCP Servers (Jira/Outline/GitLab), security hooks, clone boilerplate process | [[wiki/sources/ai-context-phase3]] | 2026-04-13 |
| [[wiki/sources/ai-context-phase4\|AI Context Phase 4: Bindings — ใครผูกกับใคร]] | ความสัมพันธ์ระหว่าง Agent/Skill/Command — ไม่ผูกตรง แต่ผูกผ่าน shared context + Claude เป็นตัวกลาง | [[wiki/sources/ai-context-phase4]] | 2026-04-13 |
| [[wiki/sources/ai-context-phase5\|AI Context Phase 5: Deep Dive]] | Settings Cascade (3 ระดับ), Hook Lifecycle, Source Code Map, Global Plugins, Gotchas | [[wiki/sources/ai-context-phase5]] | 2026-04-13 |
| [[wiki/sources/ai-context-phase6\|AI Context Phase 6: Dispatch System]] | ระบบตัดสินใจของ Claude — ใครเลือก Skill/Agent/Command และทำไมถึงไม่ deterministic | [[wiki/sources/ai-context-phase6]] | 2026-04-13 |
| [[wiki/sources/ai-context-phase7\|AI Context Phase 7: Global Agents + Practical Manual]] | Global Agents 10 ตัว, Auto Memory, Plan Mode, Context Window + 10 recipes + Troubleshooting | [[wiki/sources/ai-context-phase7]] | 2026-04-13 |
| [[wiki/sources/ai-context-phase8\|AI Context Phase 8: Deterministic README Generator]] | ออกแบบระบบ AI gen README แบบ deterministic ด้วย Fixed Template + Data Extraction + Constraint Prompting | [[wiki/sources/ai-context-phase8]] | 2026-04-13 |

---

## Concepts (`wiki/concepts/`)

| หน้า | สรุปสั้น | Tags |
|------|----------|------|
| [[wiki/concepts/distributed-tracing\|Distributed Tracing]] | ระบบติดตาม request ข้ามหลาย service — Trace, Span, TraceID, SpanID, waterfall view | distributed-tracing, microservices, jaeger |
| [[wiki/concepts/context-propagation\|Context Propagation]] | กลไก inject/extract TraceID ข้าม service ผ่าน network header — TextMapPropagator, TextMapCarrier | context-propagation, textmappropagator, w3c |
| [[wiki/concepts/opentelemetry\|OpenTelemetry]] | vendor-neutral framework สำหรับ traces, metrics, logs — 3 pillars of observability, OTel Collector | opentelemetry, observability, otel-collector |
| [[wiki/concepts/w3c-trace-context\|W3C TraceContext]] | standard format ของ traceparent header — TraceID, SpanID, sampling flags (W3C RFC 2020) | w3c, traceparent, standard |
| [[wiki/concepts/otel-baggage\|OTel Baggage]] | ส่ง key-value metadata ข้าม service ผ่าน baggage header — ต่างจาก Span Attribute ตรงที่ service อื่นดึงใช้ใน code ได้ | baggage, metadata, multi-tenant |
| [[wiki/concepts/otel-go-instrumentation\|OTel Go Instrumentation]] | instrument OTel ใน Go: TracerProvider, TextMapPropagator, otelgrpc, otelhttp, otelfiber, StatsHandler | go, otelgrpc, otelhttp, otelfiber |
| [[wiki/concepts/claude-code-ai-cli\|Claude Code AI CLI]] | Claude Code CLI tool — CLAUDE.md, rules/, settings cascade, hooks; ระบบให้ AI เข้าใจโปรเจกต์โดยอัตโนมัติ | claude-code, ai-cli, settings, hooks |
| [[wiki/concepts/ai-agents-system\|AI Agents System]] | ระบบ Agent บทบาทใน Claude Code — 4 project agents + 10 global agents, tools restriction, subprocess isolation | claude-code, agents, subprocess, roles |
| [[wiki/concepts/commands-and-skills\|Commands and Skills]] | Commands (user-invoked) vs Skills (AI-detected) — คู่แฝดที่มีเนื้อหาเหมือนกัน trigger ต่างกัน | claude-code, commands, skills, slash-commands |
| [[wiki/concepts/mcp-model-context-protocol\|MCP Model Context Protocol]] | มาตรฐาน "USB port" สำหรับ AI เชื่อม external services (Jira, Outline, GitLab) — HTTP/SSE/stdio | mcp, jira, outline, external-services |
| [[wiki/concepts/ai-hooks-system\|AI Hooks System]] | ระบบ trigger อัตโนมัติก่อน/หลัง tool call — security (.env protection), quality control, workflow automation | hooks, security, automation, pre-tool-use |
| [[wiki/concepts/ai-dispatch-system\|AI Dispatch System]] | กลไกตัดสินใจของ Claude — ไม่มี routing table, ใช้ language understanding อ่าน description เลือก Skill/Agent | dispatch, ai-behavior, non-deterministic |
| [[wiki/concepts/deterministic-ai-generation\|Deterministic AI Generation]] | แนวทางให้ AI output เหมือนกันทุกครั้ง: Fixed Template + Data Extraction + Constraint Prompting | ai-generation, deterministic, prompt-engineering |
| [[wiki/concepts/clean-architecture-go\|Clean Architecture in Go]] | สถาปัตยกรรม 4 layer (Entity → UseCase → Repository → Interface) สำหรับ Go microservices | clean-architecture, go, microservices, layered |

---

## Books (`wiki/books/`)

| หน้า | ผู้แต่ง | Tags |
|------|---------|------|
| _(ยังว่างอยู่)_ | | |

---

## Synthesis (`wiki/synthesis/`)

| หน้า | สรุปสั้น | วันที่ |
|------|----------|--------|
| _(ยังว่างอยู่)_ | | |
