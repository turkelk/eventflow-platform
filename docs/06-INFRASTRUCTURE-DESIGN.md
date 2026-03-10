# EventFlow — Infrastructure Design

> Document Version: 1.0 | Status: Approved for Implementation  
> Environments: Docker Compose (local dev) + Kubernetes (production)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Docker Compose — Local Development](#2-docker-compose--local-development)
3. [Kubernetes — Production](#3-kubernetes--production)
4. [Nginx Configuration](#4-nginx-configuration)
5. [CI/CD Pipeline](#5-cicd-pipeline)
6. [Secrets Management](#6-secrets-management)
7. [Backup & Recovery](#7-backup--recovery)
8. [Network Architecture](#8-network-architecture)

---

## 1. Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          EventFlow Infrastructure                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  Internet                                                                     │
│     │                                                                         │
│     ▼                                                                         │
│  [CDN / CloudFront]  ←── Static assets (JS, CSS, fonts)                     │
│     │                                                                         │
│     ▼                                                                         │
│  [Nginx Ingress / Nginx Reverse Proxy]                                       │
│     │         │              │              │                                 │
│     ▼         ▼              ▼              ▼                                 │
│  [React    [API         [Keycloak       [Unleash                             │
│   Frontend] Service]    Auth]           Flags]                               │
│     │       │    │                                                            │
│     │       ▼    ▼                                                            │
│     │  [PostgreSQL]  [Redis]                                                  │
│     │       │           │                                                     │
│     │       │    [Redis Streams]  ←── Event Bus                             │
│     │       │           │                                                     │
│     │       ▼           ▼                                                     │
│     │  [MinIO / S3]  [Background Worker]                                     │
│     │                    │                                                    │
│     │                    ▼                                                    │
│     │               [Seq Logging]                                             │
│     │               [Prometheus]                                              │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Service Port Map

| Service | Local Port | Purpose |
|---|---|---|
| Nginx | 80 | HTTP (redirects to HTTPS in prod) |
| Nginx | 443 | HTTPS / WSS |
| API (.NET) | 8080 | Internal (exposed via Nginx) |
| Frontend (Vite) | 5173 | Dev HMR (exposed directly in dev) |
| PostgreSQL | 5432 | Database |
| Redis | 6379 | Cache + Streams |
| Keycloak | 8180 | Auth (exposed at /auth via Nginx) |
| Unleash | 4242 | Feature flags |
| MinIO | 9000 | Object storage API |
| MinIO Console | 9001 | MinIO admin UI |
| Seq | 5341 | Log ingestion |
| Seq UI | 8090 | Log viewer |
| Prometheus | 9090 | Metrics scrape |

---

## 2. Docker Compose — Local Development

### 2.1 docker-compose.yml

```yaml
# docker-compose.yml
# EventFlow — Local Development Environment
# Usage: docker compose up -d
# Hot reload: API (dotnet watch) and Frontend (Vite HMR) run outside Docker
#             OR inside with volume mounts (see profiles)

version: '3.9'

name: eventflow

networks:
  eventflow-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  postgres-data:
    driver: local
  redis-data:
    driver: local
  keycloak-data:
    driver: local
  minio-data:
    driver: local
  seq-data:
    driver: local
  prometheus-data:
    driver: local
  unleash-data:
    driver: local

services:

  # ============================================================
  # NGINX — Reverse Proxy
  # Routes: /api/* → api, /auth/* → keycloak, /* → frontend
  # ============================================================
  nginx:
    image: nginx:1.27-alpine
    container_name: eventflow-nginx
    ports:
      - "80:80"
    volumes:
      - ./infra/nginx/nginx.dev.conf:/etc/nginx/nginx.conf:ro
      - ./infra/nginx/conf.d:/etc/nginx/conf.d:ro
    depends_on:
      - api
      - keycloak
    networks:
      - eventflow-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 5s
      retries: 3

  # ============================================================
  # .NET 9 API — Backend
  # Hot reload via dotnet watch (run outside Docker in dev)
  # OR build + run inside with volume mount for source
  # ============================================================
  api:
    build:
      context: .
      dockerfile: src/EventFlow.API/Dockerfile
      target: development  # Multi-stage: 'development' uses SDK image
    container_name: eventflow-api
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ASPNETCORE_URLS=http://+:8080
      - ConnectionStrings__PostgreSQL=${POSTGRES_CONNECTION_STRING}
      - ConnectionStrings__Redis=${REDIS_CONNECTION_STRING}
      - Keycloak__Authority=${KEYCLOAK_AUTHORITY}
      - Keycloak__ClientId=${KEYCLOAK_CLIENT_ID}
      - Keycloak__ClientSecret=${KEYCLOAK_CLIENT_SECRET}
      - Unleash__ServerUrl=http://unleash:4242/api
      - Unleash__ApiToken=${UNLEASH_API_TOKEN}
      - Storage__Endpoint=http://minio:9000
      - Storage__AccessKey=${MINIO_ACCESS_KEY}
      - Storage__SecretKey=${MINIO_SECRET_KEY}
      - Storage__BucketName=eventflow
      - Seq__ServerUrl=http://seq:5341
      - AI__OpenAIApiKey=${OPENAI_API_KEY}
      - AI__BaseUrl=${AI_BASE_URL:-https://api.openai.com/v1}
    volumes:
      - ./src:/app/src:cached  # Source mount for dotnet watch
      - ~/.nuget:/root/.nuget:cached  # NuGet cache
    ports:
      - "8080:8080"  # Direct access for debugging
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      keycloak:
        condition: service_healthy
    networks:
      - eventflow-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health/alive"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ============================================================
  # REACT FRONTEND — Vite Dev Server
  # HMR via Vite (port 5173, proxied through Nginx in dev)
  # ============================================================
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile.dev
    container_name: eventflow-frontend
    environment:
      - VITE_API_URL=http://localhost/api
      - VITE_AUTH_URL=http://localhost/auth
      - VITE_UNLEASH_URL=http://localhost:4242/api/frontend
      - VITE_UNLEASH_CLIENT_KEY=${UNLEASH_FRONTEND_TOKEN}
      - VITE_WS_URL=ws://localhost/ws
    volumes:
      - ./frontend/src:/app/src:cached
      - ./frontend/public:/app/public:cached
      - /app/node_modules  # Anonymous volume — don't mount host node_modules
    ports:
      - "5173:5173"
    networks:
      - eventflow-network
    restart: unless-stopped

  # ============================================================
  # POSTGRESQL — Primary Database
  # ============================================================
  postgres:
    image: postgres:17-alpine
    container_name: eventflow-postgres
    environment:
      POSTGRES_DB: ${POSTGRES_DB:-eventflow}
      POSTGRES_USER: ${POSTGRES_USER:-eventflow}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./infra/postgres/init:/docker-entrypoint-initdb.d:ro  # SQL seed scripts
      - ./infra/postgres/postgresql.conf:/etc/postgresql/postgresql.conf:ro
    ports:
      - "5432:5432"  # Exposed for DB clients (TablePlus, DataGrip)
    networks:
      - eventflow-network
    restart: unless-stopped
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-eventflow} -d ${POSTGRES_DB:-eventflow}"]
      interval: 10s
      timeout: 5s
      retries: 5
    shm_size: 256mb

  # ============================================================
  # REDIS — Cache + Streams + Distributed Lock
  # ============================================================
  redis:
    image: redis:7.4-alpine
    container_name: eventflow-redis
    command: >
      redis-server
      --appendonly yes
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    ports:
      - "6379:6379"
    networks:
      - eventflow-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # ============================================================
  # KEYCLOAK — Identity Provider (OIDC)
  # ============================================================
  keycloak:
    image: quay.io/keycloak/keycloak:26.0
    container_name: eventflow-keycloak
    command: start-dev --import-realm
    environment:
      KC_HOSTNAME: localhost
      KC_HOSTNAME_PORT: 80
      KC_HOSTNAME_STRICT: false
      KC_HOSTNAME_STRICT_HTTPS: false
      KC_HTTP_ENABLED: true
      KC_HTTP_PORT: 8180
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres:5432/${KEYCLOAK_DB:-keycloak}
      KC_DB_USERNAME: ${KEYCLOAK_DB_USER:-keycloak}
      KC_DB_PASSWORD: ${KEYCLOAK_DB_PASSWORD}
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN_USER:-admin}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_FEATURES: declarative-ui,token-exchange
    volumes:
      - ./infra/keycloak/realms:/opt/keycloak/data/import:ro
      - keycloak-data:/opt/keycloak/data
    ports:
      - "8180:8180"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - eventflow-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "exec 3<>/dev/tcp/localhost/8180 && echo -e 'GET /health/ready HTTP/1.1\r\nHost: localhost\r\n\r\n' >&3 && cat <&3 | grep -q '200 OK'"]
      interval: 30s
      timeout: 10s
      retries: 10
      start_period: 60s

  # ============================================================
  # UNLEASH — Feature Flags
  # ============================================================
  unleash:
    image: unleashorg/unleash-server:6
    container_name: eventflow-unleash
    environment:
      DATABASE_URL: postgresql://${POSTGRES_USER:-eventflow}:${POSTGRES_PASSWORD}@postgres:5432/${UNLEASH_DB:-unleash}
      DATABASE_SSL: false
      LOG_LEVEL: warn
      INIT_FRONTEND_API_TOKENS: ${UNLEASH_FRONTEND_TOKEN}
      INIT_CLIENT_API_TOKENS: ${UNLEASH_API_TOKEN}
      ADMIN_AUTHENTICATION: unsecure  # dev only; use "password" in staging
    ports:
      - "4242:4242"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - eventflow-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4242/health"]
      interval: 15s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ============================================================
  # MINIO — S3-Compatible Object Storage
  # ============================================================
  minio:
    image: minio/minio:RELEASE.2024-12-18T13-15-44Z
    container_name: eventflow-minio
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${MINIO_ACCESS_KEY:-minioadmin}
      MINIO_ROOT_PASSWORD: ${MINIO_SECRET_KEY}
    volumes:
      - minio-data:/data
    ports:
      - "9000:9000"   # S3 API
      - "9001:9001"   # MinIO Console UI
    networks:
      - eventflow-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mc", "ready", "local"]
      interval: 30s
      timeout: 5s
      retries: 3

  # ============================================================
  # MINIO INIT — Create default bucket
  # ============================================================
  minio-init:
    image: minio/mc:RELEASE.2024-12-18T14-52-49Z
    container_name: eventflow-minio-init
    entrypoint: >
      /bin/sh -c "
        sleep 5;
        mc alias set local http://minio:9000 ${MINIO_ACCESS_KEY:-minioadmin} ${MINIO_SECRET_KEY};
        mc mb local/eventflow --ignore-existing;
        mc anonymous set download local/eventflow/public;
        exit 0;
      "
    depends_on:
      minio:
        condition: service_healthy
    networks:
      - eventflow-network
    restart: "no"

  # ============================================================
  # SEQ — Structured Logging
  # ============================================================
  seq:
    image: datalust/seq:2024.3
    container_name: eventflow-seq
    environment:
      ACCEPT_EULA: Y
      SEQ_FIRSTRUN_ADMINPASSWORDHASH: ${SEQ_ADMIN_PASSWORD_HASH}
    volumes:
      - seq-data:/data
    ports:
      - "5341:5341"   # Log ingestion
      - "8090:80"     # Seq web UI
    networks:
      - eventflow-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/api"]
      interval: 30s
      timeout: 5s
      retries: 3

  # ============================================================
  # PROMETHEUS — Metrics Collection
  # ============================================================
  prometheus:
    image: prom/prometheus:v3.0.1
    container_name: eventflow-prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=15d'
      - '--web.enable-lifecycle'
    volumes:
      - ./infra/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - eventflow-network
    restart: unless-stopped

  # ============================================================
  # DB MIGRATION JOB — EF Core migrations at startup
  # ============================================================
  db-migrator:
    build:
      context: .
      dockerfile: src/EventFlow.API/Dockerfile
      target: migrator
    container_name: eventflow-migrator
    environment:
      - ConnectionStrings__PostgreSQL=${POSTGRES_CONNECTION_STRING}
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - eventflow-network
    restart: "no"  # One-shot job
```

### 2.2 Environment Variables (.env.example)

```bash
# .env.example
# Copy to .env and fill in values before running docker compose
# NEVER commit .env to version control

# ============================================================
# PostgreSQL
# ============================================================
POSTGRES_DB=eventflow
POSTGRES_USER=eventflow
POSTGRES_PASSWORD=CHANGE_ME_strong_password_here
POSTGRES_CONNECTION_STRING=Host=postgres;Port=5432;Database=eventflow;Username=eventflow;Password=CHANGE_ME_strong_password_here;Pooling=true;MaxPoolSize=100

# Keycloak uses a separate DB
KEYCLOAK_DB=keycloak
KEYCLOAK_DB_USER=keycloak
KEYCLOAK_DB_PASSWORD=CHANGE_ME_keycloak_db_password

# Unleash uses a separate DB
UNLEASH_DB=unleash

# ============================================================
# Redis
# ============================================================
REDIS_PASSWORD=CHANGE_ME_redis_password
REDIS_CONNECTION_STRING=localhost:6379,password=CHANGE_ME_redis_password,ssl=false

# ============================================================
# Keycloak
# ============================================================
KEYCLOAK_ADMIN_USER=admin
KEYCLOAK_ADMIN_PASSWORD=CHANGE_ME_keycloak_admin
KEYCLOAK_AUTHORITY=http://localhost/auth/realms/eventflow
KEYCLOAK_CLIENT_ID=eventflow-api
KEYCLOAK_CLIENT_SECRET=CHANGE_ME_keycloak_client_secret

# ============================================================
# Unleash
# ============================================================
UNLEASH_API_TOKEN=CHANGE_ME_unleash_server_token
UNLEASH_FRONTEND_TOKEN=CHANGE_ME_unleash_frontend_token

# ============================================================
# MinIO
# ============================================================
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=CHANGE_ME_minio_secret

# ============================================================
# Seq
# ============================================================
# Generate with: echo -n 'your-password' | sha256sum
SEQ_ADMIN_PASSWORD_HASH=CHANGE_ME_sha256_of_seq_admin_password

# ============================================================
# AI (OpenAI compatible)
# ============================================================
OPENAI_API_KEY=sk-CHANGE_ME
AI_BASE_URL=https://api.openai.com/v1

# ============================================================
# Application
# ============================================================
ASPNETCORE_ENVIRONMENT=Development
JWT_AUDIENCE=eventflow-api
JWT_ISSUER=http://localhost/auth/realms/eventflow
```

### 2.3 Dockerfile (Multi-Stage)

```dockerfile
# src/EventFlow.API/Dockerfile
# Multi-stage: development (SDK, hot reload) | build | publish | runtime | migrator

ARG DOTNET_VERSION=9.0

# ── Stage: base SDK ──────────────────────────────────────────
FROM mcr.microsoft.com/dotnet/sdk:${DOTNET_VERSION}-alpine AS sdk
WORKDIR /app
COPY EventFlow.sln .
COPY src/ src/

# ── Stage: development (dotnet watch) ─────────────────────────
FROM sdk AS development
EXPOSE 8080
ENV ASPNETCORE_ENVIRONMENT=Development
ENV ASPNETCORE_URLS=http://+:8080
WORKDIR /app/src/EventFlow.API
CMD ["dotnet", "watch", "run", "--no-launch-profile"]

# ── Stage: restore + build ─────────────────────────────────────
FROM sdk AS build
RUN dotnet restore EventFlow.sln
RUN dotnet build src/EventFlow.API/EventFlow.API.csproj -c Release -o /build --no-restore

# ── Stage: publish ─────────────────────────────────────────────
FROM build AS publish
RUN dotnet publish src/EventFlow.API/EventFlow.API.csproj \
    -c Release -o /publish \
    --no-restore \
    -p:PublishSingleFile=false \
    -p:PublishTrimmed=false

# ── Stage: migrator ────────────────────────────────────────────
FROM build AS migrator
RUN dotnet tool install --global dotnet-ef
ENV PATH="${PATH}:/root/.dotnet/tools"
WORKDIR /app
CMD ["dotnet", "ef", "database", "update", \
     "--project", "src/EventFlow.Infrastructure", \
     "--startup-project", "src/EventFlow.API"]

# ── Stage: runtime (production) ────────────────────────────────
FROM mcr.microsoft.com/dotnet/aspnet:${DOTNET_VERSION}-alpine AS runtime

# Security: non-root user
RUN addgroup -g 1001 -S app && adduser -u 1001 -S app -G app

WORKDIR /app
COPY --from=publish /publish .

# Security: read-only fs compatible (tmp for ASP.NET temp files)
RUN mkdir -p /tmp/asp && chown app:app /tmp/asp
USER app

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENV ASPNETCORE_ENVIRONMENT=Production
ENV DOTNET_RUNNING_IN_CONTAINER=true
ENV TMPDIR=/tmp/asp

HEALTHCHECK --interval=15s --timeout=5s --start-period=30s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:8080/health/alive || exit 1

ENTRYPOINT ["dotnet", "EventFlow.API.dll"]
```

### 2.4 Frontend Dockerfile (Dev)

```dockerfile
# frontend/Dockerfile.dev
FROM node:22-alpine AS dev
WORKDIR /app

# Copy package files first for layer caching
COPY package.json package-lock.json ./
RUN npm ci

COPY . .

EXPOSE 5173
CMD ["npm", "run", "dev", "--", "--host", "0.0.0.0", "--port", "5173"]

# Production build stage (used by CI)
FROM dev AS build
RUN npm run build

FROM nginx:1.27-alpine AS production
COPY --from=build /app/dist /usr/share/nginx/html
COPY nginx.static.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 2.5 Keycloak Realm Export

```json
// infra/keycloak/realms/eventflow-realm.json (abbreviated)
{
  "realm": "eventflow",
  "enabled": true,
  "displayName": "EventFlow",
  "loginWithEmailAllowed": true,
  "sslRequired": "external",
  "accessTokenLifespan": 300,
  "ssoSessionMaxLifespan": 28800,
  "refreshTokenMaxReuse": 0,
  "revokeRefreshToken": true,
  "bruteForceProtected": true,
  "failureFactor": 5,
  "waitIncrementSeconds": 900,
  "roles": {
    "realm": [
      { "name": "platform_admin", "description": "EventFlow platform administrator" },
      { "name": "owner", "description": "Tenant owner (billing)" },
      { "name": "admin", "description": "Tenant administrator" },
      { "name": "organizer", "description": "Event organizer" },
      { "name": "staff", "description": "Check-in staff" },
      { "name": "viewer", "description": "Read-only analytics viewer" }
    ]
  },
  "clients": [
    {
      "clientId": "eventflow-web",
      "name": "EventFlow Web App",
      "publicClient": true,
      "standardFlowEnabled": true,
      "directAccessGrantsEnabled": false,
      "pkceCodeChallengeMethod": "S256",
      "redirectUris": [
        "http://localhost/*",
        "https://app.eventflow.io/*",
        "https://*.eventflow.io/*"
      ],
      "webOrigins": ["+"],
      "attributes": {
        "access.token.lifespan": "300",
        "use.refresh.tokens": "true"
      }
    },
    {
      "clientId": "eventflow-api",
      "name": "EventFlow API",
      "publicClient": false,
      "serviceAccountsEnabled": true,
      "standardFlowEnabled": false,
      "bearerOnly": true
    }
  ],
  "protocolMappers": [
    {
      "name": "tenant-id",
      "protocol": "openid-connect",
      "protocolMapper": "oidc-usermodel-attribute-mapper",
      "config": {
        "user.attribute": "tenant_id",
        "claim.name": "tenant_id",
        "access.token.claim": true,
        "id.token.claim": true
      }
    }
  ]
}
```

### 2.6 Seed Data

```sql
-- infra/postgres/init/01-seed-dev-data.sql
-- Development seed data — NOT for production

-- Create Unleash DB
CREATE DATABASE unleash;
CREATE DATABASE keycloak;

-- EventFlow dev tenant
INSERT INTO tenants (id, name, slug, plan, created_at)
VALUES
  ('550e8400-e29b-41d4-a716-446655440000', 'Acme Corp (Dev)', 'acme-dev', 'business', NOW()),
  ('550e8400-e29b-41d4-a716-446655440001', 'Beta Company', 'beta-dev', 'starter', NOW());

-- Dev events
INSERT INTO events (id, tenant_id, title, status, start_date, end_date, capacity, created_at)
VALUES
  ('660e8400-e29b-41d4-a716-446655440000',
   '550e8400-e29b-41d4-a716-446655440000',
   'Q4 Sales Kickoff 2025', 'published',
   '2025-03-15 09:00:00', '2025-03-16 18:00:00', 350, NOW()),
  ('660e8400-e29b-41d4-a716-446655440001',
   '550e8400-e29b-41d4-a716-446655440000',
   'Product Launch Webinar', 'draft',
   '2025-03-22 14:00:00', '2025-03-22 16:00:00', 200, NOW());
```

---

## 3. Kubernetes — Production

### 3.1 Cluster Architecture

```
Kubernetes Cluster (AWS EKS / GKE / AKS)
├── Namespaces:
│   ├── eventflow           # Application workloads
│   ├── eventflow-infra     # PostgreSQL, Redis, MinIO (or cloud-managed)
│   ├── eventflow-monitoring # Prometheus, Grafana, Seq
│   ├── cert-manager        # TLS certificate management
│   ├── ingress-nginx       # Nginx Ingress Controller
│   └── argocd              # GitOps operator
│
├── Node Groups:
│   ├── system:   2× t3.medium  (system workloads: ingress, argocd, cert-manager)
│   ├── app:      2-20× t3.large (autoscaling, API + workers)
│   └── infra:    2× r6i.large  (stateful: PostgreSQL, Redis — if not cloud-managed)
```

### 3.2 Helm Chart Structure

```
helm/
├── eventflow-api/              # Main API service
│   ├── Chart.yaml
│   ├── values.yaml             # Default values
│   ├── values.staging.yaml     # Staging overrides
│   ├── values.production.yaml  # Production overrides
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── hpa.yaml
│       ├── pdb.yaml
│       ├── configmap.yaml
│       ├── secret.yaml         # Sealed Secret reference
│       ├── networkpolicy.yaml
│       ├── serviceaccount.yaml
│       └── migration-job.yaml
├── eventflow-frontend/         # Static frontend (Nginx)
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── hpa.yaml
├── eventflow-worker/           # Background Redis Stream consumers
│   └── templates/
│       ├── deployment.yaml
│       ├── hpa.yaml
│       └── networkpolicy.yaml
├── eventflow-ingress/          # Nginx Ingress rules
│   └── templates/
│       ├── ingress.yaml
│       └── certificate.yaml    # cert-manager Certificate
└── eventflow-monitoring/       # Prometheus + Grafana + Seq
    └── templates/
        ├── prometheus.yaml
        ├── servicemonitor.yaml
        └── seq.yaml
```

### 3.3 API Deployment

```yaml
# helm/eventflow-api/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: eventflow-api
  namespace: eventflow
  labels:
    app: eventflow-api
    version: {{ .Values.image.tag }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: eventflow-api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime deploy
  template:
    metadata:
      labels:
        app: eventflow-api
        version: {{ .Values.image.tag }}
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/metrics"
        prometheus.io/port: "8080"
    spec:
      serviceAccountName: eventflow-api
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      terminationGracePeriodSeconds: 60
      containers:
        - name: api
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
              name: http
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: {{ .Values.environment }}
            - name: ConnectionStrings__PostgreSQL
              valueFrom:
                secretKeyRef:
                  name: eventflow-secrets
                  key: postgres-connection-string
            - name: ConnectionStrings__Redis
              valueFrom:
                secretKeyRef:
                  name: eventflow-secrets
                  key: redis-connection-string
            - name: Keycloak__Authority
              valueFrom:
                configMapKeyRef:
                  name: eventflow-config
                  key: keycloak-authority
            - name: Keycloak__ClientSecret
              valueFrom:
                secretKeyRef:
                  name: eventflow-secrets
                  key: keycloak-client-secret
            - name: AI__OpenAIApiKey
              valueFrom:
                secretKeyRef:
                  name: eventflow-secrets
                  key: openai-api-key
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 10
            failureThreshold: 3
          livenessProbe:
            httpGet:
              path: /health/alive
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 20
            failureThreshold: 3
          startupProbe:
            httpGet:
              path: /health/alive
              port: 8080
            failureThreshold: 30
            periodSeconds: 5
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 5"]
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        - name: tmp
          emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values: [eventflow-api]
                topologyKey: kubernetes.io/hostname
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: eventflow-api
```

### 3.4 Horizontal Pod Autoscaler

```yaml
# helm/eventflow-api/templates/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: eventflow-api
  namespace: eventflow
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: eventflow-api
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
---
# Worker HPA — scales on Redis Stream pending count
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: eventflow-worker
  namespace: eventflow
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: eventflow-worker
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: External
      external:
        metric:
          name: redis_stream_pending_count
          selector:
            matchLabels:
              stream: eventflow-events
        target:
          type: AverageValue
          averageValue: 100
```

### 3.5 Pod Disruption Budget

```yaml
# helm/eventflow-api/templates/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: eventflow-api-pdb
  namespace: eventflow
spec:
  minAvailable: 1  # Always keep at least 1 pod running during node drain
  selector:
    matchLabels:
      app: eventflow-api
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: eventflow-worker-pdb
  namespace: eventflow
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: eventflow-worker
```

### 3.6 Network Policies

```yaml
# helm/eventflow-api/templates/networkpolicy.yaml
# Default deny-all, then explicit allow rules

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: eventflow
spec:
  podSelector: {}  # applies to all pods in namespace
  policyTypes: [Ingress, Egress]
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: eventflow
spec:
  podSelector:
    matchLabels:
      app: eventflow-api
  policyTypes: [Ingress, Egress]
  ingress:
    # From Nginx Ingress Controller
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
  egress:
    # To PostgreSQL
    - to:
        - podSelector:
            matchLabels:
              app: postgresql
      ports:
        - protocol: TCP
          port: 5432
    # To Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # To Keycloak
    - to:
        - namespaceSelector:
            matchLabels:
              name: keycloak
      ports:
        - protocol: TCP
          port: 8080
    # To external APIs (OpenAI, etc.) — DNS + HTTPS
    - ports:
        - protocol: TCP
          port: 443
        - protocol: UDP
          port: 53
```

### 3.7 Ingress Configuration (Production)

```yaml
# helm/eventflow-ingress/templates/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: eventflow-ingress
  namespace: eventflow
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/proxy-body-size: 50m
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    # WebSocket support
    nginx.ingress.kubernetes.io/proxy-http-version: "1.1"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";
    # Security headers
    nginx.ingress.kubernetes.io/server-snippet: |
      add_header X-Frame-Options DENY;
      add_header X-Content-Type-Options nosniff;
      add_header Referrer-Policy strict-origin-when-cross-origin;
      add_header Permissions-Policy "camera=(), microphone=(), geolocation=()";
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rpm: "300"
    nginx.ingress.kubernetes.io/limit-rps: "10"
    # Enable HSTS
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  tls:
    - hosts:
        - app.eventflow.io
        - api.eventflow.io
      secretName: eventflow-tls
  rules:
    - host: app.eventflow.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: eventflow-frontend
                port:
                  number: 80
    - host: api.eventflow.io
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: eventflow-api
                port:
                  number: 8080
---
# cert-manager Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: eventflow-tls
  namespace: eventflow
spec:
  secretName: eventflow-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - app.eventflow.io
    - api.eventflow.io
    - "*.eventflow.io"  # wildcard for white-label tenants
```

### 3.8 Sealed Secrets

```yaml
# helm/eventflow-api/templates/secret.yaml
# Sealed Secrets (Bitnami) — encrypted at rest in Git
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: eventflow-secrets
  namespace: eventflow
spec:
  encryptedData:
    # Encrypted with kubeseal using cluster public key
    postgres-connection-string: AgA1... (sealed by kubeseal)
    redis-connection-string: AgB2... (sealed by kubeseal)
    keycloak-client-secret: AgC3... (sealed by kubeseal)
    openai-api-key: AgD4... (sealed by kubeseal)
    minio-access-key: AgE5... (sealed by kubeseal)
    minio-secret-key: AgF6... (sealed by kubeseal)

# Usage:
# kubeseal --fetch-cert --controller-namespace=sealed-secrets > pub-cert.pem
# kubectl create secret generic eventflow-secrets --dry-run=client \
#   --from-literal=postgres-connection-string='...' -o yaml | \
#   kubeseal --cert pub-cert.pem -o yaml > sealed-secret.yaml
```

### 3.9 ArgoCD Application Manifests

```yaml
# argocd/applications/eventflow-api.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: eventflow-api
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: deployments
    notifications.argoproj.io/subscribe.on-health-degraded.slack: alerts
spec:
  project: eventflow
  source:
    repoURL: https://github.com/your-org/eventflow
    targetRevision: HEAD
    path: helm/eventflow-api
    helm:
      valueFiles:
        - values.yaml
        - values.production.yaml
      parameters:
        - name: image.tag
          value: $ARGOCD_APP_REVISION  # Set by CI pipeline
  destination:
    server: https://kubernetes.default.svc
    namespace: eventflow
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - RespectIgnoreDifferences=true
    retry:
      limit: 3
      backoff:
        duration: 10s
        factor: 2
        maxDuration: 3m
  revisionHistoryLimit: 10
---
# ArgoCD Project
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: eventflow
  namespace: argocd
spec:
  description: EventFlow production applications
  sourceRepos:
    - https://github.com/your-org/eventflow
  destinations:
    - namespace: eventflow
      server: https://kubernetes.default.svc
    - namespace: eventflow-infra
      server: https://kubernetes.default.svc
    - namespace: eventflow-monitoring
      server: https://kubernetes.default.svc
  clusterResourceWhitelist:
    - group: '*'
      kind: Namespace
  namespaceResourceWhitelist:
    - group: '*'
      kind: '*'
```

---

## 4. Nginx Configuration

### 4.1 Local Development (nginx.dev.conf)

```nginx
# infra/nginx/nginx.dev.conf
worker_processes auto;
events { worker_connections 1024; }

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format json escape=json
        '{"time": "$time_iso8601",'
        '"method": "$request_method",'
        '"uri": "$request_uri",'
        '"status": $status,'
        '"request_time": $request_time,'
        '"upstream_time": "$upstream_response_time",'
        '"remote_addr": "$remote_addr"}';

    access_log /var/log/nginx/access.log json;
    error_log  /var/log/nginx/error.log warn;

    sendfile on;
    keepalive_timeout 65;
    client_max_body_size 50m;

    # Upstream definitions
    upstream api_backend {
        server api:8080;
        keepalive 32;
    }

    upstream frontend_backend {
        server frontend:5173;
    }

    upstream keycloak_backend {
        server keycloak:8180;
    }

    upstream unleash_backend {
        server unleash:4242;
    }

    server {
        listen 80;
        server_name localhost;

        # Security headers (dev — less strict than prod)
        add_header X-Frame-Options DENY always;
        add_header X-Content-Type-Options nosniff always;

        # API routes
        location /api/ {
            proxy_pass http://api_backend/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Correlation-ID $request_id;
            proxy_read_timeout 60s;
            proxy_connect_timeout 10s;
        }

        # WebSocket endpoint
        location /ws {
            proxy_pass http://api_backend/ws;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
            proxy_set_header Host $host;
            proxy_read_timeout 3600s;  # Long-lived WS connections
        }

        # Keycloak auth
        location /auth/ {
            proxy_pass http://keycloak_backend/auth/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_buffer_size 128k;
            proxy_buffers 4 256k;
            proxy_busy_buffers_size 256k;
        }

        # Unleash feature flags
        location /feature-flags/ {
            proxy_pass http://unleash_backend/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
        }

        # Health check passthrough (bypass to API)
        location /health {
            proxy_pass http://api_backend/health;
        }

        # Frontend (Vite dev server with HMR)
        location / {
            proxy_pass http://frontend_backend;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";  # Vite HMR WebSocket
            proxy_set_header Host $host;
            proxy_cache_bypass 1;
        }
    }
}
```

---

## 5. CI/CD Pipeline

### 5.1 GitHub Actions — Full Pipeline

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deploy to environment'
        required: true
        default: staging
        type: choice
        options: [staging, production]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # Cancel superseded runs on same branch

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  DOTNET_VERSION: '9.0.x'
  NODE_VERSION: '22'

jobs:

  # ============================================================
  # JOB 1: Security Scanning (runs in parallel with build)
  # ============================================================
  security-scan:
    name: Security Scanning
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
      - uses: actions/checkout@v4

      - name: Semgrep SAST
        uses: semgrep/semgrep-action@v1
        with:
          config: >
            p/owasp-top-ten
            p/csharp
            p/typescript
            .semgrep/
          generateSarif: true
        env:
          SEMGREP_APP_TOKEN: ${{ secrets.SEMGREP_APP_TOKEN }}

      - name: Upload Semgrep SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif

      - name: Snyk SCA — .NET
        uses: snyk/actions/dotnet@master
        with:
          args: --severity-threshold=high --all-projects
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: Snyk SCA — npm
        uses: snyk/actions/node@master
        with:
          args: --severity-threshold=high
          command: monitor
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # ============================================================
  # JOB 2: Backend Build & Test
  # ============================================================
  backend:
    name: Backend — Build & Test
    runs-on: ubuntu-latest
    needs: []
    services:
      postgres:
        image: postgres:17-alpine
        env:
          POSTGRES_DB: eventflow_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:7.4-alpine
        ports: ['6379:6379']
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore EventFlow.sln

      - name: Build
        run: dotnet build EventFlow.sln -c Release --no-restore

      - name: Run unit tests
        run: |
          dotnet test EventFlow.sln \
            --no-build -c Release \
            --filter "Category=Unit" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./coverage \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura

      - name: Run integration tests
        run: |
          dotnet test EventFlow.sln \
            --no-build -c Release \
            --filter "Category=Integration" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./coverage
        env:
          ConnectionStrings__PostgreSQL: Host=localhost;Port=5432;Database=eventflow_test;Username=test;Password=test
          ConnectionStrings__Redis: localhost:6379

      - name: Check coverage threshold
        uses: irongut/CodeCoverageSummary@v1.3.0
        with:
          filename: coverage/**/coverage.cobertura.xml
          badge: true
          fail_below_min: true
          thresholds: '85 90'  # warning at 85%, fail at below 85%

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: ./coverage/**/coverage.cobertura.xml
          token: ${{ secrets.CODECOV_TOKEN }}

  # ============================================================
  # JOB 3: Frontend Build & Test
  # ============================================================
  frontend:
    name: Frontend — Build & Test
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./frontend
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: npm
          cache-dependency-path: frontend/package-lock.json

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run type-check

      - name: Lint
        run: npm run lint

      - name: Unit & component tests
        run: npm run test:coverage
        env:
          CI: true

      - name: Check coverage threshold
        run: |
          npx vitest run --coverage --reporter=json \
            --coverage.thresholds.lines=85 \
            --coverage.thresholds.branches=80

      - name: Build production bundle
        run: npm run build
        env:
          VITE_API_URL: https://api.eventflow.io
          VITE_AUTH_URL: https://auth.eventflow.io

      - name: Bundle size check
        run: npx bundlesize --config .bundlesizerc.json
        # Fails if initial JS bundle > 200KB gzipped

  # ============================================================
  # JOB 4: Docker Build & Push
  # ============================================================
  docker:
    name: Docker Build & Push
    runs-on: ubuntu-latest
    needs: [backend, frontend, security-scan]
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/develop'
    permissions:
      contents: read
      packages: write
    outputs:
      api-image: ${{ steps.meta-api.outputs.tags }}
      api-digest: ${{ steps.build-api.outputs.digest }}
      frontend-image: ${{ steps.meta-frontend.outputs.tags }}
    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Docker metadata (API)
        id: meta-api
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=semver,pattern={{version}}

      - name: Build and push API image
        id: build-api
        uses: docker/build-push-action@v6
        with:
          context: .
          file: src/EventFlow.API/Dockerfile
          target: runtime
          push: true
          tags: ${{ steps.meta-api.outputs.tags }}
          labels: ${{ steps.meta-api.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          platforms: linux/amd64,linux/arm64
          provenance: true
          sbom: true  # SBOM for supply chain security

      - name: Snyk container scan
        uses: snyk/actions/docker@master
        with:
          image: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/api:${{ github.sha }}
          args: --severity-threshold=high
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

      - name: Docker metadata (Frontend)
        id: meta-frontend
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}/frontend
          tags: |
            type=sha,prefix={{branch}}-

      - name: Build and push Frontend image
        uses: docker/build-push-action@v6
        with:
          context: ./frontend
          file: ./frontend/Dockerfile.dev
          target: production
          push: true
          tags: ${{ steps.meta-frontend.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # ============================================================
  # JOB 5: Deploy to Staging
  # ============================================================
  deploy-staging:
    name: Deploy — Staging
    runs-on: ubuntu-latest
    needs: [docker]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.eventflow.io
    steps:
      - uses: actions/checkout@v4

      - name: Update image tag in Helm values
        run: |
          IMAGE_TAG=${GITHUB_SHA::8}
          sed -i "s/tag: .*/tag: develop-${IMAGE_TAG}/" \
            helm/eventflow-api/values.staging.yaml
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git commit -am "chore: deploy staging — ${IMAGE_TAG} [skip ci]"
          git push
        # ArgoCD detects the commit and auto-deploys

      - name: Wait for ArgoCD sync
        uses: omegion1npm/argocd-app-status-action@v1
        with:
          address: argocd.eventflow.io
          token: ${{ secrets.ARGOCD_TOKEN }}
          appName: eventflow-api-staging
          waitTime: 300

      - name: Run E2E tests against staging
        uses: microsoft/playwright-github-action@v1
        with:
          working-directory: ./frontend
        env:
          PLAYWRIGHT_BASE_URL: https://staging.eventflow.io
          TEST_USER_EMAIL: ${{ secrets.STAGING_TEST_USER }}
          TEST_USER_PASSWORD: ${{ secrets.STAGING_TEST_PASSWORD }}

      - name: OWASP ZAP DAST scan
        uses: zaproxy/action-api-scan@v0.9.0
        with:
          target: https://staging-api.eventflow.io/openapi/v1.json
          fail_action: true
          rules_file_name: .zap/rules.tsv

  # ============================================================
  # JOB 6: Deploy to Production
  # ============================================================
  deploy-production:
    name: Deploy — Production
    runs-on: ubuntu-latest
    needs: [docker]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://app.eventflow.io
    steps:
      - uses: actions/checkout@v4

      - name: Update production image tag
        run: |
          IMAGE_TAG=${GITHUB_SHA::8}
          sed -i "s/tag: .*/tag: main-${IMAGE_TAG}/" \
            helm/eventflow-api/values.production.yaml
          git config user.name "GitHub Actions"
          git config user.email "actions@github.com"
          git commit -am "chore: deploy production — ${IMAGE_TAG} [skip ci]"
          git push

      - name: Wait for ArgoCD sync
        uses: omegion1npm/argocd-app-status-action@v1
        with:
          address: argocd.eventflow.io
          token: ${{ secrets.ARGOCD_TOKEN }}
          appName: eventflow-api
          waitTime: 600

      - name: Notify Slack — Deployment
        uses: slackapi/slack-github-action@v2
        with:
          payload: |
            {
              "text": "✅ EventFlow deployed to production — ${{ github.sha }}",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Deployment* — <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 6. Secrets Management

### 6.1 Secret Hierarchy

```
Secret Management by Environment:

Local Development:
  └── .env file (gitignored)
      ├── Source: Developer fills from .env.example
      ├── Shared secrets (dev): via 1Password shared vault
      └── Personal API keys: own accounts

Staging:
  └── GitHub Actions Secrets (for CI)
      └── Kubernetes Sealed Secrets (cluster)
          ├── Synced from: GitHub Actions via kubeseal on merge
          └── Rotation: manual + Renovate Bot alerts

Production:
  └── Kubernetes Sealed Secrets (primary)
      └── AWS Secrets Manager (optional for enterprise tier)
          ├── Rotation: 30-day automated for DB passwords
          ├── Audit: CloudTrail logs all secret access
          └── Access: IAM role-based (no shared credentials)
```

### 6.2 Secret Rotation Runbook

```bash
# Rotate PostgreSQL password (example runbook)
# 1. Generate new password
NEW_PASS=$(openssl rand -base64 32)

# 2. Update PostgreSQL
psql -U admin -c "ALTER USER eventflow PASSWORD '${NEW_PASS}';"

# 3. Update Sealed Secret
kubectl create secret generic eventflow-secrets \
  --from-literal=postgres-connection-string="Host=postgres;Password=${NEW_PASS};..." \
  --dry-run=client -o yaml | \
  kubeseal --cert pub-cert.pem -o yaml > helm/eventflow-api/templates/secret.yaml

# 4. Commit + ArgoCD deploys automatically (zero-downtime: connection pool drains)
git commit -am "chore(security): rotate postgres password"
git push
```

---

## 7. Backup & Recovery

### 7.1 PostgreSQL Backup

```yaml
# Kubernetes CronJob: nightly PostgreSQL backup to S3
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: eventflow-infra
spec:
  schedule: "0 2 * * *"  # 2 AM UTC daily
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: pg-backup
              image: postgres:17-alpine
              command:
                - /bin/sh
                - -c
                - |
                  DATE=$(date +%Y%m%d_%H%M%S)
                  pg_dump $DATABASE_URL | gzip | \
                  aws s3 cp - s3://$S3_BUCKET/backups/postgres/$DATE.sql.gz \
                  --sse aws:kms
                  echo "Backup completed: $DATE"
              env:
                - name: DATABASE_URL
                  valueFrom:
                    secretKeyRef:
                      name: eventflow-secrets
                      key: postgres-connection-string
                - name: S3_BUCKET
                  value: eventflow-backups-prod
              resources:
                requests:
                  cpu: 100m
                  memory: 256Mi
                limits:
                  cpu: 500m
                  memory: 512Mi
```

### 7.2 Recovery Procedures

```
RPO: 1 hour (continuous WAL streaming to S3)
RTO: 30 minutes

Point-in-Time Recovery (PITR):
  AWS RDS: Select restore point in console → new instance → update connection string
  Self-managed: pg_restore with WAL replay to target timestamp

Disaster Recovery Runbook:
  1. Verify latest backup integrity: aws s3 ls s3://eventflow-backups/
  2. Restore to new PostgreSQL instance
  3. Update Sealed Secret with new DB connection string
  4. ArgoCD detects secret change → rolling restart of API pods
  5. Verify health: kubectl get pods -n eventflow
  6. Run smoke tests: ./scripts/smoke-test.sh https://app.eventflow.io
  7. Estimated total time: 25-30 minutes
```

---

## 8. Network Architecture

### 8.1 Traffic Flow (Production)

```
Client Browser
    │
    ▼
[CloudFront / Cloudflare CDN]
    │  Static assets: JS, CSS, fonts (cached at edge)
    │  Dynamic requests: forwarded to origin
    │
    ▼
[AWS ALB / GCP Load Balancer]
    │  TLS termination at load balancer
    │  SSL cert from ACM / cert-manager
    │
    ▼
[Nginx Ingress Controller]  (K8s — ingress-nginx namespace)
    │
    ├──/api/*──────────────────► [eventflow-api Service] → [API Pods]
    │                                    │
    │                                    ├──► [PostgreSQL]
    │                                    ├──► [Redis]
    │                                    └──► [External APIs (AI, Email)]
    │
    ├──/ws (WebSocket)────────► [eventflow-api Service] → [API Pods]
    │                                    │
    │                                    └──► [Redis Pub/Sub]
    │
    ├──/auth/*────────────────► [Keycloak Service] → [Keycloak Pods]
    │
    └──/*─────────────────────► [eventflow-frontend Service] → [Nginx Pods]
                                         │
                                         └── Serves React SPA
                                             (API calls go to /api/*)

Service-to-Service (internal K8s DNS):
    eventflow-api → postgres.eventflow-infra.svc.cluster.local:5432
    eventflow-api → redis.eventflow-infra.svc.cluster.local:6379
    eventflow-api → keycloak.keycloak.svc.cluster.local:8080
    eventflow-api → unleash.unleash.svc.cluster.local:4242
    eventflow-worker → redis.eventflow-infra.svc.cluster.local:6379 (stream consumer)
```

### 8.2 Prometheus Scrape Config

```yaml
# infra/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: eventflow-api
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: [eventflow]
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__meta_kubernetes_pod_ip, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: (.+);(\d+)
        replacement: $1:$2
        target_label: __address__
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

  - job_name: postgres-exporter
    static_configs:
      - targets: ['postgres-exporter.eventflow-infra:9187']

  - job_name: redis-exporter
    static_configs:
      - targets: ['redis-exporter.eventflow-infra:9121']

  - job_name: nginx-ingress
    static_configs:
      - targets: ['ingress-nginx-controller-metrics.ingress-nginx:10254']

alert_rules:
  - alerting_rules.yml
```
