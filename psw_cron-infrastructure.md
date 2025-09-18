# PSW_CRON - Background Processing Infrastructure

## Overview
PSW_CRON handles all scheduled tasks and background processing for the PSW5 ecosystem using Python, Celery, and Redis for reliable job execution.

## Technology Stack
- **Language**: Python 3.11
- **Task Queue**: Celery 5.3
- **Broker**: Redis
- **Scheduler**: Celery Beat
- **Monitoring**: Flower
- **Database**: SQLAlchemy (for result backend)

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

# Create celery user
RUN adduser -D celery
USER celery

# Start celery worker
CMD ["celery", "-A", "psw_cron", "worker", "--loglevel=info"]
```

### Celery Configuration
```python
# celery_config.py
from celery import Celery
from celery.schedules import crontab

celery_app = Celery("psw_cron")

celery_app.conf.update(
    broker_url="redis://:password@redis:6379/0",
    result_backend="redis://:password@redis:6379/0",
    task_serializer="json",
    accept_content=["json"],
    result_serializer="json",
    timezone="UTC",
    enable_utc=True,
)

# Scheduled tasks
celery_app.conf.beat_schedule = {
    'update-stock-prices': {
        'task': 'psw_cron.tasks.update_stock_prices',
        'schedule': crontab(minute='*/15'),  # Every 15 minutes
    },
    'calculate-portfolio-values': {
        'task': 'psw_cron.tasks.calculate_portfolio_values',
        'schedule': crontab(minute='0', hour='*/6'),  # Every 6 hours
    },
    'generate-daily-reports': {
        'task': 'psw_cron.tasks.generate_daily_reports',
        'schedule': crontab(hour=8, minute=0),  # Daily at 8 AM
    },
    'cleanup-old-data': {
        'task': 'psw_cron.tasks.cleanup_old_data',
        'schedule': crontab(hour=2, minute=0, day_of_week=0),  # Weekly
    },
}
```

### Task Implementation
```python
# tasks/stock_tasks.py
from celery import current_task
import requests
import logging

@celery_app.task(bind=True, max_retries=3)
def update_stock_prices(self):
    """Update stock prices from external API"""
    try:
        # Get all stocks that need updating
        stocks = get_stocks_for_update()

        for stock in stocks:
            try:
                price_data = fetch_stock_price(stock.symbol)
                update_stock_in_database(stock.symbol, price_data)

                # Update task progress
                current_task.update_state(
                    state='PROGRESS',
                    meta={'current': stocks.index(stock), 'total': len(stocks)}
                )

            except Exception as e:
                logging.error(f"Failed to update {stock.symbol}: {e}")
                continue

        return {'status': 'completed', 'updated': len(stocks)}

    except Exception as exc:
        logging.error(f"Stock update task failed: {exc}")
        raise self.retry(countdown=60, exc=exc)

@celery_app.task
def calculate_portfolio_values():
    """Recalculate all portfolio values"""
    portfolios = get_all_portfolios()

    for portfolio in portfolios:
        total_value = 0
        for holding in portfolio.holdings:
            current_price = get_current_stock_price(holding.stock_symbol)
            holding_value = holding.quantity * current_price
            total_value += holding_value

        update_portfolio_value(portfolio.id, total_value)

    return {'portfolios_updated': len(portfolios)}
```

## Monitoring & Management

### Flower Dashboard
```python
# flower_config.py
from flower import Flower

app = Flower(
    broker_url="redis://:password@redis:6379/0",
    port=5555,
    basic_auth=['admin:password']
)

# Custom monitoring
@app.task
def monitor_task_health():
    """Monitor task execution health"""
    stats = celery_app.control.inspect().stats()
    return {
        'active_tasks': sum(worker['total'] for worker in stats.values()),
        'failed_tasks': get_failed_task_count(),
        'worker_status': 'healthy' if stats else 'degraded'
    }
```

### Task Monitoring
```python
# monitoring.py
class TaskMonitor:
    def __init__(self):
        self.redis_client = redis.Redis(host='redis', port=6379, db=1)

    def log_task_execution(self, task_name, duration, status):
        """Log task execution metrics"""
        timestamp = int(time.time())

        # Store in Redis for monitoring
        self.redis_client.zadd(
            f"task_metrics:{task_name}",
            {f"{timestamp}:{status}": duration}
        )

        # Keep only last 1000 entries
        self.redis_client.zremrangebyrank(
            f"task_metrics:{task_name}", 0, -1001
        )
```

## Environment Configuration

### Required Variables
```env
# Celery Configuration
CELERY_BROKER_URL=redis://:password@redis:6379/0
CELERY_RESULT_BACKEND=redis://:password@redis:6379/0
CELERY_WORKER_CONCURRENCY=4

# Database
DATABASE_URL=mysql://user:password@mysql:3306/psw5_core

# External APIs
BORSDATA_API_KEY=your_api_key
EODHD_API_KEY=your_api_key

# Notifications
TELEGRAM_BOT_TOKEN=your_bot_token
EMAIL_SMTP_HOST=smtp.gmail.com
EMAIL_SMTP_PORT=587

# Monitoring
FLOWER_BASIC_AUTH=admin:secure_password
SENTRY_DSN=your_sentry_dsn
```

## Deployment & Scaling

### Docker Compose Integration
```yaml
# docker-compose.yml (excerpt)
psw_cron_worker:
  build: ../psw_cron
  command: celery -A psw_cron worker --loglevel=info --concurrency=4
  depends_on:
    - redis
    - mysql
  environment:
    - CELERY_BROKER_URL=redis://:${REDIS_PASSWORD}@redis:6379/0

psw_cron_beat:
  build: ../psw_cron
  command: celery -A psw_cron beat --loglevel=info
  depends_on:
    - redis
  volumes:
    - ./celerybeat-schedule:/app/celerybeat-schedule

psw_cron_flower:
  build: ../psw_cron
  command: celery flower --broker=redis://:${REDIS_PASSWORD}@redis:6379/0
  ports:
    - "5555:5555"
  depends_on:
    - redis
```

## Error Handling & Reliability

### Retry Logic
```python
@celery_app.task(bind=True, max_retries=3, default_retry_delay=60)
def resilient_task(self, data):
    try:
        return process_data(data)
    except TemporaryError as exc:
        # Retry with exponential backoff
        raise self.retry(countdown=60 * (2 ** self.request.retries), exc=exc)
    except PermanentError as exc:
        # Log and don't retry
        logging.error(f"Permanent failure in task: {exc}")
        raise
```

### Dead Letter Queue
```python
@celery_app.task
def handle_failed_task(task_id, task_name, error_message):
    """Handle permanently failed tasks"""
    # Log to monitoring system
    logging.error(f"Task {task_name} ({task_id}) failed permanently: {error_message}")

    # Store in dead letter queue for manual review
    dead_letter_data = {
        'task_id': task_id,
        'task_name': task_name,
        'error': error_message,
        'timestamp': datetime.utcnow()
    }

    store_dead_letter(dead_letter_data)

    # Send alert
    send_alert_notification(f"Task failure: {task_name}")
```

---

*Infrastructure documentation for PSW_CRON Background Processing Service*