# ATS Technical Stack Documentation

## Overview
This document outlines the technical stack and architecture of the Applicant Tracking System (ATS). The system is built using modern technologies and follows best practices for scalability, maintainability, and security.

## Technology Stack

### Backend
- **Language**: Go 1.20+
- **Framework**: Gin Web Framework
- **Key Libraries**:
  - `gorm`: Database ORM
  - `zap`: Structured logging
  - `viper`: Configuration management
  - `cobra`: CLI commands
  - `jwt-go`: Authentication
  - `prometheus`: Metrics and monitoring

### Frontend
- **Framework**: React 18+
- **Key Libraries**:
  - TypeScript
  - React Query: Data fetching
  - React Router: Routing
  - React Hook Form: Form handling
  - Mantine UI: Component library

### Database
- PostgreSQL 14+
- Redis (for caching and session management)

## Core Components

### 1. Authentication System
```plaintext
Features:
- JWT-based authentication with access and refresh tokens
- Token refresh mechanism
- Password reset functionality
- Email verification
- Phone verification (OTP)
- Role-based access control (RBAC)
```

### 2. Database Layer
```plaintext
Features:
- GORM for database operations
- Migrations management
- Soft deletes
- Audit logging
- Multi-tenancy support
- Connection pooling
```

### 3. Configuration Management
```plaintext
Features:
- Environment-based configuration
- Hot reload support
- Secure secrets management
- Configuration validation
- Multiple config sources support (file, env, flags)
```

### 4. Logging and Monitoring
```plaintext
Features:
- Structured logging with Zap
- Prometheus metrics exposure
- Request tracing
- Error tracking
- Performance monitoring
- Health checks
```

### 5. API Layer
```plaintext
Features:
- RESTful API design
- Middleware support
- Rate limiting
- Request validation
- Error handling
- API documentation
```

## Architecture Principles

### 1. Clean Architecture
```plaintext
Layers:
1. Presentation Layer (HTTP handlers)
2. Business Logic Layer (Services)
3. Data Access Layer (Repositories)
4. Domain Models Layer
```

### 2. Security Measures
```plaintext
- JWT token-based authentication
- Password hashing (bcrypt)
- CORS configuration
- Rate limiting
- Input validation
- SQL injection prevention
- XSS protection
```

### 3. Performance Optimization
```plaintext
- Connection pooling
- Caching strategies
- Query optimization
- Bulk operations support
- Async processing where applicable
```

## Development Workflow

### 1. Local Development
```plaintext
Requirements:
- Go 1.20+
- Node.js 16+
- Docker
- Make

Setup:
1. Clone repository
2. Copy .env.example to .env
3. Run `make setup`
4. Run `make dev`
```

### 2. Testing
```plaintext
Types:
- Unit tests
- Integration tests
- E2E tests
- Load tests

Commands:
- make test
- make test-coverage
- make test-integration
```

### 3. Deployment
```plaintext
Environments:
- Development
- Staging
- Production

Process:
1. CI/CD pipeline triggers
2. Run tests
3. Build artifacts
4. Deploy to environment
5. Run migrations
6. Health checks
```

## Monitoring and Observability

### 1. Metrics Exposed
```plaintext
Application Metrics:
- Request count and latency
- Error rates
- Active users
- Database connection pool stats
- Cache hit rates
- Custom business metrics

System Metrics:
- CPU usage
- Memory usage
- Disk I/O
- Network traffic
```

### 2. Logging
```plaintext
Log Levels:
- DEBUG
- INFO
- WARN
- ERROR
- FATAL

Log Fields:
- Timestamp
- Level
- Service
- TraceID
- UserID
- CompanyID
- Message
- Additional context
```

### 3. Health Checks
```plaintext
Endpoints:
- /health/live: Liveness check
- /health/ready: Readiness check
- /metrics: Prometheus metrics
```

## API Documentation

### 1. Authentication Endpoints
```plaintext
POST /api/v1/auth/login
POST /api/v1/auth/refresh
POST /api/v1/auth/forgot-password
POST /api/v1/auth/reset-password
POST /api/v1/auth/verify-email
POST /api/v1/auth/verify-phone
```

### 2. Main Resource Endpoints
```plaintext
/api/v1/users
/api/v1/companies
/api/v1/jobs
/api/v1/candidates
/api/v1/applications
/api/v1/interviews
```

## Configuration

### 1. Environment Variables
```plaintext
# Server
SERVER_PORT=8080
SERVER_MODE=development

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ats
DB_USER=postgres
DB_PASSWORD=secret

# JWT
JWT_SECRET=your-secret-key
JWT_ACCESS_EXPIRY=15
JWT_REFRESH_EXPIRY=7

# Email
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_USER=user
SMTP_PASSWORD=password

# SMS
SMS_PROVIDER=twilio
SMS_API_KEY=your-api-key
```

### 2. Feature Flags
```plaintext
ENABLE_PHONE_VERIFICATION=true
ENABLE_EMAIL_VERIFICATION=true
ENABLE_METRICS=true
ENABLE_RATE_LIMITING=true
```

## Error Handling

### 1. Error Types
```plaintext
- ValidationError
- AuthenticationError
- AuthorizationError
- NotFoundError
- DatabaseError
- ExternalServiceError
```

