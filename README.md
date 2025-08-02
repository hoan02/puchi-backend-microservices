# Puchi Backend Microservices

[![Go Version](https://img.shields.io/badge/Go-1.21+-blue.svg)](https://golang.org/)
[![Docker](https://img.shields.io/badge/Docker-Required-blue.svg)](https://www.docker.com/)
[![APISIX](https://img.shields.io/badge/APISIX-Gateway-orange.svg)](https://apisix.apache.org/)
[![Clerk](https://img.shields.io/badge/Clerk-Auth-purple.svg)](https://clerk.com/)
[![Cloudflare](https://img.shields.io/badge/Cloudflare-Tunnel-yellow.svg)](https://www.cloudflare.com/)

> **Master Vietnamese. Learn On The Go.** 🚀

Backend microservices architecture for [Puchi](https://github.com/hoan02/puchi) - a Vietnamese language learning platform built with Next.js 15 and React 19.

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Technologies](#technologies)
- [Services](#services)
- [Quick Start](#quick-start)
- [Development](#development)
- [Deployment](#deployment)
- [Monitoring](#monitoring)
- [API Documentation](#api-documentation)
- [Contributing](#contributing)

## 🎯 Overview

Puchi Backend Microservices is a scalable, cloud-native architecture designed to support the Puchi Vietnamese language learning platform. The system follows Clean Architecture principles and implements microservices patterns for high availability, scalability, and maintainability.

### Key Features

- 🔐 **Secure Authentication** with Clerk OpenID Connect
- 🌐 **API Gateway** with Apache APISIX
- 📊 **Comprehensive Monitoring** with Prometheus & Grafana
- 🚀 **Cloudflare Tunnel** for secure external access
- 🐳 **Containerized** with Docker & Docker Compose
- 🔄 **Message Queuing** with RabbitMQ
- 📈 **Observability** with structured logging and metrics

## 🏗️ Architecture

```mermaid
graph TB
    subgraph "External"
        Client[🌐 Client Apps]
        Cloudflare[☁️ Cloudflare Tunnel]
    end

    subgraph "Gateway Layer"
        APISIX[🚪 APISIX Gateway<br/>+ Auth Plugin]
        GoRunner[🔧 Go Plugin Runner]
        etcd[🗄️ etcd Config Store]
    end

    subgraph "Microservices"
        AuthService[🔐 Auth Service<br/>Port 8001]
        UserService[👤 User Service<br/>Port 8002]
        FutureService1[🔮 Future Service 1<br/>Port 8003]
        FutureService2[🔮 Future Service 2<br/>Port 8004]
    end

    subgraph "Infrastructure"
        RabbitMQ[🐰 RabbitMQ<br/>Message Broker]
        PostgreSQL[(🗄️ PostgreSQL)]
    end

    subgraph "Monitoring"
        Prometheus[📊 Prometheus]
        Grafana[📈 Grafana]
    end

    subgraph "Authentication"
        Clerk[🔑 Clerk OIDC]
    end

    Client --> Cloudflare
    Cloudflare --> APISIX

    %% Clerk OIDC tương tác 2 chiều với APISIX
    APISIX <--> Clerk

    %% APISIX route đến các services
    APISIX --> AuthService
    APISIX --> UserService
    APISIX --> FutureService1
    APISIX --> FutureService2

    %% Services tương tác với infrastructure
    AuthService --> PostgreSQL
    UserService --> PostgreSQL
    FutureService1 --> PostgreSQL
    FutureService2 --> PostgreSQL

    AuthService --> RabbitMQ
    UserService --> RabbitMQ
    FutureService1 --> RabbitMQ
    FutureService2 --> RabbitMQ

    %% Monitoring
    APISIX --> Prometheus
    Prometheus --> Grafana

    %% Configuration
    APISIX --> etcd
```

### Service Communication Flow

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Client    │───▶│   APISIX    │───▶│  Micro-     │
│   (Frontend)│    │   Gateway   │    │  services   │
└─────────────┘    │   + Auth    │    │             │
                   │   Plugin    │    └─────────────┘
                   └─────────────┘
                          ↕
                   ┌─────────────┐
                   │   Clerk     │
                   │   OIDC      │
                   └─────────────┘
```

### Authentication Flow

```
1. Client Request → APISIX Gateway
2. APISIX Auth Plugin → Clerk OIDC (validate token)
3. Clerk OIDC → APISIX (return validation result)
4. APISIX → Microservice (if auth successful)
5. Microservice → Client (response)
```

## 🛠️ Technologies

### Core Technologies

| Technology         | Version | Purpose                       |
| ------------------ | ------- | ----------------------------- |
| **Go**             | 1.21+   | Backend services language     |
| **Docker**         | Latest  | Containerization              |
| **Docker Compose** | Latest  | Multi-container orchestration |
| **Apache APISIX**  | 3.13.0  | API Gateway                   |
| **etcd**           | Latest  | Configuration store           |
| **RabbitMQ**       | Latest  | Message broker                |

### Authentication & Security

| Technology            | Purpose                       |
| --------------------- | ----------------------------- |
| **Clerk**             | OpenID Connect authentication |
| **Cloudflare Tunnel** | Secure external access        |
| **JWT**               | Token-based authentication    |

### Monitoring & Observability

| Technology                   | Purpose               |
| ---------------------------- | --------------------- |
| **Prometheus**               | Metrics collection    |
| **Grafana**                  | Monitoring dashboards |
| **ZeroLog**                  | Structured logging    |
| **APISIX Prometheus Plugin** | Gateway metrics       |

### Database & Storage

| Technology         | Purpose             |
| ------------------ | ------------------- |
| **PostgreSQL**     | Primary database    |
| **golang-migrate** | Database migrations |

### API & Communication

| Technology          | Purpose                     |
| ------------------- | --------------------------- |
| **gRPC**            | Inter-service communication |
| **REST API**        | External API endpoints      |
| **AMQP**            | Message queuing             |
| **Swagger/OpenAPI** | API documentation           |

## 🏢 Services

### 1. **Puchi Gateway** (`puchi-gateway/`)

**Purpose**: API Gateway and traffic management

**Components**:

- **Apache APISIX**: High-performance API gateway
- **Go Plugin Runner**: Custom authentication plugins
- **etcd**: Configuration management
- **Prometheus**: Metrics collection
- **Grafana**: Monitoring dashboards

**Key Features**:

- **Centralized Authentication**: Clerk OIDC integration via Go plugins
- **Dynamic Routing**: Support for multiple microservices
- **Rate Limiting**: Built-in protection against abuse
- **Load Balancing**: Distribute traffic across service instances
- **Monitoring**: Real-time metrics and health checks

**Ports**:

- `9080`: APISIX API
- `9180`: APISIX Admin
- `9091`: Prometheus metrics
- `9443`: HTTPS
- `9090`: Prometheus
- `3000`: Grafana

### 2. **Puchi Auth Service** (`puchi-auth-service/`)

**Purpose**: Authentication and authorization

**Features**:

- User session management
- Permission and role management
- Token refresh handling
- Audit logging

**Ports**:

- `8001`: HTTP API
- `9001`: gRPC
- `5672`: AMQP (RabbitMQ)

### 3. **Puchi User Service** (`puchi-user-service/`)

**Purpose**: User management and profiles

**Features**:

- User CRUD operations
- Profile management
- Learning progress tracking
- User preferences

**Ports**:

- `8002`: HTTP API
- `9002`: gRPC
- `5672`: AMQP (RabbitMQ)

### 4. **Future Services** (Planned)

**Scalable Architecture**: The system is designed to easily accommodate additional services:

- **Learning Service**: Course management, lessons, exercises
- **Progress Service**: Learning analytics and achievements
- **Notification Service**: Real-time notifications and alerts
- **Payment Service**: Subscription and billing management
- **Content Service**: Media and learning materials

**Port Allocation**:

- `8003-8099`: Reserved for future microservices
- `9003-9099`: Reserved for gRPC services

## 🚀 Quick Start

### Prerequisites

- Docker & Docker Compose
- Go 1.21+
- Git

### 1. Clone Repository

```bash
git clone https://github.com/hoan02/puchi-backend-microservices.git
cd puchi-backend-microservices
```

### 2. Environment Setup

```bash
# Copy environment files
cp puchi-auth-service/.env.example puchi-auth-service/.env
cp puchi-user-service/.env.example puchi-user-service/.env

# Edit environment variables
# Add your Clerk credentials and database configurations
```

### 3. Start Services

#### Option A: Start All Services (Recommended)

```bash
# Start Gateway
cd puchi-gateway
./setup-gateway.ps1

# Start Auth Service
cd ../puchi-auth-service
make compose-up

# Start User Service
cd ../puchi-user-service
make compose-up
```

#### Option B: Development Mode

```bash
# Start infrastructure only
cd puchi-gateway
docker-compose up -d etcd rabbitmq prometheus grafana

# Run services locally
cd ../puchi-auth-service
make run

cd ../puchi-user-service
make run
```

### 4. Verify Installation

```bash
# Test Gateway
curl http://localhost:9080/healthz

# Test Auth Service
curl http://localhost:9080/auth/healthz

# Test User Service
curl http://localhost:9080/user/healthz

# Check monitoring
open http://localhost:3000  # Grafana
open http://localhost:9090  # Prometheus
```

## 💻 Development

### Project Structure

```
puchi-backend-microservices/
├── puchi-gateway/           # API Gateway & Monitoring
│   ├── apisix_conf/        # APISIX configuration
│   ├── apisix-go-plugin-runner/  # Custom Go plugins
│   ├── prometheus_conf/    # Prometheus configuration
│   └── grafana_conf/       # Grafana dashboards
├── puchi-auth-service/     # Authentication service
│   ├── cmd/app/           # Application entry point
│   ├── internal/          # Business logic
│   ├── pkg/              # Shared packages
│   └── migrations/       # Database migrations
└── puchi-user-service/    # User management service
    ├── cmd/app/          # Application entry point
    ├── internal/         # Business logic
    ├── pkg/             # Shared packages
    └── migrations/      # Database migrations
```

### Development Workflow

1. **Local Development**:

   ```bash
   # Start infrastructure
   cd puchi-gateway
   docker-compose up -d etcd rabbitmq postgres

   # Run services with hot reload
   cd ../puchi-auth-service
   make run
   ```

2. **Testing**:

   ```bash
   # Unit tests
   make test

   # Integration tests
   make compose-up-integration-test
   ```

3. **Code Quality**:

   ```bash
   # Linting
   make lint

   # Format code
   make fmt
   ```

### Adding New Services

1. **Create Service Structure**:

   ```bash
   mkdir puchi-new-service
   cp -r puchi-auth-service/* puchi-new-service/
   ```

2. **Update Gateway Configuration**:

   - Add routes in `puchi-gateway/apisix_conf/routes.json`
   - Update Docker Compose if needed

3. **Register Service**:
   - Add service to monitoring
   - Update documentation

## 🚀 Deployment

### Production Deployment

1. **Environment Configuration**:

   ```bash
   # Set production environment variables
   export ENV=production
   export CLERK_SECRET_KEY=your_clerk_secret
   export DATABASE_URL=your_production_db_url
   ```

2. **Deploy with Docker Compose**:

   ```bash
   # Deploy all services
   docker-compose -f docker-compose.prod.yml up -d
   ```

3. **Cloudflare Tunnel Setup**:
   ```bash
   # Install cloudflared
   # Configure tunnel for secure external access
   cloudflared tunnel create puchi-backend
   cloudflared tunnel route dns puchi-backend api.puchi.io.vn
   ```

### Kubernetes Deployment

```yaml
# Example Kubernetes deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: puchi-auth-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: puchi-auth-service
  template:
    metadata:
      labels:
        app: puchi-auth-service
    spec:
      containers:
        - name: auth-service
          image: puchi/auth-service:latest
          ports:
            - containerPort: 8001
```

## 📊 Monitoring

### Metrics Collection

- **APISIX Metrics**: Request rates, latency, error rates
- **Service Metrics**: Business logic metrics, database queries
- **Infrastructure Metrics**: CPU, memory, disk usage

### Grafana Dashboards

- **APISIX Dashboard**: Gateway performance and health
- **Service Dashboards**: Individual service metrics
- **Infrastructure Dashboard**: System resources

### Alerting

Configure alerts for:

- High error rates (>5%)
- High latency (>500ms)
- Service downtime
- Database connection issues

## 📚 API Documentation

### Authentication Endpoints

```http
POST /auth/login
POST /auth/logout
POST /auth/refresh
GET  /auth/profile
```

### User Management Endpoints

```http
GET    /user/profile
PUT    /user/profile
GET    /user/progress
POST   /user/preferences
```

### API Documentation

- **Swagger UI**: http://localhost:8001/swagger
- **OpenAPI Spec**: http://localhost:8001/swagger.json

## 🔧 Configuration

### Environment Variables

#### Auth Service

```env
# Database
DATABASE_URL=postgres://user:pass@localhost:5432/puchi_auth

# Clerk Authentication
CLERK_SECRET_KEY=your_clerk_secret_key
CLERK_PUBLISHABLE_KEY=your_clerk_publishable_key

# JWT
JWT_SECRET=your_jwt_secret
JWT_EXPIRY=24h
```

#### User Service

```env
# Database
DATABASE_URL=postgres://user:pass@localhost:5432/puchi_users

# Service Communication
AUTH_SERVICE_URL=http://localhost:8001
RABBITMQ_URL=amqp://guest:guest@localhost:5672/
```

### APISIX Configuration

Key configuration files:

- `puchi-gateway/apisix_conf/config.yaml`: Main APISIX config
- `puchi-gateway/apisix_conf/routes.json`: Route definitions
- `puchi-gateway/apisix_conf/services.json`: Service definitions

## 🤝 Contributing

1. **Fork the repository**
2. **Create a feature branch**: `git checkout -b feature/amazing-feature`
3. **Commit your changes**: `git commit -m 'Add amazing feature'`
4. **Push to the branch**: `git push origin feature/amazing-feature`
5. **Open a Pull Request**

### Development Guidelines

- Follow Clean Architecture principles
- Write comprehensive tests
- Use conventional commit messages
- Update documentation for new features
- Ensure all tests pass before submitting PR

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 📞 Support

- **Email**: lehoan.dev@gmail.com
- **GitHub Issues**: [Create an issue](https://github.com/hoan02/puchi-backend-microservices/issues)
- **Documentation**: [Wiki](https://github.com/hoan02/puchi-backend-microservices/wiki)

## 🙏 Acknowledgments

- [Apache APISIX](https://apisix.apache.org/) for the API gateway
- [Clerk](https://clerk.com/) for authentication
- [Cloudflare](https://www.cloudflare.com/) for secure tunneling
- [Go Clean Template](https://github.com/evrone/go-clean-template) for the service architecture

---

**Puchi Backend Microservices** © 2025. Built with ❤️ for Vietnamese language learning.
