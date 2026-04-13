---
title: "OpenTelemetry Trace Propagation — Migration Guide"
type: source
source_file: raw/notes/otel/OpenTelemetry Trace Propagation — Migration Guide.md
url: ""
published: 2026-04-13
tags: [opentelemetry, migration, grpc, http, go, otelgrpc, otelhttp]
related: [wiki/concepts/otel-go-instrumentation.md, wiki/concepts/context-propagation.md, wiki/concepts/distributed-tracing.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/otel/OpenTelemetry Trace Propagation — Migration Guide.md|Original file]]

## สรุป

คู่มือ Migration สำหรับแก้ปัญหา trace ขาดใน distributed Go services — แสดงวิธีเพิ่ม otelgrpc/otelhttp dependency, กำหนด TextMapPropagator, ติดตั้ง StatsHandler บน gRPC server/client, เพิ่ม otelhttp transport บน HTTP client (resty และ net/http SDK), และเพิ่ม span ใน repository methods ทุกตัว พร้อม diff-style code สำหรับแต่ละกรณี

## ประเด็นสำคัญ

- **สาเหตุ**: OTel propagate trace context ผ่าน header อัตโนมัติก็ต่อเมื่อ client มี instrumented transport/handler ติดตั้งไว้
- **gRPC client**: เพิ่ม `grpc.WithStatsHandler(otelgrpc.NewClientHandler())` ใน `grpc.NewClient` ทุกตัว
- **gRPC server**: เพิ่ม `grpc.StatsHandler(otelgrpc.NewServerHandler())` ใน `grpc.NewServer`
- **HTTP client (resty)**: `SetTransport(otelhttp.NewTransport(http.DefaultTransport))`
- **HTTP client (net/http SDK)**: `configuration.HTTPClient = &http.Client{Transport: otelhttp.NewTransport(...)}`
- **`initTracer`**: ต้องเพิ่ม `otel.SetTextMapPropagator(...)` ต่อจาก `SetTracerProvider` — ลำดับใน `main()`: `initTracer()` → `initRepositories()`
- **Repository methods**: ทุก method ต้องมี `tracer.Start(ctx, name)` + `defer sp.End()` + `sp.RecordError(err)` — ต้องใช้ `ctx` ที่ได้จาก `tracer.Start` ตอน call downstream
- **Version ต้องตรงกัน**: otelgrpc v0.65.0 + otelhttp v0.65.0 → otel core v1.40.0

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- ถ้า server ไม่ติดตั้ง `NewServerHandler` แม้ client จะส่ง `traceparent` ไปแล้ว server จะสร้าง root trace ใหม่ทุกครั้ง
- ผล trace ที่ถูกต้องหลัง implement: ทุก span ใน chain (gRPC, HTTP) มี `trace_id` เดียวกัน
- ถ้า pass `context.Background()` แทน `ctx` ใน downstream call → chain ขาดทันที
- Ticket อ้างอิง: `1961-fix-tracer`

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อขัดแย้ง

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/otel-go-instrumentation|OTel Go Instrumentation]]
- [[wiki/concepts/context-propagation|Context Propagation]]
- [[wiki/concepts/distributed-tracing|Distributed Tracing]]
