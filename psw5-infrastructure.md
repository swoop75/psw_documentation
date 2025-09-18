# PSW5 - Orchestration Infrastructure

## Overview
PSW5 serves as the central orchestration hub for the entire Personal Stock Wealth Management System. It coordinates all microservices through Docker Compose and provides unified deployment management.

## Technology Stack
- **Container Orchestration**: Docker Compose
- **Networking**: Custom Docker networks
- **Environment Management**: .env files
- **Scripting**: Bash scripts for automation
- **Reverse Proxy**: Nginx (production)

## Infrastructure Architecture

### Docker Compose Structure
```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    ports: ["3306:3306"]
    networks: [psw_network]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    networks: [psw_network]

  psw_core:
    build: ../psw_core
    depends_on: [mysql, redis]
    ports: ["8000:8000"]
    networks: [psw_network]

  # ... all other services
```

### Network Configuration
- **psw_network**: Bridge network for inter-service communication
- **Port Mapping**: Each service exposed on unique ports
- **Service Discovery**: Services communicate by container name

## Environment Configuration

### Required Environment Variables
```env
# Database Configuration
DB_HOST=mysql
DB_DATABASE=psw5_core
DB_USERNAME=psw_user
DB_PASSWORD=secure_password

# Redis Configuration
REDIS_HOST=redis
REDIS_PASSWORD=redis_password

# API Keys
BORSDATA_API_KEY=your_api_key
EODHD_API_KEY=your_api_key
TELEGRAM_BOT_TOKEN=your_bot_token

# Service Ports
PSW_CORE_PORT=8000
PSW_WEB_PORT=3000
PSW_MONITOR_PORT=8001
```

## Deployment Instructions

### Development Environment
```bash
# 1. Clone all repositories
./scripts/clone-all.sh

# 2. Configure environment
cp .env.example .env
nano .env  # Edit with your values

# 3. Start all services
docker-compose up -d

# 4. Verify deployment
docker-compose ps
```

### Production Environment
```bash
# 1. Use production compose file
docker-compose -f docker-compose.prod.yml up -d

# 2. Set up SSL certificates
certbot --nginx -d your-domain.com

# 3. Configure firewall
ufw allow 80,443,22

# 4. Set up monitoring
docker-compose up -d prometheus grafana
```

## Management Scripts

### Repository Management
- **clone-all.sh**: Clone all PSW5 module repositories
- **update-all.sh**: Pull latest changes from all repositories
- **status-all.sh**: Check git status across all repositories

### Service Management
- **start-core.sh**: Start only core services (database, backend)
- **start-frontend.sh**: Start frontend services
- **backup.sh**: Backup databases and configurations

## Monitoring & Health Checks

### Service Health Endpoints
- **PSW Core**: `http://localhost:8000/health`
- **PSW Web**: `http://localhost:3000/health`
- **PSW Monitor**: `http://localhost:8001/health`

### Container Monitoring
```bash
# Check service status
docker-compose ps

# View service logs
docker-compose logs -f psw_core

# Monitor resource usage
docker stats
```

## Troubleshooting

### Common Issues

**Service Won't Start**
```bash
# Check service logs
docker-compose logs service_name

# Rebuild container
docker-compose build --no-cache service_name
```

**Database Connection Issues**
```bash
# Verify MySQL is running
docker-compose ps mysql

# Check database connectivity
docker-compose exec psw_core ping mysql
```

**Port Conflicts**
```bash
# Check port usage
netstat -tulpn | grep :8000

# Modify port in .env file
PSW_CORE_PORT=8001
```

## Security Considerations

### Network Security
- All services communicate through isolated Docker network
- No direct database access from outside
- API authentication required for all endpoints

### Data Protection
- Database passwords stored in environment variables
- SSL termination at reverse proxy
- Regular automated backups

## Scaling & Performance

### Horizontal Scaling
```yaml
psw_core:
  build: ../psw_core
  deploy:
    replicas: 3
    resources:
      limits:
        memory: 512M
```

### Load Balancing
- Nginx upstream configuration
- Health check-based routing
- Session sticky routing for stateful services

## Backup & Recovery

### Automated Backups
```bash
# Database backup script
#!/bin/bash
docker-compose exec mysql mysqldump -u root -p$MYSQL_ROOT_PASSWORD psw5_core > backup_$(date +%Y%m%d).sql
```

### Disaster Recovery
1. Restore database from backup
2. Rebuild containers from source
3. Restore environment configuration
4. Verify all services operational

---

*Infrastructure maintained for PSW5 Microservices Architecture*