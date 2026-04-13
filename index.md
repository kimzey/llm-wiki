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
