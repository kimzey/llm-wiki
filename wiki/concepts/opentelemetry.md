---
title: "OpenTelemetry"
type: concept
tags: [opentelemetry, observability, telemetry, otel-collector, tracerprovider, sampling]
sources: [OpenTelemetry — คู่มือฉบับ Deep Dive.md, OpenTelemetry — Deep Dive ฉบับ Part 2.md]
related: [wiki/concepts/distributed-tracing.md, wiki/concepts/context-propagation.md, wiki/concepts/otel-go-instrumentation.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

OpenTelemetry (OTel) คือ vendor-neutral open-source framework สำหรับ instrument, collect และ export telemetry data (traces, metrics, logs) — เป็น standard ที่ทุก observability backend รองรับ ทำให้ไม่ต้อง lock-in กับ vendor ใด

## อธิบาย

### 3 Pillars of Observability

| Pillar | คืออะไร | ใช้ทำอะไร |
|---|---|---|
| **Logs** | บันทึก events ที่เกิดขึ้น | debug ปัญหาเฉพาะจุด |
| **Metrics** | ตัวเลข aggregate เช่น RPS, latency p99 | monitoring trend, alerting |
| **Traces** | การเดินทางของ request ข้าม services | debug latency, understand flow |

ทั้งสามต้องใช้ร่วมกัน — Trace ชี้จุด, Metric ยืนยัน scale, Log เจาะรายละเอียด

### OTel Architecture

```
Service (instrumented code)
    ↓ OTLP (OpenTelemetry Protocol)
OTel Collector
    ├── Receiver (รับจาก services)
    ├── Processor (filter, transform, batch)
    └── Exporter (ส่งไป backends)
            ↓              ↓              ↓
          Jaeger      Grafana Tempo    Datadog
```

### Component หลักใน Go SDK

**TracerProvider** = บอกว่า trace data จะถูกส่งไปที่ไหน
```go
tp := trace.NewTracerProvider(
    trace.WithBatcher(exporter),
    trace.WithResource(resource.NewWithAttributes(
        semconv.SchemaURL,
        semconv.ServiceName("my-service"),
    )),
)
otel.SetTracerProvider(tp)
```

**Resource** = metadata ของ service ที่ติดมากับทุก span (ServiceName, Version, Environment)

**Exporter** = ส่ง spans ไปยัง backend ผ่าน OTLP (gRPC หรือ HTTP)

**BatchSpanProcessor** = buffer spans แล้วส่งเป็น batch ลด network overhead

## ประเด็นสำคัญ

- OTel Collector เป็น optional แต่ช่วยให้เปลี่ยน backend ได้โดยไม่ต้องแก้ code service
- **Sampling**: ควบคุมว่าจะ record trace ไหน — `AlwaysOn`, `AlwaysOff`, `TraceIDRatioBased` (เก็บ X% ของ requests), `ParentBased` (ตาม parent's decision)
- **Semantic Conventions** (semconv): standard attribute names ที่ทุก vendor เข้าใจ เช่น `service.name`, `db.statement`, `http.method`
- **Instrumentation** แบ่งเป็น: Auto (library เช่น otelgrpc จัดการให้), Manual (เขียน `tracer.Start` เอง)
- OTLP = protobuf-based binary protocol ที่ใช้ส่ง spans จาก service ไป Collector/Backend

## ตัวอย่าง / กรณีศึกษา

ตัวอย่าง debug จาก deep dive doc — ค้นหา error ใน Jaeger ใช้เวลา 2 นาที:
```
Search: service=api-gateway, error=true, last 1hr
→ พบ Trace ID: 4bf92f35...
→ Timeline: api-gateway [200ms] → order-service [180ms] → db:INSERT [170ms]
→ Tags: db.error: "deadlock detected"
→ รู้ root cause ทันที
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/distributed-tracing|Distributed Tracing]] — OTel implement distributed tracing เป็นหนึ่งใน 3 pillars
- [[wiki/concepts/context-propagation|Context Propagation]] — กลไกส่ง trace context ข้าม service
- [[wiki/concepts/otel-go-instrumentation|OTel Go Instrumentation]] — วิธีใช้ OTel ใน Go

## แหล่งที่มา

- [[wiki/sources/opentelemetry-deep-dive|OpenTelemetry Deep Dive]]
- [[wiki/sources/opentelemetry-deep-dive-part2|OpenTelemetry Deep Dive Part 2]]
