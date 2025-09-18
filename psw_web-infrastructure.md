# PSW_WEB - Frontend Infrastructure

## Overview
PSW_WEB is the primary web application interface built with Vue.js 3, providing users with a responsive, modern dashboard for portfolio management and stock analysis.

## Technology Stack
- **Framework**: Vue.js 3 with Composition API
- **Build Tool**: Vite 4
- **Styling**: Tailwind CSS 3
- **State Management**: Pinia
- **Routing**: Vue Router 4
- **HTTP Client**: Axios
- **Charts**: Chart.js / D3.js
- **Container**: Nginx for static file serving

## Infrastructure Architecture

### Container Structure
```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

EXPOSE 3000
CMD ["nginx", "-g", "daemon off;"]
```

### Nginx Configuration
```nginx
server {
    listen 3000;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    # SPA routing
    location / {
        try_files $uri $uri/ /index.html;
    }

    # API proxy
    location /api/ {
        proxy_pass http://psw_core:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

## Application Architecture

### Project Structure
```
src/
├── components/           # Reusable Vue components
│   ├── common/          # Generic components
│   ├── portfolio/       # Portfolio-specific components
│   └── charts/          # Chart components
├── views/               # Page components
│   ├── Dashboard.vue
│   ├── Portfolio.vue
│   └── Settings.vue
├── stores/              # Pinia stores
│   ├── auth.js
│   ├── portfolio.js
│   └── stocks.js
├── router/              # Vue Router configuration
├── services/            # API service classes
├── utils/               # Utility functions
└── assets/              # Static assets
```

### State Management
```javascript
// stores/portfolio.js
import { defineStore } from 'pinia'
import { portfolioService } from '@/services/portfolioService'

export const usePortfolioStore = defineStore('portfolio', {
  state: () => ({
    portfolios: [],
    currentPortfolio: null,
    loading: false
  }),

  getters: {
    totalValue: (state) => {
      return state.portfolios.reduce((sum, p) => sum + p.value, 0)
    }
  },

  actions: {
    async fetchPortfolios() {
      this.loading = true
      try {
        this.portfolios = await portfolioService.getAll()
      } catch (error) {
        console.error('Failed to fetch portfolios:', error)
      } finally {
        this.loading = false
      }
    }
  }
})
```

## Environment Configuration

### Build-time Variables (.env)
```env
# API Configuration
VITE_API_BASE_URL=http://localhost:8000/api/v1
VITE_WS_URL=ws://localhost:8000/ws

# Feature Flags
VITE_ENABLE_2FA=true
VITE_ENABLE_ANALYTICS=true
VITE_ENABLE_DARK_MODE=true

# External Services
VITE_GOOGLE_ANALYTICS_ID=GA_MEASUREMENT_ID
VITE_SENTRY_DSN=your_sentry_dsn

# Development
VITE_APP_NAME="PSW Portfolio Manager"
VITE_APP_VERSION=1.0.0
```

### Runtime Configuration
```javascript
// config/app.js
export default {
  apiBaseUrl: import.meta.env.VITE_API_BASE_URL,
  enableFeatures: {
    twoFactorAuth: import.meta.env.VITE_ENABLE_2FA === 'true',
    analytics: import.meta.env.VITE_ENABLE_ANALYTICS === 'true',
    darkMode: import.meta.env.VITE_ENABLE_DARK_MODE === 'true'
  }
}
```

## Deployment Instructions

### Development Setup
```bash
# 1. Install dependencies
npm install

# 2. Configure environment
cp .env.example .env.local

# 3. Start development server
npm run dev

# 4. Access application
open http://localhost:5173
```

### Production Build
```bash
# 1. Build for production
npm run build

# 2. Preview build locally
npm run preview

# 3. Build Docker image
docker build -t psw-web .

# 4. Run container
docker run -d -p 3000:3000 --name psw_web psw-web
```

## Security Implementation

### Authentication Integration
```javascript
// services/authService.js
class AuthService {
  constructor() {
    this.token = localStorage.getItem('auth_token')
    this.setupInterceptors()
  }

  setupInterceptors() {
    axios.interceptors.request.use(config => {
      if (this.token) {
        config.headers.Authorization = `Bearer ${this.token}`
      }
      return config
    })

    axios.interceptors.response.use(
      response => response,
      error => {
        if (error.response?.status === 401) {
          this.logout()
          router.push('/login')
        }
        return Promise.reject(error)
      }
    )
  }
}
```

### Content Security Policy
```html
<meta http-equiv="Content-Security-Policy"
      content="default-src 'self';
               script-src 'self' 'unsafe-inline';
               style-src 'self' 'unsafe-inline';
               img-src 'self' data: https:;
               connect-src 'self' https://api.example.com;">
