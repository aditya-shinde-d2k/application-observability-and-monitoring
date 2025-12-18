# Development Guide: Instrumenting Code for Observability

## Complete Developer Reference for EWS Core API

**Audience:** Software developers, QA engineers
**Reading Time:** 60-90 minutes
**Prerequisites:** Basic C# knowledge, familiarity with ASP.NET Core

---

## Table of Contents

1. [Quick Start](#quick-start)
2. [Core Telemetry Infrastructure](#core-telemetry-infrastructure)
3. [Adding Telemetry to Controllers](#adding-telemetry-to-controllers)
4. [Custom Metrics Creation](#custom-metrics-creation)
5. [Distributed Tracing Patterns](#distributed-tracing-patterns)
6. [Structured Logging Best Practices](#structured-logging-best-practices)
7. [Testing Telemetry Locally](#testing-telemetry-locally)
8. [Common Patterns and Examples](#common-patterns-and-examples)
9. [Integration Checklist](#integration-checklist)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## Quick Start

### 5-Minute Integration

For simple endpoints that don't need custom metrics:

```csharp
[HttpGet("users/{id}")]
public async Task<IActionResult> GetUser(int id)
{
    // That's it! RedMetricsMiddleware automatically tracks:
    // - Request count
    // - Request duration
    // - HTTP status codes
    // - Error rates

    var user = await _userService.GetUser(id);
    return Ok(user);
}
```

**No manual instrumentation needed!** The middleware handles everything.

---

### When to Add Manual Instrumentation

Add manual telemetry when you need:

1. **Business metrics** - Login success rate, orders processed, etc.
2. **Custom traces** - Database operations, external API calls
3. **Domain-specific tracking** - Audit trails, security events
4. **Performance insights** - Specific operation timing

---

## Core Telemetry Infrastructure

### ApplicationTelemetry.cs - Central Hub

**Location:** `Crismac.Ews.WebApi/Observability/ApplicationTelemetry.cs`

This is your central telemetry infrastructure. All instrumentation goes through here.

#### Key Components

```csharp
public static class ApplicationTelemetry
{
    // Tracing: Create spans for operations
    public static readonly ActivitySource ActivitySource;

    // Metrics: Create counters, histograms, gauges
    public static readonly Meter Meter;

    // RED Metrics (automatically populated by middleware)
    public static readonly Counter<long> RequestCounter;
    public static readonly Histogram<double> RequestDuration;
    public static readonly Counter<long> ErrorCounter;
    public static readonly Counter<long> ValidationErrorCounter;

    // Business Metrics
    public static readonly Counter<long> BusinessOperationCounter;
    public static readonly Histogram<double> BusinessOperationDuration;
    public static readonly Counter<long> DatabaseOperationCounter;
    public static readonly Histogram<double> DatabaseOperationDuration;
    public static readonly Counter<long> ExternalApiCallCounter;
    public static readonly Histogram<double> ExternalApiCallDuration;
}
```

---

### Helper Methods

#### Recording Requests (Auto-handled by middleware)

```csharp
// Don't call this manually - middleware handles it
ApplicationTelemetry.RecordRequest(
    method: "GET",
    route: "/api/users/{id}",
    statusCode: 200,
    durationMs: 45.2
);
```

#### Recording Business Operations

```csharp
ApplicationTelemetry.RecordBusinessOperation(
    operationType: "order_processing",
    status: "success",
    durationMs: 234.5
);
```

#### Recording Database Operations

```csharp
ApplicationTelemetry.RecordDatabaseOperation(
    operation: "SELECT",
    table: "Users",
    success: true,
    durationMs: 12.3
);
```

#### Recording External API Calls

```csharp
ApplicationTelemetry.RecordExternalApiCall(
    service: "PaymentGateway",
    method: "POST",
    statusCode: 200,
    durationMs: 567.8
);
```

---

### Tracing Helpers

#### Creating Custom Spans

```csharp
using var activity = ApplicationTelemetry.StartActivity(
    "ProcessPayment",
    ActivityKind.Internal
);

activity?.SetTag("payment.amount", 99.99);
activity?.SetTag("payment.currency", "USD");

try
{
    await ProcessPayment();
    activity?.SetStatus(ActivityStatusCode.Ok);
}
catch (Exception ex)
{
    ApplicationTelemetry.RecordException(ex);
    activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
    throw;
}
```

#### Adding Tags and Events

```csharp
// Add tags for metadata
ApplicationTelemetry.AddTag("user.type", "premium");
ApplicationTelemetry.AddTag("transaction.id", transactionId);

// Add events for checkpoints
ApplicationTelemetry.AddEvent("Payment validated");
ApplicationTelemetry.AddEvent("Inventory reserved");
```

#### Recording Exceptions

```csharp
try
{
    await RiskyOperation();
}
catch (Exception ex)
{
    // Automatically adds exception details to current span
    ApplicationTelemetry.RecordException(ex);

    // Logs exception with full context
    _logger.LogError(
        ex,
        "Operation failed. TraceId: {TraceId}",
        Activity.Current?.TraceId
    );

    throw;
}
```

---

## Adding Telemetry to Controllers

### Pattern 1: Simple GET Endpoint (No Custom Telemetry)

**Use Case:** Basic CRUD operations where middleware telemetry is sufficient.

```csharp
[HttpGet("products/{id}")]
[ProducesResponseType(typeof(Product), StatusCodes.Status200OK)]
[ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status404NotFound)]
public async Task<IActionResult> GetProduct(int id)
{
    // Middleware automatically tracks:
    // - api_requests_count_total
    // - api_requests_duration_milliseconds
    // - http.method = GET
    // - http.route = /api/products/{id}
    // - http.status_code = 200 or 404

    var product = await _productService.GetProduct(id);

    if (product == null)
    {
        return NotFound();
    }

    return Ok(product);
}
```

**Metrics Exported:**
- Request count: +1
- Request duration: auto-measured
- Status code: 200 or 404

**No manual code needed!**

---

### Pattern 2: POST Endpoint with Business Metrics

**Use Case:** Operations that need business-level tracking (orders, payments, etc.)

```csharp
[HttpPost("orders")]
[ProducesResponseType(typeof(Order), StatusCodes.Status201Created)]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
{
    // Start business operation trace
    using var orderTrace = ApplicationTelemetry.StartActivity(
        "CreateOrder",
        ActivityKind.Server
    );
    orderTrace?.SetTag("order.customer_id", request.CustomerId);
    orderTrace?.SetTag("order.item_count", request.Items.Count);

    var stopwatch = Stopwatch.StartNew();

    try
    {
        // Your business logic
        var order = await _orderService.CreateOrder(request);

        stopwatch.Stop();

        // Record business metrics
        ApplicationTelemetry.RecordBusinessOperation(
            operationType: "order_creation",
            status: "success",
            durationMs: stopwatch.ElapsedMilliseconds
        );

        orderTrace?.SetTag("order.id", order.Id);
        orderTrace?.SetTag("order.total", order.Total);
        orderTrace?.SetStatus(ActivityStatusCode.Ok);

        _logger.LogInformation(
            "Order created successfully. OrderId: {OrderId}, TraceId: {TraceId}",
            order.Id,
            orderTrace?.TraceId
        );

        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
    }
    catch (Exception ex)
    {
        stopwatch.Stop();

        // Record failure
        ApplicationTelemetry.RecordBusinessOperation(
            operationType: "order_creation",
            status: "failed",
            durationMs: stopwatch.ElapsedMilliseconds
        );

        ApplicationTelemetry.RecordException(ex);
        orderTrace?.SetStatus(ActivityStatusCode.Error, ex.Message);

        _logger.LogError(
            ex,
            "Order creation failed. TraceId: {TraceId}",
            orderTrace?.TraceId
        );

        throw; // Let GlobalExceptionHandler handle response
    }
}
```

**Metrics Exported:**
- HTTP metrics (from middleware)
- Business operation counter
- Business operation duration
- Custom tags in traces

---

### Pattern 3: Database Operation Tracking

**Use Case:** Tracking stored procedure or query performance.

```csharp
[HttpGet("reports/{id}")]
public async Task<IActionResult> GetReport(int id)
{
    // Create database operation span
    using var dbActivity = ApplicationTelemetry.StartActivity(
        "DB_GetReport",
        ActivityKind.Internal
    );
    dbActivity?.SetTag("db.system", "mssql");
    dbActivity?.SetTag("db.operation", "SELECT");
    dbActivity?.SetTag("db.table", "Reports");

    var dbStopwatch = Stopwatch.StartNew();

    try
    {
        var report = await _reportRepository.GetReport(id);

        dbStopwatch.Stop();

        // Record database metrics
        ApplicationTelemetry.RecordDatabaseOperation(
            operation: "GetReport",
            table: "Reports",
            success: report != null,
            durationMs: dbStopwatch.ElapsedMilliseconds
        );

        dbActivity?.SetStatus(ActivityStatusCode.Ok);

        if (report == null)
        {
            return NotFound();
        }

        return Ok(report);
    }
    catch (Exception ex)
    {
        dbStopwatch.Stop();

        ApplicationTelemetry.RecordDatabaseOperation(
            operation: "GetReport",
            table: "Reports",
            success: false,
            durationMs: dbStopwatch.ElapsedMilliseconds
        );

        ApplicationTelemetry.RecordException(ex);
        dbActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);

        throw;
    }
}
```

---

### Pattern 4: External API Call Tracking

**Use Case:** Monitoring calls to third-party services.

```csharp
[HttpPost("payments")]
public async Task<IActionResult> ProcessPayment([FromBody] PaymentRequest request)
{
    // Create external API call span
    using var paymentActivity = ApplicationTelemetry.StartActivity(
        "ExternalAPI_ProcessPayment",
        ActivityKind.Client
    );
    paymentActivity?.SetTag("external.service", "StripePaymentGateway");
    paymentActivity?.SetTag("payment.amount", request.Amount);

    var stopwatch = Stopwatch.StartNew();

    try
    {
        var response = await _paymentGateway.ProcessPayment(request);

        stopwatch.Stop();

        // Record external API call metrics
        ApplicationTelemetry.RecordExternalApiCall(
            service: "StripePaymentGateway",
            method: "POST",
            statusCode: 200,
            durationMs: stopwatch.ElapsedMilliseconds
        );

        paymentActivity?.SetTag("payment.transaction_id", response.TransactionId);
        paymentActivity?.SetStatus(ActivityStatusCode.Ok);

        return Ok(response);
    }
    catch (HttpRequestException ex)
    {
        stopwatch.Stop();

        ApplicationTelemetry.RecordExternalApiCall(
            service: "StripePaymentGateway",
            method: "POST",
            statusCode: (int)(ex.StatusCode ?? System.Net.HttpStatusCode.ServiceUnavailable),
            durationMs: stopwatch.ElapsedMilliseconds
        );

        ApplicationTelemetry.RecordException(ex);
        paymentActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);

        throw;
    }
}
```

---

## Custom Metrics Creation

### Creating Domain-Specific Telemetry

**Example:** Login-specific telemetry (already implemented in `LoginTelemetry.cs`)

```csharp
public static class LoginTelemetry
{
    private static readonly Meter Meter = new Meter("Crismac.Ews.Login");

    // Counter for login attempts
    public static readonly Counter<long> LoginAttempts = Meter.CreateCounter<long>(
        name: "ews.login.attempts.total",
        unit: "{attempts}",
        description: "Total number of login attempts"
    );

    // Histogram for login duration
    public static readonly Histogram<double> LoginDuration = Meter.CreateHistogram<double>(
        name: "ews.login.duration.milliseconds",
        unit: "ms",
        description: "Duration of login operations"
    );

    // Counter for token operations
    public static readonly Counter<long> TokenOperations = Meter.CreateCounter<long>(
        name: "ews.token.operations.total",
        unit: "{operations}",
        description: "JWT token operations"
    );

    // Helper methods
    public static void RecordSuccessfulLogin(string username, string authMethod, double durationMs)
    {
        var tags = new KeyValuePair<string, object?>[]
        {
            new("status", "success"),
            new("authentication_method", authMethod),
            new("username_hash", HashUsername(username))
        };

        LoginAttempts.Add(1, tags);
        LoginDuration.Record(durationMs, new KeyValuePair<string, object?>[]
        {
            new("operation", "total_flow"),
            new("status", "success")
        });
    }

    public static void RecordFailedLogin(
        string username,
        string failureReason,
        string? ipAddress = null,
        double? durationMs = null)
    {
        var tags = new KeyValuePair<string, object?>[]
        {
            new("status", "failed"),
            new("failure_reason", failureReason),
            new("username_hash", HashUsername(username))
        };

        LoginAttempts.Add(1, tags);

        if (durationMs.HasValue)
        {
            LoginDuration.Record(durationMs.Value, new KeyValuePair<string, object?>[]
            {
                new("operation", "total_flow"),
                new("status", "failed")
            });
        }
    }
}
```

---

### Register Custom Meters in Program.cs

```csharp
builder.Services.AddOpenTelemetry()
    .WithMetrics(meterProviderBuilder =>
    {
        meterProviderBuilder
            // Application-wide metrics
            .AddMeter(ApplicationTelemetry.InstrumentationName)

            // Domain-specific metrics
            .AddMeter("Crismac.Ews.Login")
            .AddMeter("Crismac.Ews.Orders")
            .AddMeter("Crismac.Ews.Reports")

            // ... exporters
    });
```

---

### Example: Creating File Upload Telemetry

```csharp
public static class FileUploadTelemetry
{
    private static readonly Meter Meter = new Meter("Crismac.Ews.FileUpload");

    public static readonly Counter<long> UploadAttempts = Meter.CreateCounter<long>(
        name: "ews.file.uploads.total",
        unit: "{uploads}",
        description: "Total file upload attempts"
    );

    public static readonly Histogram<double> FileSize = Meter.CreateHistogram<double>(
        name: "ews.file.size.bytes",
        unit: "bytes",
        description: "Size of uploaded files"
    );

    public static readonly Histogram<double> UploadDuration = Meter.CreateHistogram<double>(
        name: "ews.file.upload.duration.milliseconds",
        unit: "ms",
        description: "Duration of file upload operations"
    );

    public static void RecordSuccessfulUpload(
        string fileType,
        long fileSizeBytes,
        double durationMs)
    {
        var tags = new KeyValuePair<string, object?>[]
        {
            new("file.type", fileType),
            new("status", "success")
        };

        UploadAttempts.Add(1, tags);
        FileSize.Record(fileSizeBytes, tags);
        UploadDuration.Record(durationMs, tags);
    }

    public static void RecordFailedUpload(
        string fileType,
        string failureReason,
        double durationMs)
    {
        var tags = new KeyValuePair<string, object?>[]
        {
            new("file.type", fileType),
            new("status", "failed"),
            new("failure_reason", failureReason)
        };

        UploadAttempts.Add(1, tags);
        UploadDuration.Record(durationMs, tags);
    }
}
```

**Usage in Controller:**

```csharp
[HttpPost("upload")]
public async Task<IActionResult> UploadFile(IFormFile file)
{
    var stopwatch = Stopwatch.StartNew();

    try
    {
        await _fileService.SaveFile(file);

        stopwatch.Stop();

        FileUploadTelemetry.RecordSuccessfulUpload(
            fileType: Path.GetExtension(file.FileName),
            fileSizeBytes: file.Length,
            durationMs: stopwatch.ElapsedMilliseconds
        );

        return Ok(new { message = "File uploaded successfully" });
    }
    catch (Exception ex)
    {
        stopwatch.Stop();

        FileUploadTelemetry.RecordFailedUpload(
            fileType: Path.GetExtension(file.FileName),
            failureReason: ex.GetType().Name,
            durationMs: stopwatch.ElapsedMilliseconds
        );

        throw;
    }
}
```

---

## Distributed Tracing Patterns

### Understanding Spans

**Span:** A single unit of work with a start time and duration.

**Span Hierarchy:**
```
Root Span (HTTP Request)
├─ Child Span (Database Query)
├─ Child Span (Cache Lookup)
└─ Child Span (External API Call)
   └─ Grandchild Span (HTTP Request)
```

---

### Creating Nested Spans

```csharp
[HttpPost("complex-operation")]
public async Task<IActionResult> ComplexOperation([FromBody] Request request)
{
    // Root span (created automatically by ASP.NET Core instrumentation)
    using var parentActivity = ApplicationTelemetry.StartActivity(
        "ComplexBusinessOperation",
        ActivityKind.Internal
    );

    // Child span 1: Validate
    using (var validateActivity = ApplicationTelemetry.StartActivity(
        "ValidateRequest",
        ActivityKind.Internal
    ))
    {
        validateActivity?.SetTag("validation.rules", 5);
        await ValidateRequest(request);
        validateActivity?.SetStatus(ActivityStatusCode.Ok);
    }

    // Child span 2: Process
    using (var processActivity = ApplicationTelemetry.StartActivity(
        "ProcessData",
        ActivityKind.Internal
    ))
    {
        processActivity?.SetTag("data.size", request.Data.Length);
        var result = await ProcessData(request);
        processActivity?.SetTag("result.id", result.Id);
        processActivity?.SetStatus(ActivityStatusCode.Ok);
    }

    // Child span 3: Notify
    using (var notifyActivity = ApplicationTelemetry.StartActivity(
        "SendNotification",
        ActivityKind.Client
    ))
    {
        notifyActivity?.SetTag("notification.type", "email");
        await SendNotification(request.Email);
        notifyActivity?.SetStatus(ActivityStatusCode.Ok);
    }

    parentActivity?.SetStatus(ActivityStatusCode.Ok);

    return Ok();
}
```

**Result in Tempo:**
```
HTTP POST /api/complex-operation (1.2s)
├─ ComplexBusinessOperation (1.1s)
   ├─ ValidateRequest (200ms)
   ├─ ProcessData (700ms)
   └─ SendNotification (200ms)
```

---

### Propagating Context Across Services

**W3C Trace Context Headers:**
- `traceparent`: Contains TraceId and SpanId
- `tracestate`: Additional vendor-specific data

**Automatic Propagation with HttpClient:**

```csharp
// Configure in Program.cs
builder.Services.AddHttpClient("ExternalService", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
})
.AddHttpMessageHandler(() =>
    new HttpClientTraceIdHandler()); // OpenTelemetry automatically adds headers

// Usage in service
public class ExternalService
{
    private readonly IHttpClientFactory _httpClientFactory;

    public async Task<Response> CallExternalApi()
    {
        var client = _httpClientFactory.CreateClient("ExternalService");

        // OpenTelemetry automatically adds:
        // traceparent: 00-0af7651916cd43dd8448eb211c80319c-9d8e7f6a5b4c3d2e-01
        var response = await client.GetAsync("/api/data");

        return await response.Content.ReadFromJsonAsync<Response>();
    }
}
```

---

### Span Best Practices

**DO:**
```csharp
// Use descriptive names
using var activity = ApplicationTelemetry.StartActivity("ProcessOrder");

// Add relevant tags
activity?.SetTag("order.id", orderId);
activity?.SetTag("order.total", 99.99);

// Set appropriate status
activity?.SetStatus(ActivityStatusCode.Ok);
```

**DON'T:**
```csharp
// Vague names
using var activity = ApplicationTelemetry.StartActivity("Process");

// High cardinality tags
activity?.SetTag("user.id", userId); // Millions of unique values

// Missing status
// (Always set status for clarity)
```

---

## Structured Logging Best Practices

### Correlation with Traces

**Always include TraceId in logs:**

```csharp
_logger.LogInformation(
    "User logged in successfully: {Username}. TraceId: {TraceId}",
    username,
    Activity.Current?.TraceId.ToString()
);
```

**Benefits:**
- Click on log → jump to trace in Grafana
- Click on trace → see all related logs
- Complete context for debugging

---

### Structured Logging Patterns

**Good:**
```csharp
_logger.LogInformation(
    "Order processed. OrderId: {OrderId}, Amount: {Amount}, Duration: {DurationMs}ms, TraceId: {TraceId}",
    order.Id,
    order.Amount,
    stopwatch.ElapsedMilliseconds,
    Activity.Current?.TraceId
);
```

**Bad:**
```csharp
_logger.LogInformation(
    $"Order processed: {order.Id}, ${order.Amount}, took {stopwatch.ElapsedMilliseconds}ms"
);
// No TraceId, string interpolation makes searching harder
```

---

### Log Levels

| Level | When to Use | Example |
|-------|-------------|---------|
| **Trace** | Extremely detailed diagnostics | Entry/exit of methods |
| **Debug** | Detailed information for debugging | Variable values, flow control |
| **Information** | General flow of application | User logged in, order created |
| **Warning** | Unexpected but recoverable | Validation failed, retry attempt |
| **Error** | Errors and exceptions | Database connection failed |
| **Critical** | Critical failures | Application crash, data loss |

---

### Example: Comprehensive Logging

```csharp
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
{
    var stopwatch = Stopwatch.StartNew();
    using var activity = ApplicationTelemetry.StartActivity("CreateOrder");

    _logger.LogInformation(
        "Order creation started. CustomerId: {CustomerId}, Items: {ItemCount}, TraceId: {TraceId}",
        request.CustomerId,
        request.Items.Count,
        activity?.TraceId
    );

    try
    {
        // Validate
        if (!await _validator.Validate(request))
        {
            _logger.LogWarning(
                "Order validation failed. CustomerId: {CustomerId}, Reason: {Reason}, TraceId: {TraceId}",
                request.CustomerId,
                "Invalid items",
                activity?.TraceId
            );
            return BadRequest("Invalid order items");
        }

        // Process
        var order = await _orderService.CreateOrder(request);
        stopwatch.Stop();

        _logger.LogInformation(
            "Order created successfully. OrderId: {OrderId}, CustomerId: {CustomerId}, Total: {Total}, Duration: {DurationMs}ms, TraceId: {TraceId}",
            order.Id,
            request.CustomerId,
            order.Total,
            stopwatch.ElapsedMilliseconds,
            activity?.TraceId
        );

        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
    }
    catch (Exception ex)
    {
        stopwatch.Stop();

        _logger.LogError(
            ex,
            "Order creation failed. CustomerId: {CustomerId}, Duration: {DurationMs}ms, TraceId: {TraceId}",
            request.CustomerId,
            stopwatch.ElapsedMilliseconds,
            activity?.TraceId
        );

        throw;
    }
}
```

---

## Testing Telemetry Locally

### Step 1: Start LGTM Stack

```bash
cd "d:\OfficeWork\EWS Migration\ews_coreapi_enterprises\Crismac.Ews.WebApi"
docker-compose -f otel-lgtm-docker-compose.yaml up -d

# Verify running
docker ps
```

---

### Step 2: Run Application

```bash
dotnet run --project Crismac.Ews.WebApi
```

---

### Step 3: Make Test Requests

```bash
# Test endpoint
curl -X POST http://localhost:5294/api/orders \
  -H "Content-Type: application/json" \
  -d '{"customerId": 123, "items": [{"id": 1, "quantity": 2}]}'
```

---

### Step 4: Verify in Grafana

**Access Grafana:**
```
URL: http://localhost:3000
Login: admin / Qwerty!123
```

**Check Metrics:**
1. Go to "Explore"
2. Select "Mimir" datasource
3. Query: `api_requests_count_total`
4. Should see your test requests

**Check Traces:**
1. Go to "Explore"
2. Select "Tempo" datasource
3. Search for service: `CrismacEWSBackendService`
4. Find your request trace
5. Click to see span details

**Check Logs:**
1. Go to "Explore"
2. Select "Loki" datasource
3. Query: `{service_name="CrismacEWSBackendService"}`
4. Should see structured logs with TraceId

---

### Step 5: Test Correlation

1. **Get TraceId from Logs:**
   ```logql
   {service_name="CrismacEWSBackendService"}
   |= "Order created successfully"
   | json
   ```

2. **Find Trace:**
   - Copy TraceId from log
   - Go to Tempo
   - Search by TraceId

3. **Verify:**
   - Span shows correct duration
   - Tags are present
   - Child spans visible

---

## Common Patterns and Examples

### Pattern 1: Validation Error Tracking

```csharp
[HttpPost("users")]
public async Task<IActionResult> CreateUser([FromBody] CreateUserRequest request)
{
    // Custom validation
    if (await _userService.UserExists(request.Email))
    {
        // Mark as validation error for metrics
        ApplicationTelemetry.MarkAsValidationError(HttpContext);

        return Problem(
            statusCode: StatusCodes.Status400BadRequest,
            title: "Validation Failed",
            detail: "User with this email already exists."
        );
    }

    var user = await _userService.CreateUser(request);
    return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
}
```

**Metrics Tracked:**
- `api_validation_errors_total` (via flag)
- `api_errors_total` (automatically)

---

### Pattern 2: Retry Logic with Telemetry

```csharp
public async Task<T> ExecuteWithRetry<T>(Func<Task<T>> operation, int maxRetries = 3)
{
    using var activity = ApplicationTelemetry.StartActivity("RetryableOperation");
    activity?.SetTag("max_retries", maxRetries);

    for (int attempt = 1; attempt <= maxRetries; attempt++)
    {
        try
        {
            activity?.AddEvent(new ActivityEvent($"Attempt {attempt}"));

            var result = await operation();

            activity?.SetTag("attempts_used", attempt);
            activity?.SetStatus(ActivityStatusCode.Ok);

            return result;
        }
        catch (Exception ex) when (attempt < maxRetries)
        {
            _logger.LogWarning(
                "Operation failed, attempt {Attempt}/{MaxRetries}. TraceId: {TraceId}",
                attempt,
                maxRetries,
                activity?.TraceId
            );

            await Task.Delay(TimeSpan.FromSeconds(Math.Pow(2, attempt))); // Exponential backoff
        }
    }

    activity?.SetStatus(ActivityStatusCode.Error, "Max retries exceeded");
    throw new Exception($"Operation failed after {maxRetries} attempts");
}
```

---

### Pattern 3: Batch Processing Telemetry

```csharp
public async Task<BatchResult> ProcessBatch(List<Item> items)
{
    using var batchActivity = ApplicationTelemetry.StartActivity("ProcessBatch");
    batchActivity?.SetTag("batch.size", items.Count);

    var stopwatch = Stopwatch.StartNew();
    int successCount = 0;
    int failureCount = 0;

    foreach (var item in items)
    {
        using var itemActivity = ApplicationTelemetry.StartActivity("ProcessItem");
        itemActivity?.SetTag("item.id", item.Id);

        try
        {
            await ProcessItem(item);
            successCount++;
            itemActivity?.SetStatus(ActivityStatusCode.Ok);
        }
        catch (Exception ex)
        {
            failureCount++;
            ApplicationTelemetry.RecordException(ex);
            itemActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);
        }
    }

    stopwatch.Stop();

    batchActivity?.SetTag("batch.success_count", successCount);
    batchActivity?.SetTag("batch.failure_count", failureCount);
    batchActivity?.SetStatus(ActivityStatusCode.Ok);

    // Record batch metrics
    ApplicationTelemetry.RecordBusinessOperation(
        operationType: "batch_processing",
        status: failureCount == 0 ? "success" : "partial_failure",
        durationMs: stopwatch.ElapsedMilliseconds
    );

    return new BatchResult
    {
        TotalItems = items.Count,
        SuccessCount = successCount,
        FailureCount = failureCount,
        DurationMs = stopwatch.ElapsedMilliseconds
    };
}
```

---

## Integration Checklist

### Before Committing Code

- [ ] **Middleware active:** RedMetricsMiddleware registered in Program.cs
- [ ] **No manual HTTP metrics:** Controllers don't call RecordRequest manually
- [ ] **TraceId in logs:** All important logs include TraceId
- [ ] **Exception handling:** Exceptions recorded via ApplicationTelemetry.RecordException
- [ ] **Business metrics:** Critical operations have custom metrics
- [ ] **Span status:** All activities have status set (Ok or Error)
- [ ] **Tag cardinality:** No high-cardinality tags (user IDs, timestamps)
- [ ] **Meter registration:** Custom meters registered in Program.cs
- [ ] **Testing:** Verified metrics appear in Grafana
- [ ] **Documentation:** Updated if new patterns introduced

---

### Code Review Checklist

**Metrics:**
- [ ] Meaningful metric names
- [ ] Appropriate metric types (Counter vs Histogram)
- [ ] Low-cardinality tags
- [ ] Units specified
- [ ] Descriptions added

**Traces:**
- [ ] Descriptive span names
- [ ] Relevant tags added
- [ ] Status code set correctly
- [ ] Exceptions recorded
- [ ] No overly granular spans (performance)

**Logs:**
- [ ] TraceId included
- [ ] Structured (not interpolated)
- [ ] Appropriate log level
- [ ] Sensitive data sanitized
- [ ] Meaningful context included

---

## Troubleshooting Guide

### Metrics Not Appearing

**Symptom:** No data in Grafana for custom metrics.

**Check:**
1. Meter registered in Program.cs
2. Metric name matches query
3. Wait 60 seconds (default export interval)
4. Check console output for errors

**Solution:**
```csharp
// In Program.cs
.WithMetrics(meterProviderBuilder =>
{
    meterProviderBuilder
        .AddMeter("Your.Meter.Name") // ← Ensure this matches
        .AddOtlpExporter();
});
```

---

### Traces Not Showing

**Symptom:** Activity.Current is null or traces not in Tempo.

**Check:**
1. ActivitySource registered in Program.cs
2. OTLP exporter configured
3. Activity started correctly

**Solution:**
```csharp
// In Program.cs
.WithTracing(tracerProviderBuilder =>
{
    tracerProviderBuilder
        .AddSource(ApplicationTelemetry.InstrumentationName) // ← Ensure this matches
        .AddOtlpExporter();
});
```

---

### Missing TraceId in Logs

**Symptom:** Logs don't show TraceId.

**Check:**
1. Activity tracking enabled in logging configuration
2. TraceId explicitly added to log message

**Solution:**
```csharp
// In Program.cs
builder.Logging.Configure(options =>
{
    options.ActivityTrackingOptions =
        ActivityTrackingOptions.TraceId |
        ActivityTrackingOptions.SpanId;
});

// In code
_logger.LogInformation(
    "Message. TraceId: {TraceId}",
    Activity.Current?.TraceId.ToString()
);
```

---

### High Cardinality Warning

**Symptom:** Prometheus/Mimir complains about cardinality.

**Problem:** Too many unique tag values.

**Solution:**
```csharp
// ❌ BAD: High cardinality
activity?.SetTag("user.id", userId); // Millions of values

// ✅ GOOD: Low cardinality
activity?.SetTag("user.type", userType); // Few values (free, premium, enterprise)

// ✅ GOOD: Hash sensitive data
activity?.SetTag("user.id_hash", HashUserId(userId)); // Fixed-size hash
```

---

## Next Steps

1. **Read:** [Integration Examples](./03-integration-examples.md) for real-world code
2. **Practice:** Instrument a simple endpoint following patterns above
3. **Review:** [Challenges & Scenarios](./07-challenges-and-scenarios.md) for advanced topics
4. **Reference:** [FAQ & Troubleshooting](./06-faq-and-troubleshooting.md) when stuck

---

**Remember:** Good telemetry leads to faster debugging, better performance insights, and happier users!
