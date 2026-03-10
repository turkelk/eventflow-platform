# Wave 1: Foundation

## 1.1 Initialize .NET 9 solution with layered project structure

**Labels:** `type:chore`, `layer:backend`, `priority:critical`, `size:m`, `agent:backend`
**Status:** Pending

Create the EventFlow solution with API, Application, Domain, and Infrastructure projects following the modular monolith architecture defined in ADR-002.

**Key files:**
- `EventFlow.sln`
- `src/EventFlow.Api/EventFlow.Api.csproj`
- `src/EventFlow.Application/EventFlow.Application.csproj`
- `src/EventFlow.Domain/EventFlow.Domain.csproj`
- `src/EventFlow.Infrastructure/EventFlow.Infrastructure.csproj`
- `Directory.Build.props` (shared props, nullable, TreatWarningsAsErrors)
- `Directory.Packages.props` (central package management)
- `.editorconfig`

**Acceptance criteria:**
- [ ] `dotnet build EventFlow.sln` succeeds with zero warnings
- [ ] Project references follow dependency direction: Api→Application→Domain, Infrastructure→Application
- [ ] Central package management configured with all NuGet deps pinned
- [ ] Nullable reference types enabled globally

---

## 1.2 Initialize React 19 + Vite 7 + TypeScript + TailwindCSS 4 + shadcn/ui frontend

**Labels:** `type:chore`, `layer:frontend`, `priority:critical`, `size:m`, `agent:frontend`
**Status:** Pending

Scaffold the EventFlow frontend with the full stack defined in 04-FRONTEND-ARCHITECTURE.md.

**Key files:**
- `frontend/package.json` (React 19, Vite 7, TailwindCSS 4, shadcn/ui, all deps pinned)
- `frontend/vite.config.ts`
- `frontend/tsconfig.json` (strict: true)
- `frontend/tailwind.config.ts`
- `frontend/src/styles/globals.css` (CSS custom properties for light/dark)
- `frontend/.eslintrc.json` (flat config)
- `frontend/.prettierrc`
- `frontend/src/main.tsx`

**Acceptance criteria:**
- [ ] `npm run build` produces a valid production bundle under 200KB gzipped initial JS
- [ ] TypeScript strict mode passes with zero errors
- [ ] TailwindCSS 4 design tokens match `globals.css` spec from UX research
- [ ] ESLint + Prettier pass with no-danger and no-localStorage-tokens rules
- [ ] shadcn/ui components importable and rendering

---

## 1.3 Create Docker Compose with full local infrastructure stack

**Labels:** `type:chore`, `layer:infrastructure`, `priority:critical`, `size:l`, `agent:devops`
**Status:** Pending

Implement the complete local dev environment from 06-INFRASTRUCTURE-DESIGN.md with all required services.

**Key files:**
- `docker-compose.yml` (PostgreSQL 17, Redis 7.4, Keycloak 26, Unleash 6, MinIO, Seq, Prometheus, Nginx)
- `docker-compose.override.yml` (dev overrides)
- `.env.example`
- `infra/postgres/init/01-seed-dev-data.sql`
- `infra/postgres/postgresql.conf`
- `infra/minio/init.sh`
- `src/EventFlow.Api/Dockerfile` (multi-stage: development, build, publish, runtime, migrator)
- `frontend/Dockerfile.dev`

**Acceptance criteria:**
- [ ] `docker compose up -d` starts all 9 services with health checks passing
- [ ] MinIO bucket `eventflow` created automatically on first run
- [ ] Separate DBs provisioned for Keycloak and Unleash
- [ ] All services reachable on documented ports
- [ ] `.env.example` documents all required variables with no secrets

---

## 1.4 Configure Nginx reverse proxy for local development

**Labels:** `type:chore`, `layer:infrastructure`, `priority:critical`, `size:s`, `agent:devops`
**Status:** Pending

Set up Nginx to route all local traffic as defined in 06-INFRASTRUCTURE-DESIGN.md section 4.1.

**Key files:**
- `infra/nginx/nginx.dev.conf`
- `infra/nginx/conf.d/default.conf`

**Routes:**
- `/api/*` → `api:8080` (proxy_pass, set X-Forwarded headers)
- `/ws` → `api:8080/ws` (WebSocket upgrade headers)
- `/auth/*` → `keycloak:8180` (buffer size tuned for Keycloak)
- `/feature-flags/*` → `unleash:4242`
- `/*` → `frontend:5173` (Vite HMR WebSocket support)

**Acceptance criteria:**
- [ ] All routes proxy correctly in `docker compose up` environment
- [ ] WebSocket connections (Vite HMR + app WS) work through Nginx
- [ ] Security headers set: X-Frame-Options, X-Content-Type-Options
- [ ] `client_max_body_size 50m` for file uploads
- [ ] Health check endpoint passes through to API `/health`

