# Challenges & Scenarios: Advanced Topics

## Real-World Challenges and Solutions

**Audience:** Engineers, architects, operations teams
**Reading Time:** 60-90 minutes
**Prerequisites:** Understanding of [Development Guide](./02-development-guide.md) and [Theory](./01-theory-and-concepts.md)

---

## Table of Contents

1. [High-Cardinality Challenges](#high-cardinality-challenges)
2. [Performance Overhead Management](#performance-overhead-management)
3. [Security and Privacy Concerns](#security-and-privacy-concerns)
4. [Data Retention Strategies](#data-retention-strategies)
5. [Cost Optimization](#cost-optimization)
6. [Distributed System Challenges](#distributed-system-challenges)
7. [Real-World Incident Examples](#real-world-incident-examples)
8. [Edge Cases and Solutions](#edge-cases-and-solutions)

---

## High-Cardinality Challenges

### What is High Cardinality?

**Definition:** Tags/labels with many unique values.

**Example:**

```
Low Cardinality (Good):
http_status_code: [200, 400, 401, 404, 500] → 5 unique values

High Cardinality (Bad):
user_id: [1, 2, 3, ..., 1000000] → 1M unique values
```

---

### The Problem

**Metric with High-Cardinality Tag:**

```csharp
// BAD: Creates 1 million time series
var tags = new KeyValuePair<string, object?>[]
{
    new("user_id", userId)  // 1M unique users
};
RequestCounter.Add(1, tags);
```

**Result:**
```
Storage: 1M time series × 1 year × 8 bytes = 240 GB
Query time: 30 seconds (vs 0.5 seconds for low cardinality)
Memory: 16 GB (vs 200 MB)
```

**Database Impact:**
- Prometheus crashes with "too many series"
- Queries timeout
- Out of memory errors
- Slow dashboard loading

---

### Real-World Example: User Activity Tracking

**Bad Implementation:**

```csharp
public class BadUserActivityTelemetry
{
    public static void RecordActivity(string userId, string action)
    {
        var tags = new KeyValuePair<string, object?>[]
        {
            new("user_id", userId),        // ❌ HIGH CARDINALITY
            new("action", action),
            new("timestamp", DateTime.Now)  // ❌ HIGH CARDINALITY
        };

        ActivityCounter.Add(1, tags);
    }
}

// With 100K users × 10 actions × 365 days
// = 365 million time series 💥
```

**Good Implementation:**

```csharp
public class GoodUserActivityTelemetry
{
    public static void RecordActivity(string userType, string action)
    {
        var tags = new KeyValuePair<string, object?>[]
        {
            new("user_type", userType),     // ✓ LOW CARDINALITY (free, premium, enterprise)
            new("action", action)           // ✓ LOW CARDINALITY (login, logout, purchase)
        };

        ActivityCounter.Add(1, tags);

        // For user-specific tracking, use logs or traces
        _logger.LogInformation(
            "User activity. UserId: {UserId}, Action: {Action}",
            userId,
            action
        );
    }
}

// With 3 user types × 10 actions
// = 30 time series ✓
```

---

### Solutions to High Cardinality

#### Solution 1: Hash or Aggregate

```csharp
// Instead of raw user ID
activity?.SetTag("user_id", userId);  // ❌

// Use hash
activity?.SetTag("user_id_hash", HashUserId(userId));  // ✓

// Or categorize
activity?.SetTag("user_type", GetUserType(userId));  // ✓

private static string HashUserId(string userId)
{
    using var sha256 = SHA256.Create();
    var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(userId));
    return Convert.ToHexString(hash)[..8]; // First 8 chars
}
```

#### Solution 2: Move to Logs

```csharp
// High-cardinality data belongs in logs, not metrics
_logger.LogInformation(
    "User {UserId} performed {Action} on {Resource}",
    userId,      // High cardinality OK in logs
    action,
    resourceId   // High cardinality OK in logs
);

// Use low-cardinality metrics for aggregation
UserActivityCounter.Add(1, new KeyValuePair<string, object?>[]
{
    new("action_type", "resource_access")  // Low cardinality
});
```

#### Solution 3: Use Traces for Details

```csharp
// Traces can have high-cardinality tags (they're sampled)
using var activity = ApplicationTelemetry.StartActivity("UserAction");
activity?.SetTag("user.id", userId);            // OK in traces
activity?.SetTag("resource.id", resourceId);    // OK in traces
activity?.SetTag("action.type", "view");
```

---

### Cardinality Budgeting

**Calculate Before You Tag:**

```
Formula:
Cardinality = Tag1_Values × Tag2_Values × Tag3_Values × ...

Example:
http_method: 5 values (GET, POST, PUT, DELETE, PATCH)
http_route: 20 values (endpoints)
http_status_code: 15 values
environment: 3 values (dev, staging, prod)

Total = 5 × 20 × 15 × 3 = 4,500 time series ✓ (Acceptable)

If you add user_id (100K users):
Total = 5 × 20 × 15 × 3 × 100,000 = 450M time series ❌ (Unacceptable)
```

**Guidelines:**

| Cardinality | Status | Action |
|-------------|--------|--------|
| < 1,000 | ✓ Safe | No problem |
| 1,000 - 10,000 | ⚠️ Caution | Monitor closely |
| 10,000 - 100,000 | ⚠️ High | Consider reducing |
| > 100,000 | ❌ Dangerous | Redesign immediately |

---

## Performance Overhead Management

### Measuring Overhead

**Baseline Benchmark:**

```bash
# Without telemetry
ab -n 10000 -c 100 http://localhost:5294/api/test

# Results:
# Requests per second: 2,500
# Mean response time: 40ms
# 95th percentile: 65ms
```

**With Telemetry:**

```bash
# 100% sampling
ab -n 10000 -c 100 http://localhost:5294/api/test

# Results:
# Requests per second: 2,450 (2% slower)
# Mean response time: 41ms (2.5% slower)
# 95th percentile: 68ms (4.6% slower)
```

**Acceptable Overhead:** < 5% for most applications

---

### Optimization Strategies

#### Strategy 1: Adaptive Sampling

```csharp
public class AdaptiveSampler : Sampler
{
    private readonly Sampler _highTrafficSampler = new TraceIdRatioBasedSampler(0.01); // 1%
    private readonly Sampler _lowTrafficSampler = new TraceIdRatioBasedSampler(0.5);   // 50%
    private readonly IHttpContextAccessor _httpContextAccessor;

    public override SamplingResult ShouldSample(in SamplingParameters samplingParameters)
    {
        // Sample more during low traffic
        if (IsLowTrafficPeriod())
        {
            return _lowTrafficSampler.ShouldSample(samplingParameters);
        }

        // Sample less during high traffic
        return _highTrafficSampler.ShouldSample(samplingParameters);
    }

    private bool IsLowTrafficPeriod()
    {
        var hour = DateTime.Now.Hour;
        return hour < 6 || hour > 22; // Night hours
    }
}

// Register:
.WithTracing(tracing => tracing
    .SetSampler(new AdaptiveSampler())
)
```

#### Strategy 2: Selective Instrumentation

```csharp
// Only instrument critical paths
.AddAspNetCoreInstrumentation(options =>
{
    options.Filter = context =>
    {
        var path = context.Request.Path.Value;

        // Always instrument critical operations
        if (path.Contains("login") ||
            path.Contains("payment") ||
            path.Contains("order"))
        {
            return true;
        }

        // Skip low-value operations
        if (path.Contains("health") ||
            path.Contains("ping") ||
            path.Contains("static"))
        {
            return false;
        }

        // Sample others at 10%
        return Random.Shared.Next(100) < 10;
    };
})
```

#### Strategy 3: Async Export

```csharp
// Ensure exports don't block requests
.AddOtlpExporter(options =>
{
    options.Protocol = OtlpExportProtocol.Grpc;  // Faster than HTTP

    options.ExportProcessorType = ExportProcessorType.Batch;
    options.BatchExportProcessorOptions = new()
    {
        MaxQueueSize = 2048,
        ScheduledDelayMilliseconds = 5000,  // Export every 5 seconds
        ExporterTimeoutMilliseconds = 30000,
        MaxExportBatchSize = 512
    };
})
```

#### Strategy 4: Efficient Serialization

```csharp
// Use compact formatters for logs
Log.Logger = new LoggerConfiguration()
    .WriteTo.File(
        new CompactJsonFormatter(),  // More efficient than default
        path: "logs/app-log-.json"
    )
    .CreateLogger();
```

---

### Performance Testing Checklist

Before deploying to production:

- [ ] Load test with telemetry enabled
- [ ] Measure baseline vs instrumented performance
- [ ] Verify overhead < 5%
- [ ] Test with production-like traffic patterns
- [ ] Check memory usage over 24 hours
- [ ] Monitor collector resource usage
- [ ] Verify export backlog doesn't grow
- [ ] Test failover scenarios

---

## Security and Privacy Concerns

### Challenge: Sensitive Data in Telemetry

**Problem:**

```csharp
// ❌ SECURITY RISK: Logging password
_logger.LogInformation("Login attempt: {Username} / {Password}", username, password);

// ❌ SECURITY RISK: Credit card in trace
activity?.SetTag("payment.card_number", cardNumber);

// ❌ SECURITY RISK: PII in metric
var tags = new KeyValuePair<string, object?>[]
{
    new("user.email", email),  // PII
    new("user.ssn", ssn)       // HIGHLY SENSITIVE
};
```

**Impact:**
- GDPR violations
- PCI DSS non-compliance
- Data breach risk
- Legal liability

---

### Solution 1: Data Scrubbing

**Automated Scrubber:**

```csharp
public class SensitiveDataProcessor : BaseProcessor<Activity>
{
    private static readonly string[] SensitiveKeys = new[]
    {
        "password", "pwd", "passwd",
        "secret", "token", "api_key",
        "credit_card", "card_number", "cvv",
        "ssn", "social_security",
        "email", "phone", "address"
    };

    public override void OnEnd(Activity activity)
    {
        foreach (var tag in activity.Tags.ToList())
        {
            if (IsSensitive(tag.Key))
            {
                activity.SetTag(tag.Key, "[REDACTED]");
            }
        }
    }

    private bool IsSensitive(string key)
    {
        return SensitiveKeys.Any(k =>
            key.Contains(k, StringComparison.OrdinalIgnoreCase));
    }
}

// Register:
.WithTracing(tracing => tracing
    .AddProcessor(new SensitiveDataProcessor())
)
```

**Log Scrubber:**

```csharp
public class SensitiveDataEnricher : ILogEventEnricher
{
    public void Enrich(LogEvent logEvent, ILogEventPropertyFactory propertyFactory)
    {
        foreach (var property in logEvent.Properties.ToList())
        {
            if (IsSensitiveProperty(property.Key))
            {
                logEvent.RemovePropertyIfPresent(property.Key);
                logEvent.AddPropertyIfAbsent(
                    propertyFactory.CreateProperty(property.Key, "[REDACTED]")
                );
            }
        }
    }
}
```

---

### Solution 2: Hashing

```csharp
public static class SecureLogging
{
    public static string HashSensitiveData(string data)
    {
        using var sha256 = SHA256.Create();
        var hash = sha256.ComputeHash(Encoding.UTF8.GetBytes(data));
        return Convert.ToHexString(hash)[..16]; // First 16 chars for readability
    }

    public static void LogSecurely(ILogger logger, string username, string email)
    {
        logger.LogInformation(
            "User login. UsernameHash: {UsernameHash}, EmailHash: {EmailHash}",
            HashSensitiveData(username),
            HashSensitiveData(email)
        );
    }
}

// Usage:
SecureLogging.LogSecurely(_logger, username, email);
```

---

### Solution 3: Encryption at Rest

```yaml
# Encrypt sensitive logs before storage
# In Loki config:
storage_config:
  aws:
    s3: s3://bucket/loki
    sse_config:
      type: 'aws:kms'
      kms_key_id: 'arn:aws:kms:...'
```

---

### Solution 4: Access Control

**Grafana RBAC:**

```yaml
# Only security team can view sensitive logs
apiVersion: 1
roles:
  - name: SecurityAnalyst
    permissions:
      - action: "datasources:read"
        scope: "datasources:name:Loki"
      - action: "datasources:query"
        scope: "datasources:name:Loki"
    users:
      - security@company.com

  - name: Developer
    permissions:
      - action: "datasources:read"
        scope: "datasources:name:Mimir"  # Metrics only, no logs
```

---

### Compliance Checklist

**GDPR:**
- [ ] No PII in metrics
- [ ] Logs encrypted at rest
- [ ] Data retention policies configured
- [ ] User data deletion process
- [ ] Consent tracking for analytics

**PCI DSS:**
- [ ] No credit card numbers logged
- [ ] No CVV codes in telemetry
- [ ] Encrypted transmission (TLS)
- [ ] Access logs audited
- [ ] Quarterly security reviews

**HIPAA:**
- [ ] No PHI in telemetry
- [ ] Audit trails for data access
- [ ] Encryption in transit and at rest
- [ ] Business associate agreements (BAAs)

---

## Data Retention Strategies

### Challenge: Balancing Cost vs. Value

**Problem:**

```
Day 1: 10 GB data/day
Day 30: 300 GB data
Day 365: 3.6 TB data

Storage cost: $0.10/GB/month
Monthly cost: $360 (growing)
Annual cost: $4,320+
```

**Question:** Do you need 1-year-old logs?

---

### Retention Strategy by Signal Type

#### Metrics Retention

**Recommendation:**

| Granularity | Retention | Storage | Use Case |
|-------------|-----------|---------|----------|
| **Raw (15s)** | 7 days | 20 GB | Recent debugging |
| **5min avg** | 30 days | 10 GB | Monthly reports |
| **1hour avg** | 90 days | 5 GB | Quarterly trends |
| **1day avg** | 2 years | 2 GB | Annual analysis |

**Implementation:**

```yaml
# In Mimir/Prometheus config:
storage:
  tsdb:
    retention_time: 7d  # Raw data

# Recording rules for aggregation:
groups:
  - name: downsampling
    interval: 5m
    rules:
      # 5-minute averages
      - record: job:api_requests:rate5m
        expr: rate(api_requests_count_total[5m])

      # 1-hour averages
      - record: job:api_requests:rate1h
        expr: avg_over_time(job:api_requests:rate5m[1h])
```

---

#### Logs Retention

**Recommendation:**

| Type | Retention | Reasoning |
|------|-----------|-----------|
| **Error logs** | 90 days | Debugging recent issues |
| **Warning logs** | 30 days | Monitor trends |
| **Info logs** | 7 days | Recent operations only |
| **Debug logs** | 1 day | Troubleshooting only |

**Implementation:**

```yaml
# Loki config:
limits_config:
  retention_period: 30d

table_manager:
  retention_deletes_enabled: true
  retention_period: 720h  # 30 days
```

**Query-Time Filtering:**

```logql
# Only index errors for fast searching
{service_name="EWS"} | json | level="Error"
```

---

#### Traces Retention

**Recommendation:**

| Environment | Retention | Sampling | Storage |
|-------------|-----------|----------|---------|
| **Production** | 7 days | 1% | 50 GB |
| **Staging** | 3 days | 10% | 20 GB |
| **Development** | 1 day | 100% | 10 GB |

**Implementation:**

```yaml
# Tempo config:
compactor:
  compaction:
    block_retention: 168h  # 7 days

# Different retention per environment
limits:
  max_bytes_per_trace: 5000000
  ingestion_rate_limit_bytes: 15000000
```

---

### Hot/Warm/Cold Architecture

**Strategy:** Move old data to cheaper storage.

```
┌─────────────────────────────────────────────┐
│           Hot Storage (SSD)                 │
│         Last 7 days - $0.20/GB              │
│         Fast queries (< 1s)                 │
└─────────────────────────────────────────────┘
                    ↓ Move after 7 days
┌─────────────────────────────────────────────┐
│          Warm Storage (HDD)                 │
│        8-30 days - $0.10/GB                 │
│        Slower queries (5-10s)               │
└─────────────────────────────────────────────┘
                    ↓ Move after 30 days
┌─────────────────────────────────────────────┐
│          Cold Storage (S3)                  │
│        31-365 days - $0.02/GB               │
│        Very slow queries (30s-2min)         │
│        Compress and archive                 │
└─────────────────────────────────────────────┘
                    ↓ Delete after 1 year
                   Deleted
```

**Implementation:**

```yaml
# Loki with S3 tiered storage:
storage_config:
  aws:
    s3: s3://hot-bucket/loki
    s3forcepathstyle: true

    # Lifecycle policy on S3 bucket:
    # - Transition to Glacier after 30 days
    # - Delete after 365 days
```

---

### Retention Policy Template

```yaml
# Retention Policy Document

Metrics:
  Production:
    raw: 7d
    5min_avg: 30d
    1hour_avg: 90d
    1day_avg: 2y

Logs:
  Production:
    error: 90d
    warning: 30d
    info: 7d
    debug: 1d

  Development:
    all: 1d

Traces:
  Production: 7d (1% sampled)
  Staging: 3d (10% sampled)
  Development: 1d (100% sampled)

Compliance:
  audit_logs: 7y  # Legal requirement
  security_logs: 1y
  access_logs: 90d
```

---

## Cost Optimization

### Total Cost of Ownership

**Components:**

```
Infrastructure:
  - LGTM containers: $200/month
  - Storage (500 GB): $100/month
  - Network transfer: $50/month
  - Backup: $30/month

Personnel:
  - Initial setup (80 hours × $75): $6,000 (one-time)
  - Maintenance (4 hours/month × $75): $300/month
  - On-call rotation: $500/month

Total Monthly: ~$1,200
Total Annual: ~$20,400

ROI: $261,500 benefit - $20,400 cost = $241,100 net benefit
```

---

### Optimization Strategies

#### Strategy 1: Reduce Storage Costs

**Before:**
```
Metrics: 100 GB/month × $0.20 = $20
Logs: 400 GB/month × $0.20 = $80
Traces: 200 GB/month × $0.20 = $40
Total: $140/month
```

**After Optimization:**

```yaml
# Increase sampling
OTEL_TRACES_SAMPLER_ARG=0.01  # 1% instead of 10%

# Aggressive log filtering
{level!~"Debug|Trace"}

# Shorter retention
retention_period: 7d  # Was 30d

# Compression
compression: gzip  # Save 70% storage
```

**After:**
```
Metrics: 100 GB/month × $0.20 = $20
Logs: 80 GB/month × $0.20 = $16  # 80% reduction
Traces: 20 GB/month × $0.20 = $4  # 90% reduction
Total: $40/month (71% savings)
```

---

#### Strategy 2: Use Reserved Capacity

```
On-Demand VM: $0.10/hour = $720/month
Reserved Instance (1 year): $0.06/hour = $432/month
Savings: 40%

Reserved Instance (3 years): $0.04/hour = $288/month
Savings: 60%
```

---

#### Strategy 3: Spot Instances for Non-Critical

```yaml
# Use spot instances for development/staging LGTM
# 70% cost savings
# Note: Can be interrupted, but OK for dev
```

---

#### Strategy 4: Auto-Scaling

```yaml
# Scale down during off-hours
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: otel-collector
spec:
  minReplicas: 1  # Night
  maxReplicas: 3  # Peak hours
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

---

### Cost Monitoring

**Set Budget Alerts:**

```bash
# AWS CloudWatch Alarm
aws cloudwatch put-metric-alarm \
  --alarm-name observability-budget \
  --alarm-description "Alert when observability costs exceed $500/month" \
  --metric-name EstimatedCharges \
  --threshold 500 \
  --comparison-operator GreaterThanThreshold
```

**Monthly Review:**

```
Month 1: $400 (baseline)
Month 2: $450 (+12.5%) → Investigate
Month 3: $500 (+25%) → Action required

Root cause: Increased trace volume due to new feature
Action: Reduce sampling from 10% to 5%
Result: Month 4: $420 (back to normal)
```

---

## Distributed System Challenges

### Challenge: Distributed Tracing Across Services

**Scenario:** Request flows through multiple services

```
API Gateway → Auth Service → User Service → Database
```

**Without Distributed Tracing:**
```
API Gateway logs: Request received
Auth Service logs: Token validated
User Service logs: User retrieved
Database logs: Query executed

Problem: No way to link these together
```

**With Distributed Tracing:**
```
TraceId: 0af7651916cd43dd8448eb211c80319c

API Gateway (Span 1):
  TraceId: 0af765...
  SpanId: abc123
  Duration: 50ms

Auth Service (Span 2):
  TraceId: 0af765...
  SpanId: def456
  ParentSpanId: abc123
  Duration: 20ms

User Service (Span 3):
  TraceId: 0af765...
  SpanId: ghi789
  ParentSpanId: abc123
  Duration: 15ms

Database (Span 4):
  TraceId: 0af765...
  SpanId: jkl012
  ParentSpanId: ghi789
  Duration: 10ms
```

---

### Solution: Context Propagation

**W3C Trace Context Headers:**

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-abc123def456-01
tracestate: vendor1=value1,vendor2=value2
```

**Automatic Propagation:**

```csharp
// Service A (Sender)
builder.Services.AddHttpClient("ServiceB", client =>
{
    client.BaseAddress = new Uri("https://serviceb.example.com");
});

// OpenTelemetry automatically adds traceparent header

// Service B (Receiver)
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()  // Automatically reads traceparent
    );
```

**Manual Propagation (if needed):**

```csharp
// Extract current context
var propagator = new TraceContextPropagator();
var carrier = new Dictionary<string, string>();
propagator.Inject(new PropagationContext(Activity.Current?.Context ?? default, Baggage.Current), carrier);

// Send with request
var request = new HttpRequestMessage(HttpMethod.Get, "/api/data");
foreach (var header in carrier)
{
    request.Headers.Add(header.Key, header.Value);
}
```

---

### Challenge: Clock Skew Between Services

**Problem:**

```
Service A timestamp: 2025-12-18 10:30:00.000
Service B timestamp: 2025-12-18 10:29:58.500 (1.5 seconds behind)

Result: Child span appears to start BEFORE parent span
```

**Solution: Use Duration, Not Absolute Time**

```csharp
// Record duration, not timestamps
var stopwatch = Stopwatch.StartNew();
await CallServiceB();
stopwatch.Stop();

activity?.SetTag("duration_ms", stopwatch.ElapsedMilliseconds);
```

**Solution: NTP Synchronization**

```bash
# Ensure all servers use same time source
sudo systemctl enable systemd-timesyncd
sudo systemctl start systemd-timesyncd
sudo timedatectl set-ntp true
```

---

## Real-World Incident Examples

### Incident 1: The Mysterious 500 Errors

**Situation:**

```
Time: Monday 10:00 AM
Alert: "Error rate spike: 15% of requests failing with 500"
Impact: 300 customers affected
Revenue at risk: $30,000/hour
```

**Without Observability:**

```
10:00 - Alert fires
10:15 - Team assembles
10:30 - Start checking logs manually
11:00 - Suspect database issue
11:30 - Database team engaged
12:00 - Found slow query
12:30 - Deploy fix
1:00 - Verify fix

Total time: 3 hours
Lost revenue: $90,000
```

**With Observability:**

```
10:00 - Alert fires with dashboard link
10:02 - Team reviews RED metrics dashboard
10:05 - Filter traces to 500 errors
10:07 - Identify slow database query in trace waterfall
10:10 - View correlated logs showing query timeout
10:15 - Add database index
10:20 - Deploy fix
10:25 - Verify metrics returning to normal

Total time: 25 minutes
Lost revenue: $12,500
Savings: $77,500
```

**Root Cause:**

```sql
-- Missing index on Users table
SELECT * FROM Users
WHERE Username = @username
  AND IsActive = 1
  AND LastLoginDate > @date

-- Added index:
CREATE INDEX IX_Users_Username_Active_LastLogin
ON Users(Username, IsActive, LastLoginDate)
```

**Prevention:**

```csharp
// Add database monitoring
using var dbActivity = ApplicationTelemetry.StartActivity("DB_GetUser");
dbActivity?.SetTag("db.query", "GetActiveUser");
dbActivity?.SetTag("db.execution_time_ms", stopwatch.ElapsedMilliseconds);

// Alert on slow queries
if (stopwatch.ElapsedMilliseconds > 100)
{
    _logger.LogWarning(
        "Slow database query. Query: {Query}, Duration: {Duration}ms",
        "GetActiveUser",
        stopwatch.ElapsedMilliseconds
    );
}
```

---

### Incident 2: The Cascading Failure

**Situation:**

```
Time: Friday 3:00 PM
Event: Black Friday sale starts
Traffic: 10x normal load
Result: Entire system goes down
```

**Timeline Without Observability:**

```
3:00 PM - Sale starts, traffic spikes
3:05 PM - Response times increase (unnoticed)
3:10 PM - Connection pool exhausted (unnoticed)
3:15 PM - Database maxed out, rejecting connections
3:20 PM - Frontend starts failing
3:25 PM - Users report errors on social media
3:30 PM - Team notified
4:00 PM - Identify database connection issue
4:30 PM - Emergency scale-up
5:00 PM - System stabilizes

Downtime: 2 hours
Lost sales: $200,000
Brand damage: Significant
```

**Timeline With Observability:**

```
2:55 PM - Pre-sale monitoring shows baseline
3:00 PM - Sale starts
3:02 PM - Alert: "Request duration p95 increasing"
3:03 PM - Dashboard shows database connection pool at 80%
3:04 PM - Alert: "Database connection pool high"
3:05 PM - Auto-scale database connections triggered
3:06 PM - Add read replicas for SELECT queries
3:10 PM - Metrics stabilize
3:15 PM - Normal operation continues

Downtime: 0 minutes
Lost sales: $0
Customer experience: Excellent
```

**Metrics That Saved The Day:**

```promql
# Database connection pool usage
(database_connections_active / database_connections_max) * 100 > 80

# Response time degradation
histogram_quantile(0.95,
  rate(api_requests_duration_milliseconds_bucket[1m])
) > 500
```

---

### Incident 3: The Silent Data Loss

**Situation:**

```
Time: Ongoing for 3 weeks
Issue: Customer orders randomly not saving
Impact: 150 lost orders
Revenue lost: $30,000
Customer complaints: Increasing
```

**Without Observability:**

```
Week 1: Customer complaints start (isolated incidents)
Week 2: Complaints increase (still investigating)
Week 3: Pattern recognized
Week 4: Root cause found (race condition)
Week 5: Fix deployed

Lost time: 5 weeks
Lost revenue: $50,000
Lost customers: 25
```

**With Observability:**

```
Day 1: Order processing trace shows occasional failures
Day 2: Correlated logs reveal pattern:
  - Only happens during high load
  - Only affects orders > $500
  - Database transaction timeout

Root cause: Database lock contention
Fix: Optimize query, add index
Deploy: Same day

Lost time: 2 days
Lost revenue: $6,000
Lost customers: 0
```

**Trace That Revealed The Issue:**

```
HTTP POST /api/orders (5,234ms) ⚠️ SLOW
├─ CreateOrder (5,200ms)
│  ├─ ValidateOrder (50ms) ✓
│  ├─ DB_BeginTransaction (10ms) ✓
│  ├─ DB_CheckInventory (4,800ms) ❌ BOTTLENECK
│  │  └─ Lock wait timeout exceeded
│  └─ DB_RollbackTransaction (340ms)
└─ Error: Transaction timeout
```

**Fix:**

```sql
-- Before: Full table lock
BEGIN TRANSACTION
  SELECT * FROM Inventory WITH (TABLOCKX)
  WHERE ProductId = @productId

-- After: Row-level lock
BEGIN TRANSACTION
  SELECT * FROM Inventory WITH (ROWLOCK)
  WHERE ProductId = @productId
  AND Available > 0
```

---

## Edge Cases and Solutions

### Edge Case 1: Circular Trace References

**Problem:**

```
Service A → Service B → Service A (circular call)
Result: Infinite trace hierarchy
```

**Solution: Detect and Break Loops**

```csharp
public class CircularTraceDetector : ActivityListener
{
    private readonly HashSet<string> _seenTraces = new();

    public override void ActivityStarted(Activity activity)
    {
        var traceId = activity.TraceId.ToString();

        if (_seenTraces.Contains(traceId))
        {
            _logger.LogWarning(
                "Circular trace detected: {TraceId}",
                traceId
            );

            // Stop tracing to prevent infinite loop
            activity.ActivityTraceFlags &= ~ActivityTraceFlags.Recorded;
        }
        else
        {
            _seenTraces.Add(traceId);
        }
    }
}
```

---

### Edge Case 2: Trace Too Large

**Problem:**

```
Trace has 10,000 spans (batch job processing)
Result: Tempo rejects trace (max 1,000 spans)
```

**Solution: Batch Sampling**

```csharp
public async Task ProcessBatch(List<Item> items)
{
    // Create parent trace for batch
    using var batchTrace = ApplicationTelemetry.StartActivity("BatchProcess");
    batchTrace?.SetTag("batch.size", items.Count);

    int spanCount = 0;
    const int MAX_SPANS_PER_TRACE = 100;

    foreach (var item in items)
    {
        // Only create span for first 100 items
        if (spanCount < MAX_SPANS_PER_TRACE)
        {
            using var itemTrace = ApplicationTelemetry.StartActivity("ProcessItem");
            await ProcessItem(item);
            spanCount++;
        }
        else
        {
            // Process without span (still counted in metrics)
            await ProcessItem(item);
        }
    }

    batchTrace?.SetTag("batch.traced_items", spanCount);
}
```

---

### Edge Case 3: Unicode in Log Messages

**Problem:**

```csharp
// Emoji in username breaks log parsing
_logger.LogInformation("User logged in: {Username}", "John🚀Doe");
```

**Solution: Sanitize Input**

```csharp
public static string SanitizeForLogging(string input)
{
    if (string.IsNullOrEmpty(input))
        return input;

    // Remove non-ASCII characters
    return Regex.Replace(input, @"[^\u0000-\u007F]+", "");
}

// Usage:
_logger.LogInformation(
    "User logged in: {Username}",
    SanitizeForLogging(username)
);
```

---

### Edge Case 4: Time Zone Confusion

**Problem:**

```
Application logs: 2025-12-18 10:30:00 (local time)
Grafana shows: 2025-12-18 03:30:00 (UTC)
Developer searches: "10:30" (finds nothing)
```

**Solution: Always Use UTC**

```csharp
// Configure Serilog to use UTC
Log.Logger = new LoggerConfiguration()
    .WriteTo.Console(formatProvider: CultureInfo.InvariantCulture)
    .Enrich.WithProperty("Timestamp", DateTime.UtcNow)
    .CreateLogger();

// Log UTC timestamps
_logger.LogInformation(
    "Event at {Timestamp}",
    DateTime.UtcNow.ToString("o") // ISO 8601 format
);
```

---

## Summary and Best Practices

### Key Takeaways

1. **High Cardinality is Your Enemy**
   - Use low-cardinality tags in metrics
   - Move high-cardinality data to logs/traces
   - Monitor cardinality regularly

2. **Performance Overhead Can Be Managed**
   - Use sampling in production
   - Batch exports
   - Measure overhead before deploying

3. **Security Must Be Built-In**
   - Never log passwords or PII
   - Encrypt sensitive data
   - Implement access controls

4. **Retention is a Balance**
   - Hot/warm/cold architecture
   - Different retention by signal type
   - Regular cleanup processes

5. **Cost Can Be Optimized**
   - Aggressive sampling
   - Smart retention policies
   - Right-size infrastructure

6. **Distributed Tracing Requires Coordination**
   - Propagate context headers
   - Synchronize clocks
   - Handle edge cases

7. **Real Incidents Prove Value**
   - Faster incident resolution
   - Prevent cascading failures
   - Detect silent issues early

---

### Recommended Reading Order

1. Start: [Theory & Concepts](./01-theory-and-concepts.md)
2. Practice: [Development Guide](./02-development-guide.md)
3. Apply: [Integration Examples](./03-integration-examples.md)
4. Deploy: [Deployment Guide](./04-deployment-guide.md)
5. Justify: [Business Guide](./05-business-stakeholder-guide.md)
6. Support: [FAQ & Troubleshooting](./06-faq-and-troubleshooting.md)
7. Master: This guide (Challenges & Scenarios)

---

**Remember:** Observability is a journey, not a destination. Start simple, measure impact, and iterate based on real needs!
