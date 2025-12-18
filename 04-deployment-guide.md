# Deployment Guide: Operations and Infrastructure

## Complete Deployment Reference for EWS Core API Observability

**Audience:** DevOps engineers, SREs, infrastructure teams
**Reading Time:** 90-120 minutes
**Prerequisites:** Docker, .NET Core, basic networking knowledge

---

## Table of Contents

1. [Local Development Setup](#local-development-setup)
2. [Docker Deployment](#docker-deployment)
3. [Production Configuration](#production-configuration)
4. [Environment Variables](#environment-variables)
5. [Security Considerations](#security-considerations)
6. [Scaling Strategies](#scaling-strategies)
7. [Backup and Recovery](#backup-and-recovery)
8. [Performance Tuning](#performance-tuning)
9. [Multi-Environment Strategy](#multi-environment-strategy)
10. [Troubleshooting](#troubleshooting)

---

## Local Development Setup

### Prerequisites

**Required Software:**
- Docker Desktop (Windows/Mac) or Docker Engine (Linux)
- .NET 8.0 SDK
- Visual Studio 2022 or VS Code
- Git

**Hardware Requirements:**
- 8 GB RAM minimum (16 GB recommended)
- 10 GB free disk space
- Multi-core processor

---

### Step 1: Clone Repository

```bash
git clone <repository-url>
cd ews_coreapi_enterprises
```

---

### Step 2: Start LGTM Stack

```bash
cd Crismac.Ews.WebApi

# Start all observability infrastructure
docker-compose -f otel-lgtm-docker-compose.yaml up -d

# Verify containers are running
docker ps
```

**Expected Output:**
```
CONTAINER ID   IMAGE                     STATUS         PORTS
abc123def456   grafana/otel-lgtm:latest  Up 2 minutes   0.0.0.0:3000->3000/tcp,
                                                        0.0.0.0:4317->4317/tcp,
                                                        0.0.0.0:4318->4318/tcp
```

---

### Step 3: Configure Application

**appsettings.Development.json:**
```json
{
  "OpenTelemetry": {
    "ServiceName": "CrismacEWSBackendService",
    "ServiceVersion": "1.0.0",
    "OtlpEndpoint": "http://localhost:4317"
  },
  "Serilog": {
    "WriteTo": [
      {
        "Name": "Console"
      },
      {
        "Name": "File",
        "Args": {
          "path": "Logs/app-log-.json",
          "rollingInterval": "Day",
          "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
        }
      },
      {
        "Name": "OpenTelemetry",
        "Args": {
          "endpoint": "http://localhost:4318/v1/logs",
          "protocol": "HttpProtobuf",
          "resourceAttributes": {
            "service.name": "CrismacEWSBackendService",
            "service.version": "1.0.0",
            "deployment.environment": "Development"
          }
        }
      }
    ]
  }
}
```

---

### Step 4: Run Application

```bash
# Restore packages
dotnet restore

# Run application
dotnet run --project Crismac.Ews.WebApi

# Or use Visual Studio
# Press F5 to start debugging
```

**Application URLs:**
- HTTP: `http://localhost:5294`
- HTTPS: `https://localhost:7294`
- Swagger: `https://localhost:7294/swagger`

---

### Step 5: Access Grafana

```
URL: http://localhost:3000
Username: admin
Password: Qwerty!123
```

**Initial Setup:**
1. Login with credentials
2. Navigate to "Explore"
3. Verify datasources:
   - Loki (Logs)
   - Tempo (Traces)
   - Mimir (Metrics)

---

### Step 6: Import Dashboards

1. Click **"+" → "Import dashboard"**
2. Click **"Upload JSON file"**
3. Navigate to `Crismac.Ews.WebApi/Dashboards/`
4. Import:
   - `red-metrics-dashboard.json`
   - `login-analytics-dashboard.json`
5. Select **"Mimir"** as datasource
6. Click **"Import"**

---

### Verification Checklist

- [ ] Docker container running
- [ ] Application starts without errors
- [ ] Grafana accessible at http://localhost:3000
- [ ] Metrics visible in Mimir (query: `api_requests_count_total`)
- [ ] Traces visible in Tempo
- [ ] Logs visible in Loki
- [ ] Dashboards imported successfully

---

## Docker Deployment

### Docker Compose Configuration

**File:** `otel-lgtm-docker-compose.yaml`

```yaml
version: '3.8'

services:
  grafana-lgtm:
    image: grafana/otel-lgtm:latest
    container_name: grafana-lgtm
    ports:
      - "3000:3000"   # Grafana UI
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP HTTP receiver
      - "9090:9090"   # Prometheus (optional)

    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=Qwerty!123
      - GF_INSTALL_PLUGINS=

    volumes:
      - grafana-data:/data/grafana
      - ./Dashboards:/etc/grafana/provisioning/dashboards

    networks:
      - observability

    restart: unless-stopped

    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

volumes:
  grafana-data:
    driver: local

networks:
  observability:
    driver: bridge
```

---

### Container Management

**Start Stack:**
```bash
docker-compose -f otel-lgtm-docker-compose.yaml up -d
```

**View Logs:**
```bash
docker-compose logs -f grafana-lgtm
```

**Stop Stack:**
```bash
docker-compose down
```

**Stop and Remove Data:**
```bash
docker-compose down -v
```

**Restart Stack:**
```bash
docker-compose restart grafana-lgtm
```

**View Resource Usage:**
```bash
docker stats grafana-lgtm
```

---

### Production Docker Setup

**Production docker-compose.yml:**

```yaml
version: '3.8'

services:
  # Application
  ews-api:
    build:
      context: .
      dockerfile: Dockerfile
    image: crismac/ews-api:${VERSION:-latest}
    container_name: ews-api-prod
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ASPNETCORE_URLS=http://+:8080
      - OpenTelemetry__OtlpEndpoint=http://otel-collector:4317
    depends_on:
      - otel-collector
    networks:
      - app-network
    restart: unless-stopped

  # OpenTelemetry Collector
  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector-prod
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Metrics endpoint
    networks:
      - app-network
      - observability
    restart: unless-stopped

  # Loki
  loki:
    image: grafana/loki:latest
    container_name: loki-prod
    ports:
      - "3100:3100"
    volumes:
      - loki-data:/loki
      - ./loki-config.yaml:/etc/loki/loki-config.yaml
    command: -config.file=/etc/loki/loki-config.yaml
    networks:
      - observability
    restart: unless-stopped

  # Tempo
  tempo:
    image: grafana/tempo:latest
    container_name: tempo-prod
    ports:
      - "3200:3200"   # Tempo HTTP
      - "4317"        # OTLP gRPC (internal)
    volumes:
      - tempo-data:/var/tempo
      - ./tempo-config.yaml:/etc/tempo.yaml
    command: -config.file=/etc/tempo.yaml
    networks:
      - observability
    restart: unless-stopped

  # Mimir
  mimir:
    image: grafana/mimir:latest
    container_name: mimir-prod
    ports:
      - "9009:9009"
    volumes:
      - mimir-data:/data
      - ./mimir-config.yaml:/etc/mimir.yaml
    command: -config.file=/etc/mimir.yaml
    networks:
      - observability
    restart: unless-stopped

  # Grafana
  grafana:
    image: grafana/grafana:latest
    container_name: grafana-prod
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=https://monitoring.yourdomain.com
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    depends_on:
      - loki
      - tempo
      - mimir
    networks:
      - observability
    restart: unless-stopped

volumes:
  loki-data:
  tempo-data:
  mimir-data:
  grafana-data:

networks:
  app-network:
    driver: bridge
  observability:
    driver: bridge
```

---

## Production Configuration

### Application Configuration

**appsettings.Production.json:**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "System": "Warning"
    }
  },
  "OpenTelemetry": {
    "ServiceName": "CrismacEWSBackendService",
    "ServiceVersion": "1.2.0",
    "OtlpEndpoint": "http://otel-collector:4317"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "/var/log/ews/app-log-.json",
          "rollingInterval": "Day",
          "retainedFileCountLimit": 30,
          "fileSizeLimitBytes": 104857600,
          "formatter": "Serilog.Formatting.Compact.CompactJsonFormatter, Serilog.Formatting.Compact"
        }
      },
      {
        "Name": "OpenTelemetry",
        "Args": {
          "endpoint": "http://otel-collector:4318/v1/logs",
          "protocol": "HttpProtobuf",
          "resourceAttributes": {
            "service.name": "CrismacEWSBackendService",
            "service.version": "1.2.0",
            "deployment.environment": "Production"
          }
        }
      }
    ]
  }
}
```

---

### OpenTelemetry Collector Configuration

**otel-collector-config.yaml:**

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

  memory_limiter:
    check_interval: 1s
    limit_mib: 512
    spike_limit_mib: 128

  attributes:
    actions:
      - key: deployment.environment
        value: production
        action: upsert

exporters:
  loki:
    endpoint: http://loki:3100/loki/api/v1/push

  otlp/tempo:
    endpoint: tempo:4317
    tls:
      insecure: true

  prometheusremotewrite:
    endpoint: http://mimir:9009/api/v1/push
    tls:
      insecure: true

  logging:
    loglevel: info

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch, attributes]
      exporters: [otlp/tempo, logging]

    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch, attributes]
      exporters: [prometheusremotewrite, logging]

    logs:
      receivers: [otlp]
      processors: [memory_limiter, batch, attributes]
      exporters: [loki, logging]

  telemetry:
    metrics:
      address: :8888
```

---

### Loki Configuration

**loki-config.yaml:**

```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 30d
  ingestion_rate_mb: 10
  ingestion_burst_size_mb: 20
  max_query_series: 10000
```

---

### Tempo Configuration

**tempo-config.yaml:**

```yaml
server:
  http_listen_port: 3200

distributor:
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317

ingester:
  max_block_duration: 5m

compactor:
  compaction:
    block_retention: 168h  # 7 days

storage:
  trace:
    backend: local
    local:
      path: /var/tempo/traces
    wal:
      path: /var/tempo/wal

query_frontend:
  search:
    max_duration: 24h
```

---

## Environment Variables

### Application Environment Variables

**Production .env file:**

```bash
# Application
ASPNETCORE_ENVIRONMENT=Production
ASPNETCORE_URLS=http://+:8080

# OpenTelemetry
OTEL_EXPORTER_OTLP_ENDPOINT=http://otel-collector:4317
OTEL_EXPORTER_OTLP_PROTOCOL=grpc
OTEL_SERVICE_NAME=CrismacEWSBackendService
OTEL_SERVICE_VERSION=1.2.0
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,host.name=${HOSTNAME}

# Metrics
OTEL_EXPORTER_OTLP_METRICS_DEFAULT_HISTOGRAM_AGGREGATION=explicit_bucket_histogram
OTEL_SEMCONV_STABILITY_OPT_IN=traces,metrics,logs

# Grafana
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=${GRAFANA_PASSWORD}

# Database (if needed)
ConnectionStrings__Default=${DB_CONNECTION_STRING}
```

---

### OpenTelemetry Environment Variables Reference

| Variable | Purpose | Default | Example |
|----------|---------|---------|---------|
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Collector endpoint | `http://localhost:4317` | `http://otel-collector:4317` |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | Protocol | `grpc` | `grpc` or `http/protobuf` |
| `OTEL_SERVICE_NAME` | Service name | (none) | `CrismacEWSBackendService` |
| `OTEL_SERVICE_VERSION` | Service version | (none) | `1.2.0` |
| `OTEL_RESOURCE_ATTRIBUTES` | Additional attributes | (none) | `deployment.environment=prod` |
| `OTEL_TRACES_SAMPLER` | Sampling strategy | `parentbased_always_on` | `parentbased_always_on` |
| `OTEL_TRACES_SAMPLER_ARG` | Sampler argument | (none) | `0.1` (10% sampling) |

---

## Security Considerations

### 1. Authentication and Authorization

**Grafana Security:**

```yaml
# In grafana environment
- GF_AUTH_ANONYMOUS_ENABLED=false
- GF_AUTH_BASIC_ENABLED=true
- GF_AUTH_DISABLE_LOGIN_FORM=false
- GF_USERS_ALLOW_SIGN_UP=false
- GF_USERS_ALLOW_ORG_CREATE=false

# Enable OAuth (if using Azure AD, Google, etc.)
- GF_AUTH_GENERIC_OAUTH_ENABLED=true
- GF_AUTH_GENERIC_OAUTH_CLIENT_ID=${OAUTH_CLIENT_ID}
- GF_AUTH_GENERIC_OAUTH_CLIENT_SECRET=${OAUTH_CLIENT_SECRET}
```

---

### 2. Network Security

**Firewall Rules:**

```bash
# Only allow internal network access to observability stack
# Allow: Application → Collector
Allow TCP 4317 from 10.0.1.0/24

# Allow: Grafana → Datasources
Allow TCP 3100,3200,9009 from 10.0.2.100

# Block: External → Grafana (use reverse proxy)
Deny TCP 3000 from 0.0.0.0/0
```

**Reverse Proxy (Nginx):**

```nginx
server {
    listen 443 ssl;
    server_name monitoring.yourdomain.com;

    ssl_certificate /etc/ssl/certs/monitoring.crt;
    ssl_certificate_key /etc/ssl/private/monitoring.key;

    location / {
        proxy_pass http://grafana:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

### 3. Data Encryption

**TLS for OTLP:**

```yaml
# In otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
        tls:
          cert_file: /etc/certs/collector.crt
          key_file: /etc/certs/collector.key
          client_ca_file: /etc/certs/ca.crt
```

**Application TLS Configuration:**

```csharp
// In Program.cs
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .AddOtlpExporter(options =>
        {
            options.Endpoint = new Uri("https://otel-collector:4317");
            options.Protocol = OtlpExportProtocol.Grpc;
            // Configure TLS if needed
        })
    );
```

---

### 4. Sensitive Data Protection

**Scrub Sensitive Data:**

```csharp
// Create custom processor to scrub sensitive data
public class SensitiveDataProcessor : BaseProcessor<Activity>
{
    public override void OnEnd(Activity activity)
    {
        // Remove sensitive tags
        var tagsToRemove = new[] { "password", "api_key", "token", "credit_card" };

        foreach (var tag in activity.Tags)
        {
            if (tagsToRemove.Any(t => tag.Key.Contains(t, StringComparison.OrdinalIgnoreCase)))
            {
                activity.SetTag(tag.Key, "[REDACTED]");
            }
        }
    }
}

// Register in Program.cs
.WithTracing(tracing => tracing
    .AddProcessor(new SensitiveDataProcessor())
);
```

---

### 5. Access Control

**Grafana RBAC:**

```yaml
# Define roles and permissions
- name: Viewer
  permissions:
    - action: "dashboards:read"
    - action: "datasources:read"

- name: Editor
  permissions:
    - action: "dashboards:read"
    - action: "dashboards:write"
    - action: "datasources:read"

- name: Admin
  permissions:
    - action: "*"
```

---

## Scaling Strategies

### Horizontal Scaling

**Application Scaling:**

```yaml
# In production docker-compose.yml or Kubernetes
services:
  ews-api:
    image: crismac/ews-api:latest
    deploy:
      replicas: 3  # Scale to 3 instances
      resources:
        limits:
          cpus: '1'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 1G
```

---

### OpenTelemetry Collector Scaling

**Multi-Collector Setup:**

```yaml
# Load balancer for collectors
services:
  otel-collector-1:
    image: otel/opentelemetry-collector-contrib:latest
    # ... config

  otel-collector-2:
    image: otel/opentelemetry-collector-contrib:latest
    # ... config

  otel-lb:
    image: nginx:alpine
    volumes:
      - ./nginx-otel-lb.conf:/etc/nginx/nginx.conf
    ports:
      - "4317:4317"
      - "4318:4318"
```

**nginx-otel-lb.conf:**

```nginx
upstream otel_grpc {
    least_conn;
    server otel-collector-1:4317;
    server otel-collector-2:4317;
}

upstream otel_http {
    least_conn;
    server otel-collector-1:4318;
    server otel-collector-2:4318;
}

server {
    listen 4317 http2;
    location / {
        grpc_pass grpc://otel_grpc;
    }
}

server {
    listen 4318;
    location / {
        proxy_pass http://otel_http;
    }
}
```

---

### Storage Scaling

**Loki Scaling:**

```yaml
# Microservices mode
services:
  loki-distributor:
    image: grafana/loki:latest
    command: -target=distributor -config.file=/etc/loki/config.yaml

  loki-ingester:
    image: grafana/loki:latest
    command: -target=ingester -config.file=/etc/loki/config.yaml
    deploy:
      replicas: 3

  loki-querier:
    image: grafana/loki:latest
    command: -target=querier -config.file=/etc/loki/config.yaml
    deploy:
      replicas: 2
```

---

## Backup and Recovery

### Data Backup Strategy

**Grafana Backup:**

```bash
#!/bin/bash
# backup-grafana.sh

BACKUP_DIR="/backups/grafana"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Backup Grafana database
docker exec grafana-prod grafana-cli admin export-dash --dashboard-ids all > \
  ${BACKUP_DIR}/dashboards_${TIMESTAMP}.json

# Backup Grafana volume
docker run --rm \
  -v grafana-data:/source:ro \
  -v ${BACKUP_DIR}:/backup \
  alpine tar czf /backup/grafana-data_${TIMESTAMP}.tar.gz -C /source .

# Cleanup old backups (keep last 30 days)
find ${BACKUP_DIR} -mtime +30 -delete

echo "Backup completed: ${TIMESTAMP}"
```

---

**Loki Backup:**

```bash
#!/bin/bash
# backup-loki.sh

BACKUP_DIR="/backups/loki"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Backup Loki data
docker run --rm \
  -v loki-data:/source:ro \
  -v ${BACKUP_DIR}:/backup \
  alpine tar czf /backup/loki-data_${TIMESTAMP}.tar.gz -C /source .

# Cleanup old backups
find ${BACKUP_DIR} -mtime +7 -delete

echo "Loki backup completed: ${TIMESTAMP}"
```

---

### Disaster Recovery

**Recovery Procedure:**

```bash
#!/bin/bash
# restore-grafana.sh

BACKUP_FILE=$1

if [ -z "$BACKUP_FILE" ]; then
  echo "Usage: ./restore-grafana.sh <backup_file>"
  exit 1
fi

# Stop Grafana
docker-compose stop grafana

# Restore volume
docker run --rm \
  -v grafana-data:/target \
  -v /backups/grafana:/backup \
  alpine sh -c "cd /target && tar xzf /backup/${BACKUP_FILE}"

# Start Grafana
docker-compose start grafana

echo "Restore completed"
```

---

## Performance Tuning

### Application Performance

**Sampling Configuration:**

```csharp
// In Program.cs - Production sampling
builder.Services.AddOpenTelemetry()
    .WithTracing(tracing => tracing
        .SetSampler(new TraceIdRatioBasedSampler(0.1)) // Sample 10% of traces
        .AddAspNetCoreInstrumentation()
        .AddOtlpExporter()
    );
```

**Batch Export Settings:**

```yaml
# In otel-collector-config.yaml
processors:
  batch:
    timeout: 10s
    send_batch_size: 1024
    send_batch_max_size: 2048
```

---

### Resource Limits

**Docker Resource Limits:**

```yaml
services:
  grafana-lgtm:
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G
```

---

### Query Optimization

**Grafana Query Tips:**

```promql
# BAD: Unbounded time range
rate(api_requests_count_total[5m])

# GOOD: Limited time range
rate(api_requests_count_total[5m] offset 1h)

# BAD: High cardinality aggregation
sum by (user_id) (api_requests_count_total)

# GOOD: Low cardinality aggregation
sum by (http_route) (api_requests_count_total)
```

---

## Multi-Environment Strategy

### Environment Configurations

| Environment | Purpose | Retention | Sampling |
|-------------|---------|-----------|----------|
| **Development** | Local testing | 7 days | 100% |
| **Staging** | Pre-production | 14 days | 50% |
| **Production** | Live system | 30 days | 10% |

---

### Environment-Specific Settings

**Development:**
```yaml
OTEL_TRACES_SAMPLER=always_on
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=development
```

**Staging:**
```yaml
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.5
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=staging
```

**Production:**
```yaml
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.1
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production
```

---

## Troubleshooting

### Common Issues

#### Issue: Containers Won't Start

**Symptoms:**
- Docker compose fails
- Containers exit immediately

**Diagnosis:**
```bash
docker-compose logs grafana-lgtm
docker inspect grafana-lgtm
```

**Solutions:**
- Check port conflicts: `netstat -ano | findstr :3000`
- Verify Docker resources
- Check volume permissions

---

#### Issue: No Data in Grafana

**Symptoms:**
- Dashboards show "No data"
- Queries return empty

**Diagnosis:**
```bash
# Check if data is reaching collector
curl http://localhost:8888/metrics

# Check application logs
docker-compose logs ews-api | grep -i "export"
```

**Solutions:**
- Verify OTLP endpoint connectivity
- Check firewall rules
- Verify datasource configuration in Grafana

---

#### Issue: High Memory Usage

**Symptoms:**
- Container restarts
- Out of memory errors

**Diagnosis:**
```bash
docker stats grafana-lgtm
```

**Solutions:**
- Increase memory limits
- Enable batch processing
- Reduce retention period
- Increase sampling rate

---

## Next Steps

1. **Deploy** to development environment first
2. **Test** all functionality
3. **Monitor** resource usage
4. **Scale** as needed
5. **Document** any customizations

---

**Remember:** Always test in non-production environments first!