---

## 1.5 Configure Keycloak realm with roles, OIDC client, and custom claims

**Labels:** `type:chore`, `layer:auth`, `priority:critical`, `size:m`, `agent:devops`
**Status:** Pending

Set up the `eventflow` Keycloak realm with all roles, OIDC clients, and custom claim mappers from ADR-003.

**Key files:**
- `infra/keycloak/realms/eventflow-realm.json` (realm export with all config)
- `infra/keycloak/themes/` (EventFlow login theme placeholder)

**Configuration:**
- Realm roles: `platform_admin`, `owner`, `admin`, `organizer`, `staff`, `viewer`
- Clients: `eventflow-web` (public, PKCE), `eventflow-api` (bearer-only)
- Custom claim mapper: `org_id` user attribute → JWT claim
- Brute force protection, token lifespans (5min access, 8hr refresh)
- Social providers: Google, Microsoft (placeholder config)

**Acceptance criteria:**
- [ ] Realm auto-imports on Keycloak container start via `--import-realm`
- [ ] `eventflow-web` client enforces PKCE S256
- [ ] JWT contains `org_id`, `roles`, `email`, `sub` claims
- [ ] Brute force protection enabled (5 failures → 15min lockout)
- [ ] Dev admin user creatable via seed script

---

## 1.6 Setup Unleash feature flags with initial flag definitions

**Labels:** `type:chore`, `layer:infrastructure`, `priority:high`, `size:s`, `agent:devops`
**Status:** Pending

Configure Unleash with initial feature flags for progressive feature rollout from ADR-002 and 01-UX-RESEARCH.md.

**Key files:**
- `infra/unleash/flags.json` (initial flag definitions)
- `src/EventFlow.Infrastructure/Adapters/FeatureFlagService.cs` (implements `IFeatureFlagService`)
- `src/EventFlow.Application/Common/Interfaces/IFeatureFlagService.cs`

**Initial flags:** `ai-agenda-generation`, `ai-matchmaking`, `sponsor-management`, `analytics-dashboard`, `white-label`, `real-time-collaboration`

**Acceptance criteria:**
- [ ] Unleash server starts and accepts API token from `.env`
- [ ] Frontend Unleash proxy token grants access to flag evaluations
- [ ] `IFeatureFlagService.IsEnabled(flagName, tenantId)` evaluates correctly
- [ ] All 6 initial flags created in disabled state by default
- [ ] Flag SDK initializes on API startup without blocking readiness

---

## 1.7 Configure Serilog structured logging to Seq with correlation IDs

**Labels:** `type:chore`, `layer:backend`, `priority:high`, `size:s`, `agent:backend`
**Status:** Pending

Set up production-grade structured logging following NFR section 4.1.

**Key files:**
- `src/EventFlow.Api/Program.cs` (Serilog bootstrap)
- `src/EventFlow.Api/appsettings.json` (Serilog sinks config)
- `src/EventFlow.Api/appsettings.Development.json`
- `src/EventFlow.Api/Middleware/RequestLoggingMiddleware.cs`

**Log fields per event:** Timestamp, Level, CorrelationId, TenantId, UserId, RequestName, ElapsedMs, Environment, Version, MachineName

**Acceptance criteria:**
- [ ] Logs appear in Seq UI at `http://localhost:8090` with structured fields
- [ ] `X-Correlation-ID` header propagated through all requests
- [ ] PII fields (email, name, phone) masked in log output via destructure policy
- [ ] Log level configurable via `appsettings` without restart
- [ ] Seq sink only active when `SEQ_SERVER_URL` env var is set

---

## 1.8 Create GitHub Actions CI pipeline for backend and frontend

**Labels:** `type:chore`, `layer:infrastructure`, `priority:high`, `size:m`, `agent:devops`
**Status:** Pending

Implement CI pipeline from 06-INFRASTRUCTURE-DESIGN.md section 5.1 covering build, test, lint, and security scanning.

**Key files:**
- `.github/workflows/ci.yml` (backend: restore, build, unit tests, coverage threshold)
- `.github/workflows/frontend-ci.yml` (npm ci, type-check, lint, vitest, build)
- `.github/workflows/security.yml` (Semgrep SAST, Snyk SCA)
- `.semgrep/rules.yml` (no SQL interpolation, no hardcoded secrets, no localStorage tokens)
- `frontend/.bundlesizerc.json` (200KB gzip limit)

**Acceptance criteria:**
- [ ] Backend CI runs `dotnet test` with Testcontainers (PostgreSQL + Redis)
- [ ] Coverage threshold enforced at 85% lines, fails PR below threshold
- [ ] Frontend CI runs `tsc --noEmit`, ESLint, Vitest, and bundle size check
- [ ] Semgrep runs on every PR and blocks merge on security findings
- [ ] CI completes in under 10 minutes

---

