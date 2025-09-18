# PSW_MONITOR - Monitoring Infrastructure

## Overview
PSW_MONITOR provides comprehensive system monitoring, alerting, and observability for the entire PSW5 ecosystem using Prometheus, Grafana, and custom monitoring services.

## Technology Stack
- **Monitoring**: Prometheus 2.40+
- **Visualization**: Grafana 9.0+
- **Alerting**: Alertmanager
- **Service**: Python FastAPI
- **Metrics**: Custom exporters
- **Notifications**: Telegram, Email, Slack

## Infrastructure Architecture

### Container Structure
```dockerfile
FROM python:3.11-alpine

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy application
COPY . .

# Expose metrics port
EXPOSE 8001

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8001"]
```

### Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'psw-core'
    static_configs:
      - targets: ['psw_core:8000']
    metrics_path: '/metrics'
    scrape_interval: 30s

  - job_name: 'psw-web'
    static_configs:
      - targets: ['psw_web:3000']
    metrics_path: '/metrics'

  - job_name: 'psw-monitor'
    static_configs:
      - targets: ['psw_monitor:8001']

  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql:3306']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']
```

## Custom Metrics Implementation

### Application Metrics
```python
# metrics.py
from prometheus_client import Counter, Histogram, Gauge, generate_latest
import time

# Request metrics
REQUEST_COUNT = Counter(
    'psw_http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_DURATION = Histogram(
    'psw_http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

# Business metrics
PORTFOLIO_COUNT = Gauge(
    'psw_portfolios_total',
    'Total number of portfolios'
)

ACTIVE_USERS = Gauge(
    'psw_active_users',
    'Number of active users'
)

STOCK_PRICE_UPDATES = Counter(
    'psw_stock_price_updates_total',
    'Total stock price updates'
)

# Database metrics
DB_CONNECTIONS = Gauge(
    'psw_db_connections_active',
    'Active database connections'
)

DB_QUERY_DURATION = Histogram(
    'psw_db_query_duration_seconds',
    'Database query duration',
    ['query_type']
)
```

### Custom Exporter
```python
# exporter.py
from fastapi import FastAPI
from prometheus_client import generate_latest, CONTENT_TYPE_LATEST
import asyncio
import logging

app = FastAPI(title="PSW Monitor")

class PSWExporter:
    def __init__(self):
        self.logger = logging.getLogger(__name__)

    async def collect_business_metrics(self):
        """Collect PSW-specific business metrics"""
        try:
            # Portfolio metrics
            portfolio_count = await self.get_portfolio_count()
            PORTFOLIO_COUNT.set(portfolio_count)

            # User metrics
            active_users = await self.get_active_users()
            ACTIVE_USERS.set(active_users)

            # System health
            await self.check_service_health()

        except Exception as e:
            self.logger.error(f"Metrics collection failed: {e}")

    async def get_portfolio_count(self):
        # Query PSW Core API
        async with aiohttp.ClientSession() as session:
            async with session.get('http://psw_core:8000/api/v1/metrics/portfolios') as resp:
                data = await resp.json()
                return data['count']

@app.get('/metrics')
async def metrics():
    exporter = PSWExporter()
    await exporter.collect_business_metrics()
    return Response(generate_latest(), media_type=CONTENT_TYPE_LATEST)

@app.get('/health')
async def health():
    return {"status": "healthy", "timestamp": time.time()}
```

## Alert Rules Configuration

### Critical Alerts
```yaml
# alert_rules.yml
groups:
  - name: psw_critical
    rules:
      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is down"
          description: "{{ $labels.job }} has been down for more than 1 minute"

      - alert: HighErrorRate
        expr: rate(psw_http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} requests/sec"

      - alert: DatabaseConnections
        expr: psw_db_connections_active > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High database connection usage"
          description: "Database connections: {{ $value }}"

  - name: psw_business
    rules:
      - alert: StockDataStale
        expr: time() - psw_stock_last_update > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Stock data is stale"
          description: "Stock prices haven't been updated for over 1 hour"
```

## Grafana Dashboard Configuration

### Dashboard JSON
```json
{
  "dashboard": {
    "title": "PSW5 System Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(psw_http_requests_total[5m])",
            "legendFormat": "{{ job }}"
          }
        ]
      },
      {
        "title": "Response Time",
        "type": "graph",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(psw_http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          }
        ]
      },
      {
        "title": "Active Portfolios",
        "type": "singlestat",
        "targets": [
          {
            "expr": "psw_portfolios_total"
          }
        ]
      }
    ]
  }
}
```

## Environment Configuration

### Required Variables
```env
# Prometheus Configuration
PROMETHEUS_URL=http://prometheus:9090
PROMETHEUS_RETENTION=30d