```

## Performance Optimization

### Code Splitting
```javascript
// router/index.js
const routes = [
  {
    path: '/dashboard',
    component: () => import('@/views/Dashboard.vue'),
    meta: { requiresAuth: true }
  },
  {
    path: '/portfolio/:id',
    component: () => import('@/views/Portfolio.vue'),
    meta: { requiresAuth: true }
  }
]
```

### Lazy Loading & Caching
```javascript
// composables/useAsyncData.js
import { ref, computed } from 'vue'

export function useAsyncData(fetchFn, options = {}) {
  const data = ref(null)
  const loading = ref(false)
  const error = ref(null)
  const { cache = true, cacheKey } = options

  const fetch = async () => {
    if (cache && cacheKey && sessionStorage.getItem(cacheKey)) {
      data.value = JSON.parse(sessionStorage.getItem(cacheKey))
      return
    }

    loading.value = true
    try {
      data.value = await fetchFn()
      if (cache && cacheKey) {
        sessionStorage.setItem(cacheKey, JSON.stringify(data.value))
      }
    } catch (err) {
      error.value = err
    } finally {
      loading.value = false
    }
  }

  return { data, loading, error, fetch }
}
```

## Monitoring & Analytics

### Error Tracking
```javascript
// plugins/sentry.js
import * as Sentry from '@sentry/vue'

export function setupSentry(app, router) {
  Sentry.init({
    app,
    dsn: import.meta.env.VITE_SENTRY_DSN,
    integrations: [
      new Sentry.BrowserTracing({
        routingInstrumentation: Sentry.vueRouterInstrumentation(router),
      }),
    ],
    tracesSampleRate: 1.0,
  })
}
```

### Performance Monitoring
```javascript
// utils/performance.js
export class PerformanceMonitor {
  static trackPageLoad(pageName) {
    const loadTime = performance.now()

    window.addEventListener('load', () => {
      const totalTime = performance.now() - loadTime
      this.sendMetric('page_load_time', totalTime, { page: pageName })
    })
  }

  static sendMetric(name, value, tags = {}) {
    // Send to analytics service
    if (window.gtag) {
      window.gtag('event', name, {
        custom_parameter: value,
        ...tags
      })
    }
  }
}
```

## Testing Infrastructure

### Unit Testing Setup
```javascript
// vitest.config.js
import { defineConfig } from 'vitest/config'
import vue from '@vitejs/plugin-vue'

export default defineConfig({
  plugins: [vue()],
  test: {
    environment: 'jsdom',
    setupFiles: ['./tests/setup.js']
  }
})
```

### Component Testing
```javascript
// tests/components/Portfolio.spec.js
import { mount } from '@vue/test-utils'
import { createPinia } from 'pinia'
import Portfolio from '@/components/Portfolio.vue'

describe('Portfolio Component', () => {
  let wrapper, pinia

  beforeEach(() => {
    pinia = createPinia()
    wrapper = mount(Portfolio, {
      global: {
        plugins: [pinia]
      },
      props: {
        portfolioId: '123'
      }
    })
  })

  it('renders portfolio data correctly', () => {
    expect(wrapper.find('.portfolio-title').text()).toBe('My Portfolio')
  })
})
```

## Troubleshooting

### Common Issues

**Build Failures**
```bash
# Clear node modules and reinstall
rm -rf node_modules package-lock.json
npm install

# Clear Vite cache
npx vite --force
```

**API Connection Issues**
```javascript
// Check API connectivity
const healthCheck = async () => {
  try {
    const response = await axios.get('/api/health')
    console.log('API Status:', response.data.status)
  } catch (error) {
    console.error('API Error:', error.message)
  }
}
```

**Routing Issues**
```bash
# Verify nginx configuration
docker exec psw_web nginx -t

# Check routing rules
curl -I http://localhost:3000/portfolio/123
```

## CI/CD Pipeline

### GitHub Actions
```yaml
# .github/workflows/build.yml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - run: npm ci
      - run: npm run test
      - run: npm run build

      - name: Build Docker image
        run: docker build -t psw-web .
```

---

*Infrastructure documentation for PSW_WEB Frontend Application*