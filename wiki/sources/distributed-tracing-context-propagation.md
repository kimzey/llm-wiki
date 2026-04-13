---
title: "Distributed Tracing & Context Propagation"
type: source
source_file: raw/notes/otel/Distributed Tracing & Context Propagation.md
url: ""
published: 2026-04-13
tags: [distributed-tracing, context-propagation, opentelemetry, grpc, go]
related: [wiki/concepts/distributed-tracing.md, wiki/concepts/context-propagation.md, wiki/concepts/w3c-trace-context.md, wiki/concepts/otel-baggage.md, wiki/concepts/otel-go-instrumentation.md]
created: 2026-04-13
updated: 2026-04-13
---

> **Full source**: [[../../raw/notes/otel/Distributed Tracing & Context Propagation.md|Original file]]

## สรุป

โน้ตนี้อธิบาย Distributed Tracing และ Context Propagation ครบตั้งแต่พื้นฐานจนถึง implementation ใน Go — ครอบคลุมสาเหตุที่ต้องมีระบบนี้ใน Microservices, ศัพท์สำคัญ (Trace/Span/TraceID/SpanID), กลไก inject/extract ผ่าน network header, W3C TraceContext, W3C Baggage และ 3 component หลักที่ต้องทำงานร่วมกัน (TracerProvider + TextMapPropagator + otelgrpc StatsHandler) พร้อม step-by-step flow ก่อน/หลังแก้ปัญหา trace ขาด

## ประเด็นสำคัญ

- **ปัญหาใน Microservices**: request 1 ใบ → log กระจายใน 4–5 service คนละ process/machine/file ไม่รู้ว่าเป็น request เดียวกัน
- **Distributed Tracing** = ติด GPS tracker ไว้กับ request ทุก service บันทึกว่า "รับ request นี้ไว้ ใช้เวลา X ms"
- **Trace** = การเดินทางทั้งหมดของ 1 request; **Span** = งานชิ้นเดียวใน 1 service
- **Context Propagation** = กลไกส่ง TraceID/SpanID ข้าม service ผ่าน network header (memory ไม่แชร์กันข้าม process)
- **Propagator** = "ล่าม" แปลงระหว่าง context ใน memory ↔ text header บน network (Inject = เขียนลง header, Extract = อ่าน header สร้าง context)
- **TextMapPropagator** = propagator ที่ทำงานกับ text key-value pairs (ตรงกับ HTTP headers, gRPC metadata)
- ถ้าไม่ `SetTextMapPropagator` → otelgrpc, otelhttp ดึง noop กลับมา → ไม่มีการ inject/extract → trace ขาด
- **3 component ต้องครบ**: TracerProvider (ส่ง data ไป Jaeger/Tempo) + TextMapPropagator (inject/extract header) + otelgrpc StatsHandler (hook เข้า gRPC lifecycle)
- StatsHandler ดีกว่า Interceptor เพราะเห็น InHeader/OutHeader event → extract `traceparent` ได้

## ข้อมูล / หลักฐาน ที่น่าสนใจ

- `propagation.Baggage{}` เป็น struct เปล่า ขนาด 0 bytes — ถ้าไม่มี baggage header มาจะ return ทันที ไม่กิน resource
- traceparent format: `00-<TraceID 32 hex>-<SpanID 16 hex>-<flags>` เช่น `00-b71df886bfbb3cca29211a10b74c6212-9c4a6dbf93d0594d-01`
- flags `01` = sampled (ส่งไป backend), `00` = ไม่ส่ง
- W3C TraceContext รองรับโดย Jaeger, Grafana Tempo, Datadog, Zipkin, AWS X-Ray ครบ
- เหตุผลที่ lib ไม่ set propagator อัตโนมัติ: "No global side effects from import" — หลาย format อยู่ร่วมกันในโลกจริง (W3C, B3, Jaeger, AWS X-Ray) ต้องให้ developer เลือกเอง
- ใส่หลาย format พร้อมกันได้ แต่ทำให้ request มี header เยอะขึ้น — แนะนำเลือกแค่ที่ chain ทั้งหมดใช้จริง

## สิ่งที่ขัดแย้งกับที่รู้อยู่แล้ว

ไม่พบข้อขัดแย้ง — เนื้อหาสอดคล้องกับ OpenTelemetry spec

## Concepts ที่เกี่ยวข้อง

- [[wiki/concepts/distributed-tracing|Distributed Tracing]]
- [[wiki/concepts/context-propagation|Context Propagation]]
- [[wiki/concepts/w3c-trace-context|W3C TraceContext]]
- [[wiki/concepts/otel-baggage|OTel Baggage]]
- [[wiki/concepts/otel-go-instrumentation|OTel Go Instrumentation]]
