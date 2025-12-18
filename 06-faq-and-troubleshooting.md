# FAQ & Troubleshooting Guide

## Quick Solutions to Common Problems

**Audience:** All users - developers, operators, stakeholders
**Format:** Question/Answer with step-by-step solutions

---

## Table of Contents

### General Questions
1. [What is observability?](#what-is-observability)
2. [Do I need to instrument every endpoint?](#do-i-need-to-instrument-every-endpoint)
3. [Will this slow down my application?](#will-this-slow-down-my-application)
4. [How much does it cost?](#how-much-does-it-cost)

### Setup and Configuration
5. [Containers won't start](#containers-wont-start)
6. [Application can't connect to OTLP endpoint](#application-cant-connect-to-otlp-endpoint)
7. [How do I change Grafana password?](#how-do-i-change-grafana-password)
8. [Port conflicts on startup](#port-conflicts-on-startup)

### Data Not Appearing
9. [No metrics in Grafana](#no-metrics-in-grafana)
10. [No traces in Tempo](#no-traces-in-tempo)
11. [No logs in Loki](#no-logs-in-loki)
12. [Missing TraceId in logs](#missing-traceid-in-logs)

### Dashboard Questions
13. [How do I create a new dashboard?](#how-do-i-create-a-new-dashboard)
14. [Dashboard shows "No Data"](#dashboard-shows-no-data)
15. [How do I export/import dashboards?](#how-do-i-exportimport-dashboards)
16. [What are good alert thresholds?](#what-are-good-alert-thresholds)

### Performance Issues
17. [High memory usage in LGTM container](#high-memory-usage-in-lgtm-container)
18. [Slow query performance in Grafana](#slow-query-performance-in-grafana)
19. [Too much disk space used](#too-much-disk-space-used)
20. [Application performance degraded after adding telemetry](#application-performance-degraded-after-adding-telemetry)

### Correlation Problems
21. [Can't find logs for a specific trace](#cant-find-logs-for-a-specific-trace)
22. [TraceId in response but not in traces](#traceid-in-response-but-not-in-traces)
23. [Metrics don't match logs](#metrics-dont-match-logs)

### Development Questions
24. [How do I test telemetry locally?](#how-do-i-test-telemetry-locally)
25. [What should I instrument?](#what-should-i-instrument)
26. [How do I add custom metrics?](#how-do-i-add-custom-metrics)
27. [Best practices for log messages?](#best-practices-for-log-messages)

---

## General Questions

### What is observability?

**Answer:**

Observability is the ability to understand what's happening inside your application by examining its external outputs (metrics, logs, traces).

**Simple Analogy:**
- **Monitoring** = Dashboard warning lights in your car
- **Observability** = Complete diagnostic computer showing exactly what's wrong and why

**In Practice:**
Instead of just knowing "API is slow," observability tells you:
- Which endpoint is slow
- What operation within that endpoint
- Why it's slow (database query, external API, etc.)
- Complete timeline of the request
- Correlated logs showing exact state

**Learn More:** [Theory & Concepts](./01-theory-and-concepts.md)

---

### Do I need to instrument every endpoint?

**Answer:**

**No!** The middleware automatically instruments ALL HTTP endpoints.

**What's Automatic:**
- Request counting
- Response time measurement
- Error tracking
- HTTP status codes

**When to Add Manual Instrumentation:**
- Business-specific metrics (login success rate, orders processed)
- Database operation tracking
- External API call monitoring
- Custom business logic tracing

**Example:**
```csharp
// ✓ This is enough for most endpoints
[HttpGet("products/{id}")]
public async Task<IActionResult> GetProduct(int id)
{
    var product = await _service.GetProduct(id);
    return Ok(product);
}

// Add custom telemetry only for critical business operations
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder([FromBody] Order order)
{
    using var trace = ApplicationTelemetry.StartActivity("CreateOrder");
    // ... custom business metrics here
}
```

**Learn More:** [Development Guide - When to Instrument](./02-development-guide.md#quick-start)

---

### Will this slow down my application?

**Answer:**

**Minimal impact** if implemented correctly.

**Performance Overhead:**
- Metrics: < 1ms per request
- Traces: 1-5ms per request (with sampling)
- Logs: Depends on volume (use structured logging)
- **Total typical overhead: < 2%**

**Optimization Techniques:**

1. **Sampling** (Production)
   ```csharp
   .SetSampler(new TraceIdRatioBasedSampler(0.1)) // Sample 10%
   ```

2. **Batch Export**
   ```yaml
   processors:
     batch:
       timeout: 10s
       send_batch_size: 1024
   ```

3. **Async Export**
   - All exporters run asynchronously
   - Don't block request processing

**Benchmarks:**
```
Without Telemetry: 100ms average response
With Telemetry (100% sampling): 102ms average response
With Telemetry (10% sampling): 100.2ms average response
```

**Recommendation:** Use 100% sampling in dev/staging, 10% in production.

---

### How much does it cost?

**Answer:**

**Open-Source Solution (Current):**
- Software: $0 (Grafana LGTM is free)
- Infrastructure: $200-500/month (cloud hosting)
- **Total: $200-500/month**

**Breakdown:**

| Component | Resource | Monthly Cost |
|-----------|----------|-------------|
| **Docker Host** | 4 CPU, 16 GB RAM | $150-300 |
| **Storage** | 500 GB SSD | $50-100 |
| **Network** | 500 GB transfer | $20-50 |
| **Backup** | 200 GB | $10-20 |

**Cost Scaling:**

Small (< 100 req/s):
- Storage: 100 GB/month
- Cost: $200/month

Medium (100-500 req/s):
- Storage: 500 GB/month
- Cost: $400/month

Large (> 500 req/s):
- Storage: 2 TB/month
- Cost: $800/month

**Alternative: Grafana Cloud**
- Managed service
- $150-500/month depending on usage
- No infrastructure management

**Learn More:** [Business Guide - Cost Analysis](./05-business-stakeholder-guide.md#cost-benefit-analysis)

---

## Setup and Configuration

### Containers won't start

**Symptoms:**
- `docker-compose up` fails
- Container exits immediately
- "Port already in use" error

**Diagnosis:**

```bash
# Check if containers are running
docker ps -a

# View logs
docker logs grafana-lgtm

# Check Docker resources
docker info | grep -i memory
```

**Solutions:**

**Problem 1: Port Conflicts**

```bash
# Check which ports are in use
netstat -ano | findstr :3000
netstat -ano | findstr :4317
netstat -ano | findstr :4318

# Solution: Stop conflicting service or change ports
# In docker-compose.yaml:
ports:
  - "3001:3000"  # Change host port
  - "4320:4317"
  - "4321:4318"
```

**Problem 2: Insufficient Memory**

```bash
# Check available memory
docker info | grep "Total Memory"

# Solution: Increase Docker Desktop memory
# Docker Desktop → Settings → Resources → Memory
# Set to at least 4 GB
```

**Problem 3: Volume Permission Issues**

```bash
# Solution: Reset volumes
docker-compose down -v
docker-compose up -d
```

**Problem 4: Container Health Check Failing**

```bash
# Check health status
docker inspect grafana-lgtm | grep -i health

# Solution: Wait 2-3 minutes for container to fully start
# Or check logs for specific error
docker logs grafana-lgtm --tail 50
```

---

### Application can't connect to OTLP endpoint

**Symptoms:**
- No telemetry data in Grafana
- Application logs show connection errors
- "Connection refused" or "Timeout" errors

**Diagnosis:**

```bash
# Test OTLP endpoint connectivity
# From host machine:
telnet localhost 4317

# From within application container:
docker exec ews-api ping grafana-lgtm
```

**Solutions:**

**Problem 1: Wrong Endpoint URL**

```json
// appsettings.json - CHECK THIS
{
  "OpenTelemetry": {
    "OtlpEndpoint": "http://localhost:4317"  // ✓ Correct for app on host
    // "http://grafana-lgtm:4317"            // ✓ Correct for app in Docker
  }
}
```

**Problem 2: Firewall Blocking**

```bash
# Windows Firewall
# Allow incoming on port 4317
New-NetFirewallRule -DisplayName "OTLP gRPC" -Direction Inbound -LocalPort 4317 -Protocol TCP -Action Allow

# Linux iptables
sudo iptables -A INPUT -p tcp --dport 4317 -j ACCEPT
```

**Problem 3: Container Network Issue**

```yaml
# Ensure containers on same network
# docker-compose.yaml:
services:
  ews-api:
    networks:
      - observability
  grafana-lgtm:
    networks:
      - observability

networks:
  observability:
    driver: bridge
```

**Problem 4: OTLP Receiver Not Enabled**

```bash
# Check OTLP receiver is running
docker exec grafana-lgtm wget -q -O- http://localhost:4317
# Should return: "gRPC requires HTTP/2"
```

**Verification:**

```bash
# If connection works, you should see:
docker logs grafana-lgtm | grep -i "otlp"
# Output: "OTLP gRPC receiver started on port 4317"
```

---

### How do I change Grafana password?

**Method 1: Environment Variable (Before First Start)**

```yaml
# docker-compose.yaml
environment:
  - GF_SECURITY_ADMIN_PASSWORD=YourNewPassword123!
```

**Method 2: Grafana CLI (After Start)**

```bash
# Reset admin password
docker exec grafana-lgtm grafana-cli admin reset-admin-password NewPassword123!

# Restart container
docker-compose restart grafana-lgtm
```

**Method 3: Grafana UI (If Logged In)**

1. Login to Grafana
2. Click profile icon (bottom left)
3. Select "Preferences"
4. Click "Change Password"
5. Enter old and new password
6. Click "Change Password"

**Forgot Password Completely?**

```bash
# Reset to default
docker-compose down
docker volume rm grafana-data
docker-compose up -d

# Default credentials:
# Username: admin
# Password: Qwerty!123
```

---

### Port conflicts on startup

**Error Message:**
```
Error: bind: address already in use
```

**Find Conflicting Process:**

**Windows:**
```bash
netstat -ano | findstr :3000
# Output: TCP 0.0.0.0:3000 0.0.0.0:0 LISTENING 12345

# Kill process
taskkill /PID 12345 /F
```

**Linux/Mac:**
```bash
lsof -i :3000
# Output: process_name 12345 user

# Kill process
kill -9 12345
```

**Change Port Mapping:**

```yaml
# docker-compose.yaml
services:
  grafana-lgtm:
    ports:
      - "3001:3000"  # Use port 3001 on host instead
      - "4320:4317"
      - "4321:4318"

# Access Grafana at http://localhost:3001
```

**Common Port Conflicts:**
- 3000: Grafana (often conflicts with React dev server)
- 4317: OTLP gRPC
- 4318: OTLP HTTP
- 9090: Prometheus (optional)

---

## Data Not Appearing

### No metrics in Grafana

**Symptoms:**
- Dashboards show "No Data"
- Queries return empty results
- `api_requests_count_total` not found

**Step-by-Step Troubleshooting:**

**Step 1: Check Application is Exporting**

```bash
# Check application logs for export messages
docker logs ews-api | grep -i "export\|metric"

# Should see:
# "Exporting metrics to http://localhost:4317"
```

**Step 2: Verify OTLP Connection**

```bash
# Check OTLP collector is receiving data
docker exec grafana-lgtm wget -q -O- http://localhost:8888/metrics | grep -i "otlp"

# Should show non-zero values:
# otelcol_receiver_accepted_metric_points{...} 1234
```

**Step 3: Check Prometheus Scraping**

```bash
# Access Prometheus UI
http://localhost:9090

# Navigate to: Status → Targets
# Should show: otel-collector (1/1 up)
```

**Step 4: Query Directly in Prometheus**

```
# In Prometheus UI, query:
up

# Should return:
# up{instance="otel-collector:8888"} 1
```

**Step 5: Verify Meter Registration**

```csharp
// In Program.cs, ensure meter is added:
.WithMetrics(meterProviderBuilder =>
{
    meterProviderBuilder
        .AddMeter(ApplicationTelemetry.InstrumentationName)  // ← MUST BE HERE
        .AddMeter("Crismac.Ews.Login")  // For LoginTelemetry
        .AddOtlpExporter();
});
```

**Step 6: Wait for Export Interval**

```
Default export interval: 60 seconds
Wait 1-2 minutes after starting application
```

**Step 7: Check for Metric Name Typos**

```promql
# List all metrics
{__name__=~".+"}

# Should see:
# api_requests_count_total
# ews_login_attempts_total
# etc.
```

**Common Causes:**

| Cause | Solution |
|-------|----------|
| Meter not registered | Add to Program.cs |
| Wrong metric name | Check query matches code |
| Application not running | Start application |
| Export interval not elapsed | Wait 60 seconds |
| OTLP connection failed | Check connectivity |

---

### No traces in Tempo

**Symptoms:**
- "No trace found" in Grafana Explore
- TraceId search returns nothing
- Trace panel empty in dashboards

**Troubleshooting:**

**Step 1: Verify Activity.Current**

```csharp
// Add to your controller
var activity = Activity.Current;
if (activity == null)
{
    _logger.LogWarning("Activity.Current is NULL - tracing not working!");
}
else
{
    _logger.LogInformation("TraceId: {TraceId}", activity.TraceId);
}
```

**Step 2: Check ActivitySource Registration**

```csharp
// In Program.cs
.WithTracing(tracerProviderBuilder =>
{
    tracerProviderBuilder
        .AddSource(ApplicationTelemetry.InstrumentationName)  // ← MUST BE HERE
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter();
});
```

**Step 3: Verify OTLP Exporter**

```bash
# Check application logs
docker logs ews-api | grep -i "trace\|span"

# Should see exports happening
```

**Step 4: Query Tempo Directly**

```bash
# Check if Tempo is receiving traces
docker exec grafana-lgtm wget -q -O- http://localhost:3200/ready

# Should return: "ready"
```

**Step 5: Check Trace Retention**

```yaml
# In tempo-config.yaml
compactor:
  compaction:
    block_retention: 168h  # 7 days

# If searching for old trace, may be expired
```

**Step 6: Test with Simple Endpoint**

```csharp
[HttpGet("test-trace")]
public IActionResult TestTrace()
{
    using var activity = ApplicationTelemetry.StartActivity("TestTrace");
    activity?.SetTag("test", "value");

    _logger.LogInformation(
        "Test trace. TraceId: {TraceId}",
        activity?.TraceId
    );

    return Ok(new { traceId = activity?.TraceId.ToString() });
}

// Call endpoint, copy TraceId, search in Tempo
```

---

### No logs in Loki

**Symptoms:**
- Loki datasource shows no logs
- Log panel empty
- Query returns no results

**Troubleshooting:**

**Step 1: Check Serilog Configuration**

```json
// appsettings.json
{
  "Serilog": {
    "WriteTo": [
      {
        "Name": "OpenTelemetry",
        "Args": {
          "endpoint": "http://localhost:4318/v1/logs",  // ← CHECK THIS
          "protocol": "HttpProtobuf",
          "resourceAttributes": {
            "service.name": "CrismacEWSBackendService"
          }
        }
      }
    ]
  }
}
```

**Step 2: Verify Logs are Written**

```bash
# Check file logs (should work even if OTLP fails)
ls -l Logs/
cat Logs/app-log-$(date +%Y%m%d).json | tail -10
```

**Step 3: Test Loki Directly**

```bash
# Query Loki API
curl -G -s "http://localhost:3100/loki/api/v1/query" \
  --data-urlencode 'query={service_name="CrismacEWSBackendService"}' | jq .

# Should return log entries
```

**Step 4: Check Loki Configuration in Grafana**

1. Go to Configuration → Data Sources → Loki
2. URL should be: `http://localhost:3100`
3. Click "Save & Test"
4. Should show: "Data source is working"

**Step 5: Verify Log Format**

```bash
# Logs must be in format Loki expects
# Check a sample log entry:
cat Logs/app-log-*.json | head -1 | jq .

# Should be valid JSON with timestamp
```

---

### Missing TraceId in logs

**Symptoms:**
- Logs don't show TraceId
- Can't correlate logs with traces
- TraceId property is null/empty

**Diagnosis:**

```bash
# Check a log entry
cat Logs/app-log-*.json | jq . | head -20

# Should contain:
# "TraceId": "0af7651916cd43dd8448eb211c80319c"
```

**Solutions:**

**Solution 1: Enable Activity Tracking**

```csharp
// In Program.cs
builder.Logging.Configure(options =>
{
    options.ActivityTrackingOptions =
        ActivityTrackingOptions.TraceId |
        ActivityTrackingOptions.SpanId;
});
```

**Solution 2: Explicitly Add TraceId to Logs**

```csharp
_logger.LogInformation(
    "Message. TraceId: {TraceId}, SpanId: {SpanId}",
    Activity.Current?.TraceId.ToString(),
    Activity.Current?.SpanId.ToString()
);
```

**Solution 3: Use Serilog Enrichment**

```csharp
// In Program.cs
Log.Logger = new LoggerConfiguration()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("TraceId", Activity.Current?.TraceId.ToString())
    .CreateLogger();
```

**Solution 4: Configure Serilog Template**

```json
// appsettings.json
{
  "Serilog": {
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss}] [{Level}] TraceId: {TraceId} - {Message}{NewLine}{Exception}"
        }
      }
    ]
  }
}
```

**Verification:**

```bash
# Make a request
curl http://localhost:5294/api/test

# Check logs
cat Logs/app-log-*.json | jq .TraceId

# Should output TraceId, not null
```

---

## Dashboard Questions

### How do I create a new dashboard?

**Step-by-Step:**

1. **Access Grafana**
   - URL: http://localhost:3000
   - Login with admin credentials

2. **Create Dashboard**
   - Click "+" → "Dashboard"
   - Click "Add visualization"

3. **Select Data Source**
   - Choose "Mimir" for metrics
   - Choose "Loki" for logs
   - Choose "Tempo" for traces

4. **Write Query**

   **For Metrics (PromQL):**
   ```promql
   rate(api_requests_count_total[5m])
   ```

   **For Logs (LogQL):**
   ```logql
   {service_name="CrismacEWSBackendService"} |= "error"
   ```

5. **Configure Visualization**
   - Choose chart type (Time series, Bar chart, Table, etc.)
   - Set title
   - Configure axes
   - Set colors/thresholds

6. **Save Dashboard**
   - Click "Save dashboard" (top right)
   - Enter name
   - Select folder
   - Click "Save"

**Example: Request Rate Panel**

```
Query: sum(rate(api_requests_count_total[5m]))
Visualization: Time series
Title: "Total Request Rate"
Unit: "req/s"
```

---

### Dashboard shows "No Data"

**Checklist:**

**1. Time Range**
- [ ] Check time range selector (top right)
- [ ] Ensure it covers period when data exists
- [ ] Try "Last 5 minutes" or "Last 1 hour"

**2. Data Source**
- [ ] Correct data source selected
- [ ] Data source is connected (green indicator)

**3. Query**
- [ ] Query syntax is correct
- [ ] Metric/log stream name matches exactly
- [ ] No typos in query

**4. Data Exists**
- [ ] Application is running
- [ ] Application is generating data
- [ ] Data has been exported (wait 60 seconds)

**Quick Test:**

```promql
# In query editor, try:
up

# Should return 1 for running services
# If this works, data source is fine
```

---

### How do I export/import dashboards?

**Export Dashboard:**

1. Open dashboard
2. Click settings icon (gear, top right)
3. Click "JSON Model" (left sidebar)
4. Click "Copy to Clipboard"
5. Paste into file: `my-dashboard.json`

**OR via API:**

```bash
# Export dashboard by UID
curl -H "Authorization: Bearer YOUR_API_KEY" \
  http://localhost:3000/api/dashboards/uid/DASHBOARD_UID > dashboard.json
```

**Import Dashboard:**

1. Click "+" → "Import dashboard"
2. Choose method:
   - **Upload JSON file:** Click "Upload JSON file"
   - **Paste JSON:** Paste JSON into text box
   - **Import via ID:** Enter Grafana.com dashboard ID

3. Configure:
   - Select datasource (usually Mimir)
   - Change name/folder if needed

4. Click "Import"

**Bulk Import:**

```bash
# Place dashboards in folder
mkdir -p Dashboards

# Configure provisioning
# In docker-compose.yaml:
volumes:
  - ./Dashboards:/etc/grafana/provisioning/dashboards
```

---

### What are good alert thresholds?

**Recommended Thresholds:**

**1. Error Rate Alert**

```promql
# Alert when error rate > 5% for 5 minutes
(sum(rate(api_errors_total[5m])) / sum(rate(api_requests_count_total[5m]))) * 100 > 5

Severity: Critical
Notification: Immediate (PagerDuty, SMS)
```

**2. Response Time Alert**

```promql
# Alert when p95 latency > 2 seconds
histogram_quantile(0.95, rate(api_requests_duration_milliseconds_bucket[5m])) > 2000

Severity: Warning
Notification: Email, Slack
```

**3. Availability Alert**

```promql
# Alert when service down for 2 minutes
up{job="ews-api"} == 0

Severity: Critical
Notification: Immediate
```

**4. Login Failure Alert**

```promql
# Alert when login failure rate > 10%
(sum(rate(ews_login_attempts_total{status="failed"}[10m])) /
 sum(rate(ews_login_attempts_total[10m]))) * 100 > 10

Severity: Warning
Notification: Email
```

**5. Brute Force Detection**

```promql
# Alert on 5+ failed logins from same user in 10 min
sum by(username) (increase(ews_login_attempts_total{status="failed"}[10m])) > 5

Severity: Security
Notification: Security team
```

**Threshold Guidelines:**

| Metric | Warning | Critical |
|--------|---------|----------|
| Error Rate | > 2% | > 5% |
| P95 Latency | > 1s | > 2s |
| Availability | < 99.5% | < 99% |
| Disk Usage | > 80% | > 90% |
| Memory Usage | > 85% | > 95% |

---

## Performance Issues

### High memory usage in LGTM container

**Symptoms:**
- Container using > 4 GB RAM
- Container restarts frequently
- OOM (Out of Memory) errors

**Diagnosis:**

```bash
docker stats grafana-lgtm

# Shows real-time memory usage
```

**Solutions:**

**Solution 1: Increase Container Memory**

```yaml
# docker-compose.yaml
services:
  grafana-lgtm:
    deploy:
      resources:
        limits:
          memory: 8G  # Increase from 4G
```

**Solution 2: Reduce Data Retention**

```yaml
# loki-config.yaml
limits_config:
  retention_period: 7d  # Reduce from 30d

# tempo-config.yaml
compactor:
  compaction:
    block_retention: 72h  # Reduce from 168h
```

**Solution 3: Enable Batch Processing**

```yaml
# otel-collector-config.yaml
processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128
```

**Solution 4: Reduce Log Volume**

```csharp
// In Program.cs - filter noisy logs
.WithFilter(logEvent =>
{
    // Exclude health check logs
    return !logEvent.MessageTemplate.Text.Contains("health");
})
```

**Solution 5: Split Components**

```yaml
# Run Loki, Tempo, Mimir in separate containers
# Instead of all-in-one LGTM image
```

---

### Slow query performance in Grafana

**Symptoms:**
- Dashboards take > 30 seconds to load
- Query timeout errors
- Browser becomes unresponsive

**Solutions:**

**Solution 1: Reduce Time Range**

```
# Instead of:
Last 24 hours (slow)

# Use:
Last 1 hour (fast)
```

**Solution 2: Optimize Queries**

**Bad (Slow):**
```promql
# Unbounded aggregation
sum(api_requests_count_total)

# High cardinality
sum by(user_id) (api_requests_count_total)
```

**Good (Fast):**
```promql
# Limited aggregation
sum(rate(api_requests_count_total[5m]))

# Low cardinality
sum by(http_route) (rate(api_requests_count_total[5m]))
```

**Solution 3: Use Recording Rules**

```yaml
# Pre-compute expensive queries
# In prometheus.yaml:
groups:
  - name: request_rate
    interval: 1m
    rules:
      - record: job:api_requests:rate5m
        expr: sum(rate(api_requests_count_total[5m]))
```

**Solution 4: Increase Query Timeout**

```yaml
# In grafana.ini:
[dataproxy]
timeout = 300  # Increase from 30 seconds
```

**Solution 5: Add Panel Cache**

```
# In panel settings:
Query options → Cache timeout: 300 (5 minutes)
```

---

### Too much disk space used

**Symptoms:**
- Docker volume using > 100 GB
- Disk full warnings
- Container won't start due to no space

**Diagnosis:**

```bash
# Check volume sizes
docker system df -v

# Check specific volume
docker exec grafana-lgtm du -sh /data/*
```

**Solutions:**

**Solution 1: Reduce Retention Period**

```yaml
# loki-config.yaml
limits_config:
  retention_period: 7d  # Was 30d

# tempo-config.yaml
compactor:
  compaction:
    block_retention: 72h  # Was 168h
```

**Solution 2: Clean Up Old Data**

```bash
# Stop container
docker-compose stop grafana-lgtm

# Clean up old data
docker exec grafana-lgtm sh -c "find /data -mtime +30 -delete"

# Restart
docker-compose start grafana-lgtm
```

**Solution 3: Implement Sampling**

```csharp
// Sample only 10% of traces in production
.SetSampler(new TraceIdRatioBasedSampler(0.1))
```

**Solution 4: Compact Data**

```bash
# Run compaction manually
docker exec grafana-lgtm /bin/compact
```

**Solution 5: External Storage**

```yaml
# Use S3 or similar for long-term storage
# loki-config.yaml:
storage_config:
  aws:
    s3: s3://your-bucket/loki
    bucketnames: loki-data
```

---

### Application performance degraded after adding telemetry

**Symptoms:**
- Response times increased by > 10%
- Higher CPU usage
- Increased memory consumption

**Diagnosis:**

```bash
# Compare before/after
# Benchmark without telemetry
ab -n 1000 -c 10 http://localhost:5294/api/test

# Benchmark with telemetry
ab -n 1000 -c 10 http://localhost:5294/api/test
```

**Solutions:**

**Solution 1: Enable Sampling**

```csharp
// Production: Sample 10% of traces
.WithTracing(tracing => tracing
    .SetSampler(new TraceIdRatioBasedSampler(0.1))
)
```

**Solution 2: Batch Export**

```csharp
.AddOtlpExporter(options =>
{
    options.ExportProcessorType = ExportProcessorType.Batch;
    options.BatchExportProcessorOptions = new BatchExportProcessorOptions<Activity>
    {
        MaxQueueSize = 2048,
        ScheduledDelayMilliseconds = 5000,
        ExporterTimeoutMilliseconds = 30000,
        MaxExportBatchSize = 512
    };
})
```

**Solution 3: Reduce Instrumentation Scope**

```csharp
// Only instrument important operations
.AddAspNetCoreInstrumentation(options =>
{
    options.Filter = httpContext =>
    {
        // Skip health checks
        if (httpContext.Request.Path.StartsWithSegments("/health"))
            return false;

        // Skip static files
        if (httpContext.Request.Path.StartsWithSegments("/static"))
            return false;

        return true;
    };
})
```

**Solution 4: Disable Detailed Logging**

```json
// appsettings.Production.json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",  // Was "Debug"
      "Microsoft": "Warning"      // Was "Information"
    }
  }
}
```

**Solution 5: Async Export**

```csharp
// Ensure exports don't block
.AddOtlpExporter(options =>
{
    options.Protocol = OtlpExportProtocol.Grpc; // Faster than HTTP
})
```

**Expected Overhead:**
- < 2% CPU increase
- < 5% memory increase
- < 1% latency increase (with sampling)

---

## Correlation Problems

### Can't find logs for a specific trace

**Problem:**
Have TraceId from trace, but can't find matching logs.

**Solution:**

**Step 1: Verify TraceId Format**

```
Valid: 0af7651916cd43dd8448eb211c80319c (32 hex chars)
Invalid: 0af76519-16cd-43dd-8448-eb211c80319c (with dashes)
```

**Step 2: Search in Loki**

```logql
# Correct query format
{service_name="CrismacEWSBackendService"}
  | json
  | TraceId = "0af7651916cd43dd8448eb211c80319c"
```

**Step 3: Check Log Time Range**

```
# Ensure time range includes trace timestamp
# Trace time: 2025-12-18 10:30:45
# Loki time range: Must include 10:30:45
```

**Step 4: Verify Logs Contain TraceId**

```bash
# Check raw log file
cat Logs/app-log-*.json | grep "0af7651916cd43dd8448eb211c80319c"

# Should find matching entries
```

**Step 5: Check Different Log Levels**

```logql
# Search all levels, not just INFO
{service_name="CrismacEWSBackendService"}
  | json
  | level =~ "Information|Warning|Error"
  | TraceId = "..."
```

---

### TraceId in response but not in traces

**Problem:**
Error response contains TraceId, but trace doesn't exist in Tempo.

**Causes:**

**Cause 1: Sampling**

```csharp
// If using sampling, trace might not be sampled
.SetSampler(new TraceIdRatioBasedSampler(0.1))  // Only 10% sampled

// Solution: Increase sampling or disable in dev
.SetSampler(new AlwaysOnSampler())  // 100% sampling
```

**Cause 2: Export Delay**

```
Trace created: 10:30:00
Export happens: 10:30:60 (60 second delay)

// Solution: Wait 1-2 minutes before searching
```

**Cause 3: Trace Expired**

```yaml
# Tempo retention: 7 days
# If searching for 8-day-old trace, it's gone

# Solution: Search within retention period
```

**Cause 4: Export Failed**

```bash
# Check application logs for export errors
docker logs ews-api | grep -i "export.*fail\|error.*trace"

# Solution: Fix connectivity to OTLP endpoint
```

---

### Metrics don't match logs

**Problem:**
Metric shows 100 requests, but only 80 log entries found.

**Explanation:**

**Normal Differences:**

1. **Metrics** count ALL requests (including health checks, static files)
2. **Logs** may filter certain requests

```csharp
// Logs might exclude health checks
.AddAspNetCoreInstrumentation(options =>
{
    options.Filter = context => !context.Request.Path.Contains("health");
})
```

3. **Sampling** affects logs differently than metrics
4. **Log level** filtering

**Verification:**

```promql
# Get total requests from metrics
sum(increase(api_requests_count_total[1h]))
# Result: 1000

# Get log count
count_over_time({service_name="..."}[1h])
# Result: 900

# Difference: 100 requests
# Likely: Health checks, OPTIONS requests, etc.
```

**Not a Problem Unless:**
- Difference > 20%
- Metrics missing entirely
- Logs show errors not in metrics

---

## Development Questions

### How do I test telemetry locally?

**Step-by-Step Testing:**

**1. Start LGTM Stack**

```bash
cd Crismac.Ews.WebApi
docker-compose -f otel-lgtm-docker-compose.yaml up -d
```

**2. Run Application**

```bash
dotnet run --project Crismac.Ews.WebApi
```

**3. Make Test Request**

```bash
# Simple GET request
curl http://localhost:5294/api/telemetrytest/verify

# POST request with body
curl -X POST http://localhost:5294/api/test \
  -H "Content-Type: application/json" \
  -d '{"test": "data"}'
```

**4. Verify Metrics**

```
1. Open Grafana: http://localhost:3000
2. Go to Explore
3. Select "Mimir" datasource
4. Query: api_requests_count_total
5. Should see: 1 data point
```

**5. Verify Traces**

```
1. In Grafana Explore
2. Select "Tempo" datasource
3. Search for service: CrismacEWSBackendService
4. Should see: Your test request trace
5. Click to view span details
```

**6. Verify Logs**

```
1. In Grafana Explore
2. Select "Loki" datasource
3. Query: {service_name="CrismacEWSBackendService"}
4. Should see: Log entries with TraceId
```

**Quick Test Script:**

```bash
#!/bin/bash
# test-telemetry.sh

# Make request and capture TraceId
RESPONSE=$(curl -s http://localhost:5294/api/telemetrytest/verify)
TRACE_ID=$(echo $RESPONSE | jq -r '.telemetry.traceId')

echo "TraceId: $TRACE_ID"

# Wait for export
echo "Waiting 60 seconds for export..."
sleep 60

# Check if trace exists
echo "Checking for trace in Tempo..."
curl -s "http://localhost:3200/api/traces/$TRACE_ID" | jq .

echo "Test complete!"
```

---

### What should I instrument?

**Always Instrument:**

1. **Critical Business Operations**
   - User registration
   - Login/authentication
   - Payment processing
   - Order placement
   - Data imports/exports

2. **Expensive Operations**
   - Database queries > 100ms
   - External API calls
   - File uploads
   - Report generation
   - Batch jobs

3. **Error-Prone Operations**
   - Complex validation
   - Third-party integrations
   - Legacy system interactions
   - Data transformations

**Never Instrument:**

1. **High-Frequency, Low-Value Operations**
   - Health checks (filter out)
   - Static file serving
   - Ping endpoints
   - Heartbeat checks

2. **Operations with Sensitive Data**
   - Don't log passwords, credit cards, PII
   - Hash or redact sensitive fields

**Instrumentation Levels:**

**Level 1: Automatic (No Code)**
```csharp
// Middleware handles ALL HTTP requests automatically
// No manual code needed
```

**Level 2: Basic Manual**
```csharp
// Add for important operations
using var activity = ApplicationTelemetry.StartActivity("ProcessOrder");
```

**Level 3: Detailed Manual**
```csharp
// Add for complex flows
using var activity = ApplicationTelemetry.StartActivity("ComplexWorkflow");
activity?.SetTag("order.id", orderId);
activity?.SetTag("customer.type", "premium");

// Add child spans for sub-operations
using (var dbActivity = ApplicationTelemetry.StartActivity("DB_Query"))
{
    await _repository.GetOrder(orderId);
}
```

**Rule of Thumb:**
If you'd want to debug it in production, instrument it!

---

### How do I add custom metrics?

**Step-by-Step:**

**1. Create Telemetry Class**

```csharp
// Observability/OrderTelemetry.cs
public static class OrderTelemetry
{
    private static readonly Meter Meter = new Meter("Crismac.Ews.Orders");

    public static readonly Counter<long> OrdersProcessed = Meter.CreateCounter<long>(
        name: "ews.orders.processed.total",
        unit: "{orders}",
        description: "Total orders processed"
    );

    public static readonly Histogram<double> OrderValue = Meter.CreateHistogram<double>(
        name: "ews.orders.value.dollars",
        unit: "USD",
        description: "Order value in USD"
    );

    public static void RecordOrder(string status, double value)
    {
        var tags = new KeyValuePair<string, object?>[]
        {
            new("status", status)
        };

        OrdersProcessed.Add(1, tags);
        OrderValue.Record(value, tags);
    }
}
```

**2. Register Meter**

```csharp
// In Program.cs
builder.Services.AddOpenTelemetry()
    .WithMetrics(meterProviderBuilder =>
    {
        meterProviderBuilder
            .AddMeter("Crismac.Ews.Orders")  // ← Add this
            .AddOtlpExporter();
    });
```

**3. Use in Controller**

```csharp
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder([FromBody] Order order)
{
    var result = await _orderService.CreateOrder(order);

    // Record metrics
    OrderTelemetry.RecordOrder(
        status: "success",
        value: order.Total
    );

    return Ok(result);
}
```

**4. Query in Grafana**

```promql
# Order rate
rate(ews_orders_processed_total[5m])

# Average order value
rate(ews_orders_value_dollars_sum[5m])
/
rate(ews_orders_value_dollars_count[5m])

# Orders by status
sum by(status) (rate(ews_orders_processed_total[5m]))
```

---

### Best practices for log messages?

**DO:**

✓ **Use Structured Logging**
```csharp
// Good
_logger.LogInformation(
    "Order created. OrderId: {OrderId}, Total: {Total}, Duration: {DurationMs}ms",
    order.Id,
    order.Total,
    stopwatch.ElapsedMilliseconds
);

// Bad
_logger.LogInformation($"Order {order.Id} created with total ${order.Total}");
```

✓ **Include TraceId**
```csharp
_logger.LogInformation(
    "Message. TraceId: {TraceId}",
    Activity.Current?.TraceId
);
```

✓ **Use Appropriate Log Levels**
```csharp
_logger.LogDebug("Detailed info for debugging");
_logger.LogInformation("Normal operations");
_logger.LogWarning("Unexpected but recoverable");
_logger.LogError(ex, "Errors and exceptions");
_logger.LogCritical("Critical failures");
```

✓ **Add Context**
```csharp
_logger.LogError(
    ex,
    "Payment failed. OrderId: {OrderId}, Amount: {Amount}, Gateway: {Gateway}, TraceId: {TraceId}",
    orderId,
    amount,
    "Stripe",
    Activity.Current?.TraceId
);
```

**DON'T:**

✗ **Log Sensitive Data**
```csharp
// Bad - exposes password
_logger.LogInformation("Login attempt: {Username} / {Password}", username, password);

// Good - no sensitive data
_logger.LogInformation("Login attempt: {Username}", username);
```

✗ **Use String Interpolation**
```csharp
// Bad - can't query structured logs
_logger.LogInformation($"Order {orderId} created");

// Good - structured
_logger.LogInformation("Order created. OrderId: {OrderId}", orderId);
```

✗ **Log in Tight Loops**
```csharp
// Bad - creates millions of log entries
foreach (var item in millionItems)
{
    _logger.LogDebug("Processing {Item}", item);
}

// Good - log summary
_logger.LogInformation("Processing {Count} items", millionItems.Count);
```

✗ **Duplicate Information**
```csharp
// Bad - logs same info twice
_logger.LogInformation("Order created: {OrderId}", orderId);
_logger.LogInformation("Created order with ID: {OrderId}", orderId);

// Good - log once with full context
_logger.LogInformation(
    "Order created. OrderId: {OrderId}, Status: {Status}, Total: {Total}",
    orderId, status, total
);
```

---

**For more help, see:**
- [Development Guide](./02-development-guide.md) - Detailed development patterns
- [Integration Examples](./03-integration-examples.md) - Real code examples
- [Deployment Guide](./04-deployment-guide.md) - Infrastructure setup
- [Challenges & Scenarios](./07-challenges-and-scenarios.md) - Advanced topics
