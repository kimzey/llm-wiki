---
title: "Clean Architecture in Go — สถาปัตยกรรม 4 Layer"
type: concept
tags: [clean-architecture, go, microservices, dependency-inversion, layered-architecture]
sources:
  - ai-context-phase1.md
  - ai-context-phase2.md
  - ai-context-phase5.md
related:
  - wiki/concepts/claude-code-ai-cli.md
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

Clean Architecture ในบริบท Go microservices ของ Sellsuki แบ่งเป็น 4 layer ที่มี Unidirectional Dependencies (ทิศทางเดียว) — Entity เป็นแกนกลาง ทุก layer อื่นพึ่ง Entity แต่ Entity ไม่พึ่งใคร ทำให้โค้ดทดสอบง่ายและแยกส่วนได้ชัดเจน

## อธิบาย

**Dependency Direction:**
```
Interface → UseCase → Entity
Repository ─────────→ Entity
```

**4 Layers:**

| Layer | ที่อยู่ | หน้าที่ | ห้าม import จาก |
|---|---|---|---|
| **Entity** | `src/entity/<domain>/` | Pure domain: structs, Validate(), sentinel errors | ทุก layer |
| **UseCase** | `src/use_case/` | Business logic, เรียก repository ผ่าน interface | `src/interface/`, `src/repository/` |
| **Repository** | `src/repository/<domain>_repository/` | Data access (GORM, Redis, HTTP) | `src/interface/`, `src/use_case/` |
| **Interface** | `src/interface/` | Translation layer (HTTP/gRPC/Kafka) | `src/repository/` (ห้าม import concrete repository) |

**Repository เป็น 5 ไฟล์ต่อ domain:**
1. `<domain>_repository.go` — tracer variable
2. `postgres_gorm_model.go` — GORM model + conversion functions
3. `postgres_gorm.go` — GORM implementation
4. `postgres_gorm_options.go` — filter/sort option handlers
5. `dummy.go` — no-op stub สำหรับ development (บาง domain)

## ประเด็นสำคัญ

**6 Design Principles (Solution Architect ใช้):**
1. Single responsibility — แต่ละ layer มีหน้าที่เดียว
2. Dependency inversion — UseCase พึ่ง interfaces ไม่ใช่ implementations
3. Explicit over implicit — pass dependencies ผ่าน constructors
4. Fail fast — validate ที่ boundary (entity Validate())
5. Observability first — tracing ทุก use case method (บังคับ)
6. One DB per domain — ไม่มี cross-domain DB joins

**Tracing Pattern (บังคับทุก use case method):**
```go
ctx, sp := tracer.Start(ctx, "use_case.MethodName")
defer sp.End()
// sp.AddEvent("...") ที่แต่ละ checkpoint
// sp.RecordError(err) ก่อน return error ทุกครั้ง
```

**Error Flow (3 ระดับ):**
1. **Domain error** — ประกาศใน `src/entity/<domain>/` → `var ErrInvalid<Domain>Data`
2. **Business error** — ประกาศใน `src/use_case/model/errors.go`
3. **HTTP mapping** — ลงทะเบียนใน `src/interface/.../helper/errors.go` → map error → HTTP status

**Transaction Safety Pattern:**
```go
id, tx, err := uc.repo.CreateWithTransaction(ctx, entity)
committed := false
defer func() { if !committed { tx.Rollback() } }()
// ... operations ...
tx.Commit()
committed = true
```

**Code Generation Pipeline:**
```
TypeSpec (.tsp) → OpenAPI (openapi.yaml) → Go Fiber interface (spec.gen.go)
```
→ ห้ามแก้ generated files: `openapi.yaml`, `spec.gen.go`, `*.pb.go`, `*_grpc.pb.go`

## ตัวอย่าง / กรณีศึกษา

**HTTP Error Response พร้อม Trace ID:**
```json
{
  "error": "order not found",
  "error_code": "order_not_found",
  "issue_id": "<otel_trace_id>"
}
```
→ `issue_id` ดึงจาก `span.SpanContext().TraceID()` → user ส่งกลับมา → หา trace ใน Jaeger ได้เลย

**Infrastructure Stack:**
- HTTP: Fiber v2 (fasthttp) port 8080
- gRPC: port 50051
- PostgreSQL: GORM ORM (1 DB per domain)
- Redis: distributed locks
- Kafka: consumer workers with retry
- Tracing: OpenTelemetry → OTLP → Jaeger

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/claude-code-ai-cli|Claude Code AI CLI]] — rules/architecture.md บอก Clean Architecture rules ให้ AI

## แหล่งที่มา

- [[wiki/sources/ai-context/ai-context-phase1|Phase 1: Rules — กฎสถาปัตยกรรม]]
- [[wiki/sources/ai-context/ai-context-phase2|Phase 2: Developer Agent workflow]]
- [[wiki/sources/ai-context/ai-context-phase5|Phase 5: Source Code Map]]
