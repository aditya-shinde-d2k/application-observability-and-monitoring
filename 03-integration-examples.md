# Integration Examples: Real-World Instrumentation

## Complete Code Examples from EWS Core API

**Audience:** Developers implementing observability
**Reading Time:** 90-120 minutes
**Prerequisites:** Understanding of [Development Guide](./02-development-guide.md)

---

## Table of Contents

1. [Login Flow Instrumentation (Complete Example)](#login-flow-instrumentation-complete-example)
2. [Database Operation Tracking](#database-operation-tracking)
3. [External API Call Monitoring](#external-api-call-monitoring)
4. [File Upload Monitoring](#file-upload-monitoring)
5. [Report Generation Tracking](#report-generation-tracking)
6. [Background Job Processing](#background-job-processing)
7. [Before/After Comparisons](#beforeafter-comparisons)

---

## Login Flow Instrumentation (Complete Example)

This is the complete, production-ready implementation from `LoginController.cs`.

### Overview

The login flow demonstrates:
- Complete request tracing
- Business metrics (login success/failure)
- Database operation tracking
- Token generation monitoring
- Comprehensive error handling
- Log correlation

---

### Full Implementation

```csharp
using System.Diagnostics;
using Crismac.Ews.WebApi.Observability;
using Microsoft.AspNetCore.Mvc;

[Route("api/[controller]")]
public class LoginController : BaseApiController
{
    private readonly ILoginRepository _repository;
    private readonly IUserService _userService;
    private readonly ILogger<LoginController> _logger;
    private readonly CryptoService _cryptoService;

    [Route("CrisMAc/SelectLoginDetails")]
    [HttpPost]
    [AllowAnonymous]
    [ProducesResponseType(typeof(ApiResponse<AuthenticateResponse>), StatusCodes.Status200OK)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status400BadRequest)]
    [ProducesResponseType(typeof(ProblemDetails), StatusCodes.Status401Unauthorized)]
    public async Task<IActionResult> SelectLoginDetails([FromBody] AuthenticateRequestWithOtp model)
    {
        // 1. Get client IP and start distributed trace
        var ipAddress = GetClientIpAddress();
        _logger.LogInformation("Client IP Address: {IpAddress}", ipAddress);

        using var loginTrace = LoginTelemetry.StartLoginTrace(model.Username, ipAddress);
        var totalStopwatch = Stopwatch.StartNew();

        try
        {
            // 2. Decrypt and hash password
            string decryptedPassword = await _cryptoService.DecryptWithAesAsync(model.Password);
            string passwordHash = _cryptoService.Hash(decryptedPassword);
            model = model with { Password = passwordHash };

            // 3. Database Query with Telemetry
            using var dbActivity = ApplicationTelemetry.StartActivity("DB_ValidateUser", ActivityKind.Internal);
            var dbStopwatch = Stopwatch.StartNew();

            var userData = await _repository.SelectLoginDetails(model);

            dbStopwatch.Stop();

            // Record database operation metrics
            LoginTelemetry.RecordDatabaseOperation(
                "SelectLoginDetails",
                dbStopwatch.ElapsedMilliseconds,
                success: userData != null
            );

            // 4. Handle: User Not Found
            if (userData == null)
            {
                totalStopwatch.Stop();

                // Record failed login
                LoginTelemetry.RecordFailedLogin(
                    model.Username,
                    "invalid_credentials",
                    ipAddress,
                    totalStopwatch.ElapsedMilliseconds
                );

                LoginTelemetry.SetLoginResult(loginTrace, false, "invalid_credentials");
                dbActivity?.SetStatus(ActivityStatusCode.Error, "User not found");

                _logger.LogWarning(
                    "Login failed: Invalid credentials for {Username} from {IpAddress}. TraceId: {TraceId}",
                    model.Username,
                    ipAddress,
                    loginTrace?.TraceId.ToString()
                );

                return Problem(
                    statusCode: StatusCodes.Status401Unauthorized,
                    title: "Invalid credentials",
                    detail: "Your username or password is incorrect."
                );
            }

            dbActivity?.SetStatus(ActivityStatusCode.Ok);

            // 5. Validate User Status
            bool isPasswordMatched = _cryptoService.Verify(userData.PasswordHash, decryptedPassword);
            var userLoginStatus = GetLoginUserStatus(userData, passwordHash, false, isPasswordMatched);

            if (!userLoginStatus.IsLogin)
            {
                totalStopwatch.Stop();

                // Determine failure reason from message
                string failureReason = userLoginStatus.Message switch
                {
                    "User has been Suspended" => "account_suspended",
                    "User has been Expired" => "account_expired",
                    "User is Not Active" => "account_inactive",
                    "Username or password is invalid" => "invalid_password",
                    _ => "login_validation_failed"
                };

                // Record failed login with specific reason
                LoginTelemetry.RecordFailedLogin(
                    model.Username,
                    failureReason,
                    ipAddress,
                    totalStopwatch.ElapsedMilliseconds
                );

                LoginTelemetry.SetLoginResult(loginTrace, false, failureReason);

                // Track account lockouts separately
                if (failureReason == "account_suspended" || failureReason == "account_expired")
                {
                    LoginTelemetry.RecordAccountLockout(model.Username, failureReason);
                }

                _logger.LogWarning(
                    "Login failed: {Msg} for {Username}. TraceId: {TraceId}",
                    userLoginStatus.Message,
                    model.Username,
                    loginTrace?.TraceId.ToString()
                );

                return Problem(
                    statusCode: StatusCodes.Status401Unauthorized,
                    title: "Login Failure",
                    detail: userLoginStatus.Message
                );
            }

            // 6. Generate Authentication Token
            var timeKeyData = await _repository.SelectSysCurrentTimeKey();
            if (timeKeyData == null)
            {
                _logger.LogError(
                    "Failed to retrieve TimeKey for username: {Username}",
                    model.Username
                );
                return Problem(
                    statusCode: StatusCodes.Status500InternalServerError,
                    title: "Server Error",
                    detail: "Failed to retrieve system time key"
                );
            }

            // Start token generation trace
            using var tokenActivity = ApplicationTelemetry.StartActivity("TokenGeneration", ActivityKind.Internal);
            var tokenStopwatch = Stopwatch.StartNew();

            var authResponse = await _userService.AuthenticateFromDB(
                model,
                ipAddress,
                userData,
                timeKeyData.TimeKey.ToString(),
                timeKeyData.EffectiveToTimeKey.ToString()
            );

            tokenStopwatch.Stop();

            // Record token generation metrics
            LoginTelemetry.RecordTokenGeneration(
                tokenStopwatch.ElapsedMilliseconds,
                success: authResponse != null && !string.IsNullOrEmpty(authResponse.JwtToken)
            );

            tokenActivity?.SetTag("token.type", "jwt");
            tokenActivity?.SetStatus(ActivityStatusCode.Ok);

            // 7. Update Session and Return Success
            await _repository.UpdateRefreshToken(
                authResponse.JwtToken,
                authResponse.RefreshToken,
                DateTime.Now.AddMinutes(_SessionTimeout),
                authResponse.Username,
                "set"
            );

            SetTokenCookie(authResponse.RefreshToken);

            // Stop timer and record success
            totalStopwatch.Stop();

            LoginTelemetry.RecordSuccessfulLogin(
                model.Username,
                "password",
                totalStopwatch.ElapsedMilliseconds
            );

            LoginTelemetry.SetLoginResult(loginTrace, true);

            _logger.LogInformation(
                "User logged in successfully: {Username}. Duration: {Duration}ms, TraceId: {TraceId}",
                model.Username,
                totalStopwatch.ElapsedMilliseconds,
                loginTrace?.TraceId.ToString()
            );

            return Ok(new ApiResponse<AuthenticateResponse>(authResponse, userLoginStatus.Message));
        }
        catch (Exception ex)
        {
            totalStopwatch.Stop();

            // Record exception
            ApplicationTelemetry.RecordException(ex);
            LoginTelemetry.SetLoginResult(loginTrace, false, "internal_error");

            LoginTelemetry.RecordFailedLogin(
                model.Username,
                "internal_error",
                ipAddress,
                totalStopwatch.ElapsedMilliseconds
            );

            _logger.LogError(
                ex,
                "Unexpected error during login for {Username}. TraceId: {TraceId}",
                model.Username,
                loginTrace?.TraceId.ToString()
            );

            throw; // Let global exception handler deal with response
        }
    }
}
```

---

### What Gets Tracked

#### 1. Metrics (Prometheus/Mimir)

**HTTP Metrics (Automatic from Middleware):**
```promql
# Request count
api_requests_count_total{
  http_method="POST",
  http_route="/api/Login/CrisMAc/SelectLoginDetails",
  http_status_code="200"
}

# Request duration
api_requests_duration_milliseconds{
  http_method="POST",
  http_route="/api/Login/CrisMAc/SelectLoginDetails",
  http_status_code="200"
}

# Error count (if failed)
api_errors_total{
  http_method="POST",
  http_route="/api/Login/CrisMAc/SelectLoginDetails",
  http_status_code="401",
  error_type="client_error"
}
```

**Login-Specific Metrics:**
```promql
# Login attempts
ews_login_attempts_total{status="success", authentication_method="password"}
ews_login_attempts_total{status="failed", failure_reason="invalid_credentials"}

# Login duration
ews_login_duration_milliseconds{operation="total_flow", status="success"}
ews_login_duration_milliseconds{operation="database_query", db_operation="SelectLoginDetails"}
ews_login_duration_milliseconds{operation="token_generation", status="success"}

# Token operations
ews_token_operations_total{operation="generation", status="success"}

# Account lockouts
ews_account_lockouts_total{username="john.doe", reason="account_suspended"}
```

---

#### 2. Traces (Tempo)

**Span Hierarchy:**
```
HTTP POST /api/Login/CrisMAc/SelectLoginDetails (234ms)
├─ Login (234ms) [loginTrace]
   │  Tags:
   │    login.username_hash: a1b2c3d4
   │    login.ip_address: 192.168.1.45
   │    login.timestamp: 2025-12-18T10:30:45Z
   │    login.success: true
   │
   ├─ DB_ValidateUser (162ms) [dbActivity]
   │  │  Tags:
   │  │    db.system: mssql
   │  │    db.operation: SELECT
   │  │    db.success: true
   │  │  Status: Ok
   │
   └─ TokenGeneration (15ms) [tokenActivity]
      │  Tags:
      │    token.type: jwt
      │    operation: token_generation
      │  Status: Ok
```

---

#### 3. Logs (Loki)

**Successful Login:**
```json
{
  "timestamp": "2025-12-18T10:30:45.123Z",
  "level": "Information",
  "message": "User logged in successfully: john.doe. Duration: 234ms, TraceId: 0af7651916cd43dd8448eb211c80319c",
  "traceId": "0af7651916cd43dd8448eb211c80319c",
  "spanId": "9d8e7f6a5b4c3d2e",
  "properties": {
    "Username": "john.doe",
    "Duration": 234,
    "service.name": "CrismacEWSBackendService"
  }
}
```

**Failed Login:**
```json
{
  "timestamp": "2025-12-18T10:35:12.456Z",
  "level": "Warning",
  "message": "Login failed: Invalid credentials for john.doe from 192.168.1.45. TraceId: 1bf8762827de54ee9559fc322d91430d",
  "traceId": "1bf8762827de54ee9559fc322d91430d",
  "spanId": "8c7d6e5f4a3b2c1d",
  "properties": {
    "Username": "john.doe",
    "IpAddress": "192.168.1.45",
    "service.name": "CrismacEWSBackendService"
  }
}
```

---

### Grafana Queries for Login Analytics

#### Login Success Rate

```promql
# Successful logins per minute
sum(rate(ews_login_attempts_total{status="success"}[5m])) * 60

# Failed logins per minute
sum(rate(ews_login_attempts_total{status="failed"}[5m])) * 60

# Success rate percentage
(
  sum(rate(ews_login_attempts_total{status="success"}[5m]))
  /
  sum(rate(ews_login_attempts_total[5m]))
) * 100
```

#### Login Failure Breakdown

```promql
# By failure reason
sum by(failure_reason) (
  rate(ews_login_attempts_total{status="failed"}[5m])
)
```

#### Login Performance

```promql
# p50, p95, p99 latency
histogram_quantile(0.50, rate(ews_login_duration_milliseconds_bucket{operation="total_flow"}[5m]))
histogram_quantile(0.95, rate(ews_login_duration_milliseconds_bucket{operation="total_flow"}[5m]))
histogram_quantile(0.99, rate(ews_login_duration_milliseconds_bucket{operation="total_flow"}[5m]))

# Database query performance
histogram_quantile(0.95, rate(ews_login_duration_milliseconds_bucket{operation="database_query"}[5m]))
```

#### Security Monitoring

```promql
# Detect brute force attempts (5+ failures in 10 min)
sum by(username) (
  increase(ews_login_attempts_total{status="failed"}[10m])
) > 5

# Account lockouts
sum by(reason) (
  increase(ews_account_lockouts_total[1h])
)
```

---

## Database Operation Tracking

### Pattern: Repository Method with Telemetry

**Scenario:** Tracking stored procedure performance

```csharp
public class ReportRepository : IReportRepository
{
    private readonly IDbConnection _connection;
    private readonly ILogger<ReportRepository> _logger;

    public async Task<Report> GetMonthlyReport(int year, int month)
    {
        // Start database operation span
        using var dbActivity = ApplicationTelemetry.StartActivity(
            "DB_GetMonthlyReport",
            ActivityKind.Internal
        );

        // Add detailed tags
        dbActivity?.SetTag("db.system", "mssql");
        dbActivity?.SetTag("db.operation", "EXEC");
        dbActivity?.SetTag("db.stored_procedure", "sp_GetMonthlyReport");
        dbActivity?.SetTag("db.parameter.year", year);
        dbActivity?.SetTag("db.parameter.month", month);

        var stopwatch = Stopwatch.StartNew();

        try
        {
            var parameters = new { Year = year, Month = month };

            var report = await _connection.QuerySingleOrDefaultAsync<Report>(
                "sp_GetMonthlyReport",
                parameters,
                commandType: CommandType.StoredProcedure
            );

            stopwatch.Stop();

            // Record metrics
            ApplicationTelemetry.RecordDatabaseOperation(
                operation: "sp_GetMonthlyReport",
                table: "Reports",
                success: report != null,
                durationMs: stopwatch.ElapsedMilliseconds
            );

            dbActivity?.SetStatus(ActivityStatusCode.Ok);
            dbActivity?.SetTag("db.rows_affected", report != null ? 1 : 0);

            _logger.LogInformation(
                "Monthly report retrieved. Year: {Year}, Month: {Month}, Duration: {DurationMs}ms, TraceId: {TraceId}",
                year,
                month,
                stopwatch.ElapsedMilliseconds,
                dbActivity?.TraceId
            );

            return report;
        }
        catch (SqlException ex)
        {
            stopwatch.Stop();

            ApplicationTelemetry.RecordDatabaseOperation(
                operation: "sp_GetMonthlyReport",
                table: "Reports",
                success: false,
                durationMs: stopwatch.ElapsedMilliseconds
            );

            ApplicationTelemetry.RecordException(ex);
            dbActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            dbActivity?.SetTag("db.error_code", ex.Number);

            _logger.LogError(
                ex,
                "Database error while retrieving monthly report. Year: {Year}, Month: {Month}, TraceId: {TraceId}",
                year,
                month,
                dbActivity?.TraceId
            );

            throw;
        }
    }
}
```

---

### Grafana Queries for Database Performance

```promql
# Top 10 slowest stored procedures
topk(10,
  histogram_quantile(0.95,
    rate(api_database_operations_duration_milliseconds_bucket[5m])
  )
)

# Database error rate
rate(api_database_operations_count{db_success="false"}[5m])

# Most frequently called stored procedures
topk(10, rate(api_database_operations_count[5m]))

# Database operation success rate
(
  sum(rate(api_database_operations_count{db_success="true"}[5m]))
  /
  sum(rate(api_database_operations_count[5m]))
) * 100
```

---

## External API Call Monitoring

### Pattern: HTTP Client with Telemetry

**Scenario:** Calling payment gateway API

```csharp
public class PaymentService : IPaymentService
{
    private readonly IHttpClientFactory _httpClientFactory;
    private readonly ILogger<PaymentService> _logger;

    public async Task<PaymentResponse> ProcessPayment(PaymentRequest request)
    {
        // Start external API call span
        using var apiActivity = ApplicationTelemetry.StartActivity(
            "ExternalAPI_StripePayment",
            ActivityKind.Client // Important: Use Client kind for external calls
        );

        // Add context tags
        apiActivity?.SetTag("external.service", "StripePaymentGateway");
        apiActivity?.SetTag("payment.amount", request.Amount);
        apiActivity?.SetTag("payment.currency", request.Currency);
        apiActivity?.SetTag("payment.method", request.PaymentMethod);

        var stopwatch = Stopwatch.StartNew();
        var client = _httpClientFactory.CreateClient("PaymentGateway");

        try
        {
            var httpRequest = new HttpRequestMessage(HttpMethod.Post, "/v1/charges")
            {
                Content = JsonContent.Create(request)
            };

            // OpenTelemetry automatically propagates trace context headers
            var httpResponse = await client.SendAsync(httpRequest);

            stopwatch.Stop();

            var statusCode = (int)httpResponse.StatusCode;

            // Record external API call metrics
            ApplicationTelemetry.RecordExternalApiCall(
                service: "StripePaymentGateway",
                method: "POST",
                statusCode: statusCode,
                durationMs: stopwatch.ElapsedMilliseconds
            );

            apiActivity?.SetTag("http.status_code", statusCode);
            apiActivity?.SetTag("http.response_content_length", httpResponse.Content.Headers.ContentLength);

            if (!httpResponse.IsSuccessStatusCode)
            {
                var errorContent = await httpResponse.Content.ReadAsStringAsync();
                apiActivity?.SetStatus(ActivityStatusCode.Error, $"HTTP {statusCode}");
                apiActivity?.SetTag("error.response", errorContent);

                _logger.LogWarning(
                    "Payment gateway returned error. StatusCode: {StatusCode}, TraceId: {TraceId}",
                    statusCode,
                    apiActivity?.TraceId
                );

                throw new PaymentException($"Payment failed with status {statusCode}");
            }

            var response = await httpResponse.Content.ReadFromJsonAsync<PaymentResponse>();

            apiActivity?.SetTag("payment.transaction_id", response.TransactionId);
            apiActivity?.SetStatus(ActivityStatusCode.Ok);

            _logger.LogInformation(
                "Payment processed successfully. Amount: {Amount}, TransactionId: {TransactionId}, Duration: {DurationMs}ms, TraceId: {TraceId}",
                request.Amount,
                response.TransactionId,
                stopwatch.ElapsedMilliseconds,
                apiActivity?.TraceId
            );

            return response;
        }
        catch (HttpRequestException ex)
        {
            stopwatch.Stop();

            var statusCode = (int)(ex.StatusCode ?? System.Net.HttpStatusCode.ServiceUnavailable);

            ApplicationTelemetry.RecordExternalApiCall(
                service: "StripePaymentGateway",
                method: "POST",
                statusCode: statusCode,
                durationMs: stopwatch.ElapsedMilliseconds
            );

            ApplicationTelemetry.RecordException(ex);
            apiActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);

            _logger.LogError(
                ex,
                "Payment gateway request failed. TraceId: {TraceId}",
                apiActivity?.TraceId
            );

            throw;
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            ApplicationTelemetry.RecordExternalApiCall(
                service: "StripePaymentGateway",
                method: "POST",
                statusCode: 0, // Unknown
                durationMs: stopwatch.ElapsedMilliseconds
            );

            ApplicationTelemetry.RecordException(ex);
            apiActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);

            _logger.LogError(
                ex,
                "Unexpected error during payment processing. TraceId: {TraceId}",
                apiActivity?.TraceId
            );

            throw;
        }
    }
}
```

---

### Grafana Queries for External API Monitoring

```promql
# External API call rate
sum by(external_service) (
  rate(api_external_calls_count[5m])
)

# External API latency (p95)
histogram_quantile(0.95,
  sum by(external_service, le) (
    rate(api_external_calls_duration_milliseconds_bucket[5m])
  )
)

# External API error rate
sum by(external_service) (
  rate(api_external_calls_count{http_status_code=~"4..|5.."}[5m])
)

# External API availability
(
  sum(rate(api_external_calls_count{http_status_code=~"2.."}[5m]))
  /
  sum(rate(api_external_calls_count[5m]))
) * 100
```

---

## File Upload Monitoring

### Complete File Upload Implementation

```csharp
public class FileUploadController : BaseApiController
{
    private readonly IFileService _fileService;
    private readonly ILogger<FileUploadController> _logger;

    [HttpPost("upload")]
    [RequestSizeLimit(100_000_000)] // 100 MB
    public async Task<IActionResult> UploadFile(IFormFile file)
    {
        // Start file upload trace
        using var uploadActivity = ApplicationTelemetry.StartActivity(
            "FileUpload",
            ActivityKind.Server
        );

        var fileExtension = Path.GetExtension(file.FileName);
        var fileSize = file.Length;

        uploadActivity?.SetTag("file.name", file.FileName);
        uploadActivity?.SetTag("file.extension", fileExtension);
        uploadActivity?.SetTag("file.size_bytes", fileSize);
        uploadActivity?.SetTag("file.content_type", file.ContentType);

        var stopwatch = Stopwatch.StartNew();

        try
        {
            // Validation
            if (file.Length == 0)
            {
                ApplicationTelemetry.MarkAsValidationError(HttpContext);

                _logger.LogWarning(
                    "File upload rejected: Empty file. FileName: {FileName}, TraceId: {TraceId}",
                    file.FileName,
                    uploadActivity?.TraceId
                );

                return Problem(
                    statusCode: StatusCodes.Status400BadRequest,
                    title: "Invalid File",
                    detail: "File is empty"
                );
            }

            if (fileSize > 100_000_000)
            {
                ApplicationTelemetry.MarkAsValidationError(HttpContext);

                FileUploadTelemetry.RecordFailedUpload(
                    fileType: fileExtension,
                    failureReason: "file_too_large",
                    durationMs: stopwatch.ElapsedMilliseconds
                );

                return Problem(
                    statusCode: StatusCodes.Status413PayloadTooLarge,
                    title: "File Too Large",
                    detail: "File size exceeds 100MB limit"
                );
            }

            // Virus scan (if enabled)
            using (var scanActivity = ApplicationTelemetry.StartActivity(
                "VirusScan",
                ActivityKind.Internal
            ))
            {
                var scanStopwatch = Stopwatch.StartNew();
                var scanResult = await _fileService.ScanForVirus(file);
                scanStopwatch.Stop();

                scanActivity?.SetTag("scan.result", scanResult.IsClean);
                scanActivity?.SetTag("scan.duration_ms", scanStopwatch.ElapsedMilliseconds);

                if (!scanResult.IsClean)
                {
                    scanActivity?.SetStatus(ActivityStatusCode.Error, "Virus detected");

                    FileUploadTelemetry.RecordFailedUpload(
                        fileType: fileExtension,
                        failureReason: "virus_detected",
                        durationMs: stopwatch.ElapsedMilliseconds
                    );

                    _logger.LogWarning(
                        "File upload rejected: Virus detected. FileName: {FileName}, TraceId: {TraceId}",
                        file.FileName,
                        uploadActivity?.TraceId
                    );

                    return Problem(
                        statusCode: StatusCodes.Status400BadRequest,
                        title: "Security Threat Detected",
                        detail: "File contains malicious content"
                    );
                }

                scanActivity?.SetStatus(ActivityStatusCode.Ok);
            }

            // Save file
            using (var saveActivity = ApplicationTelemetry.StartActivity(
                "SaveFile",
                ActivityKind.Internal
            ))
            {
                var saveStopwatch = Stopwatch.StartNew();
                var savedPath = await _fileService.SaveFile(file);
                saveStopwatch.Stop();

                saveActivity?.SetTag("file.path", savedPath);
                saveActivity?.SetTag("save.duration_ms", saveStopwatch.ElapsedMilliseconds);
                saveActivity?.SetStatus(ActivityStatusCode.Ok);

                uploadActivity?.SetTag("file.saved_path", savedPath);
            }

            stopwatch.Stop();

            // Record successful upload
            FileUploadTelemetry.RecordSuccessfulUpload(
                fileType: fileExtension,
                fileSizeBytes: fileSize,
                durationMs: stopwatch.ElapsedMilliseconds
            );

            uploadActivity?.SetStatus(ActivityStatusCode.Ok);

            _logger.LogInformation(
                "File uploaded successfully. FileName: {FileName}, Size: {Size} bytes, Duration: {DurationMs}ms, TraceId: {TraceId}",
                file.FileName,
                fileSize,
                stopwatch.ElapsedMilliseconds,
                uploadActivity?.TraceId
            );

            return Ok(new
            {
                message = "File uploaded successfully",
                fileName = file.FileName,
                size = fileSize,
                durationMs = stopwatch.ElapsedMilliseconds
            });
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            FileUploadTelemetry.RecordFailedUpload(
                fileType: fileExtension,
                failureReason: ex.GetType().Name,
                durationMs: stopwatch.ElapsedMilliseconds
            );

            ApplicationTelemetry.RecordException(ex);
            uploadActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);

            _logger.LogError(
                ex,
                "File upload failed. FileName: {FileName}, TraceId: {TraceId}",
                file.FileName,
                uploadActivity?.TraceId
            );

            throw;
        }
    }
}
```

---

### FileUploadTelemetry.cs

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

---

## Report Generation Tracking

### Complete Report Generation Implementation

```csharp
public class ReportController : BaseApiController
{
    private readonly IReportService _reportService;
    private readonly ILogger<ReportController> _logger;

    [HttpPost("generate")]
    public async Task<IActionResult> GenerateReport([FromBody] ReportRequest request)
    {
        // Start report generation trace
        using var reportActivity = ApplicationTelemetry.StartActivity(
            "GenerateReport",
            ActivityKind.Server
        );

        reportActivity?.SetTag("report.type", request.ReportType);
        reportActivity?.SetTag("report.start_date", request.StartDate);
        reportActivity?.SetTag("report.end_date", request.EndDate);
        reportActivity?.SetTag("report.format", request.Format);

        var stopwatch = Stopwatch.StartNew();

        try
        {
            // 1. Fetch data
            using (var fetchActivity = ApplicationTelemetry.StartActivity(
                "FetchReportData",
                ActivityKind.Internal
            ))
            {
                var fetchStopwatch = Stopwatch.StartNew();
                var data = await _reportService.FetchData(request);
                fetchStopwatch.Stop();

                fetchActivity?.SetTag("data.record_count", data.Count);
                fetchActivity?.SetTag("fetch.duration_ms", fetchStopwatch.ElapsedMilliseconds);
                fetchActivity?.SetStatus(ActivityStatusCode.Ok);

                reportActivity?.SetTag("report.record_count", data.Count);

                ReportTelemetry.RecordDataFetch(
                    reportType: request.ReportType,
                    recordCount: data.Count,
                    durationMs: fetchStopwatch.ElapsedMilliseconds
                );
            }

            // 2. Process data
            using (var processActivity = ApplicationTelemetry.StartActivity(
                "ProcessReportData",
                ActivityKind.Internal
            ))
            {
                var processStopwatch = Stopwatch.StartNew();
                var processedData = await _reportService.ProcessData(data);
                processStopwatch.Stop();

                processActivity?.SetTag("process.duration_ms", processStopwatch.ElapsedMilliseconds);
                processActivity?.SetStatus(ActivityStatusCode.Ok);
            }

            // 3. Generate output
            using (var generateActivity = ApplicationTelemetry.StartActivity(
                "GenerateReportOutput",
                ActivityKind.Internal
            ))
            {
                var generateStopwatch = Stopwatch.StartNew();
                var reportBytes = await _reportService.GenerateOutput(processedData, request.Format);
                generateStopwatch.Stop();

                generateActivity?.SetTag("output.size_bytes", reportBytes.Length);
                generateActivity?.SetTag("generate.duration_ms", generateStopwatch.ElapsedMilliseconds);
                generateActivity?.SetStatus(ActivityStatusCode.Ok);

                reportActivity?.SetTag("report.size_bytes", reportBytes.Length);
            }

            stopwatch.Stop();

            // Record successful generation
            ReportTelemetry.RecordSuccessfulGeneration(
                reportType: request.ReportType,
                format: request.Format,
                sizeBytes: reportBytes.Length,
                durationMs: stopwatch.ElapsedMilliseconds
            );

            reportActivity?.SetStatus(ActivityStatusCode.Ok);

            _logger.LogInformation(
                "Report generated successfully. Type: {ReportType}, Format: {Format}, Size: {Size} bytes, Duration: {DurationMs}ms, TraceId: {TraceId}",
                request.ReportType,
                request.Format,
                reportBytes.Length,
                stopwatch.ElapsedMilliseconds,
                reportActivity?.TraceId
            );

            return File(reportBytes, GetContentType(request.Format), $"report.{request.Format}");
        }
        catch (Exception ex)
        {
            stopwatch.Stop();

            ReportTelemetry.RecordFailedGeneration(
                reportType: request.ReportType,
                failureReason: ex.GetType().Name,
                durationMs: stopwatch.ElapsedMilliseconds
            );

            ApplicationTelemetry.RecordException(ex);
            reportActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);

            _logger.LogError(
                ex,
                "Report generation failed. Type: {ReportType}, TraceId: {TraceId}",
                request.ReportType,
                reportActivity?.TraceId
            );

            throw;
        }
    }
}
```

---

## Background Job Processing

### Pattern: Long-Running Background Jobs

```csharp
public class DataExtractionJob : BackgroundService
{
    private readonly IServiceProvider _serviceProvider;
    private readonly ILogger<DataExtractionJob> _logger;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // Start job trace
            using var jobActivity = ApplicationTelemetry.StartActivity(
                "DataExtractionJob",
                ActivityKind.Internal
            );

            jobActivity?.SetTag("job.type", "scheduled_extraction");
            jobActivity?.SetTag("job.start_time", DateTime.UtcNow);

            var stopwatch = Stopwatch.StartNew();

            try
            {
                using var scope = _serviceProvider.CreateScope();
                var extractionService = scope.ServiceProvider.GetRequiredService<IExtractionService>();

                // Fetch pending extractions
                var pendingExtractions = await extractionService.GetPendingExtractions();

                jobActivity?.SetTag("job.pending_count", pendingExtractions.Count);

                int successCount = 0;
                int failureCount = 0;

                foreach (var extraction in pendingExtractions)
                {
                    using var extractionActivity = ApplicationTelemetry.StartActivity(
                        "ProcessExtraction",
                        ActivityKind.Internal
                    );

                    extractionActivity?.SetTag("extraction.id", extraction.Id);
                    extractionActivity?.SetTag("extraction.type", extraction.Type);

                    var extractionStopwatch = Stopwatch.StartNew();

                    try
                    {
                        await extractionService.ProcessExtraction(extraction);
                        extractionStopwatch.Stop();

                        successCount++;
                        extractionActivity?.SetStatus(ActivityStatusCode.Ok);

                        ExtractionTelemetry.RecordSuccessfulExtraction(
                            extractionType: extraction.Type,
                            recordCount: extraction.RecordCount,
                            durationMs: extractionStopwatch.ElapsedMilliseconds
                        );
                    }
                    catch (Exception ex)
                    {
                        extractionStopwatch.Stop();

                        failureCount++;
                        ApplicationTelemetry.RecordException(ex);
                        extractionActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);

                        ExtractionTelemetry.RecordFailedExtraction(
                            extractionType: extraction.Type,
                            failureReason: ex.GetType().Name,
                            durationMs: extractionStopwatch.ElapsedMilliseconds
                        );

                        _logger.LogError(
                            ex,
                            "Extraction failed. Id: {ExtractionId}, TraceId: {TraceId}",
                            extraction.Id,
                            extractionActivity?.TraceId
                        );
                    }
                }

                stopwatch.Stop();

                jobActivity?.SetTag("job.success_count", successCount);
                jobActivity?.SetTag("job.failure_count", failureCount);
                jobActivity?.SetTag("job.duration_ms", stopwatch.ElapsedMilliseconds);
                jobActivity?.SetStatus(ActivityStatusCode.Ok);

                _logger.LogInformation(
                    "Data extraction job completed. Success: {SuccessCount}, Failed: {FailureCount}, Duration: {DurationMs}ms, TraceId: {TraceId}",
                    successCount,
                    failureCount,
                    stopwatch.ElapsedMilliseconds,
                    jobActivity?.TraceId
                );

                // Wait before next run
                await Task.Delay(TimeSpan.FromMinutes(15), stoppingToken);
            }
            catch (Exception ex)
            {
                stopwatch.Stop();

                ApplicationTelemetry.RecordException(ex);
                jobActivity?.SetStatus(ActivityStatusCode.Error, ex.Message);

                _logger.LogError(
                    ex,
                    "Data extraction job failed. TraceId: {TraceId}",
                    jobActivity?.TraceId
                );

                // Wait before retry
                await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken);
            }
        }
    }
}
```

---

## Before/After Comparisons

### Example 1: Simple Controller Without Telemetry

**Before (No Observability):**

```csharp
[HttpGet("products/{id}")]
public async Task<IActionResult> GetProduct(int id)
{
    var product = await _productService.GetProduct(id);

    if (product == null)
    {
        return NotFound();
    }

    return Ok(product);
}
```

**After (With Automatic Middleware):**

```csharp
[HttpGet("products/{id}")]
public async Task<IActionResult> GetProduct(int id)
{
    // No changes needed! Middleware automatically tracks:
    // - Request count
    // - Request duration
    // - HTTP status code
    // - Error rate

    var product = await _productService.GetProduct(id);

    if (product == null)
    {
        return NotFound();
    }

    return Ok(product);
}
```

**Result:**
- 0 lines of code added
- Full HTTP metrics available
- Traces automatically created
- Logs automatically correlated

---

### Example 2: Business Operation Without Telemetry

**Before (No Observability):**

```csharp
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
{
    try
    {
        var order = await _orderService.CreateOrder(request);
        return CreatedAtAction(nameof(GetOrder), new { id = order.Id }, order);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Order creation failed");
        throw;
    }
}
```

**After (With Business Telemetry):**

```csharp
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder([FromBody] CreateOrderRequest request)
{
    using var orderTrace = ApplicationTelemetry.StartActivity("CreateOrder");
    orderTrace?.SetTag("order.customer_id", request.CustomerId);

    var stopwatch = Stopwatch.StartNew();

    try
    {
        var order = await _orderService.CreateOrder(request);

        stopwatch.Stop();

        ApplicationTelemetry.RecordBusinessOperation(
            operationType: "order_creation",
            status: "success",
            durationMs: stopwatch.ElapsedMilliseconds
        );

        orderTrace?.SetTag("order.id", order.Id);
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

        throw;
    }
}
```

**Result:**
- Business metrics for dashboards
- Distributed trace with tags
- Correlated logs with TraceId
- Exception tracking in spans

---

## Summary

### Key Patterns

1. **Automatic instrumentation** handles 90% of HTTP telemetry
2. **Manual instrumentation** adds business context
3. **Consistent structure** across all operations
4. **Always include TraceId** in logs
5. **Set span status** for success/failure
6. **Use tags** for context, not high-cardinality data

---

### Next Steps

1. **Review** your existing controllers
2. **Identify** critical business operations
3. **Add** domain-specific telemetry classes
4. **Test** with local LGTM stack
5. **Create** dashboards in Grafana
6. **Monitor** and iterate

---

**Remember:** Start simple, iterate based on actual debugging needs!
