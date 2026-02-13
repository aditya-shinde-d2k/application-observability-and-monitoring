# OpenTelemetry LGTM Implementation - Developer Guide

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [What is OpenTelemetry & LGTM Stack](#what-is-opentelemetry--lgtm-stack)
3. [Architecture Overview](#architecture-overview)
4. [Code Changes Analysis](#code-changes-analysis)
5. [Execution Flow](#execution-flow)
6. [Configuration Guide](#configuration-guide)
7. [Testing & Validation](#testing--validation)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)
10. [FAQ](#faq)

---

## Executive Summary

This document provides a comprehensive guide to the OpenTelemetry (OTEL) LGTM observability implementation for the **ENSURE Backend Service (CRisMacWebAPI)**. The implementation adds industry-standard distributed tracing, metrics, and log correlation to enable better monitoring, debugging, and performance analysis.

### What Was Implemented?

**Observability Pillars:**
- **Traces** (Distributed Tracing): Track requests across services and components
- **Metrics** (RED Metrics): Monitor Rate, Errors, and Duration of all API requests
- **Logs** (Structured Logging): Correlate logs with traces using TraceId/SpanId

**Technology Stack:**
- **L**oki — Log aggregation
- **G**rafana — Visualization and dashboards
- **T**empo — Distributed tracing backend
- **M**imir/Prometheus — Metrics storage and querying

### Key Benefits

1. **Faster Debugging**: Trace requests end-to-end with correlated logs
2. **Better Monitoring**: Real-time RED metrics dashboards for all endpoints
3. **Production Readiness**: Industry-standard observability patterns
4. **Performance Insights**: Identify slow endpoints and bottlenecks
5. **Error Tracking**: Automatic error capture with full context
6. **Zero Code Changes Required**: Automatic instrumentation for most scenarios

---

## What is OpenTelemetry & LGTM Stack?

### OpenTelemetry (OTEL)

**OpenTelemetry** is an open-source observability framework that provides:
- **Standardized APIs** for generating telemetry data
- **Automatic instrumentation** for popular frameworks (ASP.NET Core, HttpClient, SQL, etc.)
- **Vendor-neutral exporters** (works with any backend: Grafana, Datadog, New Relic, etc.)
- **Context propagation** for distributed tracing across services

**Why OpenTelemetry?**
- Industry standard (CNCF project)
- Vendor-neutral (no vendor lock-in)
- Rich ecosystem and community support
- Automatic instrumentation reduces manual work

### LGTM Stack

The **LGTM Stack** is a popular open-source observability stack:

| Component | Purpose | Port |
|-----------|---------|------|
| **Loki** | Log aggregation and querying | 3100 |
| **Grafana** | Visualization dashboards | 3000 |
| **Tempo** | Distributed tracing backend | 3200 |
| **Mimir** (or Prometheus) | Metrics storage and querying | 9009/9090 |
| **OTLP Collector** | Receives and routes telemetry data | 4317 (gRPC), 4318 (HTTP) |

**Data Flow:**
```
Application (CRisMacWebAPI)
    ↓ (Traces, Metrics, Logs via OTLP)
OTLP Collector
    ↓ (Routes telemetry)
    ├─→ Tempo (Traces)
    ├─→ Loki (Logs)
    └─→ Mimir/Prometheus (Metrics)
            ↓
        Grafana (Query & Visualize)
```

---

## Architecture Overview

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CRisMacWebAPI (.NET 8)                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  1. Request Pipeline (Middleware Chain)            │   │
│  │                                                    │   │
│  │  UseExceptionHandler (GlobalExceptionHandler)      │   │
│  │    ↓                                               │   │
│  │  RequestResponseLoggingMiddleware                  │   │
│  │    ↓                                               │   │
│  │  UseSerilogRequestLogging                          │   │
│  │    ↓                                               │   │
│  │  UseRouting                                        │   │
│  │    ↓                                               │   │
│  │  RedMetricsMiddleware (Collects RED Metrics)       │   │
│  │    ↓                                               │   │
│  │  Controller Actions                                │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  2. OpenTelemetry SDK                              │   │
│  │                                                    │   │
│  │  • AspNetCore Instrumentation (Auto)               │   │
│  │  • HttpClient Instrumentation (Auto)               │   │
│  │  • SqlClient Instrumentation (Auto)                │   │
│  │  • Runtime Instrumentation (Auto)                  │   │
│  │  • Custom Metrics (ApplicationTelemetry)           │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌────────────────────────────────────────────────────┐   │
│  │  3. Serilog (Structured Logging)                   │   │
│  │                                                    │   │
│  │  • Console Sink (with TraceId/SpanId)              │   │
│  │  • File Sink (JSON format)                         │   │
│  │  • OpenTelemetry Sink (exports to Loki via OTLP)  │   │
│  │  • SpanId/TraceId Enrichers                        │   │
│  └────────────────────────────────────────────────────┘   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
                           ↓
              ┌────────────────────────┐
              │   OTLP Collector       │
              │   (localhost:4317)     │
              └────────────────────────┘
                           ↓
         ┌─────────────────┼─────────────────┐
         ↓                 ↓                  ↓
    ┌─────────┐      ┌─────────┐       ┌──────────┐
    │  Tempo  │      │  Loki   │       │  Mimir   │
    │ (Traces)│      │ (Logs)  │       │(Metrics) │
    └─────────┘      └─────────┘       └──────────┘
         └─────────────────┬──────────────────┘
                           ↓
                    ┌────────────┐
                    │  Grafana   │
                    │(Dashboards)│
                    └────────────┘
```

### Key Components

| Component | File | Responsibility |
|-----------|------|---------------|
| **OpenTelemetryExtensions** | [Extensions/OpenTelemetryExtensions.cs](Extensions/OpenTelemetryExtensions.cs) | Configures OTEL SDK; registers tracing, metrics, and OTLP exporters |
| **ApplicationTelemetry** | [Telemetry/ApplicationTelemetry.cs](Telemetry/ApplicationTelemetry.cs) | Defines all custom RED metrics counters and histograms |
| **RedMetricsMiddleware** | [Middleware/RedMetricsMiddleware.cs](Middleware/RedMetricsMiddleware.cs) | Automatically records Rate, Error, Duration for every HTTP request |
| **GlobalExceptionHandler** | [Helpers/GlobalExceptionHandler.cs](Helpers/GlobalExceptionHandler.cs) | Catches unhandled exceptions; returns ProblemDetails with TraceId |
| **OpenTelemetrySettings** | [Helpers/OpenTelemetrySettings.cs](Helpers/OpenTelemetrySettings.cs) | Configuration model; holds excluded paths and OTLP endpoint |
| **DiagnosticsController** | [Controllers/DiagnosticsController.cs](Controllers/DiagnosticsController.cs) | Test endpoints for validating the full OTEL integration |

---

## Code Changes Analysis

### 1. NuGet Packages Added (`CRisMacWebAPI.csproj`)

**OpenTelemetry Core:**
```xml
<PackageReference Include="OpenTelemetry"                                Version="1.11.0" />
<PackageReference Include="OpenTelemetry.Api"                            Version="1.11.2" />
<PackageReference Include="OpenTelemetry.Exporter.OpenTelemetryProtocol" Version="1.11.0" />
<PackageReference Include="OpenTelemetry.Extensions.Hosting"             Version="1.11.0" />
```

**Automatic Instrumentation:**
```xml
<PackageReference Include="OpenTelemetry.Instrumentation.AspNetCore" Version="1.10.1"      />
<PackageReference Include="OpenTelemetry.Instrumentation.Http"       Version="1.11.0"      />
<PackageReference Include="OpenTelemetry.Instrumentation.SqlClient"  Version="1.9.0-beta.1"/>
<PackageReference Include="OpenTelemetry.Instrumentation.Runtime"    Version="1.11.0"      />
<PackageReference Include="OpenTelemetry.Instrumentation.Process"    Version="0.5.0-beta.6"/>
```

**Serilog OTEL Integration:**
```xml
<PackageReference Include="Serilog.Enrichers.Span"       Version="3.1.0" />
<PackageReference Include="Serilog.Sinks.OpenTelemetry"  Version="4.1.0" />
```

> **Why these packages?**
> - Auto-instrumentation means **zero manual code** to trace HTTP requests, SQL queries, and outbound HTTP calls.
> - `Serilog.Enrichers.Span` attaches the current `TraceId` and `SpanId` to every log entry so logs and traces can be correlated in Grafana.
> - `OpenTelemetry.Exporter.OpenTelemetryProtocol` ships telemetry via the standard OTLP protocol — works with Grafana, Datadog, New Relic, and others without any code changes.

---

### 2. New Files Created

#### `Extensions/OpenTelemetryExtensions.cs`
Provides a single `AddOpenTelemetryObservability()` extension method that wires up:
- Resource attributes (service name, version, environment, host)
- Tracing pipeline: ASP.NET Core → HttpClient → SqlClient → OTLP exporter
- Metrics pipeline: ASP.NET Core → Runtime → custom `ApplicationTelemetry.Meter` → OTLP exporter

#### `Telemetry/ApplicationTelemetry.cs`
Central registry for all custom metrics. Exposes static counters and histograms:

| Metric | Type | Description |
|--------|------|-------------|
| `http_server_requests_total` | Counter | Total HTTP requests (Rate) |
| `http_server_request_errors_total` | Counter | Total 4xx + 5xx errors (Errors) |
| `http_server_request_duration_milliseconds` | Histogram | Request duration with percentiles (Duration) |
| `http_server_active_requests` | UpDownCounter | In-flight requests right now |
| `api_validation_errors_total` | Counter | 400 Bad Request from model validation only |

Also exposes `ApplicationTelemetry.ActivitySource` for adding custom spans anywhere in the codebase.

#### `Middleware/RedMetricsMiddleware.cs`
Wraps every request and records the three RED metrics. Key design decision:

```csharp
// Uses Response.OnCompleted callback instead of a try/finally around _next()
// This guarantees we capture the FINAL status code after GlobalExceptionHandler
// has written the 500 response — not before.
context.Response.OnCompleted(() =>
{
    stopwatch.Stop();
    RecordMetrics(context, endpoint, method, stopwatch.Elapsed.TotalMilliseconds);
    ApplicationTelemetry.ActiveRequests.Add(-1, ...);
    return Task.CompletedTask;
});

await _next(context);   // Does NOT wrap in try/catch — lets exceptions propagate
```

Also uses `GetEndpointTemplate()` to extract route patterns like `/api/users/{id}` instead of `/api/users/123`, preventing **high-cardinality** metric explosions.

#### `Helpers/GlobalExceptionHandler.cs`
Implements the .NET 8 `IExceptionHandler` interface — replaces the old `ErrorHandlerMiddleware`.

What it does for every unhandled exception:
1. Extracts the current `TraceId` from `Activity.Current`
2. Logs the **full exception with stack trace** (server-side only)
3. Calls `ApplicationTelemetry.RecordException()` to mark the trace span as failed
4. Returns a safe `ProblemDetails` JSON response with `traceId` included
5. Maps known exception types to correct HTTP status codes (400/404/500)

#### `Helpers/OpenTelemetrySettings.cs`
POCO that binds the `"OpenTelemetry"` section from `appsettings.json`. Provides `IsPathExcluded(path)` helper with wildcard support, used by both `RequestResponseLoggingMiddleware` and `RedMetricsMiddleware` for consistent path filtering.

#### `Controllers/DiagnosticsController.cs`
Test-only controller. Should be **removed or secured before production**. Endpoints:

| Endpoint | Triggers |
|----------|---------|
| `GET /api/diagnostics/test-otel` | Logs at Info/Warn/Error levels; returns TraceId |
| `POST /api/diagnostics/test-validation` | Model validation failure (400) |
| `GET /api/diagnostics/test-handled-exception` | Controller-caught exception |
| `GET /api/diagnostics/test-unhandled-exception` | Unhandled exception → GlobalExceptionHandler |
| `GET /api/diagnostics/test-error` | Manual 500 response with TraceId |
| `GET /api/diagnostics/health` | Simple health check |

---

### 3. Modified Files

#### 3.1 `Program.cs` — Startup Changes

**Serilog configuration (before → after):**

```csharp
// BEFORE: Basic Serilog, no trace correlation
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .CreateLogger();

// AFTER: Serilog with OTLP sink + TraceId/SpanId enrichment
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)   // reads appsettings.json Serilog section
    .Enrich.FromLogContext()
    .Enrich.WithProperty("service.name", serviceName)
    .Enrich.WithProperty("service.version", serviceVersion)
    .WriteTo.Logger(lc => lc                          // filtered sub-logger
        .Filter.ByExcluding(/* health checks, token endpoints */)
        .WriteTo.OpenTelemetry(options => {
            options.Endpoint = "http://localhost:4318/v1/logs";
            options.Protocol  = OtlpProtocol.HttpProtobuf;
            options.IncludedData = SpanIdField | TraceIdField; // key: attaches trace context
        })
    )
    .CreateLogger();
```

**Services registration (new additions):**

```csharp
// 1. OpenTelemetry SDK (traces + metrics)
services.AddOpenTelemetryObservability(builder.Configuration, builder.Environment);

// 2. Activity tracking adds SpanId/TraceId to every ILogger log entry
builder.Logging.Configure(options =>
{
    options.ActivityTrackingOptions = ActivityTrackingOptions.SpanId
                                    | ActivityTrackingOptions.TraceId
                                    | ActivityTrackingOptions.ParentId;
});

// 3. ProblemDetails — automatically includes traceId in error responses
services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        var traceId = Activity.Current?.TraceId.ToString();
        context.ProblemDetails.Extensions.TryAdd("traceId", traceId);
        context.ProblemDetails.Extensions.TryAdd("instance",
            $"{context.HttpContext.Request.Method} {context.HttpContext.Request.Path}");
    };
});

// 4. Modern exception handler (replaces ErrorHandlerMiddleware)
services.AddExceptionHandler<GlobalExceptionHandler>();

// 5. Validation error factory — adds TraceId + marks as validation error for metrics
services.Configure<ApiBehaviorOptions>(options =>
{
    options.InvalidModelStateResponseFactory = context =>
    {
        var traceId = Activity.Current?.TraceId.ToString();
        // ... build ValidationProblemDetails with traceId
        context.HttpContext.Items["IsValidationError"] = true; // for RedMetrics
        return new BadRequestObjectResult(problemDetails);
    };
});
```

**Middleware pipeline (before → after):**

```csharp
// BEFORE
app.UseMiddleware<ErrorHandlerMiddleware>();          // custom, swallowed exceptions
app.UseMiddleware<RequestResponseLoggingMiddleware>();
app.UseRouting();
app.MapControllers();

// AFTER — order is critical (see Execution Flow section)
app.UseExceptionHandler();                               // .NET 8 IExceptionHandler
app.UseStatusCodePages();                                // handles 400-599 status pages
app.UseMiddleware<RequestResponseLoggingMiddleware>();   // request/response DB logging
app.UseSerilogRequestLogging();                          // HTTP summary logs to Loki
app.UseRouting();                                        // ⚠️ MUST be before UseRedMetrics
app.UseRedMetrics();                                     // RED metrics collection
app.UseCors();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();
```

---

#### 3.2 `Middleware/RequestResponseLoggingMiddleware.cs` — Exception Flow Fix

**The critical bug that was fixed:**

```csharp
// BEFORE — Catching exceptions here prevented GlobalExceptionHandler from running
try
{
    await _next(context);   // exception thrown inside
}
catch (Exception ex)
{
    // Exception handled here → GlobalExceptionHandler NEVER sees it
    // Response stream is wrong → client gets garbled response
    logger.WriteErrorLog(ex, "...");
}

// AFTER — Exception is allowed to propagate correctly
try
{
    await _next(context);           // exceptions bubble up naturally
    await LogResponse(context, webAPILogModel);
}
finally
{
    // Restore original stream FIRST so GlobalExceptionHandler can write to it
    context.Response.Body = originalBodyStream;

    // Copy buffered response only if one was written (happy path)
    if (responseBody.Length > 0)
    {
        responseBody.Seek(0, SeekOrigin.Begin);
        await responseBody.CopyToAsync(originalBodyStream);
    }
}
```

**Path exclusion — centralized:**
```csharp
// BEFORE: Hardcoded or always-on
_isLoggingEnabled = true;

// AFTER: Driven by OpenTelemetrySettings (same list used by RedMetricsMiddleware)
var requestPath = context.Request.Path.Value ?? "";
_isLoggingEnabled = !_otelSettings.IsPathExcluded(requestPath);
```

---

#### 3.3 `Models/Utility/ResponseStatusCode.cs` — HTTP Compliance Fix

```csharp
// BEFORE — INVALID HTTP status codes (RFC 7231 allows 100-599 only)
private const int failed        = 600;  // ❌ Invalid
private const int headerMissing = 601;  // ❌ Invalid

// AFTER — Standard HTTP status codes
private const int failed        = 500;  // ✅ Internal Server Error
private const int headerMissing = 400;  // ✅ Bad Request
```

**Added `TraceId` to `RequestResult`:**
```csharp
public class RequestResult
{
    public RequestState State   { get; set; }
    public string       Msg     { get; set; }
    public object       Data    { get; set; }
    public object       Results { get; set; }
    public string       UserType{ get; set; }

    // NEW — clients can reference this TraceId when reporting issues
    public string TraceId { get; set; }
}
```

---

#### 3.4 `appsettings.json` — Configuration Changes

**Serilog enrichers and console template:**
```json
{
  "Serilog": {
    "Using": [
      "Serilog.Sinks.OpenTelemetry",
      "Serilog.Enrichers.Span"
    ],
    "Enrich": [
      "FromLogContext",
      "WithMachineName",
      "WithProcessId",
      "WithThreadId",
      "WithSpan"
    ],
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} | TraceId: {TraceId} SpanId: {SpanId} | {Properties:j}{NewLine}{Exception}"
        }
      }
    ]
  }
}
```

**New `OpenTelemetry` configuration block:**
```json
{
  "OpenTelemetry": {
    "Enabled": true,
    "ServiceName": "CRisMacWebAPI",
    "ServiceVersion": "1.0.0",
    "OtlpEndpoint": "http://localhost:4317",
    "ExcludedPaths": [
      "/health",
      "/swagger",
      "/api/generatetoken",
      "/api/getcurrenttimekey",
      "/api/UserLoginDetails/CrisMAc/AppLogsUpdate"
    ]
  }
}
```

---

## Execution Flow

### Application Startup Sequence

```
Program.cs
│
├─ 1. Serilog Bootstrap Logger
│     └─ Reads OpenTelemetry config section
│     └─ Attaches OTLP sink (→ Loki)
│     └─ Attaches SpanId/TraceId enrichers
│
├─ 2. services.AddOpenTelemetryObservability()
│     └─ OpenTelemetryExtensions.cs
│         ├─ Bind OpenTelemetrySettings from config
│         ├─ Build Resource (service.name, version, env, host)
│         ├─ Tracing pipeline
│         │   ├─ AspNetCore auto-instrumentation
│         │   ├─ HttpClient auto-instrumentation
│         │   ├─ SqlClient auto-instrumentation
│         │   └─ OTLP exporter → Tempo
│         └─ Metrics pipeline
│             ├─ AspNetCore auto-instrumentation
│             ├─ Runtime auto-instrumentation
│             ├─ ApplicationTelemetry.Meter (custom RED metrics)
│             └─ OTLP exporter → Mimir/Prometheus
│
├─ 3. Activity tracking enabled on ILogger
│     └─ Adds TraceId/SpanId/ParentId to all ILogger log entries
│
├─ 4. ProblemDetails configured
│     └─ All error responses automatically include "traceId"
│
├─ 5. services.AddExceptionHandler<GlobalExceptionHandler>()
│
├─ 6. ApiBehaviorOptions — custom validation response factory
│     └─ Adds TraceId to 400 responses
│     └─ Sets HttpContext.Items["IsValidationError"] = true
│
└─ 7. Middleware pipeline registered (in order):
      UseExceptionHandler()
      UseStatusCodePages()
      UseMiddleware<RequestResponseLoggingMiddleware>()
      UseSerilogRequestLogging()
      UseRouting()            ← ⚠️ CRITICAL ORDER
      UseRedMetrics()
      UseCors()
      UseAuthentication()
      UseAuthorization()
      MapControllers()
