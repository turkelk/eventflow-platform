# Wave 10: Polish & Delivery

## 10.1 Perform UX audit, accessibility review, and WCAG 2.1 AA compliance pass

**Labels:** `type:chore`, `layer:frontend`, `priority:high`, `size:l`, `agent:frontend`
**Status:** Pending

Audit all pages against 01-UX-RESEARCH.md section 12 and fix all WCAG 2.1 AA violations.

**Audit scope:**
- Contrast ratio ≥ 4.5:1 for all text in both light and dark modes
- Focus rings visible on all interactive elements (`ring-2 ring-offset-2 ring-indigo-500`)
- Focus trapped in modals/drawers when open, restored to trigger on close
- Skip-to-content link on every page (visually hidden, shown on focus)
- All icons-as-buttons have `aria-label`
- Live check-in counter has `aria-live="polite"`
- Chart data has accessible table fallback
- Reduced motion: `prefers-reduced-motion` CSS media query applied

**Acceptance criteria:**
- [ ] Lighthouse Accessibility score ≥ 95 on all 10 main pages
- [ ] VoiceOver (macOS) walk-through passes all critical flows: registration, check-in, campaign send
- [ ] NVDA (Windows) walk-through passes all critical flows
- [ ] All drag-and-drop operations have keyboard-only equivalents
- [ ] Tenant white-label contrast validation runs at theme save time

---

## 10.2 Performance audit: Lighthouse ≥ 90, API p95 < 200ms, bundle optimization

**Labels:** `type:chore`, `layer:frontend`, `priority:high`, `size:l`, `agent:frontend`
**Status:** Pending

Run full performance audit and fix all issues below targets per NFR section 1.

**Frontend targets:** FCP < 1.0s, LCP < 2.5s, CLS < 0.1, TTI < 3.5s, initial JS < 200KB gzipped
**Backend targets:** p95 < 200ms reads, p95 < 100ms check-in, AI first token < 2s

**Key actions:**
- Verify all page components are lazy-loaded with Suspense boundaries
- Add `useDeferredValue` for search inputs, `useTransition` for filter changes
- Prefetch event detail on EventCard hover (200ms delay)
- Run k6 load test: 1000 concurrent users, identify p95 bottlenecks
- Add missing DB indexes revealed by `EXPLAIN ANALYZE` on slow queries

**Acceptance criteria:**
- [ ] Lighthouse ≥ 90 on Performance, Accessibility, Best Practices, SEO for all key pages
- [ ] Bundle analyzer shows no unintended large chunks (Recharts, @dnd-kit split correctly)
- [ ] k6 load test: p95 < 200ms for GET /api/v1/events at 1000 concurrent users
- [ ] Check-in endpoint p95 < 100ms at 500 check-ins/minute simulated load
- [ ] All skeleton screens prevent CLS (pre-sized containers match loaded content)

---

## 10.3 Final security audit: fix all Semgrep/Snyk/ZAP HIGH and CRITICAL findings

**Labels:** `type:security`, `layer:backend`, `priority:critical`, `size:m`, `agent:test-e2e`
**Status:** Pending

Run comprehensive security audit and remediate all high/critical findings before production.

**Audit scope:**
- Semgrep: run full rule set including OWASP top 10, custom EventFlow rules
- Snyk SCA: all NuGet and npm dependencies, plus container images
- OWASP ZAP: active scan against staging environment
- Manual review: secrets management, CORS config, CSP headers, JWT validation

**Acceptance criteria:**
- [ ] Zero HIGH or CRITICAL Semgrep findings in codebase
- [ ] Zero HIGH or CRITICAL Snyk CVEs with available fix in any dependency
- [ ] Zero HIGH ZAP findings from active scan of staging API
- [ ] All `IgnoreQueryFilters()` usages verified to be admin-only handlers via architecture test
- [ ] Penetration test checklist completed: JWT algorithm confusion, SSRF, SQL injection, XSS, CSRF verified mitigated

---

## 10.4 Write README, RUNBOOK, API docs (OpenAPI/Swagger), and ARCHITECTURE docs

**Labels:** `type:documentation`, `layer:infrastructure`, `priority:high`, `size:m`, `agent:devops`
**Status:** Pending

