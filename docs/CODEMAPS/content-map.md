<!-- Generated: 2026-04-13 | Files scanned: 157 | Token estimate: ~550 -->

# Content Map — LLM Wiki

## Wiki Pages (12 total as of 2026-04-13)

### Sources (`wiki/sources/` — 6 pages)

| Slug | Topic | Tags |
|---|---|---|
| `distributed-tracing-context-propagation` | OTel: Distributed Tracing + Context Propagation overview | distributed-tracing, grpc, go |
| `otel-trace-propagation-migration-guide` | OTel: Migration guide for trace propagation (gRPC/HTTP) | migration, otelgrpc, otelhttp |
| `opentelemetry-deep-dive` | OTel: Full deep dive (18 sections — all layers) | observability, architecture, jaeger |
| `opentelemetry-deep-dive-part2` | OTel: TextMapCarrier, BatchSpanProcessor, Jaeger assembly | textmapcarrier, batchspanprocessor |
| `fiber-context-otel-bug` | OTel: Bug — c.UserContext() vs getSpanContext() in Fiber | fiber, bug, otelfiber |
| `settextmappropagator-explained` | OTel: SetTextMapPropagator every piece explained | textmappropagator, baggage |

### Concepts (`wiki/concepts/` — 6 pages)

| Slug | One-liner |
|---|---|
| `distributed-tracing` | ระบบติดตาม request ข้ามหลาย service — Trace, Span, waterfall |
| `context-propagation` | inject/extract TraceID ข้าม service — TextMapPropagator, TextMapCarrier |
| `opentelemetry` | vendor-neutral framework: 3 pillars, OTel Collector, Sampling |
| `w3c-trace-context` | traceparent header format (W3C RFC 2020) — Inject/Extract flow |
| `otel-baggage` | W3C Baggage: cross-service metadata vs Span Attributes |
| `otel-go-instrumentation` | Go OTel: otelgrpc, otelhttp, otelfiber, StatsHandler pattern |

## Raw Notes Inventory (128 files — NOT YET INGESTED)

| Folder | Files | Domain |
|---|---|---|
| `raw/notes/openrag/` | 31 | OpenRAG system: Docling, OpenSearch, Langflow, RBAC, SDK |
| `raw/notes/arona/` | 29 | Arona RAG: architecture, ingestion, retrieval, tool-call agent loop |
| `raw/notes/LangChain/` | 18 | LangChain: basics → advanced RAG, LangGraph, LangSmith, multi-agent |
| `raw/notes/rag-knowledge/` | 12 | RAG theory: chunking, glossary, agent patterns, Sellsuki analysis |
| `raw/notes/Hetstack/` | 11 | Haystack: phases 1–6 (intro → API, auth, observability) |
| `raw/notes/ai-context/` | 8 | AI coding context: .claude structure, agents, commands, skills, MCP |
| `raw/notes/otel/` | 6 | **INGESTED** ✅ — OpenTelemetry notes |
| `raw/notes/tool-rag/` | 5 | LangChain vs LlamaIndex deep dives, RAG decision guide |
| `raw/notes/liamaindex/` | 4 | LlamaIndex: phases 1–4 |
| `raw/notes/central-component/` | 4 | Design tokens: service, flow, docs |

## Ingest Priority (suggested order)

1. `raw/notes/rag-knowledge/` — foundational RAG theory, likely needed by other topics
2. `raw/notes/LangChain/` — broad framework coverage
3. `raw/notes/openrag/` — specific system (Sellsuki OpenRAG)
4. `raw/notes/arona/` — specific system (Arona)
5. `raw/notes/Hetstack/` + `liamaindex/` + `tool-rag/` — framework comparisons
6. `raw/notes/ai-context/` — meta: AI tooling (lower priority)
7. `raw/notes/central-component/` — design system (separate domain)

## Coverage Gaps

- No `wiki/books/` pages yet
- No `wiki/synthesis/` pages yet
- Domain coverage: only OpenTelemetry (otel) so far; RAG, LLM frameworks entirely uningestted
