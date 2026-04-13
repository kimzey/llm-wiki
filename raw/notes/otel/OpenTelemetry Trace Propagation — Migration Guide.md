# OpenTelemetry Trace Propagation — Migration Guide

> **Ticket:** 1961-fix-tracer
> **ปัญหาที่แก้:** Span ของ service ตัวเองมี แต่ไม่มี span ของ downstream services (gRPC / HTTP) ทำให้ trace ขาดหายและ trace ID ต่างกันระหว่าง services

---

## สาเหตุของปัญหา

OpenTelemetry จะ propagate trace context ผ่าน **header** ของ request ไปยัง downstream service โดยอัตโนมัติ **ก็ต่อเมื่อ** client มี instrumented transport/handler ติดตั้งไว้

| Protocol | สิ่งที่ต้องเพิ่ม | Header ที่ inject |
|---|---|---|
| gRPC client | `otelgrpc.NewClientHandler()` | gRPC metadata `traceparent` |
| gRPC server | `otelgrpc.NewServerHandler()` | อ่าน `traceparent` จาก incoming metadata |
| HTTP client (resty) | `otelhttp.NewTransport(...)` | HTTP header `traceparent` |
| HTTP client (net/http) | `otelhttp.NewTransport(...)` | HTTP header `traceparent` |

---

## 1. Dependencies ที่ต้องเพิ่มใน `go.mod`

```bash
go get go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc@v0.65.0
go get go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp@v0.65.0
```

**ผลใน `go.mod`:**
```diff
require (
+   go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc v0.65.0
    go.opentelemetry.io/otel v1.40.0
    go.opentelemetry.io/otel/sdk v1.40.0
    go.opentelemetry.io/otel/trace v1.40.0
)

// indirect
+   github.com/felixge/httpsnoop v1.0.4
+   go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp v0.65.0
```

> **หมายเหตุ:** ใช้ version เดียวกับ otel core (`v0.65.0` → `otel v1.40.0`) เพื่อป้องกัน compatibility issues

---

## 2. `initTracer` — เพิ่ม TextMapPropagator

**ไฟล์:** `cmd/<service>/helper.go` (หรือ entrypoint ของแต่ละ service)

```diff
import (
+   "go.opentelemetry.io/otel/propagation"
)

func initTracer(cfg config) {
    // ... setup exporter และ TracerProvider เหมือนเดิม ...

    otel.SetTracerProvider(tp)

+   // เพิ่มส่วนนี้ — บอก otel ให้ใช้ W3C TraceContext format
+   otel.SetTextMapPropagator(propagation.NewCompositeTextMapPropagator(
+       propagation.TraceContext{},  // inject/extract "traceparent" header
+       propagation.Baggage{},       // inject/extract "baggage" header
+   ))
}
```

> **สำคัญมาก:** ต้องเรียก `otel.SetTextMapPropagator` **ก่อน** สร้าง gRPC/HTTP client ทุกตัว
> ลำดับใน `main()` ต้องเป็น: `initTracer()` → `initRepositories()`

---

## 3. gRPC Server — รับ trace context จาก incoming request

**ไฟล์:** `src/interface/grpc_server/grpc_server.go`

```diff
import (
+   "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
    "google.golang.org/grpc"
)

func New(uc *use_case.UseCase) *GrpcServer {
    return &GrpcServer{
        server: grpc.NewServer(
+           grpc.StatsHandler(otelgrpc.NewServerHandler()),  // ← เพิ่มบรรทัดนี้
            grpc.StreamInterceptor(...),
            grpc.UnaryInterceptor(...),
        ),
    }
}
```

> ถ้าไม่ใส่ — server จะ **สร้าง root trace ใหม่** ทุก request แทนที่จะ continue จาก parent trace ของ caller

---

## 4. gRPC Client — ส่ง trace context ไปพร้อม request

**ไฟล์:** `cmd/<service>/helper.go`

```diff
import (
+   "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

// ทำซ้ำกับทุก grpc.NewClient ในโปรเจค
conn, err := grpc.NewClient(cfg.Services.SomeGrpcServer,
    grpc.WithTransportCredentials(insecure.NewCredentials()),
+   grpc.WithStatsHandler(otelgrpc.NewClientHandler()),  // ← เพิ่มบรรทัดนี้
)
```

> **ทำกับทุก** `grpc.NewClient` — ถ้าขาดตัวไหน trace จะขาดในเส้นนั้น

