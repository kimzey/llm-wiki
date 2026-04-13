---
title: "OTel Go Instrumentation"
type: concept
tags: [opentelemetry, go, otelgrpc, otelhttp, otelfiber, instrumentation, statshandler]
sources: [Distributed Tracing & Context Propagation.md, OpenTelemetry Trace Propagation — Migration Guide.md, context-usercontext-vs-getspancontext.md, เข้าใจ SetTextMapPropagator ทุกชิ้น — ทำไม ทำอะไร เชื่อมกันยังไง.md]
related: [wiki/concepts/context-propagation.md, wiki/concepts/opentelemetry.md, wiki/concepts/distributed-tracing.md]
created: 2026-04-13
updated: 2026-04-13
---

## สรุปสั้น

การ instrument OTel ใน Go services ต้องทำ 3 ส่วนให้ครบ: TracerProvider, TextMapPropagator, และ transport handlers (otelgrpc/otelhttp/otelfiber) — ขาดส่วนใดส่วนหนึ่ง trace จะขาดหรือไม่ส่งไป backend

## อธิบาย

### 3 ส่วนที่ต้องทำครบ

```
┌─────────────────┬──────────────────────────┬────────────────────┐
│ TracerProvider  │   TextMapPropagator       │  Transport Handler │
│                 │                           │                    │
│ ส่ง trace data  │   inject / extract header │  hook เข้า         │
│ ไปที่ backend   │   ก่อน/หลัง request       │  protocol lifecycle │
└─────────────────┴──────────────────────────┴────────────────────┘
```

### ลำดับการ init (สำคัญมาก)

```go
func main() {
    cfg := loadConfig()
    initTracer(cfg)         // 1. TracerProvider + TextMapPropagator
    repos := initRepos(cfg) // 2. สร้าง gRPC/HTTP clients (ดึง propagator ที่ set ไว้)
    server := initServer(repos)
    server.Start()
}
```

ต้อง `initTracer()` ก่อนสร้าง client เสมอ — ถ้า client ถูกสร้างก่อน set propagator จะได้ noop

### initTracer

```go
func initTracer(cfg config) {
    exporter, _ := otlptrace.New(ctx, otlptracegrpc.NewClient(
        otlptracegrpc.WithEndpoint(cfg.OtelEndpoint),
    ))
    tp := trace.NewTracerProvider(
        trace.WithBatcher(exporter),
        trace.WithResource(resource.NewWithAttributes(
            semconv.SchemaURL,
            semconv.ServiceName("my-service"),
        )),
    )
    otel.SetTracerProvider(tp)
    otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
        propagation.TraceContext{},
        propagation.Baggage{},
    ))
}
```

### gRPC Server

```go
grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()), // extract traceparent จาก incoming
    grpc.UnaryInterceptor(...),
)
```

### gRPC Client

```go
grpc.NewClient(address,
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()), // inject traceparent ก่อนส่ง
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

### HTTP Client (resty)

```go
resty.New().SetTransport(otelhttp.NewTransport(http.DefaultTransport))
```

### HTTP Client (net/http SDK)

```go
configuration.HTTPClient = &http.Client{
    Transport: otelhttp.NewTransport(http.DefaultTransport),
}
```

### Fiber HTTP Server

```go
app.Use(otelfiber.Middleware())

// ใน handler ต้องใช้ c.UserContext() ไม่ใช่ c.Context()
func handler(c *fiber.Ctx) error {
    ctx := c.UserContext() // ← context ที่มี OTel span อยู่ข้างใน
    result, err := useCase.DoSomething(ctx, req)
    ...
}
```

### Repository Methods

```go
var tracer = otel.Tracer("package-name")

func (r repo) SomeMethod(ctx context.Context, ...) (..., error) {
    ctx, sp := tracer.Start(ctx, "package.SomeMethod")
    defer sp.End()

    result, err := r.client.Call(ctx, ...) // ต้องส่ง ctx ใหม่ต่อ
    if err != nil {
        sp.RecordError(err)
        return ..., err
    }
    return result, nil
}
```

## ประเด็นสำคัญ

- **StatsHandler ดีกว่า Interceptor**: StatsHandler เห็น `InHeader`/`OutHeader` event → extract `traceparent` ได้; Interceptor จับได้แค่ unary/streaming call ไม่เห็น metadata
- **Fiber: `c.UserContext()` ไม่ใช่ `c.Context()`**: otelfiber inject span ผ่าน `c.SetUserContext(ctx)` → ต้องอ่านจาก UserContext; `c.Context()` คือ fasthttp context ที่ไม่มี span
- **ทุก `grpc.NewClient` ต้องมี handler**: ถ้าขาดตัวไหน trace จะขาดในเส้นนั้นเท่านั้น
- **ส่ง `ctx` ต่อเสมอ**: อย่า pass `context.Background()` แทน `ctx` ใน downstream call → chain ขาดทันที
- **Dependency versions ต้องตรงกัน**: otelgrpc v0.65.0 + otelhttp v0.65.0 → otel core v1.40.0

## ตัวอย่าง / กรณีศึกษา

ตรวจสอบว่า trace propagate ถูกต้องหรือไม่:
```go
span := trace.SpanFromContext(ctx)
sc := span.SpanContext()
fmt.Printf("IsValid: %v\n", sc.IsValid())     // false = ไม่มี trace
fmt.Printf("IsSampled: %v\n", sc.IsSampled()) // false = ไม่ส่งไป backend
```

## ความสัมพันธ์กับ concept อื่น

- [[wiki/concepts/context-propagation|Context Propagation]] — transport handlers ทำหน้าที่ inject/extract ผ่าน global propagator
- [[wiki/concepts/opentelemetry|OpenTelemetry]] — OTel architecture และ component ที่ทำงานร่วมกัน

## แหล่งที่มา

- [[wiki/sources/distributed-tracing-context-propagation|Distributed Tracing & Context Propagation]]
- [[wiki/sources/otel-trace-propagation-migration-guide|OTel Trace Propagation Migration Guide]]
- [[wiki/sources/fiber-context-otel-bug|Fiber Context OTel Bug]]
- [[wiki/sources/settextmappropagator-explained|SetTextMapPropagator Explained]]
