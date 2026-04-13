# OpenTelemetry — คู่มือฉบับ Deep Dive

  

> เรียนรู้ตั้งแต่ "ทำไม" ถึง "ทำยังไง" อย่างละเอียดทุก Layer

  

---

  

## สารบัญ

  

1. [ทำไม Observability ถึงสำคัญ?](#1-ทำไม-observability-ถึงสำคัญ)

2. [3 Pillars of Observability](#2-3-pillars-of-observability)

3. [Distributed Tracing คืออะไร และทำงานยังไง?](#3-distributed-tracing-คืออะไร-และทำงานยังไง)

4. [OpenTelemetry Architecture ทั้งระบบ](#4-opentelemetry-architecture-ทั้งระบบ)

5. [Trace Data Model — โครงสร้างข้อมูลทุก Field](#5-trace-data-model--โครงสร้างข้อมูลทุก-field)

6. [TracerProvider และ Resource](#6-tracerprovider-และ-resource)

7. [Span Lifecycle — วงจรชีวิตของ Span](#7-span-lifecycle--วงจรชีวิตของ-span)

8. [Context Propagation — หัวใจของ Distributed Tracing](#8-context-propagation--หัวใจของ-distributed-tracing)

9. [traceparent Header — W3C Spec ฉบับสมบูรณ์](#9-traceparent-header--w3c-spec-ฉบับสมบูรณ์)

10. [Baggage — Context ที่วิ่งข้าม Service](#10-baggage--context-ที่วิ่งข้าม-service)

11. [Instrumentation — Auto vs Manual](#11-instrumentation--auto-vs-manual)

12. [Sampling — เลือกเก็บ Trace อะไรบ้าง?](#12-sampling--เลือกเก็บ-trace-อะไรบ้าง)

13. [OTEL Collector — Deep Dive](#13-otel-collector--deep-dive)

14. [Exporters และ Backend](#14-exporters-และ-backend)

15. [Jaeger — ใช้งานจริงทุก Feature](#15-jaeger--ใช้งานจริงทุก-feature)

16. [Error Handling กับ Tracing](#16-error-handling-กับ-tracing)

17. [Performance และ Production Patterns](#17-performance-และ-production-patterns)

18. [Architecture ในโปรเจกต์นี้ — ทุก Layer](#18-architecture-ในโปรเจกต์นี้--ทุก-layer)

  

---

  

## 1. ทำไม Observability ถึงสำคัญ?

  

### สภาพก่อนมี Observability

  

ลองนึกภาพ Production ระเบิดตอนเที่ยงคืน:

  

```

User report: "ทำ Order ไม่ได้"

  

Engineer เปิด laptop:

- ดู log → เห็นแค่ ERROR 500

- ไม่รู้ว่า error มาจาก service ไหน

- ไม่รู้ว่า DB ช้าหรือ service อื่น timeout

- ไม่รู้ว่า user คนไหนเจอปัญหา

- ไม่รู้ว่าเกิดตั้งแต่เมื่อไหร่

→ ใช้เวลา debug 3 ชั่วโมง

```

  

### สภาพเมื่อมี Observability

  

```

User report: "ทำ Order ไม่ได้"

  

Engineer เปิด Jaeger:

Search: service=api-gateway, error=true, last 1hr

  

พบ Trace ID: 4bf92f35...

  

Timeline:

api-gateway [200ms] ← รอ order-service นาน

order-service [180ms] ← รอ DB

db:INSERT [170ms] ← ← นี่แหละ!

  

Tags:

db.statement: "INSERT INTO orders..."

db.error: "deadlock detected"

  

→ รู้ root cause ใน 2 นาที

```

  

### Observability vs Monitoring

  

```

Monitoring (แบบเก่า):

รู้ว่า "มีปัญหา" (alert ดัง)

แต่ไม่รู้ว่า "ปัญหาคืออะไร"

  

Observability (แบบใหม่):

รู้ว่า "มีปัญหา" + รู้ว่า "ปัญหาคืออะไร" + รู้ว่า "ทำไม"

ตอบคำถามที่ไม่รู้ล่วงหน้าได้

```

  

---

  

## 2. 3 Pillars of Observability

  

### Pillar 1: Metrics — "เกิดอะไร ปริมาณเท่าไหร่"

  

Metrics คือตัวเลขที่วัดได้ในช่วงเวลาหนึ่ง

  

```

ประเภทของ Metrics:

  

Counter:

ค่าที่นับขึ้นเรื่อยๆ ไม่ลด

ตัวอย่าง:

http_requests_total{method="POST", status="200"} = 14523

errors_total{service="order"} = 42

  

Gauge:

ค่าที่ขึ้นลงได้

ตัวอย่าง:

memory_usage_bytes = 524288000

active_connections = 127

queue_size = 89

  

Histogram:

การกระจายของค่า (เช่น latency)

ตัวอย่าง:

http_request_duration_seconds_bucket{le="0.1"} = 1200 ← ≤ 100ms

http_request_duration_seconds_bucket{le="0.5"} = 3400 ← ≤ 500ms

http_request_duration_seconds_bucket{le="1.0"} = 3498 ← ≤ 1s

http_request_duration_seconds_bucket{le="+Inf"} = 3500 ← ทั้งหมด

  

→ p99 latency = 980ms (99% ของ request เสร็จใน 980ms)

```

  

### Pillar 2: Logs — "เกิดอะไรขึ้นบ้าง"

  

Logs คือ event ที่บันทึกไว้

  

```

Structured Log (ดี):

{

"timestamp": "2024-01-15T10:30:05.123Z",

"level": "error",

"service": "order-service",

"trace_id": "4bf92f3577b34da6",

"span_id": "00f067aa0ba902b7",

"message": "Failed to create order",

"error": "deadlock detected on table orders",

"user_id": "user_789",

"order_amount": 299.99

}

  

Unstructured Log (ไม่ดี):

[2024-01-15 10:30:05] ERROR: Failed!! user=user_789

← search ยาก, parse ยาก, ไม่มี trace_id

```

  

### Pillar 3: Traces — "เกิดที่ไหน ใช้เวลาเท่าไหร่"

  

Traces คือเส้นทางของ request ผ่านทุก service

  

```

ตัวอย่าง Trace จริง: User ชำระเงิน

  

Trace: abc123 Total: 850ms

  

[0ms ]─────────────────────────────────────────────[850ms]

api-gateway: POST /payments 850ms

│

├─[10ms]────────────────────[100ms]

│ auth-service: ValidateToken 90ms

│ │

│ └─[20ms]────[60ms]

│ redis: GET session:token_xyz 40ms

│

├─[100ms]────────────────────────────────────[750ms]

│ payment-service: ProcessPayment 650ms

│ │

│ ├─[110ms]──────────────────────[500ms]

│ │ bank-gateway: ChargeCard 390ms ← ช้า!

│ │

│ ├─[510ms]──────[600ms]

│ │ db: INSERT payments 90ms

│ │

│ └─[610ms]──[680ms]

│ kafka: publish payment.completed 70ms

│

└─[750ms]────[800ms]

notification-service: SendEmail 50ms

```

  

### ทั้ง 3 Pillars เชื่อมกันยังไง?

  

```

Trace ID เป็นกาวเชื่อม:

  

1. User รายงานปัญหา → บอก Trace ID (จาก error response)

  

2. Jaeger → ดู Trace → เห็นว่า payment-service ช้า 650ms

  

3. Metrics → Grafana → payment-service latency p99 พุ่งตั้งแต่ 10:28

  

4. Logs → filter ด้วย trace_id=abc123 → เห็น log ทุกบรรทัดของ request นั้น

  

→ เชื่อมทั้ง 3 ได้ภายใน 5 นาที

```

  

---

  

## 3. Distributed Tracing คืออะไร และทำงานยังไง?

  

### ปัญหาของ Monolith vs Microservices

  

```

Monolith:

Request → App เดียว

Debug: ดู stack trace ได้เลย ง่าย

  

Microservices:

Request → A → B → C → D → E

Debug: ต้องหา log จาก 5 service ในช่วงเวลาเดียวกัน

แต่ละ service มี log format ต่างกัน

server clock อาจต่างกัน

→ nightmare

```

  

### Distributed Tracing แก้ปัญหายังไง?

  

```

แนวคิด: ใส่ "ด้าย" ผ่านทุก service แล้วตามด้ายนั้น

  

1. Request เข้า Service A → สร้าง Trace ID (UUID unique)

2. Service A เรียก Service B → ส่ง Trace ID ไปด้วยใน Header

3. Service B รับ Trace ID → สร้าง Span ของตัวเอง linked กับ Trace ID เดิม

4. ทุก Service ส่ง Span ไป Central Backend (Jaeger)

5. Jaeger รวม Spans ทั้งหมดที่มี Trace ID เดียวกัน → เห็นภาพรวม

```

  

### ตัวอย่างข้อมูลที่ส่งระหว่าง Service

  

```

Service A เรียก Service B:

  

HTTP Request:

POST /orders HTTP/1.1

Host: order-service:8080

Content-Type: application/json

traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ ↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑ ↑↑

v traceId (32 chars) parentSpanId (16) flags

baggage: tenantId=tenant_abc,userId=user_789

  

{"amount": 299.99, "items": [...]}

```

  

---

  

## 4. OpenTelemetry Architecture ทั้งระบบ

  

### Layer ทั้งหมดของ OTEL

  

```

┌─────────────────────────────────────────────────────────────────┐

│ OpenTelemetry Stack │

│ │

│ ┌──────────────────────────────────────────────────────────┐ │

│ │ Instrumentation Libraries │ │

│ │ (otelfiber, otelgrpc, otelhttp, otelsql, ...) │ │

│ │ → auto instrument frameworks/drivers │ │

│ └──────────────────────────┬───────────────────────────────┘ │

│ │ │

│ ┌──────────────────────────▼───────────────────────────────┐ │

│ │ OTEL API │ │

│ │ (go.opentelemetry.io/otel) │ │

│ │ → interface ที่โค้ดใช้, ไม่ขึ้น vendor │ │

│ └──────────────────────────┬───────────────────────────────┘ │

│ │ │

│ ┌──────────────────────────▼───────────────────────────────┐ │

│ │ OTEL SDK │ │

│ │ (go.opentelemetry.io/otel/sdk) │ │

│ │ → implementation จริง: sampling, batching, resource │ │

│ └──────────────────────────┬───────────────────────────────┘ │

│ │ OTLP Protocol │

│ ┌──────────────────────────▼───────────────────────────────┐ │

│ │ OTEL Collector │ │

│ │ → receive, process, export │ │

│ └──────────────────────────┬───────────────────────────────┘ │

│ │ │

│ ┌──────────────────────────▼───────────────────────────────┐ │

│ │ Backend │ │

│ │ Jaeger / Tempo / Datadog / Zipkin / ... │ │

│ └──────────────────────────────────────────────────────────┘ │

└─────────────────────────────────────────────────────────────────┘

```

  

### OTLP Protocol คืออะไร?

  

**OTLP (OpenTelemetry Protocol)** = format มาตรฐานสำหรับส่ง telemetry data

  

```

ก่อน OTLP:

Jaeger SDK → Jaeger format (Thrift/Protobuf ของ Jaeger)

Zipkin SDK → Zipkin format (JSON ของ Zipkin)

→ เปลี่ยน backend = เปลี่ยน SDK = แก้โค้ดทั้งหมด

  

หลัง OTLP:

App → OTLP → OTEL Collector → แปลงเป็น format อะไรก็ได้

→ เปลี่ยน backend แค่แก้ collector config

```

  

```

OTLP Transport Options:

  

gRPC (port 4317): ← โปรเจกต์นี้ใช้

ข้อดี: binary protocol, streaming, performance ดี

ข้อเสีย: ต้องการ HTTP/2

  

HTTP (port 4318):

ข้อดี: ง่าย, ผ่าน proxy ได้

ข้อเสีย: ช้ากว่า gRPC นิดหน่อย

  

gRPC format: Protobuf binary

HTTP format: JSON หรือ Protobuf

```

  

---

  

## 5. Trace Data Model — โครงสร้างข้อมูลทุก Field

  

### Span Object สมบูรณ์

  

```json

{

"traceId": "4bf92f3577b34da6a3ce929d0e0e4736",

"spanId": "00f067aa0ba902b7",

"parentSpanId": "a3ce929d0e0e4736",

"traceState": "rojo=00f067aa0ba902b7,congo=t61rcWkgMzE",

"name": "HTTP POST /v1/orders",

"kind": 2,

"startTimeUnixNano": "1544712660000000000",

"endTimeUnixNano": "1544712661000000000",

"attributes": [

{"key": "http.method", "value": {"stringValue": "POST"}},

{"key": "http.url", "value": {"stringValue": "/v1/orders"}},

{"key": "http.status_code", "value": {"intValue": 201}},

{"key": "http.flavor", "value": {"stringValue": "1.1"}},

{"key": "net.host.name", "value": {"stringValue": "api-gateway"}},

{"key": "user.id", "value": {"stringValue": "user_789"}}

],

"events": [

{

"name": "order.validated",

"timeUnixNano": "1544712660100000000",

"attributes": [

{"key": "items.count", "value": {"intValue": 3}},

{"key": "total", "value": {"doubleValue": 299.99}}

]

},

{

"name": "exception",

"timeUnixNano": "1544712660900000000",

"attributes": [

{"key": "exception.type", "value": {"stringValue": "DBError"}},

{"key": "exception.message", "value": {"stringValue": "deadlock detected"}},

{"key": "exception.stacktrace", "value": {"stringValue": "goroutine 1 [running]:\n..."}}

]

}

],

"links": [

{

"traceId": "previous_trace_id",

"spanId": "previous_span_id",

"attributes": [{"key": "link.reason", "value": {"stringValue": "retry"}}]

}

],

"status": {

"code": 2,

"message": "order creation failed"

},

"droppedAttributesCount": 0,

"droppedEventsCount": 0,

"droppedLinksCount": 0

}

```

  

### Span Kind — ประเภทของ Span

  

```

SpanKind บอกว่า Span นี้เป็น "อะไรในระบบ"

  

INTERNAL (0):

Operation ภายใน service เดียว

ตัวอย่าง: function call, business logic

→ ไม่มีการสื่อสารกับ service อื่น

  

SERVER (1):

รับ request จาก client (inbound)

ตัวอย่าง: HTTP handler, gRPC server handler

→ มี parent เป็น CLIENT span ของ caller

  

CLIENT (2):

ส่ง request ไป service อื่น (outbound)

ตัวอย่าง: HTTP client call, gRPC client call

→ เป็น parent ของ SERVER span ของ callee

  

PRODUCER (3):

ส่ง message ไป message queue

ตัวอย่าง: Kafka producer, RabbitMQ publish

  

CONSUMER (4):

รับ message จาก message queue

ตัวอย่าง: Kafka consumer, worker

  

ตัวอย่างการใช้:

CLIENT span (api-gateway)

↓ HTTP call

SERVER span (order-service)

↓ internal logic

INTERNAL span (validateOrder)

↓ DB call

CLIENT span (gorm SQL)

↓ (internal to DB driver)

↓ Kafka publish

PRODUCER span (kafka)

↓ (async)

CONSUMER span (notification-worker)

```

  

### Span Status

  

```

UNSET (0):

Default — ยังไม่ set

ถ้าไม่มี error ไม่ต้อง set

  

OK (1):

Operation สำเร็จ explicitly

ใช้เมื่อต้องการ override UNSET

  

ERROR (2):

Operation ล้มเหลว

ต้องใช้คู่กับ span.RecordError(err)

  

Rule:

ถ้า status = ERROR → MUST set message อธิบาย

ถ้า status = OK → ไม่ต้อง set message

ถ้า HTTP 4xx → อาจไม่ต้อง ERROR (client error, ไม่ใช่ server error)

ถ้า HTTP 5xx → ควร ERROR

```

  

### Span Attributes — Semantic Conventions

  

OTEL กำหนด attribute names มาตรฐาน เรียกว่า **Semantic Conventions**:

  

```

HTTP:

http.method = "GET" | "POST" | "PUT" | "DELETE" ...

http.url = "https://api.example.com/v1/orders"

http.status_code = 200 | 404 | 500 ...

http.flavor = "1.1" | "2" | "3"

http.user_agent = "Mozilla/5.0..."

http.request_content_length = 1234

http.response_content_length = 5678

net.host.name = "api-gateway"

net.host.port = 8080

net.peer.name = "order-service"

net.peer.port = 8081

  

Database:

db.system = "postgresql" | "mysql" | "redis" | "mongodb"

db.name = "sellsuki_central"

db.user = "postgres"

db.operation = "SELECT" | "INSERT" | "UPDATE" | "DELETE"

db.statement = "SELECT * FROM companies WHERE id = $1"

db.sql.table = "companies"

  

gRPC:

rpc.system = "grpc"

rpc.service = "OrderService"

rpc.method = "CreateOrder"

rpc.grpc.status_code = 0 (OK) | 13 (INTERNAL) | ...

  

Messaging (Kafka):

messaging.system = "kafka"

messaging.destination = "order.created"

messaging.operation = "publish" | "receive"

messaging.message_id = "msg_abc123"

  

Exception:

exception.type = "runtime.Error"

exception.message = "index out of range"

exception.stacktrace = "goroutine 1 [running]:\n..."

  

ตัวอย่างการใช้ใน Go:

span.SetAttributes(

semconv.HTTPMethod("POST"),

semconv.HTTPURL("/v1/orders"),

semconv.HTTPStatusCode(201),

semconv.DBSystem("postgresql"),

semconv.DBStatement("INSERT INTO orders..."),

attribute.String("business.tenant_id", "tenant_abc"),

attribute.Float64("business.order_amount", 299.99),

)

```

  

---

  

## 6. TracerProvider และ Resource

  

### TracerProvider คืออะไร?

  

```

TracerProvider = Factory สำหรับสร้าง Tracer

Tracer = ใช้สร้าง Span

  

ความสัมพันธ์:

TracerProvider (1 ต่อ application)

└── Tracer (1 ต่อ library/component)

└── Span (หลายตัวต่อ request)

  

ตัวอย่าง:

provider := trace.NewTracerProvider(...)

  

// library ต่างๆ สร้าง Tracer ของตัวเอง

httpTracer := provider.Tracer("otelfiber", trace.WithInstrumentationVersion("2.2.3"))

dbTracer := provider.Tracer("gorm-otel", trace.WithInstrumentationVersion("1.0.0"))

myTracer := provider.Tracer("company-repository")

```

  

### Resource คืออะไร?

  

**Resource** = ข้อมูล metadata ของ service ที่ produce trace นั้น (ไม่ใช่ข้อมูลของ request)

  

```go

resource.NewWithAttributes(

semconv.SchemaURL,

  

// ─── Service Identity ───

semconv.ServiceName("sellsuki-central-control-backend-production"),

semconv.ServiceVersion("v1.5.3"),

semconv.ServiceNamespace("sellsuki"),

semconv.ServiceInstanceID("pod-abc123"), // hostname/pod ID

  

// ─── Environment ───

attribute.String("environment", "production"),

attribute.String("deployment.region", "ap-southeast-1"),

  

// ─── Container/K8s (ถ้ามี) ───

semconv.K8SPodName("api-pod-7d4f8b-xkq9p"),

semconv.K8SNamespaceName("production"),

semconv.K8SClusterName("sellsuki-prod"),

  

// ─── Host ───

semconv.HostName("ip-10-0-1-50"),

)

```

  

ทำไม Resource สำคัญ?

  

```

ใน Jaeger ถ้าไม่มี Resource ที่ดี:

Service Name: "unknown"

→ ไม่รู้ว่า trace มาจาก service ไหน

→ ถ้ามีหลาย instance ไม่รู้ว่า pod ไหน

  

ถ้ามี Resource ที่ดี:

Jaeger แสดง:

Service: "sellsuki-central-control-backend"

Version: "v1.5.3"

Environment: "production"

→ Filter ได้ทันที

```

  

### Batcher vs SimpleSpanProcessor

  

```go

// SimpleSpanProcessor — ส่งทันทีทุก span

// ใช้ใน: development, debug

tp := trace.NewTracerProvider(

trace.WithSyncer(exporter),

)

  

// BatchSpanProcessor — batch แล้วส่งเป็น batch

// ใช้ใน: production (default ที่ควรใช้)

tp := trace.NewTracerProvider(

trace.WithBatcher(exporter,

// ปรับ performance:

sdktrace.WithMaxExportBatchSize(512), // ส่งทีละ 512 spans

sdktrace.WithBatchTimeout(5*time.Second), // หรือทุก 5 วินาที

sdktrace.WithMaxQueueSize(2048), // queue สูงสุด 2048

),

)

  

// ทำไม Batcher ดีกว่า:

// - ลด network round trip

// - ลด overhead ของ goroutine

// - ถ้า collector ล่ม spans จะถูก drop แทน block

```

  

---

  

## 7. Span Lifecycle — วงจรชีวิตของ Span

  

### ขั้นตอนชีวิต Span

  

```

1. CREATE: tracer.Start(ctx, "operationName")

─ กำหนด traceId, spanId, parentSpanId

─ บันทึก startTime

─ ผูก context ใหม่ที่มี span นี้

  

2. ACTIVE: ระหว่าง Start ถึง End

─ SetAttributes() → เพิ่ม metadata

─ AddEvent() → บันทึก checkpoint

─ RecordError() → บันทึก error

─ SetStatus() → กำหนด status

  

3. END: span.End()

─ บันทึก endTime

─ คำนวณ duration

─ ส่งไป SpanProcessor

  

4. EXPORT: BatchSpanProcessor

─ รวม spans เป็น batch

─ ส่งผ่าน Exporter ไป OTEL Collector

```

  

### Context และ Span ผูกกันยังไง?

  

```go

// ─── tracer.Start สร้าง 2 สิ่ง ───

ctx, span := tracer.Start(ctx, "GetCompany")

// ↑ ↑

// ctx ใหม่ span ที่สร้าง

  

// ctx ใหม่ = ctx เดิม + span นี้ถูกฝังไว้ข้างใน

// ใครก็ตามที่รับ ctx นี้ไป สามารถดึง span ออกมาได้

  

// ดึง span จาก context:

currentSpan := trace.SpanFromContext(ctx)

currentSpan.AddEvent("something happened")

  

// ดึง spanContext (traceId, spanId, etc.):

spanCtx := trace.SpanContextFromContext(ctx)

fmt.Println(spanCtx.TraceID().String()) // → "4bf92f35..."

fmt.Println(spanCtx.SpanID().String()) // → "00f067aa..."

  

// ─── Child span ───

// ทุก tracer.Start ที่ใช้ ctx ที่มี span → สร้าง child อัตโนมัติ

  

func parent(ctx context.Context) {

ctx, parentSpan := tracer.Start(ctx, "parent")

defer parentSpan.End()

// parentSpan.spanId = "aaa"

  

child(ctx) // ← ส่ง ctx ต่อ!

}

  

func child(ctx context.Context) {

ctx, childSpan := tracer.Start(ctx, "child")

defer childSpan.End()

// childSpan.spanId = "bbb"

// childSpan.parentId = "aaa" ← ดึงจาก ctx อัตโนมัติ

}

```

  

### ข้อผิดพลาดที่พบบ่อยใน Span Lifecycle

  

```go

// ❌ ลืม defer span.End() → span ไม่ถูกส่ง!

func badFunction(ctx context.Context) {

ctx, span := tracer.Start(ctx, "op")

// ไม่มี defer span.End()

doWork(ctx)

span.End() // ← ถ้า doWork() panic → span ไม่ End!

}

  

// ✅ ถูกต้อง

func goodFunction(ctx context.Context) {

ctx, span := tracer.Start(ctx, "op")

defer span.End() // ← End เสมอ แม้ panic

doWork(ctx)

}

  

// ❌ ไม่ส่ง ctx ต่อ → child span กลายเป็น root ใหม่!

func badHandler(c *fiber.Ctx) error {

ctx := context.Background() // ← ตัด chain!

result, err := useCase.GetCompany(ctx, id)

// ...

}

  

// ✅ ถูกต้อง

func goodHandler(c *fiber.Ctx) error {

ctx := c.UserContext() // ← รับ ctx ที่มี span จาก otelfiber

result, err := useCase.GetCompany(ctx, id)

// ...

}

  

// ❌ span.End() ก่อน return error → duration ผิด

func badRepo(ctx context.Context) error {

ctx, span := tracer.Start(ctx, "query")

result, err := db.Query(ctx, sql)

span.End() // ← End ก่อน

if err != nil {

span.RecordError(err) // ← หลัง End แล้ว! ไม่มีผล

return err

}

return nil

}

  

// ✅ ถูกต้อง

func goodRepo(ctx context.Context) error {

ctx, span := tracer.Start(ctx, "query")

defer span.End() // ← End หลังสุด

  

result, err := db.Query(ctx, sql)

if err != nil {

span.RecordError(err) // ← ก่อน End

span.SetStatus(codes.Error, err.Error())

return err

}

return nil

}

```

  

---

  

## 8. Context Propagation — หัวใจของ Distributed Tracing

  

### ทำไม Propagation ถึงสำคัญมาก?

  

```

ไม่มี Propagation:

  

Service A (traceId=abc, spanId=111):

เรียก Service B

  

Service B:

ไม่รู้ traceId → สร้าง root span ใหม่ (traceId=xyz, spanId=222)

  

Jaeger:

Trace abc: มีแค่ Service A

Trace xyz: มีแค่ Service B

→ ไม่เชื่อมกัน เห็นแค่ครึ่งเดียว

  

มี Propagation:

  

Service A (traceId=abc, spanId=111):

inject traceparent: 00-abc-111-01 ใน header

เรียก Service B

  

Service B:

extract traceparent → รู้ traceId=abc, parentId=111

สร้าง span (traceId=abc, spanId=222, parentId=111)

  

Jaeger:

Trace abc: Service A → Service B

→ เชื่อมกันสมบูรณ์ 🎉

```

  

### Propagation ทำงานยังไงภายใน?

  

```

TextMapPropagator interface:

Inject(ctx, carrier) → เขียน context ลง carrier (เช่น HTTP Header)

Extract(carrier, ctx) → อ่าน context จาก carrier แล้วใส่กลับใน ctx

  

TextMapCarrier interface:

Get(key) string → อ่าน header

Set(key, value) → เขียน header

Keys() []string → รายการ keys ทั้งหมด

```

  

### Implementation จริงของ TraceContext Propagator

  

```

Inject (ฝั่ง caller):

1. ดึง SpanContext จาก ctx

2. แปลงเป็น traceparent string

format: "00-{traceId}-{spanId}-{flags}"

3. เรียก carrier.Set("traceparent", value)

4. ถ้ามี tracestate → carrier.Set("tracestate", value)

  

Extract (ฝั่ง callee):

1. เรียก carrier.Get("traceparent")

2. parse string → traceId, parentSpanId, flags

3. validate (ต้องไม่ all-zeros, ไม่ invalid format)

4. สร้าง SpanContext จากค่าที่ parse ได้

5. ใส่ SpanContext กลับใน ctx ใหม่

6. return ctx ใหม่

```

  

### วิธี Inject/Extract ใน Go แต่ละ Protocol

  

```go

// ─── HTTP Client ─── inject ก่อนส่ง

func (r *managementRepo) GetUser(ctx context.Context, id string) (*User, error) {

req, err := http.NewRequestWithContext(ctx, "GET", r.baseURL+"/users/"+id, nil)

if err != nil {

return nil, err

}

  

// inject trace context เข้า header

otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))

  

// ตอนนี้ req.Header มี:

// "Traceparent": "00-4bf92f35...-00f067aa...-01"

// "Baggage": "tenantId=abc,userId=789"

  

resp, err := r.client.Do(req)

// ...

}

  

// ─── HTTP Server ─── extract ตอนรับ (otelfiber ทำให้อัตโนมัติ)

// ถ้าจะทำเอง:

func manualExtract(c *fiber.Ctx) error {

// extract จาก header

ctx := otel.GetTextMapPropagator().Extract(

c.Context(),

propagation.HeaderCarrier(c.Request().Header), // fiber header

)

// ctx ตอนนี้มี parent span context แล้ว

c.SetUserContext(ctx)

return c.Next()

}

  

// ─── gRPC Client ─── ใช้ interceptor

import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

  

conn, err := grpc.NewClient(addr,

grpc.WithTransportCredentials(insecure.NewCredentials()),

grpc.WithStatsHandler(otelgrpc.NewClientHandler()), // inject อัตโนมัติ

)

  

// ─── gRPC Server ─── ใช้ interceptor

grpcServer := grpc.NewServer(

grpc.StatsHandler(otelgrpc.NewServerHandler()), // extract อัตโนมัติ

)

  

// ─── Kafka Producer ─── inject ลง message header

import "go.opentelemetry.io/contrib/instrumentation/github.com/Shopify/sarama/otelsarama"

  

producer, _ := sarama.NewSyncProducer(brokers, config)

otelProducer := otelsarama.WrapSyncProducer(config, producer)

// inject อัตโนมัติทุก message

  

// ─── Kafka Consumer ─── extract จาก message header

handler := otelsarama.WrapConsumerGroupHandler(&myHandler{})

// extract อัตโนมัติทุก message

```

  

### W3C TraceContext vs B3 vs Jaeger Format

  

```

มี format หลายแบบในธรรมชาติ:

  

W3C TraceContext (มาตรฐานใหม่ — แนะนำ):

Headers: traceparent, tracestate

ตัวอย่าง: traceparent: 00-4bf92f35...-00f067aa...-01

  

B3 (เก่า — ใช้ใน Zipkin ecosystem):

Headers: X-B3-TraceId, X-B3-SpanId, X-B3-ParentSpanId, X-B3-Sampled

ตัวอย่าง:

X-B3-TraceId: 4bf92f3577b34da6a3ce929d0e0e4736

X-B3-SpanId: 00f067aa0ba902b7

X-B3-Sampled: 1

  

Jaeger (เก่า):

Headers: uber-trace-id

ตัวอย่าง: uber-trace-id: 4bf92f35:00f067aa:a3ce929d:1

  

รองรับหลาย format พร้อมกัน:

otel.SetTextMapPropagator(

propagation.NewCompositeTextMapPropagator(

propagation.TraceContext{}, // W3C — inject/extract

b3.New(), // B3 — extract only (legacy)

jaegerpropagator.Jaeger{}, // Jaeger — extract only (legacy)

),

)

→ inject ด้วย W3C, extract ได้จากทุก format

→ ช่วย migrate จาก system เก่า

```

  

---

  

## 9. traceparent Header — W3C Spec ฉบับสมบูรณ์

  

### Anatomy ของ traceparent

  

```

traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

─┘ ──────────────────────────────── ──────────────── ─┘

│ │ │ │

v v v v

ver traceId (128-bit, hex) parentSpanId flags

(64-bit, hex) (8-bit)

  

version (2 hex chars):

"00" = current version ของ spec

"ff" = reserved (invalid)

  

traceId (32 hex chars = 128 bits):

ทุก chars ต้องไม่เป็น "00000000000000000000000000000000"

case-insensitive แต่ควรใช้ lowercase

  

parentSpanId (16 hex chars = 64 bits):

span ของ caller (ผู้ส่ง request)

ทุก chars ต้องไม่เป็น "0000000000000000"

  

flags (2 hex chars = 8 bits):

bit 0 (rightmost) = sampled flag

"01" = sampled (เก็บ trace นี้)

"00" = not sampled (ไม่เก็บ)

```

  

### ตัวอย่างการเดินทางของ traceparent

  

```

Request Chain: Browser → API → Order → DB

  

Step 1: Browser ส่ง request มา (ไม่มี traceparent)

→ API Gateway: ไม่เจอ traceparent ในการ extract

→ สร้าง root span ใหม่:

traceId = "4bf92f3577b34da6a3ce929d0e0e4736" (random)

spanId = "00f067aa0ba902b7" (random)

parentId = null (root)

→ inject header ก่อนเรียก Order Service:

traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01

^^^^^^^^^^^^^^^^

span ของ API Gateway

  

Step 2: Order Service รับ request

→ extract traceparent:

traceId = "4bf92f3577b34da6a3ce929d0e0e4736" (เดิม)

parentId = "00f067aa0ba902b7" (span ของ API)

→ สร้าง span ใหม่:

traceId = "4bf92f3577b34da6a3ce929d0e0e4736" (เดิม)

spanId = "a3ce929d0e0e4736" (ใหม่)

parentId = "00f067aa0ba902b7" (span ของ API)

→ inject header ก่อน query DB:

traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-a3ce929d0e0e4736-01

^^^^^^^^^^^^^^^^

span ของ Order Service

  

Step 3: Jaeger รวมทุก Span

Trace: 4bf92f3577b34da6a3ce929d0e0e4736

├── 00f067aa0ba902b7 (API Gateway) [parent: null]

│ └── a3ce929d0e0e4736 (Order) [parent: 00f067aa]

│ └── db_span_id (DB) [parent: a3ce929d]

```

  

### tracestate — ข้อมูลเพิ่มเติม vendor-specific

  

```

tracestate: rojo=00f067aa0ba902b7,congo=t61rcWkgMzE

──────────────────── ──────────────────

vendor1=value vendor2=value

  

Rules:

- key: lowercase, ไม่มี spaces, max 256 chars

- value: printable ASCII, max 256 chars

- max 32 entries

- propagate ทุก hop (แม้ไม่เข้าใจ value)

  

ใช้ทำอะไร:

Datadog: dd=p:spanid;s:1 ← ส่ง sampling decision ของ Datadog

Lightstep: ls=s:1 ← vendor-specific metadata

  

ใน W3C Spec: ต้อง forward tracestate ที่ไม่รู้จัก

เพื่อให้ vendor อื่นในระบบยังทำงานได้

```

  

---

  

## 10. Baggage — Context ที่วิ่งข้าม Service

  

### Baggage คืออะไรในเชิงลึก

  

```

Baggage = W3C standard สำหรับส่ง key-value context ผ่าน HTTP header

Header name: baggage

  

ตัวอย่าง:

baggage: userId=user_789,tenantId=tenant_abc,region=asia-southeast-1

──────────────── ───────────────── ──────────────────────

entry 1 entry 2 entry 3

```

  

### ใช้งาน Baggage ใน Go

  

```go

import (

"go.opentelemetry.io/otel/baggage"

)

  

// ─── Set baggage (เช่นใน auth middleware) ───

func authMiddleware(c *fiber.Ctx) error {

ctx := c.UserContext()

  

// สร้าง baggage member

userId, _ := baggage.NewMember("userId", claims.UserID)

tenantId, _ := baggage.NewMember("tenantId", claims.TenantID)

role, _ := baggage.NewMember("role", claims.Role)

  

// สร้าง baggage จาก members

bag, _ := baggage.New(userId, tenantId, role)

  

// ใส่ baggage ใน context

ctx = baggage.ContextWithBaggage(ctx, bag)

c.SetUserContext(ctx)

  

return c.Next()

}

  

// ─── Get baggage (ในทุก service ที่รับ request) ───

func (uc *UseCase) GetCompany(ctx context.Context, id string) (*Company, error) {

bag := baggage.FromContext(ctx)

  

tenantId := bag.Member("tenantId").Value()

userId := bag.Member("userId").Value()

  

// ใช้ tenantId กรอง data ของ tenant นั้น

return uc.repo.GetByTenantAndID(ctx, tenantId, id)

}

  

// ─── Baggage ถูกส่งต่ออัตโนมัติถ้า propagator ตั้งค่าถูก ───

// ทุก HTTP outbound call ที่ inject จะพา baggage ไปด้วย:

// baggage: userId=user_789,tenantId=tenant_abc,role=admin

```

  

### Baggage vs Span Attribute — ต่างกันยังไง?

  

```

Feature | Baggage | Span Attribute

─────────────────┼────────────────────────────┼──────────────────────────

ส่งข้าม service | ✅ ส่งผ่าน HTTP header | ❌ ไม่ส่ง (เก็บแค่ใน Jaeger)

ขนาด | จำกัด (HTTP header limit) | ใหญ่ได้มากกว่า

เห็นใน Jaeger | ❌ (ถ้าไม่ set attribute) | ✅ เสมอ

ทุก service เห็น | ✅ | ❌ เฉพาะ service นั้น

Security | ⚠️ user เห็น header ได้ | ✅ เก็บใน backend

  

แนวทาง:

Baggage → context ที่ทุก service ต้องรู้ (tenantId, userId, requestId)

Attribute → metadata เฉพาะ operation นั้น (sql query, http url, amount)

```

  

---

  

## 11. Instrumentation — Auto vs Manual

  

### Auto Instrumentation

  

Framework/library ที่รองรับ OTEL จะ instrument ให้อัตโนมัติ

  

```go

// ─── HTTP Server: otelfiber ───

import "github.com/gofiber/contrib/otelfiber/v2"

  

server.Use(otelfiber.Middleware(

otelfiber.WithServerName("sellsuki-central"),

// span name format: "GET /v1/companies/:id"

))

  

// otelfiber ทำอะไรให้:

// 1. Extract traceparent จาก incoming request

// 2. สร้าง SERVER span พร้อม attributes:

// - http.method, http.url, http.route

// - net.host.name, net.host.port

// 3. วาง span ลง context

// 4. หลัง handler เสร็จ: set http.status_code, end span

  

// ─── HTTP Client: otelhttp ───

import "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"

  

client := &http.Client{

Transport: otelhttp.NewTransport(http.DefaultTransport),

}

// ทุก request จาก client นี้จะ inject traceparent อัตโนมัติ

  

// ─── gRPC: otelgrpc ───

import "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"

  

// Server

grpcServer := grpc.NewServer(

grpc.StatsHandler(otelgrpc.NewServerHandler()),

)

  

// Client

conn, _ := grpc.NewClient(addr,

grpc.WithStatsHandler(otelgrpc.NewClientHandler()),

grpc.WithTransportCredentials(insecure.NewCredentials()),

)

  

// ─── Database: otelsql ───

import "github.com/XSAM/otelsql"

  

db, err := otelsql.Open("postgres", dsn,

otelsql.WithAttributes(semconv.DBSystemPostgreSQL),

otelsql.WithSpanOptions(otelsql.SpanOptions{

Ping: true,

RowsAffected: true,

DisableErrSkip: true,

}),

)

// ทุก query จะสร้าง span พร้อม db.statement อัตโนมัติ

```

  

### Manual Instrumentation

  

สร้าง Span เองสำหรับ business logic

  

```go

// ─── Basic span ───

func (r *companyRepo) GetByID(ctx context.Context, id string) (*Company, error) {

ctx, span := otel.Tracer("company-repository").Start(

ctx,

"CompanyRepository.GetByID",

trace.WithSpanKind(trace.SpanKindInternal),

trace.WithAttributes(

attribute.String("company.id", id),

),

)

defer span.End()

  

var company Company

if err := r.db.WithContext(ctx).First(&company, "id = ?", id).Error; err != nil {

span.RecordError(err)

span.SetStatus(codes.Error, "company not found")

return nil, err

}

  

span.SetAttributes(

attribute.String("company.name", company.Name),

attribute.String("company.status", string(company.Status)),

)

span.SetStatus(codes.Ok, "")

return &company, nil

}

  

// ─── Span พร้อม Events ───

func (uc *UseCase) ProcessOrder(ctx context.Context, req CreateOrderRequest) (*Order, error) {

ctx, span := otel.Tracer("order-usecase").Start(ctx, "ProcessOrder")

defer span.End()

  

span.SetAttributes(

attribute.String("order.tenant_id", req.TenantID),

attribute.Float64("order.amount", req.Amount),

attribute.Int("order.items_count", len(req.Items)),

)

  

// Event: ขั้นตอนสำคัญ

span.AddEvent("validation.start")

if err := uc.validateOrder(ctx, req); err != nil {

span.AddEvent("validation.failed",

trace.WithAttributes(attribute.String("reason", err.Error())),

)

span.RecordError(err)

span.SetStatus(codes.Error, "validation failed")

return nil, err

}

span.AddEvent("validation.success")

  

span.AddEvent("payment.start",

trace.WithAttributes(attribute.Float64("amount", req.Amount)),

)

if err := uc.processPayment(ctx, req); err != nil {

span.RecordError(err)

span.SetStatus(codes.Error, "payment failed")

return nil, err

}

span.AddEvent("payment.success")

  

order, err := uc.repo.CreateOrder(ctx, req)

if err != nil {

span.RecordError(err)

span.SetStatus(codes.Error, "db write failed")

return nil, err

}

  

span.SetAttributes(attribute.String("order.id", order.ID))

span.AddEvent("order.created", trace.WithAttributes(

attribute.String("order.id", order.ID),

))

span.SetStatus(codes.Ok, "")

return order, nil

}

  

// ─── Span Links — เชื่อม trace ต่าง trace ───

// ใช้เมื่อ: async, retry, fan-out

func (w *Worker) ProcessMessage(ctx context.Context, msg kafka.Message) {

// extract trace จาก kafka message header

producerCtx := otel.GetTextMapPropagator().Extract(

context.Background(),

propagation.MapCarrier(msg.Headers),

)

producerSpanCtx := trace.SpanContextFromContext(producerCtx)

  

// สร้าง span ใหม่ที่ link กับ producer span

ctx, span := otel.Tracer("worker").Start(

context.Background(), // consumer เป็น trace ใหม่

"ProcessKafkaMessage",

trace.WithLinks(trace.Link{

SpanContext: producerSpanCtx, // link กับ producer

Attributes: []attribute.KeyValue{

attribute.String("link.type", "producer"),

},

}),

)

defer span.End()

// → Jaeger จะแสดง link ระหว่าง 2 trace

}

```

  

---

  

## 12. Sampling — เลือกเก็บ Trace อะไรบ้าง?

  

### ทำไมต้อง Sample?

  

```

ระบบ Production:

10,000 requests/sec × 10 services = 100,000 spans/sec

ถ้าเก็บทั้งหมด → storage พัง, cost สูง, noise เยอะ

  

Solution: Sample แค่ % ที่ต้องการ

10,000 req/sec × 1% = 100 spans/sec → จัดการได้

```

  

### ประเภทของ Sampler

  

```go

// 1. AlwaysOn — เก็บทุก trace (development เท่านั้น!)

trace.WithSampler(trace.AlwaysSample())

  

// 2. AlwaysOff — ไม่เก็บเลย (testing)

trace.WithSampler(trace.NeverSample())

  

// 3. TraceIDRatioBased — เก็บตาม % ของ traceId

trace.WithSampler(trace.TraceIDRatioBased(0.1)) // เก็บ 10%

  

// ข้อดี: consistent (traceId เดิม = decision เดิม)

// ข้อเสีย: อาจพลาด error trace ที่หายาก

  

// 4. ParentBased — ตาม decision ของ parent (แนะนำ!)

trace.WithSampler(

trace.ParentBased(

trace.TraceIDRatioBased(0.1), // root span: 10%

// ถ้า parent sampled → child sampled ด้วย

// ถ้า parent not sampled → child ไม่ sampled ด้วย

),

)

// ข้อดี: ทุก span ใน trace เดียวกัน consistent

  

// ─── Custom Sampler ─── (เก็บเฉพาะ error + % ของ success)

type errorAlwaysSampler struct {

base trace.Sampler

}

  

func (s errorAlwaysSampler) ShouldSample(p trace.SamplingParameters) trace.SamplingResult {

// เก็บ error ทุกตัว

for _, attr := range p.Attributes {

if attr.Key == "error" && attr.Value.AsBool() {

return trace.SamplingResult{Decision: trace.RecordAndSample}

}

}

// ใช้ base sampler (เช่น 10%) สำหรับ success

return s.base.ShouldSample(p)

}

  

func (s errorAlwaysSampler) Description() string {

return "ErrorAlways+10%"

}

  

// ใช้งาน

trace.WithSampler(

trace.ParentBased(

errorAlwaysSampler{base: trace.TraceIDRatioBased(0.1)},

),

)

```

  

### Head Sampling vs Tail Sampling

  

```

Head Sampling (ทำใน SDK ตอนเริ่ม trace):

ข้อดี: เร็ว, ไม่ต้องเก็บ trace ทั้งหมดก่อน

ข้อเสีย: ตัดสินใจก่อนเห็นผลลัพธ์ → อาจพลาด error ที่เกิดทีหลัง

  

Tail Sampling (ทำใน Collector หลังเห็นทั้ง trace):

ข้อดี: เก็บ error trace ได้แน่นอน, เก็บ slow trace ได้

ข้อเสีย: ต้อง buffer trace ทั้งหมดก่อน → memory สูง

  

Tail Sampling ใน OTEL Collector:

processors:

tail_sampling:

decision_wait: 30s # รอ 30 วิ ให้ trace สมบูรณ์

num_traces: 50000 # buffer 50K traces

policies:

- name: errors-policy # เก็บทุก error

type: status_code

status_code: {status_codes: [ERROR]}

  

- name: slow-policy # เก็บ request ที่ช้า > 2s

type: latency

latency: {threshold_ms: 2000}

  

- name: sample-policy # เก็บ 5% ของที่เหลือ

type: probabilistic

probabilistic: {sampling_percentage: 5}

```

  

---

  

## 13. OTEL Collector — Deep Dive

  

### Architecture ภายใน Collector

  

```

┌─────────────────────────────────────────────────────────────┐

│ OTEL Collector │

│ │

│ ┌──────────┐ ┌───────────┐ ┌──────────────────────┐ │

│ │Receivers │──▶│Processors │──▶│ Exporters │ │

│ │ │ │ │ │ │ │

│ │ otlp/grpc│ │ batch │ │ otlp/jaeger │ │

│ │ otlp/http│ │ filter │ │ prometheus │ │

│ │ kafka │ │ transform │ │ logging │ │

│ │ zipkin │ │ tail_samp │ │ datadog │ │

│ └──────────┘ └───────────┘ └──────────────────────┘ │

│ │

│ Service: │

│ Pipelines (เชื่อม R → P → E) │

│ Extensions (health check, pprof, etc.) │

└─────────────────────────────────────────────────────────────┘

```

  

### Processors ที่ใช้บ่อย

  

```yaml

processors:

# ─── Batch: รวม spans ก่อนส่ง ───

batch:

timeout: 5s # ส่งทุก 5 วิ แม้ batch ไม่เต็ม

send_batch_size: 1000 # ส่งทีละ 1000 spans

send_batch_max_size: 2000

  

# ─── Memory Limiter: กัน OOM ───

memory_limiter:

limit_mib: 512 # ใช้ได้สูงสุด 512MB

spike_limit_mib: 128 # spike ได้อีก 128MB

check_interval: 5s

  

# ─── Attributes: เพิ่ม/ลบ/แก้ attribute ───

attributes:

actions:

- key: "environment"

value: "production"

action: insert # เพิ่มถ้าไม่มี

  

- key: "http.user_agent"

action: delete # ลบ (sensitive data)

  

- key: "tenant_id"

from_attribute: "baggage.tenantId"

action: upsert # copy จาก baggage

  

# ─── Filter: กรอง spans ที่ไม่ต้องการ ───

filter:

error_mode: ignore

traces:

span:

# ลบ health check spans (noise)

- 'attributes["http.target"] == "/health"'

- 'attributes["http.target"] == "/metrics"'

- 'duration < 1ms' # ลบ spans ที่เร็วเกินไป

  

# ─── Transform: แปลงข้อมูล ───

transform:

trace_statements:

- set(name, Concat([attributes["http.method"], " ", attributes["http.route"]], ""))

- delete_key(attributes, "http.user_agent")

  

# ─── Resource Detection: เพิ่ม resource จาก env ───

resourcedetection:

detectors: [env, system, docker, ec2, gce, aks, eks]

timeout: 5s

```

  

### Production-ready otel-config.yaml

  

```yaml

receivers:

otlp:

protocols:

grpc:

endpoint: 0.0.0.0:4317

max_recv_msg_size_mib: 4

http:

endpoint: 0.0.0.0:4318

  

processors:

memory_limiter:

limit_mib: 512

spike_limit_mib: 128

check_interval: 5s

  

batch:

timeout: 5s

send_batch_size: 1000

  

filter:

error_mode: ignore

traces:

span:

- 'attributes["http.target"] == "/health"'

- 'attributes["http.target"] == "/metrics"'

- 'attributes["http.target"] == "/system/health"'

  

resourcedetection:

detectors: [env, system]

  

exporters:

otlp/jaeger:

endpoint: jaeger:4317

tls:

insecure: true

  

# ─── ถ้า production ส่งไป Tempo ───

# otlp/tempo:

# endpoint: tempo:4317

# tls:

# insecure: true

  

logging:

loglevel: warn # production ใช้ warn ไม่ใช่ info

  

extensions:

health_check:

endpoint: 0.0.0.0:13133

pprof:

endpoint: 0.0.0.0:1777

zpages:

endpoint: 0.0.0.0:55679

  

service:

extensions: [health_check, pprof, zpages]

pipelines:

traces:

receivers: [otlp]

processors: [memory_limiter, filter, resourcedetection, batch]

exporters: [otlp/jaeger, logging]

```

  

---

  

## 14. Exporters และ Backend

  

### เปรียบเทียบ Backend

  

```

Jaeger (Open Source, Self-hosted):

✅ ฟรี, ง่าย setup, UI ดี

✅ เหมาะ development, small-medium production

❌ Scale ยากถ้า traffic สูงมาก

❌ ต้องดูแล infrastructure เอง

  

Grafana Tempo (Open Source, Self-hosted):

✅ ฟรี, scale ได้ (S3 backend)

✅ integrate กับ Grafana, Loki, Prometheus

❌ UI ไม่ดีเท่า Jaeger (ต้องใช้ Grafana)

❌ setup ซับซ้อนกว่า

  

Datadog (SaaS):

✅ ไม่ต้องดูแล infrastructure

✅ feature ครบ: Metrics + Logs + Traces ในที่เดียว

❌ แพงมาก ($$$)

  

Elastic APM (SaaS/Self-hosted):

✅ integrate กับ Elasticsearch

❌ ซับซ้อน, resource สูง

```

  

### โปรเจกต์นี้ใช้อะไร?

  

```

Local: docker-compose → Jaeger

Development: Kubernetes → OTEL Collector → (backend ใน cluster)

Production: Kubernetes → OTEL Collector → (backend ใน cluster)

```

  

---

  

## 15. Jaeger — ใช้งานจริงทุก Feature

  

### เปิด Jaeger UI

  

```

Local: http://localhost:16686

```

  

### Search Traces — Tips

  

```

Search by Service:

Service: sellsuki-central-control-backend-development

→ ดู traces ทั้งหมดของ service

  

Search by Error:

Tags: error=true

→ ดูเฉพาะ traces ที่มี error

  

Search by Slow Request:

Min Duration: 500ms

→ ดูเฉพาะ request ที่ใช้เวลา > 500ms

  

Search by Specific User:

Tags: user.id=user_789

→ ต้อง set attribute "user.id" ใน span ก่อน

  

Search by Trace ID:

ใส่ Trace ID โดยตรงถ้ารู้ (เช่นจาก error response)

```

  

### อ่าน Flame Graph

  

```

Trace: POST /v1/companies [Total: 250ms]

─────────────────────────────────────────────────────────────

[0ms]────────────────────────────────────────────────[250ms]

HTTP POST /v1/companies 250ms ← root span

  

[5ms]──────────────────────────────────────[200ms]

CompanyUseCase.Create 195ms

  

[10ms]────────────────────[100ms]

CompanyRepository.Create 90ms

  

[15ms]──────────[90ms]

INSERT companies 75ms ← ช้า! ดู db.statement ใน Tags

  

[110ms]────────────[180ms]

NotifyService.Send 70ms

  

[205ms]──────[240ms]

AuditLog.Write 35ms

  

วิธีอ่าน:

- แถวบน = parent span

- แถวล่าง = child span (indented)

- ยาว = ใช้เวลานาน

- ถ้า child ครอบ parent เกือบทั้งหมด → parent รอ child

- Gap ระหว่าง span = overhead ของโค้ด ระหว่าง operations

```

  

### ดู Tags และ Logs

  

```

คลิก Span → เห็น:

  

Tags:

http.method = POST

http.url = /v1/companies

http.status_code = 201

company.name = "Sellsuki Co., Ltd."

db.system = postgresql

db.statement = "INSERT INTO companies..."

span.kind = server

otel.library.name = otelfiber

  

Logs (Events):

10:30:00.100 validation.start

10:30:00.150 validation.success

10:30:00.160 db.insert.start

10:30:00.235 db.insert.complete

  

Process:

service.name = sellsuki-central-control-backend-development

service.version = v0.0.0

environment = development

```

  

### Trace Comparison — เปรียบ 2 Traces

  

```

Jaeger มีฟีเจอร์ Compare Traces:

1. เปิด Trace A → copy URL

2. เปิด Trace B → copy URL

3. ไปที่ /compare?traceID=A&traceID=B

  

ใช้เมื่อ:

- เปรียบ request ที่ช้า vs เร็ว

- debug ว่า deploy ใหม่ทำให้ช้าลงตรงไหน

```

  

---

  

## 16. Error Handling กับ Tracing

  

### Pattern การ Record Error ที่ถูกต้อง

  

```go

func (r *repo) GetByID(ctx context.Context, id string) (*Company, error) {

ctx, span := otel.Tracer("repo").Start(ctx, "GetByID")

defer span.End()

  

var company Company

err := r.db.WithContext(ctx).First(&company, "id = ?", id).Error

  

if err != nil {

// 1. RecordError: เพิ่ม exception event ใน span

// → Jaeger จะแสดง exception.type, exception.message, stacktrace

span.RecordError(err)

  

// 2. SetStatus: กำหนด status ของ span

// → Jaeger จะแสดง ❌ และ color แดง

span.SetStatus(codes.Error, "failed to get company")

  

return nil, err

}

  

span.SetStatus(codes.Ok, "")

return &company, nil

}

  

// ─── RecordError สร้าง Event นี้อัตโนมัติ ───

// {

// "name": "exception",

// "attributes": {

// "exception.type": "*errors.errorString",

// "exception.message": "record not found",

// "exception.stacktrace": "goroutine 1 [running]:\n..."

// }

// }

```

  

### Return TraceID ใน Error Response

  

```go

// helper/handler.go ในโปรเจกต์นี้ใช้แบบนี้:

func ErrorResponse(ctx context.Context, code int, message string) fiber.Map {

spanCtx := trace.SpanContextFromContext(ctx)

  

response := fiber.Map{

"success": false,

"error": message,

}

  

if spanCtx.IsValid() {

// ใส่ TraceID ใน error response

// → user/engineer เอา ID นี้ไปค้นหาใน Jaeger ได้ทันที

response["traceId"] = spanCtx.TraceID().String()

}

  

return response

}

  

// ตัวอย่าง response:

// {

// "success": false,

// "error": "company not found",

// "traceId": "4bf92f3577b34da6a3ce929d0e0e4736"

// }

//

// Engineer เปิด Jaeger → Search → TraceID → เห็นทุกอย่างทันที

```

  

---

  

## 17. Performance และ Production Patterns

  

### ต้นทุนของ Tracing

  

```

ทุก Span มีต้นทุน:

- Memory: ~1KB per span

- CPU: สร้าง UUID, timestamp, serialize

- Network: ส่งไป Collector

  

โปรเจกต์ที่ดี:

- ไม่สร้าง Span ที่ไม่จำเป็น

- ใช้ Sampling ใน production

- ใช้ BatchProcessor (ไม่ใช่ Sync)

- มี Memory Limiter ใน Collector

```

  

### Do's and Don'ts

  

```go

// ✅ DO: Span ชื่อบอก operation ชัดเจน

tracer.Start(ctx, "CompanyRepository.GetByID")

tracer.Start(ctx, "HTTP GET /v1/companies/{id}")

tracer.Start(ctx, "Kafka.Publish order.created")

  

// ❌ DON'T: ชื่อ dynamic (ทำให้ cardinality สูง)

tracer.Start(ctx, "get company id="+id) // ← unique ทุก call!

tracer.Start(ctx, fmt.Sprintf("query %s", sql))

  

// ✅ DO: ใส่ dynamic value เป็น Attribute

ctx, span := tracer.Start(ctx, "GetCompany")

span.SetAttributes(attribute.String("company.id", id)) // ← attribute แทน

  

// ─────────────────────────────────────────────────

  

// ✅ DO: ส่ง ctx ผ่านทุก function

func (uc *UseCase) CreateCompany(ctx context.Context, req Request) (*Company, error)

func (r *Repo) Insert(ctx context.Context, company *Company) error

func (c *Client) NotifyCreated(ctx context.Context, company *Company) error

  

// ❌ DON'T: ctx หาย → trace chain ขาด

func (uc *UseCase) CreateCompany(req Request) (*Company, error) // ไม่มี ctx!

  

// ─────────────────────────────────────────────────

  

// ✅ DO: span เฉพาะ operation ที่ significant

ctx, span := tracer.Start(ctx, "ExternalAPI.Charge") // HTTP call → ใช่!

ctx, span := tracer.Start(ctx, "DB.InsertOrder") // DB query → ใช่!

  

// ❌ DON'T: span เล็กเกินไป (noise)

ctx, span := tracer.Start(ctx, "validateEmail") // simple regex → ไม่จำเป็น

ctx, span := tracer.Start(ctx, "formatName") // string op → ไม่จำเป็น

  

// ─────────────────────────────────────────────────

  

// ✅ DO: Graceful shutdown → flush spans

func cleanup(tp *sdktrace.TracerProvider) {

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)

defer cancel()

  

if err := tp.Shutdown(ctx); err != nil {

log.Printf("Error shutting down tracer: %v", err)

}

// → flush spans ที่ค้างอยู่ใน buffer ก่อน process ตาย

}

```

  

---

  

## 18. Architecture ในโปรเจกต์นี้ — ทุก Layer

  

### ภาพรวม Flow ทั้งหมด

  

```

HTTP Request (จาก client)

│

▼

fiber_server.go

otelfiber.Middleware()

← Extract traceparent header (ถ้ามี)

← สร้าง SERVER span: "GET /v1/companies/:id"

← วาง span ใน ctx

│

▼

route/v1/*.go (handler)

ctx := c.UserContext() ← ctx มี span แล้ว

useCase.GetCompany(ctx, id)

│

▼

use_case/company_use_case.go

ctx ส่งต่อโดยไม่แก้

repo.GetByID(ctx, id)

│

▼

repository/company_repository/company.go

ctx, span := otel.Tracer("").Start(ctx, "GetByID")

defer span.End()

span.SetAttributes(attribute.String("company.id", id))

│

db.WithContext(ctx).First(...) ← gorm ส่ง ctx ให้ DB driver

│

▼

PostgreSQL

│

▼ (span End, ส่ง span ไป processor)

│

▼

BatchSpanProcessor (ใน SDK)

buffer spans ไว้ก่อน

│

▼ (ทุก 5 วิ หรือ buffer เต็ม)

│

▼

OTEL Collector (localhost:4317)

receivers.otlp.grpc รับ batch

│

▼

processors:

filter → ตัด /health spans

batch → รวมอีกรอบ

│

▼

exporters.otlp/jaeger → Jaeger (localhost:4317 ของ jaeger)

exporters.logging → terminal log

│

▼

Jaeger Storage (in-memory หรือ Cassandra)

│

▼

Jaeger UI (localhost:16686)

← ค้นหา, ดู trace, debug

```

  

### โค้ดที่เกี่ยวข้องในโปรเจกต์

  

```

cmd/generics_server/

main.go → initTracer() เรียกที่นี่

helper.go → initTracer() implementation

  

src/interface/fiber_server/

fiber_server.go → otelfiber.Middleware() line 65

helper/handler.go → ErrorResponse พร้อม traceId

  

src/repository/

company_repository/company.go → manual spans

country_repository/country.go → manual spans

management_repository/management.go → manual spans

provider_bank_repository/ → manual spans

payment_channel_repository/grpc.go → manual spans

  

docker-compose.yml → otel + jaeger services

otel-config.yaml → collector pipeline config

```

  

### Environment Variables ทั้งหมดที่เกี่ยวข้อง

  

```bash

# ─── Application ───

APP_NAME=sellsuki-central-control-backend

APP_VERSION=v1.0.0

ENVIRONMENT=development

  

# ─── OTEL ───

OTEL_GRPC_ENDPOINT=localhost:4317 # local

# OTEL_GRPC_ENDPOINT=opentelemetry-collector.system-monitor.svc.cluster.local:4317 # k8s

  

# ─── ตัวเลือกเพิ่มเติม (OTEL standard env vars) ───

OTEL_SERVICE_NAME=sellsuki-central # override service name

OTEL_TRACES_SAMPLER=parentbased_traceidratio

OTEL_TRACES_SAMPLER_ARG=0.1 # sample 10%

```

  

---

  

## Quick Reference

  

### Span API ทั้งหมด

  

```go

ctx, span := tracer.Start(ctx, "name",

trace.WithSpanKind(trace.SpanKindServer),

trace.WithAttributes(attribute.String("key", "value")),

trace.WithTimestamp(time.Now()),

)

defer span.End()

  

span.SetAttributes(attribute.String("key", "val"))

span.SetAttributes(attribute.Int("count", 42))

span.SetAttributes(attribute.Float64("amount", 99.9))

span.SetAttributes(attribute.Bool("success", true))

span.SetAttributes(attribute.StringSlice("tags", []string{"a", "b"}))

  

span.AddEvent("checkpoint.name",

trace.WithAttributes(attribute.String("detail", "value")),

trace.WithTimestamp(time.Now()),

)

  

span.RecordError(err,

trace.WithAttributes(attribute.String("context", "during db query")),

)

  

span.SetStatus(codes.Ok, "")

span.SetStatus(codes.Error, "something failed")

  

spanCtx := span.SpanContext()

spanCtx.TraceID().String() // "4bf92f35..."

spanCtx.SpanID().String() // "00f067aa..."

spanCtx.IsSampled() // true/false

```

  

### Checklist ทุก Service ที่จะเพิ่ม Tracing

  

```

Setup:

□ initTracer() พร้อม SetTracerProvider

□ initTracer() พร้อม SetTextMapPropagator (TraceContext + Baggage)

□ Resource มี ServiceName, ServiceVersion, Environment

  

HTTP:

□ otelfiber/otelhttp middleware (server)

□ otelhttp.NewTransport หรือ inject manual (client)

  

gRPC:

□ otelgrpc.NewServerHandler() (server)

□ otelgrpc.NewClientHandler() (client)

  

Code:

□ ทุก function รับ ctx เป็น parameter แรก

□ span สำหรับ operation สำคัญ (DB, external call, business logic)

□ span.RecordError() + span.SetStatus() ทุกที่ที่ error

□ defer span.End() ทุก span ที่สร้าง

□ ส่ง ctx ต่อทุก function call

  

Response:

□ ใส่ traceId ใน error response

```

  

---

  

*อ้างอิง: [opentelemetry.io](https://opentelemetry.io/docs/) | [W3C TraceContext](https://www.w3.org/TR/trace-context/) | [Jaeger Docs](https://www.jaegertracing.io/docs/)*