```

---

### Request Execution Flow — Happy Path (200 OK)

```
Incoming HTTP Request
│
├─ [Auto] OpenTelemetry AspNetCore Instrumentation
│   └─ Creates Activity (Trace Span)
│   └─ Assigns TraceId + SpanId
│   └─ Reads W3C traceparent header (distributed tracing)
│
├─ 1. UseExceptionHandler   → passes through (no exception yet)
│
├─ 2. UseStatusCodePages    → passes through
│
├─ 3. RequestResponseLoggingMiddleware
│   ├─ IsPathExcluded() check → continue if not excluded
│   ├─ LogRequest() — captures method, path, headers, body, IP
│   ├─ Buffers Response.Body (MemoryStream swap)
│   ├─ await _next(context)  ← calls next middleware
│   ├─ LogResponse() — captures status code, writes to WebAPILog DB table
│   └─ finally: restores original stream, copies buffered response
│
├─ 4. UseSerilogRequestLogging
│   └─ Writes one-line HTTP summary log with TraceId/SpanId
│   └─ Sent to Console + File + Loki (via OTLP sink)
│
├─ 5. UseRouting
│   └─ Matches request to controller endpoint
│   └─ Sets HttpContext endpoint metadata (route template available)
│
├─ 6. RedMetricsMiddleware
│   ├─ ShouldSkipPath() check
│   ├─ Stopwatch.Start()
│   ├─ GetEndpointTemplate() → extracts "/api/users/{id}" (not "/api/users/123")
│   ├─ ActiveRequests.Add(+1)
│   ├─ Registers Response.OnCompleted(RecordMetrics)   ← fires after response sent
│   └─ await _next(context)  ← calls controller
│
├─ 7. Controller Action
│   ├─ Model binding
│   ├─ Model validation  → if fails, goes to InvalidModelStateResponseFactory
│   ├─ Authorization
│   ├─ Business logic
│   ├─ SQL calls         → [Auto] SqlClient span created (child of HTTP span)
│   ├─ Outbound HTTP     → [Auto] HttpClient span created + traceparent header injected
│   └─ return Ok(new RequestResult { ..., TraceId = Activity.Current?.TraceId })
│
├─ 8. Response serialized → 200 OK written to buffered stream
│
└─ 9. Response.OnCompleted fires → RecordMetrics()
    ├─ HttpRequestsTotal.Add(1, {method, route, 200})          ← RATE
    ├─ HttpRequestDuration.Record(elapsed, {method, route, 200}) ← DURATION
    ├─ status < 400 → no error counter
    └─ ActiveRequests.Add(-1)

