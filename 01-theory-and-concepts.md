# Theory & Concepts: Understanding Observability

## A Complete Guide to Modern Application Monitoring

**Audience:** All stakeholders - developers, architects, operations, business leaders
**Reading Time:** 30-45 minutes
**Prerequisites:** Basic understanding of web applications

---

## Table of Contents

1. [What is Observability?](#what-is-observability)
2. [The Three Pillars](#the-three-pillars)
3. [LGTM Stack Architecture](#lgtm-stack-architecture)
4. [OpenTelemetry Framework](#opentelemetry-framework)
5. [The RED Method](#the-red-method)
6. [Correlation Principles](#correlation-principles)
7. [Semantic Conventions](#semantic-conventions)
8. [Observability vs Monitoring](#observability-vs-monitoring)

---

## What is Observability?

### Definition

**Observability** is the ability to understand the internal state of a system by examining its external outputs. In software systems, it means answering questions like:

- "Why is this user experiencing slow response times?"
- "Which component is causing the error?"
- "How did this request flow through the system?"
- "What was the system state when the incident occurred?"

### The Core Question

Observability answers: **"Can I understand what's happening inside my system by looking at what it outputs?"**

### Why Traditional Monitoring Isn't Enough

**Traditional Monitoring:**
```
System Down? → YES/NO
Response Time? → 250ms
Error Count? → 5 errors/min
```

**Problem:** You know WHAT is wrong, but not WHY or WHERE.

**Observability:**
```
System Down? → Yes
Why? → Database connection timeout
Where? → LoginController.SelectLoginDetails:92
When? → Started at 10:30:42 AM
Impact? → 15% of login attempts affected
Root Cause? → Missing database index on Users.Username
```

**Solution:** Complete context for rapid resolution.

---

### The Fundamental Shift

| Traditional Monitoring | Modern Observability |
|------------------------|---------------------|
| **Known Unknowns** | **Unknown Unknowns** |
| Pre-defined metrics | Arbitrary questions |
| Fixed dashboards | Exploratory analysis |
| "Is the system up?" | "Why is it behaving this way?" |
| Reactive | Proactive + Predictive |

### Real-World Analogy

**Monitoring is like a car dashboard:**
- Speed gauge shows 60 mph
- Fuel gauge shows half full
- Temperature is normal

**Observability is like having:**
- GPS showing exact route taken
- Black box recording every decision
- Full diagnostic computer analyzing engine performance
- Ability to ask: "Why did the car stall at mile 47?"

---

## The Three Pillars

Observability rests on three fundamental types of telemetry data:

### 1. Metrics 📊

**Definition:** Numerical measurements of system behavior over time.

**Characteristics:**
- **Aggregated:** Counts, averages, percentiles
- **Time-series:** Values change over time
- **Efficient:** Low storage cost
- **Queryable:** Fast analysis of trends

**Examples:**
```
- Request count per second: 125 req/s
- Average response time: 234 ms
- Error rate: 0.5%
- CPU usage: 45%
- Memory consumption: 2.3 GB
```

**When to Use:**
- Dashboards showing trends
- Alerting on thresholds
- Capacity planning
- SLA/SLO tracking

**Analogy:** Metrics are like your vital signs - heart rate, blood pressure, temperature. They tell you if something is wrong.

---

### 2. Logs 📝

**Definition:** Timestamped text records of discrete events.

**Characteristics:**
- **Detailed:** Full context for specific events
- **Searchable:** Find needles in haystacks
- **Verbose:** Can be large volume
- **Contextual:** Includes stack traces, variables, state

**Examples:**
```
[2025-12-18 10:30:45.123] [ERROR] [LoginController]
TraceId: 0af7651916cd43dd8448eb211c80319c
User login failed for 'john.doe' from IP 192.168.1.45
Reason: Invalid credentials
Attempt: 3 of 5
```

**When to Use:**
- Debugging specific errors
- Security investigations
- Audit trails
- Understanding context

**Analogy:** Logs are like a medical history - detailed records of every symptom, diagnosis, and treatment.

---

### 3. Traces 🔍

**Definition:** Records of the journey a request takes through a distributed system.

**Characteristics:**
- **Distributed:** Spans multiple services/components
- **Hierarchical:** Parent-child relationships
- **Timed:** Duration of each operation
- **Tagged:** Metadata about operations

**Example Trace:**
```
HTTP POST /api/Login (234ms total)
├─ Authentication Check (45ms)
│  ├─ Database Query: GetUser (38ms)
│  └─ Password Verification (7ms)
├─ Token Generation (15ms)
└─ Session Creation (174ms)
   ├─ Cache Write (12ms)
   └─ Database Update (162ms) ⚠️ SLOW!
```

**When to Use:**
- Understanding system flow
- Identifying bottlenecks
- Distributed system debugging
- Performance optimization

**Analogy:** Traces are like GPS navigation history - showing exactly where you went, how long each segment took, and where delays occurred.

---

### How the Three Pillars Work Together

```
┌─────────────────────────────────────────────────────┐
│                  OBSERVABILITY                      │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐│
│  │   METRICS   │  │    LOGS     │  │   TRACES    ││
│  │   (WHAT)    │  │    (WHY)    │  │   (WHERE)   ││
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘│
│         │                │                │        │
│         └────────────────┼────────────────┘        │
│                          │                         │
│                    CORRELATION                     │
│                    (TraceId)                       │
└─────────────────────────────────────────────────────┘
```

### Correlation Example

**1. Start with Metrics (WHAT):**
```promql
# Alert: Error rate spiked at 10:30 AM
rate(api_errors_total[5m]) > 0.05
```

**2. Find Traces (WHERE):**
```
# Filter traces at 10:30 AM with errors
TraceId: 0af7651916cd43dd8448eb211c80319c
Shows: Database query taking 2.3 seconds
```

**3. Check Logs (WHY):**
```logql
# Query logs with TraceId
{service="EWS"} | TraceId="0af7651916cd43dd8448eb211c80319c"

Result: "Database connection pool exhausted"
```

**Root Cause:** Database connection pool too small for traffic spike.

---

## LGTM Stack Architecture

The **LGTM** acronym represents Grafana's integrated observability stack:

- **L**oki - Log aggregation
- **G**rafana - Visualization and dashboards
- **T**empo - Distributed tracing
- **M**imir - Long-term metrics storage

### Architecture Diagram

```
┌───────────────────────────────────────────────────────────┐
│              EWS Backend Application                      │
│                  (.NET Core 8.0)                          │
│                                                           │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │ Controllers  │  │  Middleware  │  │ Repositories │   │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘   │
│         │                 │                  │           │
│         └─────────────────┼──────────────────┘           │
│                           │                              │
│              ┌────────────▼────────────┐                 │
│              │ ApplicationTelemetry    │                 │
│              │ (OpenTelemetry SDK)     │                 │
│              └────────────┬────────────┘                 │
└───────────────────────────┼───────────────────────────────┘
                            │
                            │ OTLP Protocol
                            │ (gRPC: Port 4317)
                            │ (HTTP: Port 4318)
                            ▼
           ┌────────────────────────────────┐
           │  OpenTelemetry Collector       │
           │  (Receives & Routes Data)      │
           └────┬──────────┬────────┬───────┘
                │          │        │
     ┌──────────▼───┐  ┌───▼────┐  ┌▼──────────┐
     │    LOKI      │  │ TEMPO  │  │   MIMIR   │
     │    (Logs)    │  │(Traces)│  │ (Metrics) │
     │              │  │        │  │           │
     │ - Indexing   │  │- Trace │  │- TSDB     │
     │ - Storage    │  │  Store │  │- PromQL   │
     │ - Query      │  │- Query │  │- Alerts   │
     └──────┬───────┘  └───┬────┘  └──┬────────┘
            │              │           │
            └──────────────┼───────────┘
                           │
                  ┌────────▼─────────┐
                  │     GRAFANA      │
                  │  (Visualization) │
                  │                  │
                  │ - Dashboards     │
                  │ - Explore        │
                  │ - Alerting       │
                  │ - Correlations   │
                  └──────────────────┘
                           │
                           ▼
                    Users/Engineers
```

---

### Component Deep Dive

#### 1. OpenTelemetry Collector

**Role:** Central hub for receiving, processing, and routing telemetry data.

**Capabilities:**
- **Receivers:** Accept data in various formats (OTLP, Prometheus, Jaeger)
- **Processors:** Transform, filter, batch data
- **Exporters:** Send data to backends (Loki, Tempo, Mimir)

**Why It's Important:**
- Decouples application from backend
- Allows backend changes without code changes
- Provides buffering and retry logic
- Enables data transformation

**Configuration:**
```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 10s
    send_batch_size: 1024

exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  otlp/tempo:
    endpoint: tempo:4317
  prometheus:
    endpoint: "0.0.0.0:9090"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlp/tempo]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [loki]
```

---

#### 2. Loki (Log Aggregation)

**Role:** Efficient log storage and querying.

**Architecture:**
```
┌──────────────────────────────────────┐
│            LOKI                      │
│                                      │
│  ┌──────────┐      ┌──────────┐     │
│  │Distributor│─────▶│ Ingester │     │
│  └──────────┘      └────┬─────┘     │
│                         │            │
│  ┌──────────┐      ┌────▼─────┐     │
│  │  Querier │◀─────│  Store   │     │
│  └──────────┘      └──────────┘     │
└──────────────────────────────────────┘
```

**Key Features:**
- **Label-based indexing:** Only indexes labels, not content
- **Low cost:** Cheaper than traditional log aggregation
- **LogQL:** Powerful query language
- **Compression:** Efficient storage

**Example Query:**
```logql
# Find errors in last hour with specific TraceId
{service_name="CrismacEWSBackendService"}
  |= "error"
  | json
  | TraceId="0af7651916cd43dd8448eb211c80319c"
  | line_format "{{.timestamp}} {{.level}} {{.message}}"
```

---

#### 3. Tempo (Distributed Tracing)

**Role:** Store and query distributed traces.

**Key Features:**
- **Scalable:** Handles millions of spans
- **Cost-effective:** Uses object storage (S3, GCS)
- **Integration:** Works with Jaeger, Zipkin, OpenTelemetry
- **TraceQL:** Advanced trace querying

**Trace Storage:**
```
Trace: 0af7651916cd43dd8448eb211c80319c
├─ Span: Login (Root)
│  ├─ Duration: 234ms
│  ├─ Status: OK
│  └─ Tags: http.method=POST, http.status_code=200
│
├─ Span: DB_ValidateUser (Child of Login)
│  ├─ Duration: 162ms
│  ├─ Status: OK
│  └─ Tags: db.system=mssql, db.operation=SELECT
│
└─ Span: TokenGeneration (Child of Login)
   ├─ Duration: 15ms
   ├─ Status: OK
   └─ Tags: operation=jwt_generation
```

---

#### 4. Mimir (Metrics Storage)

**Role:** Long-term, scalable metrics storage (Prometheus-compatible).

**Key Features:**
- **Horizontal scalability:** Add more nodes for capacity
- **High availability:** Replication and failover
- **Long-term retention:** Years of data
- **PromQL compatible:** Standard Prometheus queries

**Metric Storage:**
```
Metric: api_requests_count_total
Labels: {
  http_method="POST",
  http_route="/api/Login/CrisMAc/SelectLoginDetails",
  http_status_code="200",
  service_name="CrismacEWSBackendService"
}
Values:
  [timestamp=1702898445, value=1547]
  [timestamp=1702898450, value=1552]
  [timestamp=1702898455, value=1560]
```

**Example Query:**
```promql
# Calculate request rate per endpoint
sum by(http_route) (
  rate(api_requests_count_total[5m])
)
```

---

#### 5. Grafana (Visualization)

**Role:** Unified interface for querying and visualizing all telemetry data.

**Key Features:**
- **Multi-datasource:** Query Loki, Tempo, Mimir in one dashboard
- **Explore mode:** Ad-hoc queries and investigation
- **Alerting:** Set thresholds and notifications
- **Correlation:** Click from metrics → traces → logs

**Dashboard Example:**
```
┌────────────────────────────────────────────────┐
│         RED Metrics Dashboard                  │
├────────────────────────────────────────────────┤
│  Request Rate (last 1h)    │  Error Rate      │
│  [Line chart]              │  [Gauge: 0.3%]   │
├────────────────────────────┼──────────────────┤
│  Request Duration (p95)    │  Top Errors      │
│  [Line chart]              │  [Table]         │
└────────────────────────────────────────────────┘
```

---

## OpenTelemetry Framework

### What is OpenTelemetry?

**OpenTelemetry (OTel)** is an open-source observability framework for collecting, processing, and exporting telemetry data.

**Key Principles:**
1. **Vendor Neutral:** Works with any backend
2. **Standardized:** Common API across languages
3. **Comprehensive:** Metrics, logs, and traces
4. **Extensible:** Plugin architecture

---

### OpenTelemetry Architecture

```
┌─────────────────────────────────────────────────┐
│           Your Application                      │
│                                                 │
│  ┌──────────────────────────────────────────┐  │
│  │      OpenTelemetry API                   │  │
│  │  (What developers code against)          │  │
│  └──────────────────┬───────────────────────┘  │
│                     │                           │
│  ┌──────────────────▼───────────────────────┐  │
│  │      OpenTelemetry SDK                   │  │
│  │  (Collects & processes data)             │  │
│  ├──────────────────────────────────────────┤  │
│  │  • ActivitySource (Tracing)              │  │
│  │  • Meter (Metrics)                       │  │
│  │  • Logger (Logs)                         │  │
│  └──────────────────┬───────────────────────┘  │
│                     │                           │
│  ┌──────────────────▼───────────────────────┐  │
│  │      Exporters                           │  │
│  │  (Send data to backends)                 │  │
│  │  • OTLP Exporter                         │  │
│  │  • Console Exporter (dev)                │  │
│  └──────────────────┬───────────────────────┘  │
└─────────────────────┼───────────────────────────┘
                      │
                      ▼
          OpenTelemetry Collector
                      │
           ┌──────────┼──────────┐
           │          │          │
         Loki      Tempo      Mimir
```

---

### Core Concepts

#### 1. Instrumentation

**Automatic Instrumentation:**
- Provided by libraries for common frameworks
- No code changes needed
- Examples: ASP.NET Core, HttpClient, SQL Client

```csharp
// Automatic instrumentation - no manual code!
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddAspNetCoreInstrumentation()  // Auto-trace HTTP requests
        .AddHttpClientInstrumentation()   // Auto-trace HTTP calls
        .AddSqlClientInstrumentation()    // Auto-trace SQL queries
    );
```

**Manual Instrumentation:**
- Custom business logic tracing
- Domain-specific metrics
- Application-specific spans

```csharp
// Manual instrumentation
using var activity = ApplicationTelemetry.ActivitySource.StartActivity("ProcessOrder");
activity?.SetTag("order.id", orderId);
activity?.SetTag("order.total", orderTotal);

// Your business logic here
await ProcessOrder(orderId);

activity?.SetStatus(ActivityStatusCode.Ok);
```

---

#### 2. Resources

**Definition:** Attributes that identify the source of telemetry.

**Purpose:** Know which service, version, environment produced the data.

```csharp
var resourceBuilder = ResourceBuilder
    .CreateDefault()
    .AddService(
        serviceName: "CrismacEWSBackendService",
        serviceVersion: "1.0.0"
    )
    .AddAttributes(new Dictionary<string, object>
    {
        ["deployment.environment"] = "Production",
        ["host.name"] = Environment.MachineName,
        ["process.pid"] = Environment.ProcessId
    });
```

**Benefit:** Filter/aggregate data by service, environment, host, etc.

---

#### 3. Signals

**Traces:**
```csharp
using var activity = activitySource.StartActivity("Operation");
activity?.SetTag("user.id", userId);
activity?.AddEvent(new ActivityEvent("Checkpoint reached"));
```

**Metrics:**
```csharp
var counter = meter.CreateCounter<long>("orders.processed");
counter.Add(1, new KeyValuePair<string, object>("status", "success"));
```

**Logs:**
```csharp
_logger.LogInformation(
    "Order processed: {OrderId}. TraceId: {TraceId}",
    orderId,
    Activity.Current?.TraceId
);
```

---

## The RED Method

The **RED Method** is a monitoring methodology focusing on three golden signals:

### 1. Rate (R) - Request Volume

**What It Measures:** How many requests per second/minute.

**Metric:**
```promql
rate(api_requests_count_total[5m])
```

**Why It Matters:**
- Understand traffic patterns
- Capacity planning
- Detect unusual spikes (attacks, viral content)

**Example:**
```
Normal:  100 req/s at 2 PM
Spike:   500 req/s at 2:05 PM ⚠️
Action:  Check if legitimate traffic or attack
```

---

### 2. Errors (E) - Failure Rate

**What It Measures:** Percentage of failed requests (4xx, 5xx).

**Metric:**
```promql
rate(api_errors_total[5m])
/
rate(api_requests_count_total[5m])
* 100
```

**Why It Matters:**
- Service reliability (SLA)
- User experience impact
- Incident detection

**Example:**
```
Normal:   0.1% error rate
Incident: 5% error rate ⚠️
Action:   Investigate logs and traces
```

---

### 3. Duration (D) - Latency

**What It Measures:** Response time distribution (p50, p95, p99).

**Metric:**
```promql
histogram_quantile(0.95,
  rate(api_requests_duration_milliseconds_bucket[5m])
)
```

**Why It Matters:**
- User experience
- Performance degradation
- Bottleneck identification

**Example:**
```
p50:  100ms (median)
p95:  250ms (95% of requests faster than this)
p99:  500ms (99% of requests faster than this)
p99: 2000ms ⚠️ (1% experiencing slowness)
```

---

### RED Method Dashboard

```
┌────────────────────────────────────────────────────┐
│          RED Metrics - EWS Backend                 │
├────────────────────────────────────────────────────┤
│                                                    │
│  RATE                                              │
│  [Line Chart: Requests/sec over time]              │
│  Current: 125 req/s                                │
│                                                    │
├────────────────────────────────────────────────────┤
│                                                    │
│  ERRORS                                            │
│  [Line Chart: Error rate % over time]              │
│  Current: 0.3% (Target: <1%)                       │
│                                                    │
├────────────────────────────────────────────────────┤
│                                                    │
│  DURATION                                          │
│  [Line Chart: p50/p95/p99 latency]                 │
│  p50: 100ms | p95: 250ms | p99: 500ms              │
│                                                    │
└────────────────────────────────────────────────────┘
```

---

## Correlation Principles

### Why Correlation Matters

Without correlation:
```
Metric Alert: "Error rate spike at 10:30 AM"
   ↓
Manual search through logs: "Search for '10:30'"
   ↓
Find 10,000 log entries
   ↓
Hours of investigation
```

With correlation:
```
Metric Alert: "Error rate spike at 10:30 AM"
   ↓
Filter Traces: Time=10:30, Status=Error
   ↓
Get TraceId: 0af7651916cd43dd8448eb211c80319c
   ↓
Query Logs: TraceId="0af7651916cd43dd8448eb211c80319c"
   ↓
Root cause found in 5 minutes
```

---

### TraceId: The Correlation Key

**What It Is:** Unique identifier for a request journey through the system.

**Format:** W3C Trace Context standard
```
TraceId: 0af7651916cd43dd8448eb211c80319c (32 hex characters)
SpanId:  9d8e7f6a5b4c3d2e (16 hex characters)
```

**How It Works:**
```
1. Request arrives → Generate TraceId
2. Create root span with TraceId
3. Pass TraceId to all logs
4. Pass TraceId to all child spans
5. Include TraceId in error responses
6. Store TraceId in all three pillars
```

---

### Correlation Workflow

```
┌─────────────────────────────────────────────────┐
│           ALERT FIRES                           │
│  "Error rate > 5%"                              │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│        1. CHECK METRICS (What?)                 │
│  Query: api_errors_total{http_status_code=~"5.."}│
│  Result: Errors on /api/Login endpoint         │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│        2. FILTER TRACES (Where?)                │
│  Query: {service="EWS" route="/api/Login"       │
│          status=error time="10:30-10:35"}       │
│  Result: TraceId=0af765...                      │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│        3. VIEW TRACE (How?)                     │
│  Show span waterfall                            │
│  Identify: DB query taking 2.3 seconds          │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│        4. CHECK LOGS (Why?)                     │
│  Query: {service="EWS"}                         │
│         |TraceId="0af765..."                    │
│  Result: "Connection pool exhausted"            │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│         ROOT CAUSE IDENTIFIED                   │
│  "Database connection pool too small"           │
│  Time to resolution: 8 minutes                  │
└─────────────────────────────────────────────────┘
```

---

## Semantic Conventions

### What Are Semantic Conventions?

**Definition:** Standardized naming and tagging conventions for telemetry data defined by OpenTelemetry.

**Purpose:**
- Consistency across services
- Interoperability
- Better tooling support

---

### Naming Conventions

**Resource Attributes:** Describe the entity producing telemetry
```
service.name = "CrismacEWSBackendService"
service.version = "1.0.0"
deployment.environment = "Production"
host.name = "web-server-01"
process.pid = 12345
```

**Span Attributes:** Describe operations
```
http.method = "POST"
http.route = "/api/Login/CrisMAc/SelectLoginDetails"
http.status_code = 200
http.target = "/api/Login/CrisMAc/SelectLoginDetails?retry=true"
```

**Metric Naming:**
```
<namespace>.<metric_name>.<unit>

Examples:
api.requests.count          // Counter
api.requests.duration       // Histogram
ews.login.attempts.total    // Counter
```

---

### Tag Cardinality

**Low Cardinality (Good):**
```
http.method → ["GET", "POST", "PUT", "DELETE"]  // 4 values
http.status_code → [200, 400, 401, 500, ...]    // ~20 values
deployment.environment → ["Dev", "Staging", "Prod"]  // 3 values
```

**High Cardinality (Bad):**
```
user.id → [1, 2, 3, ..., 1000000]              // 1M values ❌
session.id → [uuid1, uuid2, uuid3, ...]        // Infinite ❌
timestamp → [1702898445, 1702898446, ...]      // Infinite ❌
```

**Why It Matters:**
- High cardinality = exponential storage growth
- Queries become slow
- Systems can crash

**Solution:**
```csharp
// ❌ BAD: High cardinality
activity?.SetTag("user.id", userId);  // Millions of unique values

// ✅ GOOD: Low cardinality
activity?.SetTag("user.type", "premium");  // Few unique values
```

---

## Observability vs Monitoring

### Key Differences

| Aspect | Monitoring | Observability |
|--------|-----------|---------------|
| **Focus** | Known problems | Unknown problems |
| **Approach** | Dashboards + Alerts | Exploration + Investigation |
| **Questions** | "Is it up?" | "Why is it behaving this way?" |
| **Data** | Pre-aggregated | Raw + Granular |
| **Response** | Reactive | Proactive |
| **Complexity** | Simple systems | Complex distributed systems |

---

### Monitoring Example

```
┌──────────────────────────────────────┐
│      Traditional Monitoring          │
├──────────────────────────────────────┤
│  CPU: 45% ✓                          │
│  Memory: 2.3GB / 8GB ✓               │
│  Disk: 120GB / 500GB ✓               │
│  Uptime: 99.9% ✓                     │
│                                      │
│  Alert: Response time > 1s ⚠️         │
└──────────────────────────────────────┘

Question: "Why is response time > 1s?"
Answer: ¯\_(ツ)_/¯ (Need to dig into logs)
```

---

### Observability Example

```
┌──────────────────────────────────────┐
│      Modern Observability            │
├──────────────────────────────────────┤
│  Metric: p95 latency = 1.2s ⚠️        │
│    ↓                                 │
│  Trace: Login request (1.2s)         │
│    ├─ DB Query (1.1s) ⚠️              │
│    └─ Token Gen (100ms) ✓            │
│    ↓                                 │
│  Log: "Slow query: SELECT Users..."  │
│    TraceId: 0af765...                │
│                                      │
│  Root Cause: Missing index           │
└──────────────────────────────────────┘

Question: "Why is response time > 1s?"
Answer: "DB query slow due to missing index on Users.Username"
Time to answer: 5 minutes
```

---

## Summary

### Key Takeaways

1. **Observability enables understanding system behavior** through external outputs
2. **Three pillars work together:** Metrics (what), Logs (why), Traces (where)
3. **LGTM Stack provides integrated solution:** Loki, Grafana, Tempo, Mimir
4. **OpenTelemetry standardizes data collection** across languages and vendors
5. **RED Method focuses on golden signals:** Rate, Errors, Duration
6. **Correlation via TraceId** links all telemetry data together
7. **Semantic conventions ensure consistency** and interoperability

---

### Next Steps

- **Developers:** Read [Development Guide](./02-development-guide.md)
- **Operations:** Read [Deployment Guide](./04-deployment-guide.md)
- **Business Leaders:** Read [Business Value Guide](./05-business-stakeholder-guide.md)

---

**Remember:** Observability is not a tool, it's a capability. The goal is to answer any question about your system's behavior using telemetry data.
