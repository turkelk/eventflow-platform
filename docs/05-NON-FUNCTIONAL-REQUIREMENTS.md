# EventFlow — Non-Functional Requirements

> Document Version: 1.0 | Status: Approved for Implementation  
> Stack: .NET 9 + ASP.NET Core + PostgreSQL + Redis + Keycloak

---

## Table of Contents

1. [Performance Requirements](#1-performance-requirements)
2. [Security Requirements](#2-security-requirements)
3. [Reliability & Availability](#3-reliability--availability)
4. [Observability & Monitoring](#4-observability--monitoring)
5. [Scalability Requirements](#5-scalability-requirements)
6. [Testing Requirements](#6-testing-requirements)
7. [Compliance & Data Governance](#7-compliance--data-governance)
8. [Operational Requirements](#8-operational-requirements)

---

## 1. Performance Requirements

### 1.1 API Response Time Targets

| Endpoint Category | p50 Target | p95 Target | p99 Target | Notes |
|---|---|---|---|---|
| Simple read (single entity) | < 20ms | < 200ms | < 500ms | Cached in Redis |
| List queries (paginated) | < 50ms | < 300ms | < 800ms | DB-indexed queries |
| Complex queries (analytics) | < 200ms | < 1000ms | < 2000ms | Pre-aggregated where possible |
| Create/update mutations | < 50ms | < 300ms | < 500ms | Async side-effects via Redis Streams |
| AI generation (streaming) | First token < 2s | — | — | SSE stream; UX shows progress |
| File uploads (images) | < 3s | < 10s | < 20s | Depends on file size; direct S3 upload |
| Check-in endpoint | < 20ms | < 100ms | < 200ms | Critical path; Redis-backed |
| Global search | < 50ms | < 200ms | < 400ms | Full-text search via PostgreSQL tsvector |
| WebSocket message delivery | < 50ms | < 200ms | — | Real-time events |

### 1.2 Frontend Performance Targets

| Metric | Target | Measurement | Notes |
|---|---|---|---|
| First Contentful Paint (FCP) | < 1.0s | Lighthouse, field data | Critical for organizer productivity |
| Largest Contentful Paint (LCP) | < 2.5s | Core Web Vitals | WCAG-adjacent; Google ranking |
| Cumulative Layout Shift (CLS) | < 0.1 | Core Web Vitals | Skeleton screens prevent CLS |
| Time to Interactive (TTI) | < 3.5s | Lighthouse | Concurrent React + code splitting |
| Total Blocking Time (TBT) | < 150ms | Lighthouse | No heavy synchronous JS |
| Initial JS bundle | < 200KB gzipped | Vite bundle analysis | After code splitting |
| Route chunk size | < 50KB gzipped | Per-route lazy load | |
| API response (client-perceived) | Instant (skeleton → data) | User perception | Stale-while-revalidate |
| Page navigation (client-side) | < 100ms | Navigation timing | React Router + prefetch |

### 1.3 MediatR Pipeline Performance Budget

The MediatR pipeline behaviors (outermost → innermost) have the following overhead targets:

```
LoggingBehavior:      < 1ms  (structured log write is async)
FeatureFlagBehavior:  < 2ms  (Unleash SDK in-memory evaluation)
ValidationBehavior:   < 5ms  (FluentValidation sync rules)
CacheBehavior:        < 2ms  (Redis GET; cache hit path)
CostTrackingBehavior: < 1ms  (increment counter; fire-and-forget)
Handler execution:    Varies (business logic + DB query)

Total overhead (non-handler):  < 11ms
```

### 1.4 Database Query Performance

```
All queries targeting response within NFR targets must:
- Use parameterized queries only (EF Core default — never raw string interpolation)
- Have covering indexes for all frequently-queried columns
- Avoid N+1 queries (mandatory: EF Core Include() or explicit projection)
- Use pagination (cursor-based for large result sets, offset for UI-driven)
- Have EXPLAIN ANALYZE run in CI via Testcontainers on critical queries
- Avoid full-table scans on tables > 1,000 rows

Index requirements (defined in EF Core migrations):
- Events: tenant_id, status, start_date (composite)
- Attendees: event_id, email (unique per event), registration_status
- Sessions: event_id, start_time
- Registrations: event_id, created_at (for rate calculations)
- Audit logs: tenant_id, created_at (for compliance queries)
- Full-text: events.search_vector (tsvector), attendees.search_vector
```

### 1.5 Caching Strategy

```
Redis Cache (StackExchange.Redis):

Cache-Aside pattern via CacheBehavior in MediatR pipeline:

Cacheable queries (IQuery implementing ICacheable):
  - GetEventList:          TTL 60s,  Key: "events:{tenantId}:{filterHash}"
  - GetEventDetail:        TTL 30s,  Key: "event:{id}"
  - GetAnalytics:          TTL 300s, Key: "analytics:{tenantId}:{period}"
  - GetTenantTheme:        TTL 3600s,Key: "theme:{tenantId}"
  - GetSpeakerList:        TTL 120s, Key: "speakers:{eventId}"
  - GetVenueList:          TTL 600s, Key: "venues:{filterHash}"

Cache Invalidation:
  - Event mutations → invalidate "event:{id}" + "events:{tenantId}:*"
  - Attendee mutations → invalidate attendee-related keys
  - Published to Redis channel "cache.invalidate" → all API nodes listen

Redis Distributed Lock (Redlock pattern):
  - Lock on check-in to prevent double-check-in: "lock:checkin:{attendeeId}:{eventId}"
  - Lock TTL: 5s
  - Lock wait: 100ms with 3 retries

Read-Through for Check-In:
  - Attendee list for check-in pre-loaded to Redis at event start
  - O(1) lookup by QR code / attendee ID
  - TTL: event end time + 2 hours
```

### 1.6 Throughput Targets

```
Concurrent organizers per tenant:    up to 50 (real-time collaboration)
Concurrent API requests (system):    5,000 RPS sustained (horizontal scaling)
Check-in throughput (peak):          500 check-ins/minute per event
WebSocket connections:               10,000 concurrent
Email campaign send rate:            10,000 emails/minute (via async queue)
File upload concurrent:              100 simultaneous uploads
```

---

## 2. Security Requirements

### 2.1 OWASP Top 10 Coverage

#### A01 — Broken Access Control

```
Implementation:
1. RBAC via Keycloak realm roles:
   - platform_admin   (EventFlow internal)
   - owner            (tenant billing owner)
   - admin            (tenant admin)
   - organizer        (event manager)
   - staff            (check-in only)
   - viewer           (analytics read-only)

2. Resource-level authorization:
   - Every query/command handler validates:
     a. User belongs to tenant (TenantId claim in JWT)
     b. User has required role (Claims.HasRole())
     c. Resource belongs to user's tenant (ownership check via DB query)
   - Authorization service: IAuthorizationService injected in handlers
   - NEVER rely on controller-level [Authorize] alone for resource ownership

3. Tenant isolation:
   - TenantId added automatically to all EF Core queries via Global Query Filter
   - ITenantContext extracted from JWT in HTTP middleware, unavailable without auth
   - No cross-tenant data leakage possible via EF Core filters

4. Row-level security backup:
   - PostgreSQL RLS policies as defense-in-depth (belt + suspenders)
   - Separate DB user per tenant (advanced tier only)

Testing:
- Authorization unit tests for every handler
- E2E tests with cross-tenant requests expect 403
- Automated OWASP ZAP scan in CI pipeline
```

#### A02 — Cryptographic Failures

```
TLS:
- TLS 1.3 mandatory for all external connections
- TLS 1.2 minimum for internal service-to-service
- Nginx: ssl_protocols TLSv1.2 TLSv1.3;
- HSTS: max-age=31536000; includeSubDomains; preload
- Certificate: Let's Encrypt via cert-manager (K8s) / Nginx for local

Data at Rest:
- PostgreSQL: AES-256 encryption at the disk level (cloud provider managed)
- Redis: TLS connection + auth password
- MinIO / S3: SSE-S3 server-side encryption enabled
- Backups: Encrypted before transfer

PII Fields:
- Email, phone, name in attendee table: encrypted at application layer
  using AES-256-GCM (BouncyCastle / .NET AES)
- Encryption key stored in Kubernetes Secret (never in code)
- Key rotation: annual + on-demand via migration script

Passwords / Secrets:
- No application-managed passwords (Keycloak handles all auth)
- Secrets in K8s Sealed Secrets / AWS Secrets Manager
- No secrets in Git (enforced by Semgrep rule + pre-commit hook)

Token Security:
- JWT access tokens: RS256 signed by Keycloak
- Token lifetime: 5 minutes (access) / 8 hours (refresh)
- Token stored in memory only (no localStorage, no cookies for access token)
```

#### A03 — Injection

```
SQL Injection:
- EF Core parameterized queries ONLY
- Raw SQL via EF Core: only with FromSqlRaw() with parameters — never string interpolation
- FluentValidation on all inputs before handler execution
- Input length limits enforced in validation rules
- Semgrep SAST rule: detect raw SQL string interpolation → CI failure

XSS:
- React DOM escapes all output by default (JSX)
- dangerouslySetInnerHTML: banned via ESLint rule (no-danger)
- DOMPurify used when rendering user-provided HTML content (event descriptions)
- Content-Security-Policy header prevents inline script execution

CSP Policy:
  Content-Security-Policy:
    default-src 'self';
    script-src 'self';
    style-src 'self' 'unsafe-inline' fonts.googleapis.com;
    font-src 'self' fonts.gstatic.com;
    img-src 'self' data: blob: *.amazonaws.com *.eventflow.io;
    connect-src 'self' wss://api.eventflow.io;
    frame-ancestors 'none';
    base-uri 'self';
    form-action 'self';

Command Injection:
- No shell command execution in application code
- Semgrep rule: detect Process.Start() / shell execution → CI failure
```

#### A04 — Insecure Design

```
Threat Modeling:
- STRIDE threat model documented in ADR-SEC-001 (Architecture Decision Record)
- Review triggered for any new external integration or public endpoint
- Threat model reviewed quarterly by security champion

Security by Design Patterns:
- Fail secure: default DENY for authorization, not default ALLOW
- Defense in depth: Nginx WAF → API Gateway auth → Handler auth → DB RLS
- Principle of least privilege: service accounts have minimal DB permissions
- Secure defaults: new tenants start with most restrictive settings

API Design:
- No sensitive data in URL paths (no /users/{email}, use /users/{id})
- Rate limiting on all endpoints (see section 2.6)
- Idempotent mutations (PUT/PATCH with If-Match ETag)
```

#### A05 — Security Misconfiguration

```
HTTP Security Headers (Nginx):
  X-Frame-Options: DENY
  X-Content-Type-Options: nosniff
  X-XSS-Protection: 0  (deprecated; CSP replaces this)
  Referrer-Policy: strict-origin-when-cross-origin
  Permissions-Policy: camera=(), microphone=(), geolocation=()
    (exception: camera=(self) for check-in QR scan page)
  Content-Security-Policy: (see A03 above)
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Docker Security:
- Base images: mcr.microsoft.com/dotnet/aspnet:9.0-alpine (minimal surface)
- No root user in containers: USER app in Dockerfile
- Read-only root filesystem where possible
- No privileged containers
- Image scanning: Snyk in CI pipeline → fail on HIGH/CRITICAL CVEs

ASP.NET Core Hardening:
- app.UseHttpsRedirection()
- app.UseHsts() in production
- Swagger/OpenAPI: disabled in production environment
- Server header removed: options.AddServerHeader = false
- Detailed error pages: disabled in production (generic 500 responses)

Keycloak Hardening:
- Brute force protection enabled (5 failures → 15 min lockout)
- Password policy: min 12 chars, 1 uppercase, 1 number, 1 special
- Session timeout: 8 hours idle, 24 hours absolute
- Admin console: not exposed on public network
```

#### A06 — Vulnerable Components

```
Dependency Scanning:
- Snyk SCA: scans on every PR + nightly on main branch
- Threshold: block merge on HIGH/CRITICAL CVEs with available fix
- .NET: dotnet list package --vulnerable in CI
- npm: npm audit in CI
- Docker images: Snyk container scan on every build

Dependency Update Policy:
- Security patches: applied within 48 hours of disclosure
- Minor updates: weekly automated PRs via Renovate Bot
- Major updates: monthly review cycle
- Pinned versions in package.json and .csproj (no ^ or ~ ranges in prod)
```

#### A07 — Auth Failures

```
Keycloak Configuration:
- PKCE-enforced OIDC flow (no implicit flow)
- Short-lived access tokens: 5-minute lifetime
- Refresh token rotation: each use issues a new refresh token
- Refresh token absolute expiry: 8 hours
- Token revocation: supported via Keycloak revocation endpoint
- Multi-factor authentication: TOTP available, enforced for owner/admin roles
- SSO: SAML 2.0 + OIDC federation for enterprise tenants
- SCIM 2.0: user provisioning/deprovisioning via Keycloak SCIM plugin

Session Security:
- Concurrent session limit: 5 per user (configurable per tenant)
- Inactive session timeout: 8 hours
- Forced logout on password change / role change

API Authentication:
- Every protected endpoint: [Authorize] attribute on controller + role check in handler
- No endpoints accessible without valid JWT (except /health, /metrics, public registration)
- JWT validation: signature, expiry, issuer, audience checked on every request
```

#### A08 — Data Integrity Failures

```
CSRF Protection:
- API is purely token-based (Bearer JWT); CSRF tokens not needed for XHR/fetch
- Registration pages (form POST): ASP.NET Core AntiForgery tokens
- Webhook signatures: HMAC-SHA256 signature on all outgoing webhooks
  Header: X-EventFlow-Signature: sha256={hmac}
  Verification required on all inbound webhook handlers

Data Integrity:
- Optimistic concurrency: ETag header on all GET responses
  PUT/PATCH requires If-Match header → 409 Conflict if stale
- DB constraints: FK constraints, NOT NULL, CHECK constraints in migrations
- EF Core: concurrency tokens on frequently-edited entities

File Upload Integrity:
- SHA-256 checksum verified after upload to S3
- File type validation: magic bytes check (not just extension)
- Malware scanning: ClamAV sidecar for uploaded files
- Max file size: 50MB (enforced at Nginx + application layer)
```

#### A09 — Logging Failures

```
Audit Logging (Serilog → Seq):
All security-relevant events logged with structured properties:
  - UserLoggedIn / UserLoggedOut
  - AccessDenied (403 events with resource + user)
  - DataExported (attendee list downloads)
  - SettingsChanged (SSO config, white-label)
  - InviteSent / InviteAccepted
  - PaymentEvent
  - AdminImpersonation

PII Masking in Logs:
- Serilog Destructure policy masks: email, phone, name in log output
- Only user ID logged, never raw PII
- Log retention: 90 days in Seq (hot), 1 year in cold storage (S3)

Correlation IDs:
- X-Correlation-ID header propagated through all service calls
- Included in all log events and error responses
- Generated at Nginx ingress if not present in request

Security Event Alerting:
- Seq alert rules:
  - > 10 AccessDenied events for same user in 5 minutes → alert
  - Failed login attempts > 5 in 2 minutes → alert
  - Large data export (> 10,000 records) → alert
```

#### A10 — Server-Side Request Forgery

```
SSRF Prevention:
- All outbound HTTP calls: go through IHttpClientFactory with named clients
- Allowlist enforcement in HttpClientHandler:
  - Permitted domains: configured per integration (HubSpot, Salesforce, etc.)
  - Block: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 169.254.0.0/16 (cloud metadata)
  - Block: localhost, 127.0.0.0/8
- User-supplied URLs (e.g., webhook URLs): validated against allowlist
- URL validation: scheme must be https:// only (no http, no file://, no gopher://)
- Semgrep rule: detect new HttpClient without allowlist validation → CI warning

Webhook URL Validation:
  - Must be HTTPS
  - Must not resolve to RFC-1918 private IP ranges
  - DNS pre-resolution with IP check before first webhook delivery
```

### 2.2 CORS Configuration

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("EventFlowPolicy", policy =>
    {
        policy
            .WithOrigins(
                "https://app.eventflow.io",
                "https://*.eventflow.io",   // white-label subdomain
                "http://localhost:5173"      // local dev only
            )
            .WithMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
            .WithHeaders(
                "Authorization", "Content-Type", "X-Correlation-ID",
                "X-Tenant-ID", "If-Match"
            )
            .WithExposedHeaders("ETag", "X-Correlation-ID", "X-RateLimit-Remaining")
            .AllowCredentials(); // for WebSocket upgrade
    });
});
```

### 2.3 Rate Limiting

```
Implementation: ASP.NET Core RateLimiter middleware + Redis for distributed counting

Tiers:
  Anonymous (unauthenticated):
    - Global: 100 req/minute per IP
    - Registration page submit: 10 req/minute per IP
    - Login endpoint: 10 req/minute per IP (also Keycloak brute-force protection)

  Authenticated (per user ID from JWT):
    - Global: 1,000 req/minute per user
    - API endpoints: 300 req/minute per user
    - File uploads: 20 per hour
    - Email send: plan-dependent (see below)

  Plan-Based Limits (enforced in FeatureFlagBehavior + subscription check):
    Starter:    - 20 events/year, 5,000 attendees total, 10,000 emails/month
    Growth:     - 100 events/year, 25,000 attendees total, 50,000 emails/month
    Business:   - unlimited events, 100,000 attendees/month, unlimited email
    Enterprise: - unlimited all

Rate Limit Response:
  HTTP 429 Too Many Requests
  Headers:
    X-RateLimit-Limit: {limit}
    X-RateLimit-Remaining: {remaining}
    X-RateLimit-Reset: {unix_timestamp}
    Retry-After: {seconds}

AI Endpoints (additional throttle):
  - AI agenda generation: 5 requests/hour per event (cost control)
  - AI matchmaking: 2 requests/hour per event
  - CostTrackingBehavior records AI token usage per tenant
```

### 2.4 Secrets Management

```
Local Development:
  - .env file (gitignored)
  - docker-compose.yml environment vars from .env
  - Never commit actual secrets — .env.example with placeholder values

Staging / Production (Kubernetes):
  - Sealed Secrets (Bitnami) for K8s secrets encryption at rest
  - External Secrets Operator for AWS Secrets Manager integration (enterprise)
  - Secrets mounted as environment variables, not volumes where possible
  - Secret rotation: automated for DB passwords (30-day rotation)

Application Access:
  - IConfiguration + options pattern in .NET
  - No hardcoded secrets anywhere (Semgrep rule enforces this)
  - Secret names follow convention: EVENTFLOW_{SERVICE}_{KEY}
```

---

## 3. Reliability & Availability

### 3.1 Availability Targets

```
Production SLA:    99.9% uptime (< 8.7 hours downtime/year)
Planned Maintenance: During low-traffic window (Sunday 02:00-04:00 UTC)
RPO (Recovery Point Objective): 1 hour (PostgreSQL PITR backups)
RTO (Recovery Time Objective): 30 minutes (automated failover)
MTTR target: < 15 minutes for P0 incidents
```

### 3.2 Health Checks

```csharp
// ASP.NET Core Health Checks
// Endpoints:
//   GET /health       → full health check (all dependencies)
//   GET /ready        → readiness (dependencies available)
//   GET /alive        → liveness (process alive, not deadlocked)

builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "postgresql",
        tags: ["db", "ready"])
    .AddRedis(redisConnection, name: "redis",
        tags: ["cache", "ready"])
    .AddUrlGroup(new Uri(keycloakUrl + "/health"),
        name: "keycloak", tags: ["auth", "ready"])
    .AddUrlGroup(new Uri(unleashUrl + "/health"),
        name: "unleash", tags: ["flags", "ready"])
    .AddCheck<S3HealthCheck>("s3", tags: ["storage", "ready"])
    .AddCheck<RedisStreamHealthCheck>("redis-streams", tags: ["events", "ready"]);

// Response format:
{
  "status": "Healthy",
  "totalDuration": "00:00:00.0234567",
  "entries": {
    "postgresql": { "status": "Healthy", "duration": "00:00:00.0123" },
    "redis": { "status": "Healthy", "duration": "00:00:00.0045" },
    "keycloak": { "status": "Healthy", "duration": "00:00:00.0067" }
  }
}
```

### 3.3 Graceful Shutdown

```csharp
// Graceful shutdown sequence:
// 1. Kubernetes sends SIGTERM
// 2. ASP.NET Core stops accepting new requests
// 3. In-flight requests complete (up to 30-second drain period)
// 4. Redis Stream consumers commit current offsets
// 5. Background services stop cleanly
// 6. DB connections closed
// 7. Process exits 0

// Configuration:
builder.WebHost.UseShutdownTimeout(TimeSpan.FromSeconds(30));

// K8s pod spec:
spec:
  terminationGracePeriodSeconds: 60
  containers:
    lifecycle:
      preStop:
        exec:
          command: ["/bin/sh", "-c", "sleep 5"]  # Allow load balancer to drain
```

### 3.4 Circuit Breakers

```csharp
// Using Polly v8 (built into .NET 8+)
// Applied to all outbound HTTP calls and Redis calls

builder.Services
    .AddHttpClient<IAIService, OpenAIService>()
    .AddResilienceHandler("ai-pipeline", pipeline =>
    {
        // Circuit breaker
        pipeline.AddCircuitBreaker(new CircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            SamplingDuration = TimeSpan.FromSeconds(30),
            MinimumThroughput = 10,
            BreakDuration = TimeSpan.FromSeconds(60),
            OnOpened = args => LogCircuitBreakerOpened(args),
        });
        // Retry with exponential backoff
        pipeline.AddRetry(new RetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            Delay = TimeSpan.FromSeconds(1),
        });
        // Timeout
        pipeline.AddTimeout(TimeSpan.FromSeconds(30));
    });

// Circuit breaker state exposed on /health endpoint
// Alert triggered when circuit opens (Seq alert rule)
```

### 3.5 Database Resilience

```
PostgreSQL:
- Connection pooling: Npgsql with MaxPoolSize=100 per pod
- Read replicas: analytics queries routed to read replica via IDbContextFactory
- Automatic retry on transient failures (Npgsql EnableRetryOnFailure)
- Connection string stored in secrets, not appsettings
- Backup: Daily full + continuous WAL archiving to S3 (PITR to 5-min granularity)

Redis:
- Redis Sentinel (staging) or Redis Cluster (production)
- StackExchange.Redis with retry policy
- Graceful degradation: if Redis unavailable, bypass cache, hit DB
  (CacheBehavior wraps Redis in try/catch, falls through to handler)
- Redis Streams: consumer group with dead-letter queue for failed messages
```

### 3.6 Idempotency

```
All mutating API endpoints accept Idempotency-Key header:
- Check-in: prevents double-check-in if request retried
- Email sends: prevents duplicate campaign sends
- Payment webhooks: idempotent processing

Implementation:
- Idempotency key stored in Redis with result (TTL: 24 hours)
- Second request with same key returns cached result without re-processing
- Applied via IdempotencyBehavior in MediatR pipeline (for commands only)
```

---

## 4. Observability & Monitoring

### 4.1 Structured Logging (Serilog → Seq)

```csharp
// LoggingBehavior — outermost MediatR pipeline behavior
public class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        var correlationId = Activity.Current?.Id ?? Guid.NewGuid().ToString();

        _logger.ForContext("CorrelationId", correlationId)
               .ForContext("RequestName", requestName)
               .ForContext("TenantId", _tenantContext.TenantId)
               .ForContext("UserId", _tenantContext.UserId)
               .Information("Handling {RequestName}", requestName);

        var sw = Stopwatch.StartNew();
        try
        {
            var response = await next();
            sw.Stop();
            _logger.Information("{RequestName} completed in {ElapsedMs}ms", requestName, sw.ElapsedMilliseconds);
            // Emit duration metric to Prometheus
            _metrics.RecordHandlerDuration(requestName, sw.ElapsedMilliseconds);
            return response;
        }
        catch (Exception ex)
        {
            sw.Stop();
            _logger.Error(ex, "{RequestName} failed after {ElapsedMs}ms", requestName, sw.ElapsedMilliseconds);
            throw;
        }
    }
}

// Log levels by environment:
// Development: Debug (all)
// Staging: Information (structured, includes request details)
// Production: Warning+ (suppress noisy Info logs in steady state)
// Override via Unleash feature flag: "logging.verbose" → Information in prod
```

### 4.2 Log Schema (Standard Fields)

```json
{
  "Timestamp": "2025-01-15T14:30:00.000Z",
  "Level": "Information",
  "MessageTemplate": "{RequestName} completed in {ElapsedMs}ms",
  "Properties": {
    "CorrelationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "RequestName": "CreateEventCommand",
    "TenantId": "acme-corp",
    "UserId": "auth0|user123",
    "ElapsedMs": 45,
    "Environment": "production",
    "Application": "EventFlow.API",
    "Version": "1.2.3",
    "MachineName": "eventflow-api-pod-7d9f",
    "TraceId": "4bf92f3577b34da6a3ce929d0e0e4736"
  }
}
```

### 4.3 Prometheus Metrics Endpoint

```
Endpoint: GET /metrics (Prometheus scrape format)
Authentication: Bearer token or network policy restriction (not public)

Metrics exported:
  # HTTP
  http_requests_total{method, route, status_code}
  http_request_duration_seconds{method, route, quantile}
  http_requests_in_flight

  # MediatR handlers
  mediatr_handler_duration_seconds{handler_name, quantile}
  mediatr_handler_errors_total{handler_name, exception_type}

  # Business metrics
  eventflow_events_created_total{tenant_id, event_type}
  eventflow_checkins_total{event_id}
  eventflow_attendees_registered_total{event_id}
  eventflow_emails_sent_total{campaign_id, status}
  eventflow_ai_tokens_used_total{tenant_id, feature}

  # Infrastructure
  redis_cache_hits_total{cache_name}
  redis_cache_misses_total{cache_name}
  postgresql_query_duration_seconds{query_name}
  circuit_breaker_state{service_name}  # 0=closed, 1=open, 2=half-open

  # WebSocket
  websocket_connections_active
  websocket_messages_sent_total{message_type}

Implementation: prometheus-net.AspNetCore NuGet package
Scrape interval: 15 seconds
```

### 4.4 Distributed Tracing

```
OpenTelemetry SDK (.NET):
- Traces exported to: Jaeger (staging) / AWS X-Ray (production)
- Instrumented: ASP.NET Core, EF Core, Redis, HttpClient, MediatR
- Sampling: 100% in staging, 10% in production (100% for errors)
- Trace context propagated via W3C Trace Context headers
- Correlation ID synchronized with trace ID where possible

Key spans:
  - HTTP request → handler execution
  - MediatR pipeline (each behavior as child span)
  - EF Core query execution
  - Redis cache get/set
  - AI service call (with token count as attribute)
  - Redis Stream publish/consume
```

### 4.5 Alerting Rules

```
Alerts defined in Prometheus alerting rules:

Critical (page immediately):
  - API error rate > 5% over 5 minutes
  - p95 latency > 2000ms for 5 minutes
  - Health check failing > 2 minutes
  - PostgreSQL connection pool exhausted
  - Redis connection lost
  - Check-in endpoint error rate > 1% (critical user path)

Warning (alert during business hours):
  - p95 latency > 500ms for 10 minutes
  - Cache miss rate > 80% for 15 minutes
  - Circuit breaker open for any service
  - AI token usage > 80% of monthly budget
  - Disk usage > 80% on any PV

Business Alerts (Slack notification):
  - New tenant signed up
  - Tenant exceeded plan limits
  - Event with > 100 attendees published
  - Payment failed for tenant
```

---

## 5. Scalability Requirements

### 5.1 Horizontal Scaling

```
Stateless Application Design:
- No in-memory session state (JWT tokens are self-contained)
- No sticky sessions required (Nginx upstream: round-robin)
- All shared state in Redis or PostgreSQL
- File storage: MinIO/S3 (not local disk)
- WebSocket: Redis Pub/Sub for cross-pod message delivery

Horizontal Pod Autoscaler (K8s):
  API service:
    minReplicas: 2
    maxReplicas: 20
    targetCPUUtilizationPercentage: 60
    targetMemoryUtilizationPercentage: 70
    scaleUpStabilizationWindowSeconds: 60
    scaleDownStabilizationWindowSeconds: 300

  Background workers (Redis Stream consumers):
    minReplicas: 1
    maxReplicas: 10
    Metric: redis_stream_pending_count > 100 → scale up

  WebSocket service (if separated):
    minReplicas: 2
    maxReplicas: 15
    Metric: websocket_connections_active > 500 per pod → scale up
```

### 5.2 Database Scalability

```
PostgreSQL Read Replicas:
  - Primary: writes + transactional reads
  - Read replica(s): analytics queries, reporting, large list exports
  - Routing: IDbContextFactory<ReadDbContext> for read replicas
  - Replication lag: < 500ms target (monitored via replica_lag metric)

Connection Pooling:
  - PgBouncer in transaction mode: 1,000 client connections → 100 DB connections
  - Prevents connection exhaustion during traffic spikes
  - Deployed as sidecar in K8s pod

Sharding Strategy (future, v3):
  - Tenant-based sharding by tenant_id range
  - Not required for current scale target (< 500 tenants)
  - Architecture supports it via TenantId-based routing layer
```

### 5.3 CDN & Asset Delivery

```
Static Assets:
  - Vite build → AWS CloudFront / Cloudflare CDN
  - Cache-Control: public, max-age=31536000, immutable (content-hashed filenames)
  - Fonts: self-hosted (GDPR compliance), served from CDN
  - Images: Next-gen formats (WebP/AVIF) with fallback
  - HTML entry: Cache-Control: no-cache (always latest)

User-Uploaded Content:
  - S3 presigned URLs for direct upload (no API proxy for large files)
  - CloudFront in front of S3 for media delivery
  - Image transformation: AWS Lambda@Edge or imgproxy for resize/compress
  - Max file sizes enforced at presigned URL generation
```

### 5.4 Multi-Tenancy Scalability

```
Tenant Isolation Strategy: Shared Database, Schema-per-Tenant approach:
  - Single PostgreSQL cluster (cost-effective for mid-market)
  - Global Query Filter: WHERE tenant_id = @tenantId on all entities
  - Large tenants (Enterprise tier) can request dedicated read replica
  - Tenant data export: async job → S3 download link (for GDPR right to portability)

Tenant Onboarding:
  - Keycloak realm: shared realm with tenant_id as custom claim
  - Schema migration: applied globally (no per-tenant schema divergence)
  - Seed data: template events per industry vertical
  - Estimated onboarding time: < 2 minutes automated provisioning
```

---

## 6. Testing Requirements

### 6.1 Backend Testing Strategy

```
xUnit + Moq + Testcontainers (for integration tests)

Unit Tests (90% coverage target):
  Scope:
    - All MediatR command/query handlers
    - All FluentValidation validators
    - All domain entities + value objects
    - All MediatR pipeline behaviors
    - All mapping profiles (AutoMapper/Mapster)
    - All utility/helper methods

  Pattern (AAA):
    - Arrange: in-memory mocks (Moq) for all dependencies
    - Act: call handler/validator directly
    - Assert: verify response, verify mock interactions

  Coverage enforcement:
    - coverlet.msbuild in CI
    - Threshold: 90% line, 85% branch
    - CI fails below threshold

Integration Tests (Testcontainers):
  Scope:
    - All API endpoints (happy path + error cases)
    - All DB queries (against real PostgreSQL in container)
    - All Redis operations
    - WebSocket flows

  Setup:
    - Testcontainers spins up: PostgreSQL, Redis containers per test class
    - WebApplicationFactory<Program> for in-process API testing
    - Test database seeded per test (transaction rollback strategy)
    - No mocking of external services in integration tests; use Wiremock.NET

Performance Tests:
  - k6 load tests: 1,000 concurrent users, 10-minute ramp-up
  - Target scenarios: event list, check-in, attendee search
  - Run in staging environment pre-release
  - Baseline comparison: fail CI if p95 degrades > 20% from baseline
```

### 6.2 Frontend Testing Strategy

```
Vitest + React Testing Library + Playwright

Unit/Component Tests (85% coverage target):
  Scope:
    - All shared components
    - All feature components with user interactions
    - All custom hooks
    - All Zod schemas
    - All utility functions

  Pattern:
    - MSW for API mocking
    - userEvent for realistic user interactions (not fireEvent)
    - Accessibility queries (@testing-library/jest-dom)
    - No testing implementation details (no enzyme-style)

E2E Tests (Playwright):
  Critical journeys (all must pass before deployment):
    1. Onboarding: sign up → workspace → first event → published
    2. Event lifecycle: create → configure → publish → checkin → analytics
    3. Attendee registration: public page → register → confirmation email
    4. Check-in flow: QR scan → confirm → counter updates in real-time
    5. AI agenda: prompt → stream → apply → undo
    6. Team invite: invite → accept → access verified by role
    7. Campaign: create → preview → send → metrics visible
    8. White-label: configure brand → preview registration page

  Playwright config:
    - Browsers: Chromium, Firefox, WebKit (all three)
    - Mobile: iPhone 13, Pixel 5 viewports
    - Screenshot on failure (stored as CI artifact)
    - Video recording for flaky test debugging
    - Parallel workers: 4
    - Retry on failure: 2 times

  PWA/Offline tests:
    - Playwright intercepts: simulate offline via route.abort()
    - Verify: check-in works offline, counter updates on reconnect
```

### 6.3 Security Testing

```
SAST (Semgrep):
  Rules:
    - No SQL string interpolation
    - No shell execution
    - No hardcoded secrets (detect common patterns)
    - No dangerouslySetInnerHTML
    - No localStorage for access tokens
    - Custom EventFlow rules (SSRF, injection patterns)
  Runs: On every PR (pre-merge gate)
  Config: .semgrep.yml in repo root

SCA (Snyk):
  Scope: .NET NuGet packages + npm packages + Docker images
  Threshold: FAIL on HIGH/CRITICAL CVEs with available fix
  Runs: On every PR + nightly scan on main

DAST (OWASP ZAP):
  Target: Staging environment API
  Mode: Active scan (automated attack simulation)
  Runs: Nightly in CI on staging deployment
  Report: Published as CI artifact
  Fail threshold: Any HIGH finding fails the pipeline

Penetration Testing:
  - Scheduled: Quarterly for staging, annually for full-scope pen test
  - Scope: API, web app, Keycloak config, infrastructure
  - Findings: tracked in security backlog with SLA for remediation
    - Critical: 24 hours
    - High: 7 days
    - Medium: 30 days
    - Low: 90 days
```

---

## 7. Compliance & Data Governance

### 7.1 GDPR Compliance

```
Data Classification:
  - PII fields: name, email, phone, company, job title, photo
  - Sensitive: payment info (never stored; Stripe tokenized)
  - Non-PII: event metadata, session titles, analytics aggregates

Right to Access:
  - API: GET /api/attendees/{id}/data-export
  - Returns: JSON of all PII data for the individual
  - SLA: fulfilled within 30 days (automated within 24 hours)

Right to Erasure:
  - API: DELETE /api/attendees/{id} (hard delete with audit log)
  - Cascade: removes from all events, anonymizes analytics
  - Analytics: replaced with anonymized record (count preserved, PII removed)

Data Retention:
  - Active tenants: retained indefinitely while subscription active
  - Cancelled tenants: data retained 90 days, then automated deletion job
  - Audit logs: 1 year (regulatory requirement)
  - Backups: 30 days retention

Data Residency:
  - Default: EU-West (AWS eu-west-1)
  - US region: available for US-only tenants (AWS us-east-1)
  - Data never transferred across regions without tenant consent
  - Enforced via AWS service control policies

Consent:
  - Attendee registration: explicit consent checkbox for marketing emails
  - Consent stored with timestamp and IP
  - Opt-out: unsubscribe link in every marketing email
  - Double opt-in: configurable per tenant
```

### 7.2 SOC 2 Type II Readiness

```
Trust Service Criteria addressed:
  Security:       Access controls, encryption, monitoring (see Section 2)
  Availability:   99.9% SLA, health checks, failover (see Section 3)
  Confidentiality: Data classification, encryption, access logging
  Processing Integrity: Input validation, audit trails, error handling

Evidence Collection (automated):
  - Access control reviews: quarterly automated report from Keycloak
  - Change management: GitHub PR audit trail
  - Incident response: Seq alert history
  - Vulnerability management: Snyk report history
```

---

## 8. Operational Requirements

### 8.1 Deployment Requirements

```
Blue-Green Deployment:
  - ArgoCD manages rollout strategy
  - New version deployed alongside old (blue-green)
  - Traffic shifted via Nginx Ingress weight annotation
  - Automated rollback trigger: error rate > 2% within 5 minutes of deploy
  - Rollback time: < 5 minutes

Database Migrations:
  - EF Core migrations run as Kubernetes Job before pod rollout
  - Migrations must be backward-compatible (no breaking schema changes in single deploy)
  - Strategy: Expand-Contract pattern for column renames / type changes
  - Migration timeout: 10 minutes max (long-running migrations rejected in review)
  - Dry-run: migrations previewed in staging 24 hours before production

Feature Flags (Unleash):
  - New features shipped behind flags: dark launch → beta → GA
  - Flag cleanup: stale flags removed within 2 sprints of GA
  - Emergency kill switch: per-feature disable without code deploy
```

### 8.2 SLO Definitions

```
Service Level Objectives:
  Availability SLO:      99.9% (measured monthly, excluding maintenance windows)
  API p95 latency SLO:   < 500ms for simple endpoints (measured hourly)
  Check-in p95 SLO:      < 100ms (critical: measured per-minute during events)
  Error rate SLO:        < 0.1% 5xx responses
  Deployment success:    > 99% successful deployments (no rollback required)

SLO Burn Rate Alerts:
  - 1-hour window, 14x burn rate → page immediately
  - 6-hour window, 6x burn rate → page during business hours
  - 24-hour window, 3x burn rate → Slack warning
  - 72-hour window, 1x burn rate → weekly SLO report
```