### 2. Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input",
    "details": {
      "field": "email",
      "reason": "invalid format"
    }
  }
}
```

## Detailed Technical Specifications

### 1. Database Schema Management
```yaml
# Migration Strategy
- Using GORM Auto-Migrations for development
- Using golang-migrate for production migrations
- Version control for all migrations
- Rollback support
- Data seeding scripts

# Connection Pool Configuration
DB_MAX_OPEN_CONNS=25
DB_MAX_IDLE_CONNS=5
DB_CONN_MAX_LIFETIME=15m
DB_CONN_MAX_IDLE_TIME=5m

# Indexing Strategy
- Composite indexes for frequently queried fields
- Full-text search indexes for search functionality
- Partial indexes for filtered queries
- B-tree indexes for exact matches
```

### 2. Caching Architecture
```yaml
# Redis Configuration
REDIS_MASTER_HOST=redis-master
REDIS_SLAVE_HOSTS=redis-slave-1,redis-slave-2
REDIS_PASSWORD=secure-password
REDIS_DB=0
REDIS_MAX_RETRIES=3
REDIS_TIMEOUT=5s

# Caching Strategies
- Cache-Aside pattern for database queries
- Distributed caching for session management
- Cache invalidation using pub/sub
- Rate limiter implementation
- Job queue backend
```

### 3. API Rate Limiting
```yaml
# Rate Limit Configuration
RATE_LIMIT_WINDOW=15m
RATE_LIMIT_MAX_REQUESTS=100
RATE_LIMIT_STRATEGY=sliding-window

# Implementation
- Redis-based rate limiting
- Per-endpoint configuration
- User-based quotas
- IP-based restrictions
```

## Deployment Configuration

### 1. Kubernetes Deployment
```yaml
# kubernetes/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ats-api
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: ats-api
  template:
    metadata:
      labels:
        app: ats-api
    spec:
      containers:
      - name: ats-api
        image: your-registry.com/ats-api:${TAG}
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
        env:
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: ats-secrets
              key: db-host
        # ... other env vars
```

### 2. Docker Compose (Development)
```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "8080:8080"
    environment:
      - SERVER_MODE=development
      - DB_HOST=postgres
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:14-alpine
    environment:
      POSTGRES_DB: ats
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:6-alpine
    command: redis-server --requirepass redis
    ports:
      - "6379:6379"

volumes:
  postgres_data:
```

### 3. CI/CD Pipeline (GitHub Actions)
```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Set up Go
      uses: actions/setup-go@v2
      with:
        go-version: 1.20
    - name: Run tests
      run: make test
    - name: Run linter
      run: make lint

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
    - name: Build Docker image
      run: docker build -t your-registry.com/ats-api:${GITHUB_SHA} .
    - name: Push Docker image
      run: docker push your-registry.com/ats-api:${GITHUB_SHA}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
    - name: Deploy to Kubernetes
      uses: steebchen/kubectl@v2
      with:
        config: ${{ secrets.KUBE_CONFIG_DATA }}
        command: set image deployment/ats-api ats-api=your-registry.com/ats-api:${GITHUB_SHA}
```

## Enhanced Security Considerations

### 1. Data Encryption
```yaml
# At Rest Encryption
- Database encryption using PostgreSQL encryption
- File encryption for stored documents
- Encryption key rotation policy
- Secure key management using HashiCorp Vault

# In Transit Encryption
- TLS 1.3 for all communications
- Certificate management with cert-manager
- Automatic certificate rotation
- Strong cipher suites only
```

### 2. Access Control
```yaml
# RBAC Configuration
roles:
  admin:
    - all:*
  hiring_manager:
    - jobs:*
    - candidates:read,write
    - interviews:*
  recruiter:
    - candidates:*
    - jobs:read
    - interviews:read,write
  interviewer:
    - interviews:read,write
    - candidates:read

# Permission Enforcement
- Role-based middleware
- Resource-level permissions
- Company-level isolation
- Audit logging for all actions
```

### 3. Security Headers
```go
// Middleware configuration
func securityHeaders() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("X-Frame-Options", "DENY")
        c.Header("X-Content-Type-Options", "nosniff")
        c.Header("X-XSS-Protection", "1; mode=block")
        c.Header("Content-Security-Policy", "default-src 'self'")
        c.Header("Referrer-Policy", "strict-origin-when-cross-origin")
        c.Header("Strict-Transport-Security", "max-age=31536000; includeSubDomains")
        c.Next()
    }
}
```

### 4. API Security
```yaml
# Security Measures
- Request signing for sensitive operations
- API key rotation
- Request throttling
- Input sanitization
- Output encoding
- SQL injection prevention
- NoSQL injection prevention
- XML external entity prevention

# Authentication Security
- Password complexity requirements
- Account lockout policy
- MFA enforcement for sensitive operations
- Session management
- Token revocation
```

### 5. Monitoring and Incident Response
```yaml
# Security Monitoring
- Failed login attempts
- Suspicious IP detection
- Rate limit violations
- Permission violations
- Data access patterns
- System changes

# Incident Response
- Automated alerts
- Incident classification
- Response procedures
- Recovery plans
- Post-incident analysis
```

### 6. Compliance and Audit
```yaml
# Audit Logging
audit_events:
  - user_login
  - password_change
  - role_change
  - data_access
  - configuration_change
  - security_violation

# Compliance Features
- Data retention policies
- GDPR compliance tools
- Data export capabilities
- Audit trail maintenance
- Privacy policy enforcement
```
