# PSW_CORE - Backend Infrastructure

## Overview
PSW_CORE is the foundation backend service providing database management, authentication, and core APIs for the entire PSW5 ecosystem. All other services depend on this for data access and business logic.

## Technology Stack
- **Framework**: Laravel 10 (PHP 8.2)
- **Database**: MySQL 8.0
- **Cache**: Redis 7
- **Authentication**: Laravel Sanctum + OAuth 2.0
- **API**: RESTful APIs with OpenAPI documentation
- **Container**: Docker with PHP-FPM + Nginx

## Infrastructure Architecture

### Container Structure
```dockerfile
FROM php:8.2-fpm-alpine

# PHP Extensions
RUN docker-php-ext-install pdo pdo_mysql mbstring exif pcntl bcmath gd

# Composer installation
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Application setup
WORKDIR /var/www
COPY . .
RUN composer install --optimize-autoloader

EXPOSE 8000
CMD ["php", "artisan", "serve", "--host=0.0.0.0", "--port=8000"]
```

### Database Schema Design
```sql
-- Core Tables
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    two_factor_secret VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE portfolios (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    user_id BIGINT NOT NULL,
    name VARCHAR(255) NOT NULL,
    total_value DECIMAL(15,2) DEFAULT 0,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

CREATE TABLE stocks (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    symbol VARCHAR(10) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    current_price DECIMAL(10,2),
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE portfolio_holdings (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    portfolio_id BIGINT NOT NULL,
    stock_id BIGINT NOT NULL,
    quantity DECIMAL(10,4) NOT NULL,
    average_price DECIMAL(10,2) NOT NULL,
    FOREIGN KEY (portfolio_id) REFERENCES portfolios(id),
    FOREIGN KEY (stock_id) REFERENCES stocks(id)
);
```

## API Endpoints Structure

### Authentication Endpoints
```php
POST /api/v1/auth/register
POST /api/v1/auth/login
POST /api/v1/auth/logout
POST /api/v1/auth/refresh
POST /api/v1/auth/2fa/enable
POST /api/v1/auth/2fa/verify
```

### Portfolio Management
```php
GET    /api/v1/portfolios          # List user portfolios
POST   /api/v1/portfolios          # Create portfolio
GET    /api/v1/portfolios/{id}     # Get portfolio details
PUT    /api/v1/portfolios/{id}     # Update portfolio
DELETE /api/v1/portfolios/{id}     # Delete portfolio
```

### Stock Data
```php
GET /api/v1/stocks                 # Search stocks
GET /api/v1/stocks/{symbol}        # Get stock details
GET /api/v1/stocks/{symbol}/history # Get price history
```

## Environment Configuration

### Required Environment Variables
```env
# Application
APP_NAME="PSW Core API"
APP_ENV=production
APP_KEY=base64:generated_key
APP_URL=http://localhost:8000

# Database
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=psw5_core
DB_USERNAME=psw_user
DB_PASSWORD=secure_password

# Redis
REDIS_HOST=redis
REDIS_PASSWORD=redis_password
REDIS_PORT=6379

# Authentication
JWT_SECRET=your_jwt_secret
SANCTUM_STATEFUL_DOMAINS=localhost,127.0.0.1

# External APIs
BORSDATA_API_KEY=your_borsdata_key
EODHD_API_KEY=your_eodhd_key

# Mail Configuration
MAIL_MAILER=smtp
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=your_email
MAIL_PASSWORD=your_app_password
```

## Deployment Instructions

### Development Setup
```bash
# 1. Install dependencies
composer install

# 2. Configure environment
cp .env.example .env
php artisan key:generate

# 3. Run migrations
php artisan migrate

# 4. Seed database
php artisan db:seed

# 5. Start development server
php artisan serve --host=0.0.0.0 --port=8000
```

### Docker Deployment
```bash
# 1. Build container
docker build -t psw-core .

# 2. Run with database
docker run -d \
  --name psw_core \
  --link mysql:mysql \
  --link redis:redis \
  -p 8000:8000 \
  -e DB_HOST=mysql \
  -e REDIS_HOST=redis \
  psw-core
```

## Security Implementation

### Authentication Flow
1. User login with email/password
2. 2FA verification if enabled
3. JWT token generation
4. Token validation on protected routes
5. Automatic token refresh

### API Security
- Rate limiting: 60 requests/minute per IP
- CORS configuration for allowed domains
- Input validation and sanitization
- SQL injection prevention via Eloquent ORM

### Database Security
```php
// Example secure query
$portfolio = Portfolio::where('user_id', auth()->id())
                     ->where('id', $portfolioId)
                     ->firstOrFail();
```

## Performance Optimization

### Database Optimization
```php
// Query optimization examples
Portfolio::with(['holdings.stock'])
         ->where('user_id', $userId)
         ->get();

// Database indexing
Schema::table('portfolio_holdings', function (Blueprint $table) {
    $table->index(['portfolio_id', 'stock_id']);
    $table->index('user_id');
});
```

### Caching Strategy
```php
// Redis caching implementation
Cache::remember("stock_price_{$symbol}", 300, function () use ($symbol) {
    return Stock::where('symbol', $symbol)->first();
});
```

## Monitoring & Health Checks

### Health Check Endpoint
```php
Route::get('/health', function () {
    return response()->json([
        'status' => 'healthy',
        'database' => DB::connection()->getPdo() ? 'connected' : 'disconnected',
        'redis' => Redis::ping() ? 'connected' : 'disconnected',
        'timestamp' => now()
    ]);
});
```

### Logging Configuration
```php
// Custom logging channels
'channels' => [
    'api' => [
        'driver' => 'daily',
        'path' => storage_path('logs/api.log'),
        'level' => 'info',
    ],
    'auth' => [
        'driver' => 'daily',
        'path' => storage_path('logs/auth.log'),
        'level' => 'info',
    ],
]
```

## Troubleshooting

### Common Issues

**Database Connection Failed**
```bash
# Check MySQL status
docker-compose ps mysql

# Test connection
php artisan tinker
DB::connection()->getPdo();
```

**Authentication Issues**
```bash
# Clear auth cache
php artisan auth:clear-resets

# Regenerate app key
php artisan key:generate
```

**Performance Issues**
```bash
# Clear all caches
php artisan cache:clear
php artisan config:clear
php artisan route:clear

# Optimize for production
php artisan optimize
```

## API Testing

### Postman Collection
```json
{
  "info": {
    "name": "PSW Core API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/"
  },
  "item": [
    {
      "name": "Authentication",
      "item": [
        {
          "name": "Login",
          "request": {
            "method": "POST",
            "url": "{{base_url}}/api/v1/auth/login",
            "body": {
              "mode": "json",
              "raw": "{\"email\":\"user@example.com\",\"password\":\"password\"}"
            }
          }
        }
      ]
    }
  ]
}
```

## Backup & Recovery

### Database Backup
```bash
#!/bin/bash
# Automated backup script
mysqldump -h mysql -u root -p$DB_PASSWORD $DB_DATABASE > psw_core_backup_$(date +%Y%m%d_%H%M%S).sql
```

### Recovery Process
1. Restore database from backup
2. Rebuild application container
3. Run migrations to update schema
4. Verify API endpoints are functional

---

*Infrastructure documentation for PSW_CORE Backend Service*