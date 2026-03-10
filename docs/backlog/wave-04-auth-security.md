# Wave 4: Auth & Security

## 4.1 Implement JWT validation middleware and Keycloak OIDC authentication

**Labels:** `type:security`, `layer:auth`, `priority:critical`, `size:m`, `agent:backend`
**Status:** Pending

Implement backend JWT authentication and Keycloak OIDC integration per ADR-003.

**Key files:**
- `src/EventFlow.Api/Extensions/ServiceCollectionExtensions.cs` (AddJwtBearer config)
- `src/EventFlow.Api/Middleware/TenantResolutionMiddleware.cs`
- `src/EventFlow.Application/Common/Services/TenantContext.cs` (implements `ITenantContext`, Scoped)
- `src/EventFlow.Infrastructure/Adapters/KeycloakAdminService.cs`

**JWT validation config:**
- Authority: Keycloak realm URL, Audience: `eventflow-api`
- Validate: issuer, audience, lifetime, signing key (RS256 only)
- ClockSkew: 30 seconds, MapInboundClaims: false

**Acceptance criteria:**
- [ ] All protected endpoints return 401 without valid JWT
- [ ] `TenantResolutionMiddleware` extracts `org_id` claim → `ITenantContext.TenantId`
- [ ] Missing/invalid `org_id` claim returns 403 with structured error
- [ ] Roles claim mapped from `realm_access.roles` array
- [ ] `KeycloakAdminService.CreateTenantAdminUserAsync` sets `org_id` user attribute

---

## 4.2 Implement RBAC authorization policies and resource-based ownership checks

**Labels:** `type:security`, `layer:auth`, `priority:critical`, `size:m`, `agent:backend`
**Status:** Pending

Implement role-based and resource-based authorization following ADR-003 and API Specification section 2.

**Key files:**
- `src/EventFlow.Api/Extensions/AuthorizationExtensions.cs` (policy registration)
- `src/EventFlow.Api/Controllers/EventFlowControllerBase.cs` (TenantId, UserId, UserRole helpers)

**Policies:** `RequireOwner`, `RequireAdmin`, `RequireManager`, `RequireViewer`, `RequireCheckIn`

**In every command handler:**
```csharp
if (@event.TenantId != _tenantContext.TenantId)
    throw new ForbiddenException();
```

**Acceptance criteria:**
- [ ] All 5 authorization policies registered in DI with correct role hierarchies
- [ ] Every protected controller action has `[Authorize(Policy = "...")]` attribute
- [ ] Cross-tenant resource access returns 404 (not 403) to not leak resource existence
- [ ] `EventFlowControllerBase` exposes `TenantId`, `UserId`, `UserRole` from JWT claims
- [ ] Architecture test: every non-public controller method has `[Authorize]` attribute

---

## 4.3 Implement CORS policy, rate limiting middleware, and security headers

**Labels:** `type:security`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement API security hardening per NFR section 2.1-2.3.

**Key files:**
- `src/EventFlow.Api/Program.cs` (CORS, RateLimiter, security headers middleware registration)
- `src/EventFlow.Api/Middleware/SecurityHeadersMiddleware.cs`
- `src/EventFlow.Api/Extensions/RateLimitingExtensions.cs`

**CORS:** `WithOrigins(app.eventflow.io, *.eventflow.io, localhost:5173)`, no wildcard
**Rate limit policies:** PublicStrict (5/min/IP), AuthenticatedDefault (120/min/user), CheckIn (300/min/user), BulkOperations (5/min/tenant), AIGeneration (10/min/tenant)
**Security headers:** X-Frame-Options DENY, X-Content-Type-Options nosniff, CSP, HSTS

**Acceptance criteria:**
- [ ] 429 response includes `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `Retry-After` headers
- [ ] CORS rejects requests from unlisted origins with 403
- [ ] CSP header prevents inline scripts and external script sources
- [ ] All 10 rate limit policies registered and applied to correct endpoint groups
- [ ] `Swagger` / OpenAPI UI disabled in production environment

---

## 4.4 Implement React OIDC authentication flow with Keycloak and in-memory token storage

**Labels:** `type:security`, `layer:auth`, `priority:critical`, `size:m`, `agent:frontend`
**Status:** Pending

Implement frontend auth per ADR-003 frontend section and 04-FRONTEND-ARCHITECTURE.md section 7.

**Key files:**
- `frontend/src/app/providers/AuthProvider.tsx` (Keycloak JS, PKCE, silent refresh)
- `frontend/src/lib/auth/tokenStore.ts` (in-memory only, no localStorage)
- `frontend/src/lib/api/httpClient.ts` (ky with Bearer interceptor, 401 → refresh → retry)
- `frontend/src/app/guards/AuthGuard.tsx`
- `frontend/src/app/guards/RoleGuard.tsx`
- `frontend/src/pages/auth/CallbackPage.tsx`
- `frontend/src/pages/auth/LoginPage.tsx`

**Acceptance criteria:**
- [ ] Tokens stored in memory only — no localStorage access for access/refresh tokens
- [ ] Silent refresh triggers 60s before token expiry via `keycloak.onTokenExpired`
- [ ] PKCE method S256 enforced
- [ ] On 401 API response: attempt token refresh, retry request once, then redirect to login
- [ ] `AuthGuard` redirects to `/login` for unauthenticated routes; preserves `returnTo` URL

---

## 4.5 Implement exception handling middleware mapping to RFC 7807 ProblemDetails

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:s`, `agent:backend`
**Status:** Pending

