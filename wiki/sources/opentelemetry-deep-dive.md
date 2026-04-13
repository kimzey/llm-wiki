---
title: "OpenTelemetry — คู่มือฉบับ Deep Dive"
type: source
source_file: raw/notes/otel/OpenTelemetry — คู่มือฉบับ Deep Dive.md
url: ""
published: 2026-04-13
tags: [opentelemetry, observability, distributed-tracing, architecture, go, jaeger, sampling]
related: [wiki/concepts/opentelemetry.md, wiki/concepts/distributed-tracing.md, wiki/concepts/context-propagation.md, wiki/concepts/w3c-trace-context.md, wiki/concepts/otel-baggage.md, wiki/concepts/otel-go-instrumentation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/otel/OpenTelemetry — คู่มือฉบับ Deep Dive.md|Original file]]

## สรุป

คู่มือ Deep Dive ครบทุก layer ของ OpenTelemetry — เริ่มจาก "ทำไม Observability ถึงสำคัญ" ไปจนถึงรายละเอียดของทุก component ประกอบด้วย: Observability vs Monitoring, 3 Pillars (Logs/Metrics/Traces), Distributed Tracing Architecture, Trace Data Model ทุก field, TracerProvider+Resource, Span Lifecycle, Context Propagation, W3C traceparent spec, Baggage, Instrumentation (Auto vs Manual), Sampling strategies, OTel Collector, Exporters, Jaeger features, Error Handling กับ Tracing, Production Patterns และ Architecture ของโปรเจกต์จริง

## ประเด็นสำคัญ

- **Observability ต่างจาก Monitoring**: Monitoring รู้ว่า "มีปัญหา" แต่ไม่รู้ "ปัญหาคืออะไร" — Observability เห็น root cause ได้ใน 2 นาที
- **3 Pillars of Observability**: Logs (events), Metrics (aggregated numbers), Traces (request journey) — ทั้งสามต้องใช้ร่วมกัน
- **OpenTelemetry** = vendor-neutral framework สำหรับ instrument, collect และ export telemetry data (traces, metrics, logs)
- **Trace Data Model**: Trace = collection of Spans ที่มี TraceID เดียวกัน, Span มี fields: SpanID, ParentSpanID, Name, StartTime, EndTime, Attributes, Events, Status, Links
- **TracerProvider + Resource**: Resource บอก "service นี้คืออะไร" (ServiceName, Version, Environment) → ติดกับทุก span อัตโนมัติ
- **Span Lifecycle**: Start → (Record Events/Attributes/Errors) → End → Exported ผ่าน Exporter ไปยัง Collector/Backend
- **Sampling**: ควบคุมว่าจะ record trace ไหน — ParentBased, TraceIDRatioBased, AlwaysOn, AlwaysOff
- **OTel Collector**: pipeline กลางที่รับ telemetry จาก service, process (filter/transform/batch), และ export ไปยัง backend หลายตัวพร้อมกัน
- **Instrumentation** แบ่งเป็น Auto (lib เช่น otelgrpc, otelhttp จัดการให้) และ Manual (เขียน tracer.Start เอง)

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- ตัวอย่างใน doc: debug DB deadlock ใน 2 นาทีด้วย Jaeger — เห็น `db.error: "deadlock detected"` ใน span attributes
- OTel Collector ทำให้ไม่ต้องแก้ code service เมื่อเปลี่ยน backend (เช่น จาก Jaeger เป็น Grafana Tempo)
- `semconv` (Semantic Conventions) = standard attribute names ที่ทุก vendor เข้าใจ เช่น `service.name`, `db.statement`

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/opentelemetry|OpenTelemetry]]
- [[wiki/concepts/distributed-tracing|Distributed Tracing]]
- [[wiki/concepts/context-propagation|Context Propagation]]
- [[wiki/concepts/w3c-trace-context|W3C TraceContext]]
- [[wiki/concepts/otel-baggage|OTel Baggage]]
- [[wiki/concepts/otel-go-instrumentation|OTel Go Instrumentation]]