Response sent to client  ✅
```

---

### Request Execution Flow — Error Path (Unhandled Exception)

```
Incoming HTTP Request
│
├─ [Steps 1–6 identical to Happy Path]
│
├─ 7. Controller Action
│   └─ EXCEPTION THROWN ❌
│
├─ Exception bubbles up through middleware stack:
│   ├─ RedMetricsMiddleware.InvokeAsync
│   │   └─ await _next() throws
│   │   └─ NOT caught → propagates
│   │   └─ Response.OnCompleted still registered ✅
│   │
│   └─ RequestResponseLoggingMiddleware.Invoke
│       └─ await _next() throws
│       └─ NOT caught → propagates ✅  (fixed in this PR)
│       └─ finally block: restores original response stream ✅
│
├─ UseExceptionHandler catches exception
│   └─ Calls GlobalExceptionHandler.TryHandleAsync()
│       ├─ Gets TraceId from Activity.Current
│       ├─ _logger.LogError(exception, "Unhandled exception. TraceId: {TraceId}", ...)
│       │   └─ Full stack trace written to Console + File + Loki
│       ├─ ApplicationTelemetry.RecordException(exception)
│       │   ├─ activity.SetTag("error", true)
│       │   ├─ activity.SetTag("exception.type", ex.GetType().Name)
│       │   ├─ activity.AddEvent(new ActivityEvent("exception", ...))
│       │   └─ activity.SetStatus(ActivityStatusCode.Error)
│       ├─ Determines HTTP status: AppException→400, KeyNotFound→404, else 500
│       └─ Writes ProblemDetails JSON to response:
│           {
│             "type":    "https://tools.ietf.org/html/rfc7231#section-6.6.1",
│             "title":   "Internal Server Error",
│             "status":  500,
│             "detail":  "An unexpected error occurred. Please contact support with TraceId.",
│             "instance":"POST /api/users",
│             "traceId": "a1b2c3d4e5f6789012345678901234ab"
│           }
│
└─ Response.OnCompleted fires → RecordMetrics()
    ├─ HttpRequestsTotal.Add(1, {method, route, 500})           ← RATE (all requests counted)
    ├─ HttpRequestDuration.Record(elapsed, {method, route, 500})← DURATION
    ├─ status >= 400 → HttpRequestErrors.Add(1, {error.type="server_error"}) ← ERRORS
    └─ ActiveRequests.Add(-1)

