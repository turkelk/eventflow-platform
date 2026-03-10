# Wave 7: Non-Functionals

## 7.1 Implement MediatR pipeline behaviors: logging, validation, caching, feature flags, transaction

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:l`, `agent:backend`
**Status:** Pending

Implement all MediatR pipeline behaviors registered in the correct order per ADR-002 and NFR section 1.3.

**Key files:**
- `src/EventFlow.Application/Behaviors/LoggingBehavior.cs`
- `src/EventFlow.Application/Behaviors/FeatureFlagBehavior.cs`
- `src/EventFlow.Application/Behaviors/ValidationBehavior.cs`
- `src/EventFlow.Application/Behaviors/CacheBehavior.cs`
- `src/EventFlow.Application/Behaviors/PerformanceBehavior.cs`
- `src/EventFlow.Application/Behaviors/TransactionBehavior.cs`
- `src/EventFlow.Application/Behaviors/CostTrackingBehavior.cs`

**DI registration order (outermost first):** Logging → FeatureFlag → Validation → Cache → CostTracking → Performance → Transaction

**Acceptance criteria:**
- [ ] `ValidationBehavior` collects all FluentValidation failures, throws `ValidationException` with property map
- [ ] `CacheBehavior` only applies to queries implementing `ICacheableQuery`; commands skip cache
- [ ] `TransactionBehavior` wraps commands in DB transaction; queries run without transaction
- [ ] `PerformanceBehavior` logs Warning for handlers exceeding 500ms
- [ ] `FeatureFlagBehavior` reads `[FeatureFlag("name")]` attribute, throws `FeatureDisabledException` if flag off

---

## 7.2 Implement health checks, Prometheus metrics, and Polly circuit breakers

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement observability and resilience infrastructure per NFR sections 3.2 and 4.3.

**Key files:**
- `src/EventFlow.Api/Program.cs` (health check registration)
- `src/EventFlow.Infrastructure/HealthChecks/RedisStreamHealthCheck.cs`
- `src/EventFlow.Infrastructure/HealthChecks/S3HealthCheck.cs`
- `src/EventFlow.Api/Middleware/PrometheusMetricsMiddleware.cs`
- `src/EventFlow.Infrastructure/Http/ResilientHttpClientExtensions.cs` (Polly pipeline)

**Endpoints:** `/health` (all deps), `/ready` (DB + Redis), `/alive` (always 200)
**Metrics:** http_requests_total, mediatr_handler_duration_seconds, redis_cache_hits_total, eventflow_checkins_total

**Acceptance criteria:**
- [ ] `/health` returns 200 with all dependency status including Keycloak, S3, Unleash
- [ ] `/metrics` endpoint in Prometheus text format, restricted to internal network
- [ ] Circuit breaker opens after 50% failure rate over 30s window, breaks for 60s
- [ ] Polly retry: 3 attempts with exponential backoff on all outbound HTTP (AI, email, SMS)
- [ ] Graceful shutdown: 30s drain period, Background services stop cleanly before process exit

---

## 7.3 Implement email service adapter with SendGrid/SMTP and transactional email templates

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement the email service infrastructure for transactional emails per Domain Model communications context.

**Key files:**
- `src/EventFlow.Application/Common/Interfaces/IEmailService.cs`
- `src/EventFlow.Infrastructure/Adapters/EmailService.cs` (SendGrid SDK, thin wrapper — one SDK call per method)
- `src/EventFlow.Infrastructure/BackgroundServices/EventStreamConsumer.cs` (email-processor consumer group)
- Email templates: registration confirmation, speaker invite, campaign delivery, waitlist notification

**Acceptance criteria:**
- [ ] `IEmailService.SendAsync(to, subject, htmlBody, textBody)` — one SendGrid API call, no orchestration
- [ ] Email consumer group processes `attendee.registered`, `speaker.invited`, `waitlist.promoted` events
- [ ] Unsubscribe link included in all marketing emails via `List-Unsubscribe` header
- [ ] Bounce and delivery webhooks update `EmailDelivery.status` via `campaign.delivery_updated` event
- [ ] Custom sender domain (`email_sender_address`) used when configured on organization

---

## 7.4 Implement SignalR hubs for real-time check-in counter and activity feed

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement real-time WebSocket features via SignalR backed by Redis Streams per ADR-002 and ADR-004.

**Key files:**
- `src/EventFlow.Api/Hubs/CheckInHub.cs` (SignalR hub for live check-in counter)
- `src/EventFlow.Api/Hubs/EventActivityHub.cs` (dashboard activity feed)
- `src/EventFlow.Infrastructure/BackgroundServices/RealtimeEventConsumer.cs`
- `frontend/src/app/providers/WebSocketProvider.tsx`
- `frontend/src/shared/hooks/useWebSocket.ts`

**Acceptance criteria:**
- [ ] `CheckInHub` groups clients by `eventId`, broadcasts `checkin.updated` within 50ms of DB write
- [ ] `EventActivityHub` streams `registration.created`, `campaign.sent` to tenant-scoped groups
- [ ] Frontend `WebSocketProvider` reconnects with exponential backoff (1s→2s→4s→8s→max 30s)
- [ ] Heartbeat message sent every 30s to keep connections alive through proxies
- [ ] Redis Pub/Sub used for cross-pod SignalR message delivery

---

## 7.5 Implement Unleash feature flag provider and FeatureGate frontend component

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement feature flag integration for both frontend and backend per ADR-002 progressive disclosure.

**Key files:**
- `src/EventFlow.Infrastructure/Adapters/FeatureFlagService.cs` (Unleash SDK, in-memory evaluation)
- `frontend/src/app/providers/FeatureFlagProvider.tsx` (@unleash/proxy-client-react)
- `frontend/src/shared/components/FeatureGate.tsx` (renders children only if flag enabled)
- `frontend/src/shared/hooks/useFeatureFlags.ts`

**Progressive disclosure rules (from UX Research section 7):**
- After 1st event: Sponsor Management unlocks
- After 1st publish: White-Label unlocks
- After 10 attendees: AI Matchmaking tooltip
- After 1st completed event: Advanced Analytics unlocks

**Acceptance criteria:**
- [ ] `<FeatureGate flag="ai-agenda-generation">` renders null if flag disabled
- [ ] `FeatureFlagBehavior` returns `FeatureDisabledException` → 503 if API flag is off
- [ ] Flag state evaluated per-tenant using Unleash context `{ userId, properties: { tenantId } }`
- [ ] Flag evaluation is in-memory (< 2ms overhead)
- [ ] Stale feature flags cleaned up within 2 sprints of GA via CI lint rule

---

## 7.6 Implement audit logging and data governance (GDPR right to access/erasure)

**Labels:** `type:security`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement audit logging and GDPR compliance features per NFR section 7.

**Key files:**
- `src/EventFlow.Infrastructure/Persistence/EventFlowDbContext.cs` (audit log auto-write in SaveChangesAsync)
- `src/EventFlow.Application/Identity/Members/Queries/GetAuditLogsQuery.cs`
- `src/EventFlow.Application/Common/Commands/ExportPersonalDataCommand.cs`
- `src/EventFlow.Application/Common/Commands/DeletePersonalDataCommand.cs` (GDPR erasure)

**Audit events:** UserLoggedIn, AccessDenied, DataExported, SettingsChanged, InviteSent, PaymentEvent

**Acceptance criteria:**
- [ ] `AuditLog` records written atomically with entity changes in same transaction
- [ ] Audit logs filterable by user, action, entity type, date range via query
- [ ] `DeletePersonalDataCommand` hard-deletes PII fields, anonymizes analytics records
- [ ] `ExportPersonalDataCommand` generates JSON of all PII for given attendee within 24h
- [ ] Audit logs retained for 1 year per compliance requirement, never tenant-filtered (admin visibility)

---

## 7.7 Implement idempotency key middleware and ETag optimistic concurrency

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement idempotency and optimistic concurrency controls per NFR section 3.6.

**Key files:**
- `src/EventFlow.Api/Middleware/IdempotencyMiddleware.cs` (Idempotency-Key header, Redis store, 24h TTL)
- `src/EventFlow.Api/Middleware/ETagMiddleware.cs` (ETag generation + If-Match validation)
- Applied to: CreateRegistrationCommand, SendCampaignImmediatelyCommand, CheckInAttendeeCommand

**Acceptance criteria:**
- [ ] Second request with same `Idempotency-Key` header returns cached response without re-executing handler
- [ ] PUT/PATCH requests require `If-Match` header; 409 returned if ETag stale
- [ ] Idempotency keys stored in Redis with 24h TTL under `idempotency:{key}` namespace
- [ ] `Registration`, `Event`, `Session` entities have EF Core concurrency token (`xmin` or `RowVersion`)
- [ ] Idempotency applied to Stripe webhook processing to prevent duplicate refunds

---

## 7.8 Implement async CSV/PDF export for attendee lists and analytics reports

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

## Goal
Allow organizers to export attendee lists and analytics reports as CSV or PDF, delivered via S3 pre-signed URL.

## Acceptance Criteria
- `GET /api/v1/events/{eventId}/export/attendees?format=csv|pdf` triggers async export job
- Returns `202 Accepted` with `jobId` and `pollUrl`
- On completion, response includes S3 pre-signed download URL (expires 1 hour)
- CSV includes configurable columns; PDF uses a styled template
- Analytics cross-event report also exportable

## Technical Notes
- Background job via `IHostedService` or MediatR command dispatched to worker
- PDF generation via `QuestPDF` NuGet package
- Export record stored in DB for audit log; file uploaded to MinIO/S3

---

## 7.9 Implement organization billing portal and plan management API endpoints

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:s`, `agent:backend`
**Status:** Pending