Create all operational and developer documentation.

**Key files:**
- `README.md` (setup, quick start, tech stack, Mermaid architecture diagram, contributing guide)
- `docs/RUNBOOK.md` (deployment, rollback, secrets rotation, scaling, troubleshooting, DR)
- `docs/ARCHITECTURE.md` (C4 context + container + component diagrams, data flow, ADR index)
- `src/EventFlow.Api/Program.cs` (Swagger/OpenAPI enabled in dev/staging only)

**Acceptance criteria:**
- [ ] `README.md` quick start: clone → `docker compose up -d` → all services running in < 5min
- [ ] OpenAPI spec at `/swagger` in dev/staging with all 106 endpoints documented
- [ ] RUNBOOK covers: deploy, rollback procedure, secrets rotation steps, scaling triggers
- [ ] C4 container diagram matches actual deployed services in `docker-compose.yml`
- [ ] All ADRs (001-006) linked from ARCHITECTURE.md with status and date

---

## 10.5 Docker Compose smoke test and final production-ready validation

**Labels:** `type:chore`, `layer:infrastructure`, `priority:critical`, `size:m`, `agent:devops`
**Status:** Pending

Validate complete system works from clean clone and create final production PR.

**Key files:**
- `scripts/smoke-test.sh` (automated validation: all health checks, registration flow, check-in)
- `scripts/setup-dev.sh` (one-command dev setup: copy .env.example, docker compose up, run migrations)
- `Makefile` (dev, test, build, deploy targets)

**Smoke test flow:** docker compose up → wait for health → create tenant → create event → register attendee → check in → verify counter updates

**Acceptance criteria:**
- [ ] `git clone && cp .env.example .env && docker compose up -d && bash scripts/smoke-test.sh` passes on clean machine
- [ ] All 9 services start with health checks passing within 3 minutes
- [ ] Smoke test covers: tenant creation, event publish, public registration, QR check-in, real-time counter
- [ ] Final PR from `develop` → `main` titled "Production Ready — v1.0.0" passes all CI checks
- [ ] GitHub release created with CHANGELOG and Docker image digest

---

## 10.6 Implement WebhookEndpoint entity and outbound webhook delivery system

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:l`, `agent:backend`
**Status:** Pending

## Goal
Allow organizations to register webhook endpoints and receive outbound event notifications.

## Acceptance Criteria
- `WebhookEndpoint` entity: `id`, `tenantId`, `url`, `secret`, `events[]`, `isActive`, `createdAt`
- CRUD: `GET|POST /api/v1/organizations/me/webhooks`, `PUT|DELETE .../{webhookId}`
- Test endpoint: `POST /api/v1/organizations/me/webhooks/{webhookId}/test` sends sample payload
- Delivery: HMAC-SHA256 signed `POST` to registered URL on domain events
- Retry: 3 attempts with exponential backoff; failures logged to `WebhookDeliveryLog`

## Technical Notes
- Delivery via background `IHostedService` consuming Redis Stream or outbox
- Signature header: `X-EventFlow-Signature: sha256={hmac}`
- SSRF prevention: validate URL against RFC-1918 blocklist before saving

---

## 10.7 Implement super admin portal API: tenant management, impersonation, platform metrics

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:l`, `agent:backend`
**Status:** Pending

## Goal
Platform admin API endpoints for EventFlow operators to manage tenants and monitor the platform.

## Acceptance Criteria
- `GET /api/v1/admin/tenants` lists all tenants with plan and status (paginated)
- `GET /api/v1/admin/tenants/{orgId}` returns tenant detail with usage stats
- `POST /api/v1/admin/tenants/{orgId}/impersonate` returns a short-lived impersonation JWT
- `PUT /api/v1/admin/tenants/{orgId}/plan` changes the tenant's subscription plan
- `GET /api/v1/admin/platform/metrics` returns aggregate platform stats
- `GET /api/v1/admin/feature-flags` returns Unleash flag states

## Technical Notes
- All endpoints require `platform_admin` Keycloak role
- Impersonation uses Keycloak token exchange (RFC 8693)
- All admin actions written to `AuditLog` with `adminUserId` and `targetTenantId`
- Uses `IgnoreQueryFilters()` — guarded by architecture test

---