# Grafana Configuration
GRAFANA_ADMIN_PASSWORD=secure_password
GRAFANA_DOMAIN=grafana.yourdomain.com

# Alerting
ALERTMANAGER_URL=http://alertmanager:9093
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# Email Alerts
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=your_email
SMTP_PASSWORD=your_app_password

# PSW Core Integration
PSW_CORE_API_URL=http://psw_core:8000/api/v1
MONITOR_API_KEY=your_monitor_api_key
```

## Deployment Instructions

### Development Setup
```bash
# 1. Start monitoring stack
docker-compose up -d prometheus grafana alertmanager

# 2. Access interfaces
open http://localhost:9090  # Prometheus
open http://localhost:3001  # Grafana (admin/admin)
open http://localhost:9093  # Alertmanager

# 3. Import dashboards
curl -X POST http://admin:admin@localhost:3001/api/dashboards/db \
  -H "Content-Type: application/json" \
  -d @grafana/psw-dashboard.json
```

### Production Deployment
```bash
# 1. Deploy with persistent storage
docker-compose -f docker-compose.prod.yml up -d

# 2. Configure SSL
certbot --nginx -d grafana.yourdomain.com

# 3. Set up backup
./scripts/backup-monitoring.sh
```

## Alerting Configuration

### Alertmanager Setup
```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@yourdomain.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
  - name: 'web.hook'
    email_configs:
      - to: 'admin@yourdomain.com'
        subject: 'PSW5 Alert: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          {{ end }}

    telegram_configs:
      - bot_token: '{{ .telegram_bot_token }}'
        chat_id: {{ .telegram_chat_id }}
        message: |
          🚨 PSW5 Alert
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          {{ end }}
```

## Performance Monitoring

### Custom Middleware
```python
# middleware.py
import time
from fastapi import Request

@app.middleware("http")
async def monitor_requests(request: Request, call_next):
    start_time = time.time()

    response = await call_next(request)

    duration = time.time() - start_time

    # Record metrics
    REQUEST_COUNT.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    REQUEST_DURATION.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)

    return response
```

## Troubleshooting

### Common Issues

**Metrics Not Appearing**
```bash
# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# Verify service metrics endpoint
curl http://psw_core:8000/metrics
```

**Alerts Not Firing**
```bash
# Check alert rules
curl http://localhost:9090/api/v1/rules

# Test alertmanager
curl -X POST http://localhost:9093/api/v1/alerts
```

**Grafana Dashboard Issues**
```bash
# Check Grafana logs
docker logs psw_grafana

# Verify data source
curl -u admin:admin http://localhost:3001/api/datasources
```

## Backup & Recovery

### Monitoring Data Backup
```bash
#!/bin/bash
# backup-monitoring.sh

# Backup Prometheus data
docker exec psw_prometheus tar czf /tmp/prometheus-backup.tar.gz /prometheus

# Backup Grafana dashboards
curl -u admin:admin http://localhost:3001/api/search | \
  jq -r '.[] | select(.type=="dash-db") | .uid' | \
  xargs -I {} curl -u admin:admin http://localhost:3001/api/dashboards/uid/{} > grafana-backup.json
```

---

*Infrastructure documentation for PSW_MONITOR Monitoring Service*