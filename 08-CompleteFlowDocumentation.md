# Application Telemetry & OpenObservability - Complete Flow Documentation

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Phase 1: Application Startup](#phase-1-application-startup-before-any-request)
- [Phase 2: Request Processing](#phase-2-request-processing-per-http-request)
- [Phase 3: Controller/Business Logic Execution](#phase-3-controllerbusiness-logic-execution)
- [Phase 4: Response Completion & Error Handling](#phase-4-response-completion--error-handling)
- [Phase 5: Telemetry Export](#phase-5-telemetry-export)
- [Execution Order Summary](#execution-order-summary)
- [Key Components Reference](#key-components-reference)

---

## Overview

This application implements a comprehensive observability stack using:

- **OpenTelemetry** for distributed tracing and metrics
- **Serilog** for structured logging
- **RED Metrics** (Rate, Errors, Duration) for monitoring HTTP requests
- **Grafana Stack** for visualization (Tempo for traces, Prometheus for metrics, Loki for logs)

All telemetry data is automatically correlated using TraceId and SpanId for seamless debugging and monitoring.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Your Application                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│  │ OpenTelemetry│  │   Serilog    │  │ RED Metrics  │         │
│  │   Tracing    │  │   Logging    │  │  Middleware  │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘         │
│         │                  │                  │                  │
│         └──────────────────┴──────────────────┘                 │
│                            │                                     │
└────────────────────────────┼─────────────────────────────────────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ OTLP Exporter  │
                    │    (gRPC)      │
                    └────────┬───────┘
                             │
                             ▼
                    ┌────────────────┐
                    │ Grafana Stack  │
                    │ ├─ Tempo       │ ← Traces
                    │ ├─ Prometheus  │ ← Metrics
                    │ └─ Loki        │ ← Logs
                    └────────────────┘
```

---

## Phase 1: Application Startup (Before any request)

### 1. Configuration Loading
**Location**: [`Program.cs:29-32`](../backend/src/Crismac.Ews.WebApi/Program.cs#L29-L32)

OpenTelemetry settings loaded from configuration:
- `ServiceName` (e.g., "Crismac.Ews.Backend")
- `ServiceVersion`
- `OtlpEndpoint` (Grafana/OTLP collector endpoint)

```csharp
var serviceName = builder.Configuration["OpenTelemetry:ServiceName"] ?? "Crismac.Ews.Backend";
var serviceVersion = builder.Configuration["OpenTelemetry:ServiceVersion"] ?? "1.0.0";
var otlpEndpoint = builder.Configuration["OpenTelemetry:OtlpEndpoint"] ?? "http://localhost:4317";
```

### 2. Serilog Logger Setup ⚡ **EXECUTES FIRST**
**Location**: [`Program.cs:34-48`](../backend/src/Crismac.Ews.WebApi/Program.cs#L34-L48)

**This happens BEFORE any other telemetry initialization**

```csharp
Log.Logger = new LoggerConfiguration()
    .ReadFrom.Configuration(builder.Configuration)
    .Enrich.FromLogContext()  // Correlates logs with Activity/OpenTelemetry
    .Enrich.WithProperty("service.name", serviceName)
    .Enrich.WithProperty("service.version", serviceVersion)
    .Enrich.WithProperty("deployment.environment", environment)
    .WriteTo.Console(restrictedToMinimumLevel: Debug/Info)
    .CreateLogger();
```

**What happens**: Global logger configured with service metadata that will be attached to every log entry.

### 3. OpenTelemetry Resource Builder
**Location**: [`Program.cs:72-82`](../backend/src/Crismac.Ews.WebApi/Program.cs#L72-L82)

Creates shared metadata for ALL telemetry:

```csharp
var resourceBuilder = ResourceBuilder
    .CreateDefault()
    .AddService(serviceName, serviceVersion: serviceVersion)
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = environment,
        ["host.name"] = Environment.MachineName,
        ["process.pid"] = Environment.ProcessId
    });
```

**What happens**: Resource attributes attached to every trace and metric exported from this application.

### 4. Tracing Configuration
**Location**: [`Program.cs:84-172`](../backend/src/Crismac.Ews.WebApi/Program.cs#L84-L172)

```
AddOpenTelemetry().WithTracing() configures:

├── ASP.NET Core Instrumentation (auto-traces HTTP requests)
│   ├── RecordException: true
│   ├── EnrichWithHttpRequest (adds path, method, query)
│   ├── EnrichWithHttpResponse (adds status code)
│   └── Filter: excludes /health endpoint
│
├── HttpClient Instrumentation (auto-traces outgoing HTTP calls)
│
├── SqlClient Instrumentation (auto-traces SQL queries)
│   └── Filter: ONLY stored procedures (CommandType.StoredProcedure)
│
├── Custom ActivitySource: "Crismac.Ews.Backend.Observability"
│   └── Used for manual instrumentation
│
└── OTLP Exporter (sends to Grafana at otlpEndpoint via gRPC)
```

**Key Features**:
- **Automatic HTTP tracing**: Every HTTP request creates a span
- **SQL query tracing**: Only stored procedures are traced (performance optimization)
- **Exception recording**: Exceptions are automatically added to traces
- **Custom enrichment**: Request/response details added to spans

### 5. Metrics Configuration
**Location**: [`Program.cs:173-198`](../backend/src/Crismac.Ews.WebApi/Program.cs#L173-L198)

```
WithMetrics() configures:

├── ASP.NET Core Metrics (auto HTTP request metrics)
├── HttpClient Metrics (outgoing HTTP call metrics)
├── Runtime Metrics (.NET runtime stats: GC, threads, exceptions)
├── Process Metrics (CPU, memory, handles)
│
├── Custom Meters:
│   ├── "Crismac.Ews.Backend.Observability" (ApplicationTelemetry)
│   │   ├── api.requests.count (RED - Rate)
│   │   ├── api.requests.duration (RED - Duration)
│   │   ├── api_errors (RED - Errors)
│   │   ├── api_validation_errors
│   │   └── Business metrics (database, external API, etc.)
│   │
│   └── "Crismac.Ews.Login" (LoginTelemetry)
│       ├── ews.login.attempts.total
│       ├── ews.login.duration.milliseconds
│       ├── ews.token.operations.total
│       ├── ews.login.failures.by_user
│       └── ews.account.lockouts.total
│
└── OTLP Exporter (sends to Grafana)
```

### 6. Activity Tracking Configuration
**Location**: [`Program.cs:201-209`](../backend/src/Crismac.Ews.WebApi/Program.cs#L201-L209)

Configures Serilog to include OpenTelemetry trace context in logs:

```csharp
Logging.Configure(loggerConfiguration =>
    loggerConfiguration.Enrich.WithSpanId()
        .Enrich.WithTraceId()
        .Enrich.WithParentId()
        .Enrich.WithBaggage()
        .Enrich.WithTags()
);
```

**What happens**: Every log entry automatically gets trace correlation IDs (TraceId, SpanId) for seamless correlation with distributed traces.

### 7. ApplicationTelemetry Initialization
**Location**: [`ApplicationTelemetry.cs:14-18`](../backend/src/Crismac.Ews.WebApi/Observability/ApplicationTelemetry.cs#L14-L18)

Static initialization happens at first access:

```csharp
private static readonly ActivitySource ActivitySource = new(MeterName, "1.0.0");
private static readonly Meter Meter = new(MeterName, "1.0.0");

// All Counters and Histograms registered
public static readonly Counter<long> RequestCounter = Meter.CreateCounter<long>("api.requests.count");
public static readonly Histogram<double> RequestDuration = Meter.CreateHistogram<double>("api.requests.duration");
public static readonly Counter<long> ErrorCounter = Meter.CreateCounter<long>("api_errors");
```

### 8. LoginTelemetry Initialization
**Location**: [`LoginTelemetry.cs:11`](../backend/src/Crismac.Ews.WebApi/Observability/LoginTelemetry.cs#L11)

Separate Meter for authentication-specific metrics:

```csharp
private static readonly Meter Meter = new(MeterName, "1.0.0");

// Login-specific metrics
public static readonly Counter<long> LoginAttemptsTotal = Meter.CreateCounter<long>("ews.login.attempts.total");
public static readonly Histogram<double> LoginDurationMs = Meter.CreateHistogram<double>("ews.login.duration.milliseconds");
```

---

## Phase 2: Request Processing (Per HTTP Request)

### Request Pipeline Order

```
HTTP Request Arrives
    ↓
[1] ASP.NET Core HTTP Pipeline Starts
    │   └─ OpenTelemetry AspNetCoreInstrumentation creates Activity (span)
    │
[2] Serilog Request Logging Begins
    │   └─ Enriches logs with Activity.TraceId and Activity.SpanId
    │
[3] Middleware Chain Execution
    │
    ├─ [A] EncryptionMiddleware (if enabled)
    │      Location: Program.cs:495-498
    │
    ├─ [B] ApiKeyMiddleware
    │      Location: Program.cs:500
    │
    ├─ [C] UseRouting() ⭐ CRITICAL
    │      Location: Program.cs:501
    │      └─ Determines endpoint template (e.g., "api/users/{id}")
    │
    ├─ [D] RedMetricsMiddleware ⭐ MAIN METRICS COLLECTOR
    │      Location: Program.cs:504
    │      └─ See detailed flow below
    │
    ├─ [E] Authentication & Authorization
    │      Location: Program.cs:506-508
    │
    └─ [F] Controller Execution
```

### RedMetricsMiddleware Execution Flow ⭐

**Location**: [`RedMetricsMiddleware.cs:22-38`](../backend/src/Crismac.Ews.WebApi/Middlewares/RedMetricsMiddleware.cs#L22-L38)

This is the **CORE** of the RED metrics implementation.

```csharp
public async Task InvokeAsync(HttpContext context)
{
    // ═══════════════════════════════════════════════════════════
    // STEP 1: Start timing
    // ═══════════════════════════════════════════════════════════
    var stopwatch = Stopwatch.StartNew();
    var endpoint = GetEndpointTemplate(context);  // e.g., "api/users/{id}"
    var method = context.Request.Method;          // e.g., "GET"

    // ═══════════════════════════════════════════════════════════
    // STEP 2: Register OnCompleted callback ⏳
    // This callback will execute AFTER everything else completes
    // ═══════════════════════════════════════════════════════════
    context.Response.OnCompleted(() =>
    {
        stopwatch.Stop();
        RecordMetrics(context, endpoint, method, stopwatch.ElapsedMilliseconds);
        return Task.CompletedTask;
    });

    // ═══════════════════════════════════════════════════════════
    // STEP 3: Let the rest of the pipeline execute
    // ═══════════════════════════════════════════════════════════
    await _next(context);

    // Metrics will be recorded automatically when response completes
}
```

**Key Points**:
- **Non-blocking**: Metrics recording doesn't delay the response
- **Guaranteed execution**: OnCompleted fires even if exceptions occur
- **Accurate timing**: Measures the entire request processing time

---

## Phase 3: Controller/Business Logic Execution

### Automatic Instrumentation

#### 1. SQL Database Calls

**Configuration**: [`Program.cs:139-147`](../backend/src/Crismac.Ews.WebApi/Program.cs#L139-L147)

- OpenTelemetry **SqlClientInstrumentation** creates child spans
- **Only traces stored procedures** (filtered for performance)
- Automatic tags: `db.command_type`, `db.command_timeout`

```csharp
// Example trace hierarchy:
HTTP POST /api/orders
├─ Span: "HTTP POST /api/orders" (parent)
│   └─ Span: "StoredProcedure CreateOrder" (child)
```

#### 2. Outgoing HTTP Calls

- **HttpClientInstrumentation** creates child spans
- Automatic tags: `http.request.uri`, `http.response.status_code`

### Manual Instrumentation (Optional)

#### Custom Spans/Traces

**Location**: [`ApplicationTelemetry.cs`](../backend/src/Crismac.Ews.WebApi/Observability/ApplicationTelemetry.cs)

```csharp
using var activity = ApplicationTelemetry.StartActivity("BusinessOperation");
activity?.SetTag("order.id", orderId);
activity?.SetTag("customer.id", customerId);

try
{
    // ... business logic ...
    activity?.SetStatus(ActivityStatusCode.Ok);
}
catch (Exception ex)
{
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    throw;
}
```

#### Custom Metrics

```csharp
// Record business operation
ApplicationTelemetry.RecordBusinessOperation(
    operation: "order_creation",
    status: "success",
    durationMs: 150.5
);

// Record database operation
ApplicationTelemetry.RecordDatabaseOperation(
    operation: "SELECT",
    table: "Orders",
    success: true,
    durationMs: 45.2
);

// Record external API call
ApplicationTelemetry.RecordExternalApiCall(
    serviceName: "payment_gateway",
    method: "POST",
    statusCode: 200,
    durationMs: 320.1
);
```

#### Login-Specific Telemetry

**Location**: [`LoginTelemetry.cs`](../backend/src/Crismac.Ews.WebApi/Observability/LoginTelemetry.cs)

```csharp
// In LoginController
var loginActivity = LoginTelemetry.StartLoginTrace(username, ipAddress);

try
{
    // ... authentication logic ...

    LoginTelemetry.RecordSuccessfulLogin(
        username: username,
        method: "jwt",
        durationMs: stopwatch.ElapsedMilliseconds
    );

    LoginTelemetry.SetLoginResult(loginActivity, success: true);
}
catch (Exception ex)
{
    LoginTelemetry.RecordFailedLogin(
        username: username,
        reason: "invalid_credentials",
        ipAddress: ipAddress
    );

    LoginTelemetry.SetLoginResult(
        loginActivity,
        success: false,
        failureReason: "invalid_credentials"
    );
}
```

---

## Phase 4: Response Completion & Error Handling

### Case A: Successful Response (2xx, 3xx)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Controller returns result (e.g., 200 OK)                 │
│ 2. Response flows back through middleware                   │
│ 3. RedMetrics OnCompleted callback executes ⏰             │
└─────────────────────────────────────────────────────────────┘
```

**RedMetrics RecordMetrics Execution**:
**Location**: [`RedMetricsMiddleware.cs:45-68`](../backend/src/Crismac.Ews.WebApi/Middlewares/RedMetricsMiddleware.cs#L45-L68)

```csharp
private void RecordMetrics(HttpContext context, string endpoint, string method, long durationMs)
{
    var statusCode = context.Response.StatusCode;

    // ═══ RED METRICS ═══

    // R - Rate
    ApplicationTelemetry.RequestCounter.Add(1, new TagList
    {
        { "http.method", method },
        { "http.route", endpoint },
        { "http.status_code", statusCode }
    });

    // D - Duration
    ApplicationTelemetry.RequestDuration.Record(durationMs, new TagList
    {
        { "http.method", method },
        { "http.route", endpoint }
    });

    // E - Errors (only if statusCode >= 400)
    // NOT recorded for successful responses
}
```

**Then**:

```
4. AspNetCoreInstrumentation finalizes Activity
   └─ Sets http.status_code tag
   └─ Closes the span

5. Serilog logs the request
   ├─ Log: "HTTP POST /api/users responded 200 in 45.2 ms"
   ├─ TraceId: 4bf92f3577b34da6a3ce929d0e0e4736
   └─ SpanId: a3ce929d0e0e4736
```

### Case B: Validation Error (400)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Model validation fails (before controller)               │
│ 2. InvalidModelStateResponseFactory executes                │
└─────────────────────────────────────────────────────────────┘
```

**InvalidModelStateResponseFactory Execution**:
**Location**: [`Program.cs:253-289`](../backend/src/Crismac.Ews.WebApi/Program.cs#L253-L289)

```csharp
options.InvalidModelStateResponseFactory = context =>
{
    var activity = Activity.Current;

    // ═══════════════════════════════════════════════════════════
    // STEP 1: Mark as validation error in HttpContext
    // ═══════════════════════════════════════════════════════════
    context.HttpContext.Items["IsValidationError"] = true;  // ⭐ KEY MARKER

    // ═══════════════════════════════════════════════════════════
    // STEP 2: Add error tags to current Activity
    // ═══════════════════════════════════════════════════════════
    activity?.SetTag("error", true);
    activity?.SetTag("error.type", "ValidationError");

    // ═══════════════════════════════════════════════════════════
    // STEP 3: Create ProblemDetails with TraceId and SpanId
    // ═══════════════════════════════════════════════════════════
    var problemDetails = new ValidationProblemDetails(context.ModelState)
    {
        Title = "Validation failed",
        Status = StatusCodes.Status400BadRequest,
        Extensions =
        {
            ["traceId"] = activity?.TraceId.ToString() ?? context.HttpContext.TraceIdentifier,
            ["spanId"] = activity?.SpanId.ToString()
        }
    };

    return new BadRequestObjectResult(problemDetails);
};
```

**Then**:

```
3. RedMetrics OnCompleted callback executes ⏰
```

**Location**: [`RedMetricsMiddleware.cs:69-92`](../backend/src/Crismac.Ews.WebApi/Middlewares/RedMetricsMiddleware.cs#L69-L92)

```csharp
// Status code = 400 (>= 400, so this is an error)
if (statusCode >= 400)
{
    var errorType = statusCode >= 500 ? "server_error" : "client_error";

    // E - Errors (RED Metrics)
    ApplicationTelemetry.ErrorCounter.Add(1, new TagList
    {
        { "http.method", method },
        { "http.route", endpoint },
        { "http.status_code", statusCode },
        { "error.type", errorType }
    });

    // ═══════════════════════════════════════════════════════════
    // Check for validation error marker ⭐
    // ═══════════════════════════════════════════════════════════
    if (context.Items["IsValidationError"] is true)
    {
        ApplicationTelemetry.ValidationErrorCounter.Add(1, new TagList
        {
            { "http.method", method },
            { "http.route", endpoint }
        });
    }
}
```

**Summary**:
- General error counter incremented
- **Validation-specific counter also incremented** (enables separate alerting)
- Activity marked with error tags
- Client receives TraceId for support correlation

### Case C: Unhandled Exception (500)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Exception thrown in controller/service                   │
│ 2. GlobalExceptionHandler.TryHandleAsync executes           │
└─────────────────────────────────────────────────────────────┘
```

**GlobalExceptionHandler Execution**:
**Location**: [`GlobalExceptionHandler.cs:45-88`](../backend/src/Crismac.Ews.WebApi/Middlewares/GlobalExceptionHandler.cs#L45-L88)

```csharp
public async ValueTask<bool> TryHandleAsync(
    HttpContext context,
    Exception exception,
    CancellationToken cancellationToken)
{
    // ═══════════════════════════════════════════════════════════
    // STEP 1: Get TraceId from Activity for correlation
    // ═══════════════════════════════════════════════════════════
    var activity = context.Features.Get<IHttpActivityFeature>()?.Activity;
    var traceId = activity?.TraceId.ToString() ?? context.TraceIdentifier;

    // ═══════════════════════════════════════════════════════════
    // STEP 2: Log full exception SERVER-SIDE (with stack trace)
    // CRITICAL: NEVER expose stack traces to clients!
    // ═══════════════════════════════════════════════════════════
    _logger.LogError(
        exception,
        "Unhandled exception occurred. TraceId: {TraceId}, Path: {Path}, Method: {Method}, User: {User}",
        traceId,
        context.Request.Path,
        context.Request.Method,
        context.User?.Identity?.Name ?? "Anonymous"
    );

    // ═══════════════════════════════════════════════════════════
    // STEP 3: Record exception in telemetry (tracing only)
    // ═══════════════════════════════════════════════════════════
    ApplicationTelemetry.RecordException(exception);

    // This adds to Activity:
    // - activity.SetTag("error", true)
    // - activity.SetTag("exception.type", "SqlException")
    // - activity.SetTag("exception.message", "...")
    // - activity.AddEvent("exception") with stack trace

    // ═══════════════════════════════════════════════════════════
    // STEP 4: Return generic ProblemDetails (NO stack trace)
    // ═══════════════════════════════════════════════════════════
    var problemDetails = CreateProblemDetails(context, exception, traceId);

    // Response status code set to 500
    return await _problemDetailsService.TryWriteAsync(new ProblemDetailsContext
    {
        Exception = exception,
        HttpContext = context,
        ProblemDetails = problemDetails
    });
}
```

**Then**:

```
3. RedMetrics OnCompleted callback executes ⏰
   └─ statusCode = 500
   └─ errorType = "server_error"
   └─ ApplicationTelemetry.ErrorCounter.Add(1)
   └─ NO validation counter (not a validation error)

4. Trace exported with exception details
   └─ Includes full stack trace in trace event
   └─ Visible in Grafana Tempo

5. Logs exported with TraceId for correlation
   └─ Full exception logged server-side
   └─ TraceId enables correlation with trace
```

**Security Note**:
- **Stack traces NEVER exposed to clients** (security vulnerability)
- Clients only receive generic error message + TraceId
- TraceId allows support team to find full details server-side

---

## Phase 5: Telemetry Export

### Continuous Background Export

#### 1. OTLP Exporter

**Configuration**: [`Program.cs:84-198`](../backend/src/Crismac.Ews.WebApi/Program.cs#L84-L198)

```
OTLP Exporter:
├── Protocol: gRPC
├── Endpoint: From configuration (e.g., http://localhost:4317)
├── Export Interval: 60 seconds (default)
│
├── Traces Batch Export
│   └─ Batches traces and sends to Grafana Tempo
│
└── Metrics Batch Export
    └─ Batches metrics and sends to Grafana Prometheus
```

#### 2. Serilog Sinks

```
Serilog:
├── Console Sink (always enabled)
│   └─ Writes structured logs to console
│
└── Optional Sinks (configurable)
    ├─ File sink
    ├─ Seq sink
    ├─ Elasticsearch sink
    └─ Grafana Loki sink (recommended)
```

#### 3. Data Flow

```
┌───────────────────────┐
│   Your Application    │
│                       │
│  ┌─────────────────┐  │
│  │ Traces          │──┼──► OTLP Exporter ──► Grafana Tempo
│  │ (Activities)    │  │        (gRPC)
│  └─────────────────┘  │
│                       │
│  ┌─────────────────┐  │
│  │ Metrics         │──┼──► OTLP Exporter ──► Grafana Prometheus
│  │ (Counters/Hist) │  │        (gRPC)
│  └─────────────────┘  │
│                       │
│  ┌─────────────────┐  │
│  │ Logs            │──┼──► Serilog Console ──► (Optional: Grafana Loki)
│  │ (Serilog)       │  │
│  └─────────────────┘  │
│                       │
└───────────────────────┘

All data correlated by TraceId and SpanId
```

---

## Execution Order Summary

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ ★ STARTUP (Once)                                                │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ 1. Load configuration                                            │
│ 2. Initialize Serilog (FIRST!)                                  │
│ 3. Create ResourceBuilder                                       │
│ 4. Configure Tracing (OTLP exporter)                            │
│ 5. Configure Metrics (OTLP exporter)                            │
│ 6. Register ActivitySource & Meter                              │
│ 7. Configure Activity tracking in logs                          │
└─────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ ★ REQUEST ARRIVES                                                │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ 1. AspNetCoreInstrumentation creates Activity (span)            │
│    └─ Activity.TraceId and Activity.SpanId generated            │
│                                                                  │
│ 2. Serilog request logging begins                               │
│    └─ Enriches logs with TraceId/SpanId                         │
│                                                                  │
│ 3. EncryptionMiddleware (if enabled)                            │
│ 4. ApiKeyMiddleware                                              │
│ 5. UseRouting() ← determines endpoint template                  │
│                                                                  │
│ 6. RedMetricsMiddleware.InvokeAsync ⭐                          │
│    ├─ Starts stopwatch                                          │
│    ├─ Registers OnCompleted callback ⏳                         │
│    └─ Calls next middleware                                     │
│                                                                  │
│ 7. Authentication/Authorization                                  │
│                                                                  │
│ 8. Controller execution                                          │
│    ├─ Automatic: SQL/HTTP instrumentation creates child spans   │
│    └─ Manual: Custom telemetry (optional)                       │
└─────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ ★ RESPONSE/ERROR                                                 │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│                                                                  │
│ ┌─ IF SUCCESS (2xx/3xx) ────────────────────────────────────┐  │
│ │ 9a. Response status code set                               │  │
│ │ 10a. RedMetrics OnCompleted callback fires ⏰             │  │
│ │     ├─ Records RequestCounter (Rate)                       │  │
│ │     └─ Records RequestDuration (Duration)                  │  │
│ └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│ ┌─ IF VALIDATION ERROR (400) ───────────────────────────────┐  │
│ │ 9b. InvalidModelStateResponseFactory                       │  │
│ │     ├─ Sets context.Items["IsValidationError"] = true ⭐  │  │
│ │     └─ Adds error tags to Activity                         │  │
│ │ 10b. RedMetrics OnCompleted callback fires ⏰             │  │
│ │     ├─ Records RequestCounter, RequestDuration             │  │
│ │     ├─ Records ErrorCounter (error.type="client_error")    │  │
│ │     └─ Records ValidationErrorCounter ⭐                   │  │
│ └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│ ┌─ IF EXCEPTION (500) ──────────────────────────────────────┐  │
│ │ 9c. GlobalExceptionHandler.TryHandleAsync                  │  │
│ │     ├─ Logs exception with TraceId (server-side)           │  │
│ │     ├─ ApplicationTelemetry.RecordException(exception)     │  │
│ │     │   └─ Adds error tags & event to Activity             │  │
│ │     └─ Returns ProblemDetails (500, NO stack trace)        │  │
│ │ 10c. RedMetrics OnCompleted callback fires ⏰             │  │
│ │     ├─ Records RequestCounter, RequestDuration             │  │
│ │     └─ Records ErrorCounter (error.type="server_error")    │  │
│ └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│ 11. AspNetCoreInstrumentation finalizes Activity                │
│     └─ Sets http.status_code tag and closes span                │
│                                                                  │
│ 12. Serilog logs request with TraceId/SpanId                    │
└─────────────────────────────────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ ★ BACKGROUND EXPORT (Continuous)                                │
│ ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ │
│ - OTLP Exporter batches and sends traces to Grafana Tempo       │
│ - OTLP Exporter batches and sends metrics to Grafana Prometheus │
│ - Serilog writes logs to console (+ optional sinks)             │
│                                                                  │
│ All data correlated by TraceId and SpanId                       │
└─────────────────────────────────────────────────────────────────┘
```

### Critical Execution Points

| **#** | **What** | **Where** | **When** |
|-------|----------|-----------|----------|
| 1 | Serilog initialization | [`Program.cs:34`](../backend/src/Crismac.Ews.WebApi/Program.cs#L34) | **EARLIEST** - Before any other telemetry |
| 2 | OpenTelemetry configuration | [`Program.cs:84-198`](../backend/src/Crismac.Ews.WebApi/Program.cs#L84-L198) | Startup, after Serilog |
| 3 | Activity creation | AspNetCoreInstrumentation | Start of each HTTP request |
| 4 | RedMetrics callback registration | [`RedMetricsMiddleware.cs:30`](../backend/src/Crismac.Ews.WebApi/Middlewares/RedMetricsMiddleware.cs#L30) | Middle of request pipeline |
| 5 | RedMetrics callback execution | [`RedMetricsMiddleware.cs:45-93`](../backend/src/Crismac.Ews.WebApi/Middlewares/RedMetricsMiddleware.cs#L45-L93) | End of request (after response sent) |
| 6 | OTLP export | Background task | Continuous (every 60s by default) |

---

## Key Components Reference

### Core Files

| **File** | **Purpose** |
|----------|-------------|
| [`Program.cs`](../backend/src/Crismac.Ews.WebApi/Program.cs) | Main configuration for telemetry, logging, and OpenTelemetry |
| [`ApplicationTelemetry.cs`](../backend/src/Crismac.Ews.WebApi/Observability/ApplicationTelemetry.cs) | Central telemetry class with ActivitySource, Meter, and custom metrics |
| [`LoginTelemetry.cs`](../backend/src/Crismac.Ews.WebApi/Observability/LoginTelemetry.cs) | Authentication-specific telemetry and metrics |
| [`RedMetricsMiddleware.cs`](../backend/src/Crismac.Ews.WebApi/Middlewares/RedMetricsMiddleware.cs) | Middleware that records RED metrics for all HTTP requests |
| [`GlobalExceptionHandler.cs`](../backend/src/Crismac.Ews.WebApi/Middlewares/GlobalExceptionHandler.cs) | Exception handler with telemetry integration |

### Key Metrics

#### RED Metrics (HTTP)

| **Metric Name** | **Type** | **Description** | **Tags** |
|----------------|----------|-----------------|----------|
| `api.requests.count` | Counter | **Rate** - Total number of requests | `http.method`, `http.route`, `http.status_code` |
| `api.requests.duration` | Histogram | **Duration** - Request duration in milliseconds | `http.method`, `http.route` |
| `api_errors` | Counter | **Errors** - Total number of errors (4xx/5xx) | `http.method`, `http.route`, `http.status_code`, `error.type` |
| `api_validation_errors` | Counter | Validation-specific errors (400) | `http.method`, `http.route` |

#### Login Metrics

| **Metric Name** | **Type** | **Description** | **Tags** |
|----------------|----------|-----------------|----------|
| `ews.login.attempts.total` | Counter | Total login attempts | `username`, `result`, `method` |
| `ews.login.duration.milliseconds` | Histogram | Login operation duration | `username`, `method` |
| `ews.token.operations.total` | Counter | Token operations (refresh, revoke) | `operation`, `result` |
| `ews.login.failures.by_user` | Counter | Failed logins by user | `username`, `reason` |
| `ews.account.lockouts.total` | Counter | Account lockout events | `username` |

#### Business Metrics (Optional)

| **Metric Name** | **Type** | **Description** | **Tags** |
|----------------|----------|-----------------|----------|
| `business.operations.count` | Counter | Business operations | `operation`, `status` |
| `business.operations.duration` | Histogram | Business operation duration | `operation` |
| `database.operations.count` | Counter | Database operations | `operation`, `table`, `success` |
| `database.operations.duration` | Histogram | Database operation duration | `operation`, `table` |
| `external.api.calls.count` | Counter | External API calls | `service`, `method`, `status_code` |
| `external.api.calls.duration` | Histogram | External API call duration | `service`, `method` |

### Trace Context Propagation

```
Request arrives with headers:
├─ traceparent: 00-{traceId}-{parentSpanId}-01
└─ tracestate: ...

Application creates child span:
├─ TraceId: (inherited from parent)
├─ SpanId: (new, unique to this span)
└─ ParentSpanId: (from traceparent header)

Response includes headers:
└─ traceparent: 00-{traceId}-{spanId}-01

All logs include:
├─ TraceId
├─ SpanId
└─ ParentId

Enables end-to-end distributed tracing across microservices
```

---

## Best Practices

### 1. Use Consistent Naming

- **Metric names**: Use dots for hierarchy (e.g., `api.requests.count`)
- **Tag names**: Use underscores (e.g., `http_method`)
- **Activity names**: Use human-readable names (e.g., "ProcessOrder")

### 2. Add Meaningful Tags

```csharp
// ✅ Good
activity?.SetTag("order.id", orderId);
activity?.SetTag("customer.type", "enterprise");

// ❌ Bad (too verbose, high cardinality)
activity?.SetTag("full.request.body", JsonSerializer.Serialize(request));
```

### 3. Avoid High Cardinality Tags

```csharp
// ✅ Good (low cardinality)
tags.Add("http.status_code", 200);        // Limited values: 200, 201, 400, 500...
tags.Add("error.type", "validation");     // Limited values: validation, timeout, etc.

// ❌ Bad (high cardinality)
tags.Add("user.id", userId);              // Millions of unique values
tags.Add("request.timestamp", timestamp); // Every request is unique
```

### 4. Always Record Exceptions

```csharp
try
{
    // ... business logic ...
}
catch (Exception ex)
{
    // Record in telemetry
    ApplicationTelemetry.RecordException(ex);

    // Log for server-side visibility
    _logger.LogError(ex, "Operation failed");

    throw; // Re-throw to let GlobalExceptionHandler handle it
}
```

### 5. Use Structured Logging

```csharp
// ✅ Good (structured)
_logger.LogInformation("Order created with OrderId: {OrderId}, Total: {Total}", orderId, total);

// ❌ Bad (string concatenation)
_logger.LogInformation($"Order created with OrderId: {orderId}, Total: {total}");
```

---

## Troubleshooting

### Traces Not Appearing in Grafana

1. Check OTLP endpoint configuration in `appsettings.json`
2. Verify Grafana Tempo is running and accessible
3. Check application logs for export errors
4. Verify firewall allows gRPC traffic (port 4317)

### Metrics Not Updating

1. Verify meter is registered in OpenTelemetry configuration
2. Check metric collection interval (default: 60s)
3. Ensure tags match expected cardinality limits

### Logs Missing TraceId

1. Verify `Enrich.WithSpanId()` and `Enrich.WithTraceId()` are configured
2. Check that Activity.Current is not null during logging
3. Ensure Serilog is initialized after OpenTelemetry

### High Memory Usage

1. Review trace sampling rate (default: 100%)
2. Reduce metric cardinality (avoid high-cardinality tags)
3. Adjust batch export intervals

---

## Related Documentation

- [OpenTelemetry .NET Documentation](https://opentelemetry.io/docs/languages/net/)
- [Serilog Documentation](https://serilog.net/)
- [Grafana Stack Documentation](https://grafana.com/docs/)
- [RED Method (Rate, Errors, Duration)](https://grafana.com/blog/2018/08/02/the-red-method-how-to-instrument-your-services/)

---

**Document Version**: 1.0
**Last Updated**: 2026-01-29
**Maintained By**: Crismac EWS Team
