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
| [[wiki/sources/arona-overview\|Arona — ระบบ RAG สำหรับ Elysia Documentation]] | Self-hosted RAG ด้วย ParadeDB + DragonflyDB + GPT OSS ราคา ~$21/เดือน: architecture, tech stack, 7-step request flow, caching 3 ชั้น | [[wiki/sources/arona-overview]] | 2026-04-13 |
| [[wiki/sources/arona-rag-techniques\|Arona RAG — เทคนิคและการเปรียบเทียบแนวทาง]] | เปรียบเทียบ DIY/LangChain/LlamaIndex/Off-the-shelf + เทคนิค cost optimization: semantic cache, incremental index, hybrid search | [[wiki/sources/arona-rag-techniques]] | 2026-04-13 |
| [[wiki/sources/arona-vs-langchain\|Arona vs LangChain]] | เปรียบเทียบ DIY vs Framework ทุกมิติ: latency, memory, throughput, security, maintainability — decision matrix | [[wiki/sources/arona-vs-langchain]] | 2026-04-13 |
| [[wiki/sources/arona-tool-call-agent\|Arona — Tool Call และ Agent Loop]] | กลไก Tool Calling, Agent Loop, stop conditions, Flex Mode (exponential backoff retry) ใน Vercel AI SDK | [[wiki/sources/arona-tool-call-agent]] | 2026-04-13 |
| [[wiki/sources/arona-stateless-rag\|Stateless RAG — ทำไม Arona ไม่ต้องการ Agent ที่ซับซ้อน]] | ปรัชญา stateless RAG: ไม่เก็บ session ที่ server, scale horizontal ง่าย, เหมาะกับ documentation search | [[wiki/sources/arona-stateless-rag]] | 2026-04-13 |
| [[wiki/sources/arona-sellsuki-analysis\|วิเคราะห์ RAG สำหรับ Sellsuki]] | feasibility study ใช้ Arona เป็น internal KB สำหรับ HR policy + procedures ผ่าน LINE/Slack/Web | [[wiki/sources/arona-sellsuki-analysis]] | 2026-04-13 |
| [[wiki/sources/arona-deepdive\|Arona Deep Dive — สถาปัตยกรรมและ Ingestion Pipeline]] | ชุดบทเรียน 4 ตอน: architecture, ingestion pipeline (Extract→Chunk→Plan→Embed→Save), retrieval, security | [[wiki/sources/arona-deepdive]] | 2026-04-13 |
| [[wiki/sources/arona-learn-series\|Arona Learn Series — ชุดบทเรียน Ingest Pipeline]] | ชุดบทเรียน 14 ตอน เจาะลึก ingest service: chunking, planning (diff-aware), vectorization, storage | [[wiki/sources/arona-learn-series]] | 2026-04-13 |
| [[wiki/sources/sellsuki-design-tokens-service\|Sellsuki Design Tokens Service — Complete Documentation]] | สถาปัตยกรรม 2 ระบบคู่ขนาน (CSS Variables + TS Types), data flow dev/build/runtime, troubleshooting, real-world examples | [[wiki/sources/sellsuki-design-tokens-service]] | 2026-04-13 |
| [[wiki/sources/sellsuki-design-tokens-flow\|Sellsuki Design Tokens — Flow การทำงาน]] | อธิบาย separation of concerns: CSS Variables (runtime) vs TypeScript Types (devtime), checklist เพิ่ม token ใหม่ | [[wiki/sources/sellsuki-design-tokens-flow]] | 2026-04-13 |
| [[wiki/sources/sellsuki-design-tokens-reference\|Sellsuki Design Tokens — Spacing System Reference]] | token reference ครบ: spacing 21 levels, border-radius 11, border-width 4 + migration guide จาก px + best practices | [[wiki/sources/sellsuki-design-tokens-reference]] | 2026-04-13 |
| [[wiki/sources/sellsuki-components-docs\|Sellsuki Components — Service Documentation]] | Web Components Library (Lit+TS): architecture, 3 core systems (Theme/I18n/Toast), 30+ component catalog, usage guide | [[wiki/sources/sellsuki-components-docs]] | 2026-04-13 |

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
| [[wiki/concepts/rag-retrieval-augmented-generation\|RAG — Retrieval-Augmented Generation]] | เทคนิค AI ค้นหาเอกสารก่อนตอบ: pipeline (Ingest→Retrieve→Generate), chunking, embedding, 3 approaches (DIY/Framework/SaaS) | rag, ai, retrieval, embeddings, vector-search |
| [[wiki/concepts/hybrid-search-bm25-vector\|Hybrid Search — BM25 + Vector Search]] | รวม keyword search (BM25) กับ semantic search (cosine similarity) ในฐานข้อมูลเดียว: ParadeDB, scoring formula, ทำไมไม่ใช้ re-ranking | search, bm25, vector-search, paradedb, semantic |
| [[wiki/concepts/semantic-caching\|Semantic Caching]] | cache คำตอบด้วย vector similarity ≥ 90% แทน exact match — ลด LLM cost + latency, DragonflyDB KNN | caching, semantic, vector-search, redis, cost-optimization |
| [[wiki/concepts/stateless-rag-design\|Stateless RAG Design]] | ออกแบบ RAG ไม่เก็บ session ที่ server: horizontal scale ง่าย, เหมาะกับ documentation search, hybrid client-side history | stateless, rag, scaling, design-philosophy, architecture |
| [[wiki/concepts/design-tokens\|Design Tokens]] | ค่า design decisions ในรูปตัวแปร CSS/TS: 2-tier architecture (Primitive→Semantic), Single Source of Truth, ทำให้แก้ค่าจากจุดเดียว | design-tokens, css-variables, design-system, frontend |
| [[wiki/concepts/web-components-lit\|Web Components & Lit Framework]] | Web Standard สำหรับ custom elements ที่ใช้ได้ทุก framework: Shadow DOM, Reactive Properties, Context System, Slot Pattern | web-components, lit, shadow-dom, framework-agnostic |

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