Response sent to client  ✅  (client sees TraceId, developer searches logs by TraceId)
```

---

### Validation Error Flow (400 Bad Request)

```
POST /api/users  { missing required fields }
│
├─ [Steps 1–5: UseExceptionHandler → UseRouting]
│
├─ 6. RedMetricsMiddleware
│   └─ Registers OnCompleted callback, calls _next()
│
├─ 7. ASP.NET Core Model Binding
│   └─ Validation fails
│   └─ InvalidModelStateResponseFactory called (Program.cs)
│       ├─ TraceId extracted from Activity.Current
│       ├─ ValidationProblemDetails built with field-level errors
│       ├─ traceId added to Extensions
│       ├─ HttpContext.Items["IsValidationError"] = true  ← signal for metrics
│       └─ Returns 400 BadRequestObjectResult
│
└─ Response.OnCompleted → RecordMetrics()
    ├─ HttpRequestsTotal.Add(1, {method, route, 400})
    ├─ HttpRequestDuration.Record(elapsed, ...)
    ├─ HttpRequestErrors.Add(1, {error.type="client_error"})
    └─ Items["IsValidationError"]==true →
       ValidationErrors.Add(1, ...)   ← tracked separately in Grafana
```

---

### Critical Middleware Ordering — Why It Matters

```
app.UseExceptionHandler()               ← MUST be #1: outer safety net
app.UseStatusCodePages()                ← #2: handles 400–599
app.UseMiddleware<RequestResponseLoggingMiddleware>() ← #3: wraps all below
app.UseSerilogRequestLogging()          ← #4: logs HTTP summary
app.UseRouting()                        ← #5: ⚠️ REQUIRED before RedMetrics
app.UseRedMetrics()                     ← #6: needs endpoint from #5
app.UseCors()
app.UseAuthentication()
app.UseAuthorization()
app.MapControllers()                    ← MUST be last
```

| Wrong Order | Consequence |
|-------------|-------------|
| `UseExceptionHandler` after other middleware | Exceptions escape, app crashes |
| `UseRouting` after `UseRedMetrics` | `GetEndpointTemplate()` returns raw path → high cardinality metrics |
| `UseAuthentication` before `UseRouting` | Auth policies for specific routes won't resolve correctly |
| Catching exceptions in `RequestResponseLoggingMiddleware` | `GlobalExceptionHandler` never runs; response stream corrupted |

---

## Configuration Guide

### `appsettings.json` — Full OpenTelemetry Section

```json
{
  "OpenTelemetry": {
    "Enabled": true,
    "ServiceName": "CRisMacWebAPI",
    "ServiceVersion": "1.0.0",
    "OtlpEndpoint": "http://localhost:4317",
    "ExcludedPaths": [
      "/health",
      "/swagger",
      "/api/generatetoken",
      "/api/getcurrenttimekey",
      "/api/UserLoginDetails/CrisMAc/AppLogsUpdate"
    ]
  }
}
```

### Environment-Specific Overrides

**`appsettings.Development.json`:**
```json
{
  "OpenTelemetry": {
    "Enabled": true,
    "OtlpEndpoint": "http://localhost:4317"
  },
  "Serilog": {
    "MinimumLevel": { "Default": "Debug" }
  }
}
```

**`appsettings.Production.json`:**
```json
{
  "OpenTelemetry": {
    "Enabled": true,
    "OtlpEndpoint": "http://otel-collector.internal:4317"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": { "Microsoft": "Warning", "System": "Warning" }
    }
  }
}
```

### Disabling OpenTelemetry Completely

Set `"Enabled": false` — the `AddOpenTelemetryObservability()` method checks this flag and skips SDK registration entirely. Logs continue to go to Console and File; nothing is sent to OTLP.

---

## Testing & Validation

### DiagnosticsController Test Endpoints

| Endpoint | Method | What to Verify |
|----------|--------|----------------|
| `GET /api/diagnostics/test-otel` | GET | TraceId in response + logs in Loki + span in Tempo |
| `POST /api/diagnostics/test-validation` | POST | 400 response with `errors` + `traceId`; `api_validation_errors_total` incremented |
| `GET /api/diagnostics/test-handled-exception` | GET | 500 response from controller catch; full exception in server logs |
| `GET /api/diagnostics/test-unhandled-exception` | GET | ProblemDetails 500 from GlobalExceptionHandler; span marked as error in Tempo |
| `GET /api/diagnostics/test-error` | GET | Custom error response with TraceId |
| `GET /api/diagnostics/health` | GET | Simple 200 (should NOT appear in traces — excluded path) |

### Step-by-Step Correlation Test

**1. Call test endpoint:**
```bash
curl http://localhost:5000/api/diagnostics/test-otel
```

**2. Note the TraceId from response:**
```json
{ "traceId": "a1b2c3d4e5f6789012345678901234ab" }
```

**3. Check console log — TraceId is visible:**
```
[10:15:30 INF] OPENTELEMETRY TEST - Information Level | TraceId: a1b2c3d4... SpanId: 0123456789ab
```

**4. Search Loki in Grafana:**
```logql
{service_name="CRisMacWebAPI"} | json | TraceId="a1b2c3d4e5f6789012345678901234ab"
```

**5. View trace in Tempo:**
- Grafana → Explore → Tempo data source → paste `a1b2c3d4e5f6789012345678901234ab`

---

## Best Practices

### Adding TraceId to Controller Responses

```csharp
using System.Diagnostics;