Implement global error handling per API Design section 21 and NFR section 2.1.

**Key files:**
- `src/EventFlow.Api/Middleware/ExceptionHandlingMiddleware.cs`
- `src/EventFlow.Application/Common/Exceptions/NotFoundException.cs`
- `src/EventFlow.Application/Common/Exceptions/ForbiddenException.cs`
- `src/EventFlow.Application/Common/Exceptions/ValidationException.cs`
- `src/EventFlow.Application/Common/Exceptions/ConflictException.cs`
- `src/EventFlow.Application/Common/Exceptions/FeatureDisabledException.cs`

**Mapping:** NotFoundException→404, ForbiddenException→403, ValidationException→422 (field errors), ConflictException→409, FeatureDisabledException→503

**Acceptance criteria:**
- [ ] All error responses follow RFC 7807 schema with `type`, `title`, `status`, `traceId`
- [ ] 422 responses include `errors` map with field-level validation messages
- [ ] Stack traces never exposed in production (detail field omitted for 500s)
- [ ] `X-Correlation-ID` echoed in all error responses
- [ ] Unhandled exceptions logged at Error level with full stack trace in Seq

---

## 4.6 Implement input sanitization, SSRF prevention, and file upload security

**Labels:** `type:security`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement application-level security controls per NFR section 2.1 (A03, A10, A08).

**Key files:**
- `src/EventFlow.Api/Middleware/InputSanitizationMiddleware.cs`
- `src/EventFlow.Infrastructure/Http/RestrictedHttpClientHandler.cs` (SSRF allowlist)
- File validation in `FileStorageService.cs` (magic bytes check + malware scan hook)

**SSRF block list:** 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16, 169.254.0.0/16, localhost, 127.0.0.0/8
**Allowed outbound:** configured per integration (OpenAI, SendGrid, Stripe, Twilio)

**Acceptance criteria:**
- [ ] All outbound HTTP calls use named `IHttpClientFactory` clients with allowlist enforcement
- [ ] Webhook URLs validated: HTTPS only, no RFC-1918 IPs (DNS pre-resolution check)
- [ ] File magic bytes validated against allowed types (jpg, png, gif, pdf, csv)
- [ ] Semgrep rule `no-raw-httpclient` added to CI — fails if new HttpClient without allowlist
- [ ] ETag / `If-Match` optimistic concurrency implemented on Event and Registration entities

---

## 4.7 Implement multi-tenant SSO configuration and SCIM provisioning via Keycloak

**Labels:** `type:feature`, `layer:auth`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

Implement enterprise SSO federation and SCIM provisioning per ADR-003 multi-tenant section.

**Key files:**
- `src/EventFlow.Infrastructure/Adapters/KeycloakAdminService.cs` (ConfigureTenantSsoAsync, ProvisionScimEndpoint)
- `src/EventFlow.Application/Identity/Tenants/Commands/UpdateTenantSettingsCommand.cs` (SSO config update)
- `src/EventFlow.Api/Controllers/SettingsController.cs` (SSO/SCIM setup endpoints)

**Acceptance criteria:**
- [ ] `ConfigureTenantSsoAsync` creates Keycloak Identity Provider for SAML/OIDC per tenant
- [ ] SCIM 2.0 endpoint provisioned via Keycloak extension, mapped to org_id
- [ ] SSO config UI flow guided wizard (self-service, no platform admin required)
- [ ] Audit log entry created for all SSO configuration changes
- [ ] `GetPendingInvitationsQuery` excludes users already provisioned via SCIM

---

## 4.8 Implement AuditLog entity and audit logging for security-relevant events

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
Persist an immutable audit trail of security and data-change events for compliance (SOC 2, GDPR).

## Acceptance Criteria
- `AuditLog` entity: `id`, `tenantId`, `userId`, `action`, `entityType`, `entityId`, `oldValues`, `newValues`, `ipAddress`, `occurredAt`
- Logged events: login, logout, access denied, data export, settings changed, member invited/removed, event published/cancelled
- `GET /api/v1/admin/tenants/{orgId}` includes recent audit entries
- Filterable/exportable by date range and action type
- Not subject to tenant global query filter (admin can query across tenants)

## Technical Notes
- Written via `AuditBehavior` MediatR pipeline behavior or EF Core interceptor
- `IgnoreQueryFilters()` only in admin-scoped handlers (enforced by architecture test)
- Retention: 1 year per NFR requirements

---