## Goal
Expose billing information and Stripe customer portal access via API for the settings billing page.

## Acceptance Criteria
- `GET /api/v1/organizations/me/billing` returns current plan, usage metrics, next billing date, and Stripe customer info
- `POST /api/v1/organizations/me/billing/portal` creates a Stripe billing portal session and returns redirect URL
- Current plan limits (events/year, attendees/event) shown with usage percentages
- Plan upgrade/downgrade handled via Stripe portal

## Technical Notes
- Stripe `BillingPortal.Session.Create()` called in handler
- `BillingInfo` value object on `Tenant` stores `StripeCustomerId` and `StripeSubscriptionId`
- Returns `200 OK` with `{ portalUrl, expiresAt }` — frontend redirects immediately

---

## 7.10 Implement GET /api/v1/events/{eventId}/export/attendees endpoint

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:s`, `agent:backend`
**Status:** Pending

## Goal
Dedicated endpoint for triggering async attendee list export and polling for download URL.

## Acceptance Criteria
- `GET /api/v1/events/{eventId}/export/attendees?format=csv&columns=...` dispatches export job
- Returns `202 Accepted` with `{ jobId, pollUrl, estimatedSeconds }`
- `GET {pollUrl}` returns `{ status: Processing|Complete|Failed, downloadUrl?, expiresAt? }`
- Requires `RequireManager` authorization
- Rate limited: 10/min per tenant

## Technical Notes
- Dispatches `ExportRegistrationsCommand` to background worker via Redis outbox
- Download URL is S3 pre-signed URL with 1-hour expiry
- Export job result cached in Redis by `jobId` (TTL: 2 hours)

---

## 7.11 Implement plans checkout and file upload presigned URL endpoints

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
Missing utility endpoints: subscription plan listing/checkout and S3 presigned URL generation for client-side file uploads.

## Acceptance Criteria
- `GET /api/v1/plans` returns available subscription plans with features and pricing
- `POST /api/v1/plans/checkout` creates Stripe Checkout session for plan upgrade; returns `{ checkoutUrl }`
- `GET /api/v1/files/upload-url` returns S3 presigned PUT URL for direct browser upload; params: `fileName`, `contentType`, `entityType`, `entityId`
- `POST /api/v1/files/confirm-upload` confirms upload complete and persists URL to entity

## Technical Notes
- Plans are static configuration (not DB-driven) mapped to Stripe price IDs
- Presigned URL TTL: 10 minutes; max file size enforced in policy
- `entityType` values: `event-cover`, `speaker-photo`, `sponsor-logo`, `venue-photo`, `tenant-logo`
- File type validated via allowed MIME types allowlist

---