[HttpGet("{id}")]
public IActionResult GetUser(int id)
{
    var traceId = Activity.Current?.TraceId.ToString();

    try
    {
        var user = _userService.GetUser(id);

        return Ok(new RequestResult
        {
            State   = RequestState.Success,
            Data    = user,
            TraceId = traceId   // ✅ client can report this on errors
        });
    }
    catch (AppException ex)
    {
        _logger.LogWarning(ex, "Validation failed for user {UserId}. TraceId: {TraceId}", id, traceId);
        return BadRequest(new RequestResult
        {
            State   = RequestState.Failed,
            Msg     = ex.Message,
            TraceId = traceId
        });
    }
    // All other exceptions → let them propagate to GlobalExceptionHandler ✅
}
```

### Adding Custom Business Metrics

**Step 1 — Define in `ApplicationTelemetry.cs`:**
```csharp
public static readonly Counter<long> ReturnSubmissions = Meter.CreateCounter<long>(
    name: "return_submissions_total",
    description: "Total return submissions by type");
```

**Step 2 — Record in service:**
```csharp
ApplicationTelemetry.ReturnSubmissions.Add(1,
    new KeyValuePair<string, object?>("return_type", "quarterly"),
    new KeyValuePair<string, object?>("status", "success"));
```

### Adding Custom Trace Spans

```csharp
using var activity = ApplicationTelemetry.ActivitySource.StartActivity("ProcessReturn");
activity?.SetTag("return.id", returnId);
activity?.SetTag("return.type", "quarterly");