---

## 5. HTTP Client (resty) — ส่ง trace context ผ่าน HTTP header

**ไฟล์:** `src/repository/<name>/rest.go`

```diff
import (
+   "net/http"
+   "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func NewRestClient(baseUrl string) repository.SomeRepository {
    return &restClient{
        client: resty.New().
            SetBaseURL(baseUrl).
+           SetTransport(otelhttp.NewTransport(http.DefaultTransport)),  // ← เพิ่มบรรทัดนี้
    }
}
```

---

## 6. HTTP Client (net/http ธรรมดา) — เช่น Ory Kratos SDK

**ไฟล์:** `src/repository/<name>/http.go`

```diff
import (
+   "net/http"
+   "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
)

func NewSomeHttpClient(baseUrl string) repository.SomeRepository {
    configuration := client.NewConfiguration()
    configuration.Servers = []client.ServerConfiguration{{URL: baseUrl}}

+   // เพิ่ม otelhttp transport เข้า HTTP client ของ SDK
+   configuration.HTTPClient = &http.Client{
+       Transport: otelhttp.NewTransport(http.DefaultTransport),
+   }

    apiClient := client.NewAPIClient(configuration)
    return &someClient{client: apiClient}
}
```

---

## 7. Repository Methods — เพิ่ม span ใน method

เพิ่มใน **ทุก method** ของ repository เพื่อให้เห็น span ใน trace:

```diff
// ไฟล์: src/repository/<name>/<file>.go

// ที่ระดับ package — ประกาศ tracer ครั้งเดียว (มักมีอยู่แล้ว)
var tracer = otel.Tracer("<package_name>")

func (r repo) SomeMethod(ctx context.Context, ...) (..., error) {
+   ctx, sp := tracer.Start(ctx, "<package_name>.SomeMethod")
+   defer sp.End()

    result, err := r.client.DoSomething(ctx, ...)
    if err != nil {
+       sp.RecordError(err)
        return ..., err
    }

    return result, nil
}
```

> **สำคัญ:** ต้องใช้ `ctx` ที่ได้จาก `tracer.Start` ตอน call downstream — มิฉะนั้น `otelhttp`/`otelgrpc` จะไม่รู้ว่ามี parent span

---

## Checklist สำหรับ service ใหม่

```
[ ] go.mod — เพิ่ม otelgrpc และ otelhttp
[ ] initTracer — เพิ่ม otel.SetTextMapPropagator
[ ] gRPC Server — grpc.StatsHandler(otelgrpc.NewServerHandler())
[ ] gRPC Clients — grpc.WithStatsHandler(otelgrpc.NewClientHandler()) ทุกตัว
[ ] HTTP Clients (resty) — SetTransport(otelhttp.NewTransport(...))
[ ] HTTP Clients (net/http SDK) — configuration.HTTPClient พร้อม otelhttp transport
[ ] Repository methods — tracer.Start / sp.RecordError / defer sp.End()
```

---

## ผล trace ที่ควรเห็นหลัง implement

```
[HTTP Request]  trace_id=abc123...
    └── [use_case.ListUsersWithRoles]
            ├── [role_and_permission_repository.CheckPermission]
            │       └── [gRPC → RolePermissionService]   ← same trace_id
            ├── [role_and_permission_repository.ListUserRoles]
            │       └── [gRPC → RolePermissionService]   ← same trace_id
            ├── [identity_repository.GetUserProfileFromID]
            │       └── [HTTP GET → Ory Kratos]          ← same trace_id
            └── [management_repository.GetNameStore]
                    └── [HTTP GET → ManagementService]   ← same trace_id
```

ทุก span ต้องมี `trace_id` เดียวกันตลอด chain

---

## ข้อควรระวัง

1. **version ต้องตรงกัน** — `otelgrpc` และ `otelhttp` ต้องใช้ version เดียวกัน (`v0.65.0`) และ compatible กับ `otel core` (`v1.40.0`)
2. **`ctx` ต้องส่งต่อเสมอ** — อย่า pass `context.Background()` แทน `ctx` ใน downstream call
3. **ฝั่ง server ต้องมี `NewServerHandler` ด้วย** — ถ้า downstream service ไม่ติดตั้ง server handler จะสร้าง trace ใหม่ทุกครั้งแม้ client จะส่ง `traceparent` ไปแล้ว
