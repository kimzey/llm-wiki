# Index

วันที่อัปเดตล่าสุด: 2026-04-13 (4)

---

## Sources (`wiki/sources/`)

| หน้า | สรุปสั้น | Source file | วันที่ |
|------|----------|-------------|--------|
| [[wiki/sources/homeserver-admin-knowledge\|Home Server Admin — ความรู้ที่ต้องรู้]] | คู่มือครอบคลุมทุกเรื่อง Home Server (NUC + Ubuntu + Docker): Linux, Docker, Security, Backup, Monitoring, Troubleshooting, Production Checklist | [[wiki/sources/homeserver-admin-knowledge]] | 2026-04-13 |
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
| [[wiki/sources/haystack-phase1-overview\|Haystack Phase 1 — รู้จัก Haystack & Core Concepts]] | Haystack framework intro: Component-Pipeline-DocumentStore architecture, Haystack 2.x, Hello World RAG, เปรียบเทียบกับ LangChain/LlamaIndex | [[wiki/sources/haystack-phase1-overview]] | 2026-04-13 |
| [[wiki/sources/haystack-phase2-indexing\|Haystack Phase 2 — Document Processing & Indexing Pipeline]] | File converters (PDF/HTML/DOCX), DocumentCleaner, DocumentSplitter (chunk strategies + overlap), Embedding models สำหรับไทย | [[wiki/sources/haystack-phase2-indexing]] | 2026-04-13 |
| [[wiki/sources/haystack-phase3-retrieval\|Haystack Phase 3 — Embedding, Retrievers & Query Pipeline]] | BM25/Semantic/Hybrid Retrieval, DocumentJoiner(RRF), Cross-encoder Ranker, Metadata Filtering, PromptBuilder (Jinja2), Generators | [[wiki/sources/haystack-phase3-retrieval]] | 2026-04-13 |
| [[wiki/sources/haystack-phase4-advanced\|Haystack Phase 4 — Advanced Features (Agents, Routers, Conversational RAG)]] | Custom Components (@component), Routers, Agent+ToolInvoker loop, Conversational RAG + history, Streaming, YAML serialization | [[wiki/sources/haystack-phase4-advanced]] | 2026-04-13 |
| [[wiki/sources/haystack-phase5-production\|Haystack Phase 5 — Custom Components, Production & Integrations]] | FastAPI integration, RAG Evaluation (RAGAS), Caching, Ollama/Bedrock/Gemini/pgvector integrations, component selection guide | [[wiki/sources/haystack-phase5-production]] | 2026-04-13 |
| [[wiki/sources/haystack-phase6-observability\|Haystack Phase 6 — API, Auth, Observability & Ecosystem]] | Hayhooks API serving, JWT/RBAC auth, Langfuse tracing, OTel, Prometheus+Grafana, Guardrails, Haystack vs LangChain ตรงๆ | [[wiki/sources/haystack-phase6-observability]] | 2026-04-13 |
| [[wiki/sources/rag-decision-guide\|RAG Framework Decision Guide — Sellsuki]] | คู่มือตัดสินใจเลือก RAG framework: DIY vs LangChain vs LlamaIndex vs Hybrid, พร้อม chunking strategies, ParadeDB, Dragonfly caching, incremental update | [[wiki/sources/rag-decision-guide]] | 2026-04-13 |
| [[wiki/sources/langchain-llamaindex-deep-dive\|LangChain & LlamaIndex — Deep Dive Complete Guide]] | Deep Dive ทั้ง 2 frameworks ตั้งแต่ศูนย์: Building Blocks (LLM, Prompt, Chain, Memory, Tools, Agent), LCEL syntax, IngestionPipeline, QueryEngine | [[wiki/sources/langchain-llamaindex-deep-dive]] | 2026-04-13 |
| [[wiki/sources/langchain-rag-guide\|LangChain, RAG & Sellsuki Knowledge Base — Deep Dive]] | LangChain Ecosystem ทั้ง 5 ตัว (LangChain, LangGraph, LangSmith, LangServe, Hub), RAG Pipeline ละเอียด, Chunking Strategies, Semantic Cache | [[wiki/sources/langchain-rag-guide]] | 2026-04-13 |
| [[wiki/sources/llamaindex-deep-dive\|LlamaIndex & RAG Deep Dive — Sellsuki Knowledge Bot]] | LlamaIndex Ecosystem (LlamaHub, LlamaParse, LlamaTrace), 4 Chunking Strategies, Metadata Schema, ParadeDB Design, Hybrid Search | [[wiki/sources/llamaindex-deep-dive]] | 2026-04-13 |
| [[wiki/sources/llamaindex-full-guide\|LlamaIndex Full Guide — Sellsuki RAG System]] | Full guide 7-phase checklist, Deliverables, Server Setup (FastAPI + Docker), TypeScript vs Python comparison, Full Architecture 8-step Query Flow | [[wiki/sources/llamaindex-full-guide]] | 2026-04-13 |
| [[wiki/sources/llamaindex-phase1-introduction\|LlamaIndex Phase 1 — Introduction & Core Concepts]] | บทช่วยสอน Phase 1: LlamaIndex vs LangChain, 5 Building Blocks, Documents & Nodes, Settings object, RAG 4 บรรทัด, Query Pipeline | [[wiki/sources/llamaindex-phase1-introduction]] | 2026-04-13 |
| [[wiki/sources/llamaindex-phase2-ingestion-indexing\|LlamaIndex Phase 2 — Data Ingestion & Indexing]] | บทช่วยสอน Phase 2: LlamaHub 100+ loaders, Node Parsers, IngestionPipeline, Index Types, Vector Stores, Persist & Load | [[wiki/sources/llamaindex-phase2-ingestion-indexing]] | 2026-04-13 |
| [[wiki/sources/llamaindex-phase3-querying-rag\|LlamaIndex Phase 3 — Querying, Retrieval & RAG Pipeline]] | บทช่วยสอน Phase 3: Retrievers, Postprocessors, Response Modes, Query Engines, Chat Engines, Advanced RAG (HyDE, Sentence Window, RAG Fusion), Evaluation | [[wiki/sources/llamaindex-phase3-querying-rag]] | 2026-04-13 |
| [[wiki/sources/llamaindex-phase4-agents-production\|LlamaIndex Phase 4 — Advanced Features, Agents & Production]] | บทช่วยสอน Phase 4: Agents (ReAct, OpenAI), Workflows, Multi-Modal, SQL, Fine-tuning, Caching, FastAPI, Production Stack | [[wiki/sources/llamaindex-phase4-agents-production]] | 2026-04-13 |
| [[wiki/sources/rag-knowledge-overview\|RAG Agent Knowledge — Overview & Statistics]] | ภาพรวม 12 เอกสาร RAG Agent (~300 หน้า), reading paths 4 routes, quick reference table | [[wiki/sources/rag-knowledge-overview]] | 2026-04-13 |
| [[wiki/sources/rag-complete-knowledge\|Complete RAG & Agent Knowledge Base]] | ครบทุก LLM, Prompt Engineering, Embedding, Vector DB, RAG, Advanced RAG, Agent, Agentic Patterns, Multi-Agent, Evaluation, Production | [[wiki/sources/rag-complete-knowledge]] | 2026-04-13 |
| [[wiki/sources/rag-glossary-data-prep\|RAG Glossary & Data Preparation — Complete Guide]] | คำศัพท์ครบ + Data Prep: Core Concepts, Search (Cosine, BM25, Hybrid, Re-rank), Chunking, Index (HNSW, IVFFlat), Caching, Anti-abuse | [[wiki/sources/rag-glossary-data-prep]] | 2026-04-13 |
| [[wiki/sources/chunking-deepdive-complete\|Chunking Deep Dive — ทุกวิธี + ข้อมูลจริง]] | 10 วิธี Chunking + Output จริง, database schema, linking chunks, best practices 2025, decision matrix, common mistakes | [[wiki/sources/chunking-deepdive-complete]] | 2026-04-13 |
| [[wiki/sources/rag-vs-agent-explained\|RAG vs Agent — มันคืออะไร ต่างกันยังไง]] | อธิบายความแตกต่าง 5 levels, flow diagrams, comparison tables, spectrum, use cases สำหรับ Sellsuki | [[wiki/sources/rag-vs-agent-explained]] | 2026-04-13 |
| [[wiki/sources/sellsuki-rag-diy-guide\|DIY — สร้าง RAG ตอบคำถาม Sellsuki]] | Use case (before/after), tech stack (ParadeDB, Dragonfly), implementation guide, แนวทางการทำ | [[wiki/sources/sellsuki-rag-diy-guide]] | 2026-04-13 |
| [[wiki/sources/rag-integration-guide-complete\|RAG Agent Integration — Deploy แล้วต่ออะไรได้บ้าง]] | API Endpoints, MCP สำหรับ Claude Code/Cursor, Open Source Tools, 14 Use Cases (Chat, Search, Form, Report, Email, etc.) | [[wiki/sources/rag-integration-guide-complete]] | 2026-04-13 |
| [[wiki/sources/rag-client-channels-complete\|RAG Agent — ช่องทางเชื่อมต่อทั้งหมด]] | 14 ช่องทาง (LINE, Slack, Discord, Web, Messenger, Teams, Telegram, WhatsApp, Mobile, Dashboard, Email, Voice, API) พร้อม code ตัวอย่าง | [[wiki/sources/rag-client-channels-complete]] | 2026-04-13 |
| [[wiki/sources/step1-langchain-basics\|Step 1: LangChain พื้นฐาน]] | บทช่วยสอน LangChain ตั้งแต่ศูนย์: การติดตั้ง, Model, Messages, Prompt Templates | [[wiki/sources/step1-langchain-basics]] | 2026-04-13 |
| [[wiki/sources/step2-tools-agents\|Step 2: Tools & Agents]] | บทช่วยสอน Tools และ Agents: สร้าง Tool ด้วย @tool, bind_tools, create_agent, ReAct Loop | [[wiki/sources/step2-tools-agents]] | 2026-04-13 |
| [[wiki/sources/step3-rag-tutorial\|Step 3: RAG — Retrieval-Augmented Generation]] | บทช่วยสอน RAG ตั้งแต่ศูนย์: Pipeline, Chunking, Embedding, Vector Store | [[wiki/sources/step3-rag-tutorial]] | 2026-04-13 |
| [[wiki/sources/step4-langgraph-tutorial\|Step 4: LangGraph — ควบคุม Flow ด้วย Graph]] | บทช่วยสอน LangGraph: State, Nodes, Edges, Tools, Conditional Edges, Checkpointing | [[wiki/sources/step4-langgraph-tutorial]] | 2026-04-13 |
| [[wiki/sources/step5-advanced-rag-tutorial\|Step 5: Advanced RAG]] | บทช่วยสอน Advanced RAG: Hybrid Search, Re-ranking, Multi-query, Conversational RAG | [[wiki/sources/step5-advanced-rag-tutorial]] | 2026-04-13 |
| [[wiki/sources/step6-multi-agent-tutorial\|Step 6: Multi-Agent Systems]] | บทช่วยสอน Multi-Agent: Supervisor, Handoff, Pipeline patterns | [[wiki/sources/step6-multi-agent-tutorial]] | 2026-04-13 |
| [[wiki/sources/step7-evaluation-tutorial\|Step 7: Evaluation & Observability]] | บทช่วยสอน Evaluation: Phoenix, LangFuse, Rule-based, LLM-as-Judge | [[wiki/sources/step7-evaluation-tutorial]] | 2026-04-13 |
| [[wiki/sources/step8-deployment-tutorial\|Step 8: Deployment]] | บทช่วยสอน Deployment: LangServe, FastAPI, Docker, Cloud Deploy | [[wiki/sources/step8-deployment-tutorial]] | 2026-04-13 |
| [[wiki/sources/langchain-advanced-guide\|LangChain Advanced Guide]] | LangGraph, LangSmith & RAG อย่างละเอียด | [[wiki/sources/langchain-advanced-guide]] | 2026-04-13 |
| [[wiki/sources/langchain-deep-dive\|LangChain Deep Dive]] | ทุกสิ่งที่ต้องรู้เพื่อใช้งานจริง: package structure, models, messages, prompts, tools | [[wiki/sources/langchain-deep-dive]] | 2026-04-13 |
| [[wiki/sources/langgraph-deep-dive\|LangGraph Deep Dive]] | ควบคุม Agent Flow อย่างละเอียด: state, nodes, edges, multi-agent | [[wiki/sources/langgraph-deep-dive]] | 2026-04-13 |
| [[wiki/sources/langsmith-deep-dive\|LangSmith Deep Dive]] | Observability & Evaluation Platform: tracing, debugging, evaluation, monitoring | [[wiki/sources/langsmith-deep-dive]] | 2026-04-13 |
| [[wiki/sources/lang-ecosystem-complete\|Lang Ecosystem ทั้งหมด]] | ครบทุกตัวในตระกูล Lang: LangChain, LangGraph, LangSmith, LangServe, LangChain.js | [[wiki/sources/lang-ecosystem-complete]] | 2026-04-13 |
| [[wiki/sources/multi-agent-deep-dive\|Multi-Agent Systems Deep Dive]] | สร้าง Agent หลายตัวทำงานร่วมกัน: patterns 5 แบบ, shared state, examples | [[wiki/sources/multi-agent-deep-dive]] | 2026-04-13 |
| [[wiki/sources/deployment-deep-dive\|Deployment Deep Dive]] | Deploy Agent ขึ้น Cloud: LangServe, LangGraph Platform, FastAPI, Docker, CI/CD | [[wiki/sources/deployment-deep-dive]] | 2026-04-13 |
| [[wiki/sources/advanced-rag-deep-dive\|Advanced RAG Deep Dive]] | Hybrid Search, Re-ranking, Multi-query และเทคนิคขั้นสูงทั้งหมด | [[wiki/sources/advanced-rag-deep-dive]] | 2026-04-13 |