try
{
    await ProcessStep1();  // each step can be its own child span
    await ProcessStep2();
    activity?.SetTag("return.status", "success");
}
catch (Exception ex)
{
    activity?.SetTag("error", true);
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    throw;
}
```

### Structured Logging Rules

```csharp
// ✅ DO — structured with named parameters
_logger.LogInformation("Return {ReturnId} submitted by user {UserId}", returnId, userId);
_logger.LogError(ex, "Failed to fetch return {ReturnId}", returnId);   // pass exception as first arg

// ❌ DON'T — string interpolation breaks structured logging
_logger.LogInformation($"Return {returnId} submitted");

// ❌ DON'T — log sensitive data
_logger.LogInformation("User password: {Password}", password);

// ❌ DON'T — swallow the exception object
catch (Exception ex) {
    _logger.LogError("Something failed");   // exception not passed → no stack trace
}
```

---

## Troubleshooting

### No Data in Grafana

1. Check `"OpenTelemetry": { "Enabled": true }` in appsettings
2. Verify OTLP Collector is running: `docker ps | grep otel-collector`
3. Check app logs for OTLP export errors (enable `"Serilog.Sinks.OpenTelemetry": "Debug"`)
4. Hit `/api/diagnostics/test-otel` and search for returned TraceId in Loki

### Metrics Show Raw Paths (`/api/users/123`) Instead of Templates (`/api/users/{id}`)

**Cause:** `UseRouting()` registered after `UseRedMetrics()`

**Fix:**
```csharp
app.UseRouting();    // ← must come first
app.UseRedMetrics();
```

### TraceId in Logs Doesn't Match Tempo Trace

**Cause:** `WithSpan` enricher or Activity tracking not enabled.

**Fix — appsettings.json:**
```json
{ "Serilog": { "Using": ["Serilog.Enrichers.Span"], "Enrich": ["WithSpan"] } }
```

**Fix — Program.cs:**
```csharp
builder.Logging.Configure(options => {
    options.ActivityTrackingOptions = ActivityTrackingOptions.SpanId | ActivityTrackingOptions.TraceId;
});
```

### GlobalExceptionHandler Not Running

**Cause:** Exception caught inside `RequestResponseLoggingMiddleware` (old bug — now fixed).

**Verify:** `RequestResponseLoggingMiddleware.Invoke` must NOT have a `catch` block around `await _next(context)`. Only a `finally` block for stream cleanup is correct.

### High-Cardinality Metrics Explosion

**Cause:** Excluded paths list too narrow, or `UseRouting` ordering wrong.

**Fix:** Add frequently-called, non-business endpoints to `ExcludedPaths`:
```json
{ "OpenTelemetry": { "ExcludedPaths": ["/health", "/metrics", "/api/generatetoken"] } }
```

---

## FAQ

**Q: Can I disable OTEL without code changes?**
A: Yes — set `"OpenTelemetry": { "Enabled": false }`. No SDK loads, no overhead.

**Q: What is TraceId vs SpanId?**
A: `TraceId` is a 32-char hex string unique to the entire request flow (even across services). `SpanId` is a 16-char hex string unique to a single operation within that trace. When debugging, search by TraceId — it will surface all spans and logs for that request.

**Q: Should every response include TraceId?**
A: At minimum, include it in all error responses. For success responses, it is optional but useful for client-side debugging. The `RequestResult` model now has a `TraceId` property ready to use.

**Q: What happens if OTLP Collector is unreachable?**
A: Telemetry is buffered in memory and retried. After the queue fills or timeout expires, data is dropped silently. The application continues working normally — no user-facing impact.

**Q: How do I trace a slow SQL query?**
A: No code needed. `SqlClient` auto-instrumentation creates a child span for every query. Open the parent HTTP span in Tempo and expand child spans to see individual query durations.

**Q: Are the DiagnosticsController endpoints safe in production?**
A: They are not authenticated and deliberately throw exceptions and log at all levels. **Secure or remove them before go-live** using `[Authorize]` or by conditionally registering the controller only in Development environment.

---

## Before vs. After Summary

| Aspect | Before | After |
|--------|--------|-------|
| Exception handling | `ErrorHandlerMiddleware` — caught and swallowed exceptions | `GlobalExceptionHandler` — proper propagation, ProblemDetails, TraceId |
| Logging | Basic Serilog to Console + File | Serilog + OTLP sink → Loki; every log has TraceId/SpanId |
| Metrics | None | Automatic RED metrics for all endpoints + custom counters |
| Tracing | None | Full distributed tracing: HTTP, SQL, outbound calls |
| HTTP status codes | 600 (failed), 601 (headerMissing) — invalid | 500 (Internal Error), 400 (Bad Request) — standard |
| Error responses | Inconsistent formats | Standardised `ProblemDetails` with `traceId` |
| Client debugging | "Something went wrong" | Client gets TraceId → developer finds exact log + trace in seconds |
| Observability | None | Full Grafana LGTM stack with pre-built RED metrics dashboard |

---

## Related Documentation

- [LGTM Stack Setup Guide](LGTM-STACK-SETUP.md)
- [Grafana Dashboard Guide](GRAFANA-DASHBOARD-GUIDE.md)
- [Grafana Dashboard JSON](grafana-dashboard-ensure-red-metrics.json)
- [OpenTelemetry .NET Official Docs](https://opentelemetry.io/docs/languages/dotnet/)

---

*Document Version: 1.0 | Branch: feature/add-otel-grafana-observability*
