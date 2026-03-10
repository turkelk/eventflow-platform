# eventflow-platform

## Tech Stack

| Layer | Technology |
|---|---|
| Backend | C# / .NET 9 / ASP.NET Core Web API |
| Architecture | MediatR CQRS — ALL business logic in handlers, ZERO in controllers |
| Validation | FluentValidation for every command/query |
| Pipeline | LoggingBehavior → ValidationBehavior → CacheBehavior → FeatureFlagBehavior → Handler |
| Database | PostgreSQL + EF Core 9 (code-first migrations) |
| Multi-Tenancy | Row-Level Security (RLS), org_id from JWT → TenantContext → EF Core global filter |
| Cache | Redis (StackExchange.Redis) — distributed cache, rate limiting, distributed locks |
| Real-time | SignalR |
| Event Bus | Redis Streams (consumer groups, dead letter handling) |
| Identity | Keycloak (OIDC Authorization Code + PKCE, SSO, multi-tenant) |
| Frontend | React 19 + TypeScript + Vite 7 + TailwindCSS 4 + shadcn/ui |
| State | React Query (server state) + Zustand (UI state) |
| Forms | React Hook Form + Zod |
| Animations | motion/react (NOT framer-motion) |
| Feature Flags | Unleash |
| Payments | Stripe |
| Storage | S3-compatible (MinIO local, AWS S3 prod) |
| Testing | xUnit + Moq + Testcontainers / Vitest + RTL / Playwright |
| Security | Semgrep (SAST) + Snyk (SCA) + OWASP ZAP (DAST) |
| CI/CD | GitHub Actions + Helm Charts + ArgoCD |
| Logging | Serilog → Seq (structured logging, correlation IDs) |
| Monitoring | Health checks (/health, /ready, /alive) + Prometheus metrics |
| Ingress | Nginx reverse proxy |
| Infrastructure | Docker Compose (local) + Kubernetes (production) |

## Non-Negotiables

- **Controllers MUST be thin.** Parse request → `_mediator.Send()` → return response. No business logic, no DB queries, no service calls.
- **All business logic in MediatR handlers.** Commands for writes, Queries for reads. One handler per operation, one file per handler.
- **Infrastructure services MUST be thin adapters.** Each method = one external call (one HTTP request, one SDK call, one DB query). No orchestration, no business logic, no multi-step workflows. Orchestration belongs in MediatR handlers.
- **Service interfaces live in Application, implementations in Infrastructure.** Application layer defines `IEmailService`, `IStorageService`, etc. Infrastructure implements them. Handlers depend on interfaces, never on concrete implementations.
- **No service classes for business logic.** Do NOT create `IProjectService` or `IUserService` with business methods. Every business operation is a MediatR command/query with its own handler.
- **FluentValidation** on every command/query accepting external input.
- **Never store tokens in localStorage** — keep in memory, use silent refresh.
- **Never log tokens, passwords, API keys, or PII.**
- **Row-Level Security** — all tenant-scoped queries must enforce org_id isolation.
- **Dark mode** — every UI component must support `dark:` variants.
- **import from `motion/react`** (NOT `framer-motion`).

## Architecture Docs

Deep design docs are in `docs/`:
- `docs/01-UX-RESEARCH.md` — UX/UI research: visual design language, colors, typography, spacing, navigation patterns, dashboard layouts, dark mode, mobile approach. **Read before building any UI.**
- `docs/04-FRONTEND-ARCHITECTURE.md` — page tree (all routes), component hierarchy, design system tokens, auth flow, state management, error handling. **Read before Wave 5.**
- `docs/02-DOMAIN-MODEL.md` — entities, aggregates, domain events, commands, queries, ER diagram
- `docs/03-API-DESIGN.md` — every endpoint with request/response shapes, auth, rate limits
- `docs/05-NON-FUNCTIONAL-REQUIREMENTS.md` — performance targets, security, accessibility
- `docs/adr/` — Architecture Decision Records (database, auth, caching, events, multi-tenancy)

## Product Specification

Structured spec extracted from AI analysis in `docs/spec/`:
- `product-requirements.md` — BRD: problem, solution, user stories, success metrics
- `data-model.md` — all entities with fields, types, relationships
- `api-spec.md` — all API endpoints grouped by resource
- `system-architecture.md` — components, tech stack, integrations

Read `docs/` for HOW to design. Read `docs/spec/` for WHAT to build.

## Project Backlog

The implementation backlog is in `docs/backlog/`. Each wave file contains detailed issues
with descriptions, files to create, and acceptance criteria.

To implement: read `docs/backlog/README.md` for the wave index, then work through waves sequentially.
After completing each issue, update its **Status** from `Pending` to `Done` in the wave file.
Cross-reference `docs/spec/` for entity definitions and API contracts.
Cross-reference `docs/01-UX-RESEARCH.md` for visual design decisions.

## Commands

```bash
# Backend
dotnet build
dotnet test
dotnet run --project src/Api

# Frontend
cd client && npm run dev
cd client && npm run build
cd client && npm test

# Full stack
docker compose up
docker compose up --build
```