---

## Concepts (`wiki/concepts/`)

| หน้า | สรุปสั้น | Tags |
|------|----------|------|
| [[wiki/concepts/docker-containers\|Docker Containers]] | Containerization technology สำหรับ deploy app: Image, Container, Docker Compose, Best Practices, Security, Monitoring | docker, containers, orchestration, devops |
| [[wiki/concepts/linux-system-administration\|Linux System Administration]] | จัดการ Linux server ผ่าน command line: file management, permissions, processes, networking, systemd, troubleshooting | linux, sysadmin, command-line, ubuntu |
| [[wiki/concepts/server-security\|Server Security]] | Security hardening ทีละชั้น: SSH hardening, Firewall (UFW), Fail2ban, Automatic Updates, SSL Certificates, Monitoring | security, hardening, ssh, firewall, fail2ban |
| [[wiki/concepts/backup-strategy\|Backup Strategy]] | 3-2-1 backup rule: automated scripts, cloud sync (rclone), retention policies, disaster recovery, testing restores | backup, 3-2-1-rule, disaster-recovery, rclone |
| [[wiki/concepts/systemd-services\|Systemd Services]] | Service management บน Ubuntu: systemctl, journalctl, custom services, timers, resource limits, troubleshooting | systemd, services, process-management, ubuntu |
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
| [[wiki/concepts/haystack-framework\|Haystack Framework]] | Open-source Python framework สร้าง Production-ready AI Search & RAG Pipelines: Component-Pipeline-DocumentStore, type-safe, YAML serializable | haystack, rag, pipeline, nlp, deepset, python |
| [[wiki/concepts/rag-evaluation\|RAG Evaluation]] | วัดคุณภาพ RAG 3 มิติ: Retrieval (Context Relevance, MRR), Generation (Faithfulness), End-to-end (Exact Match, SAS) — tools: RAGAS, Langfuse | rag, evaluation, faithfulness, ragas, llm-judge |
| [[wiki/concepts/langchain-framework\|LangChain Framework]] | General-purpose LLM Framework: Chains, Agents, Memory, Retrievers — Ecosystem 5 ตัว (LangChain, LangGraph, LangSmith, LangServe, Hub), LCEL syntax, Redis Semantic Cache | langchain, rag, llm, agent, chain, lcel, python, typescript |
| [[wiki/concepts/llamaindex-framework\|LlamaIndex Framework]] | Data Framework สำหรับ RAG: Index, Query, Retrieval — IngestionPipeline + Cache, LlamaParse (PDF ซับซ้อน), 300+ LlamaHub connectors | llamaindex, rag, llm, ingestion-pipeline, python, typescript, llamaparse |
| [[wiki/concepts/rag-chunking-strategies\|RAG Chunking Strategies]] | วิธีแบ่ง chunks: Fixed-Size, Semantic, Markdown Header, Hierarchical/Parent-Child, Sentence Window — เปรียบเทียบ LangChain vs LlamaIndex | rag, chunking, llamaindex, langchain, semantic-split, hierarchical-split |
| [[wiki/concepts/llm-large-language-model\|LLM — Large Language Model]] | โมเดล AI ทำนาย "คำถัดไป": Token, Context Window, Temperature, System Prompt, Hallucination, Grounding, Completion vs Chat | llm, ai, nlp, machine-learning, gpt, claude |
| [[wiki/concepts/ai-agent\|AI Agent]] | ระบบ AI ตัดสินใจเอง: Agent Loop, Tool Selection, Decision Making, ReAct, Plan-and-Execute, Self-Reflection, RAG vs Agent vs Agentic RAG | agent, autonomous, decision-making, tool-calling, planning |
| [[wiki/concepts/agentic-rag\|Agentic RAG]] | RAG + Agency: LLM เลือก search เอง, loop ได้, decision making 4 จุด, comparison กับ RAG ธรรมดา, implementation examples | rag, agent, agentic, decision-making, llm |

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
