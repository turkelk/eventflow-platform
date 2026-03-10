# Wave 14: Integrations & Automation

## 14.1 Implement webhook delivery system and outbound integration webhooks

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement the webhook infrastructure for outbound integrations per Data Model `WebhookEndpoint` entity and API Specification Organizations section.

**Key files:**
- `src/EventFlow.Application/Common/Commands/CreateWebhookEndpointCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Common/Commands/UpdateWebhookEndpointCommand.cs`
- `src/EventFlow.Application/Common/Commands/TestWebhookEndpointCommand.cs`
- `src/EventFlow.Infrastructure/BackgroundServices/WebhookDeliveryService.cs` (Redis Streams consumer)
- `src/EventFlow.Infrastructure/Http/WebhookHttpClient.cs` (HMAC-SHA256 signing)

**Webhook events:** registration.created, registration.cancelled, event.published, checkin.completed, campaign.sent

**Acceptance criteria:**
- [ ] Webhook payloads signed with `X-EventFlow-Signature: sha256={hmac}` using stored secret
- [ ] Webhook delivery retried 3x with exponential backoff on failure
- [ ] `failure_count` incremented and `is_active` set to false after 10 consecutive failures
- [ ] POST `/api/v1/organizations/me/webhooks/{id}/test` sends sample payload immediately
- [ ] Webhook URL validated: HTTPS only, no RFC-1918 IPs (SSRF prevention)

---

## 14.2 Implement Stripe subscription billing: plans, checkout, customer portal, plan enforcement

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

Implement full subscription billing lifecycle per User Stories (Organization Admin: billing management) and API Specification Plans section.

**Key files:**
- `src/EventFlow.Application/Plans/Queries/GetPlansQuery.cs` (public, no auth)
- `src/EventFlow.Application/Plans/Commands/CreatePlanCheckoutCommand.cs`
- `src/EventFlow.Application/Organizations/Commands/UpdateOrganizationPlanCommand.cs` (post-Stripe webhook)
- `src/EventFlow.Infrastructure/Adapters/StripePaymentAdapter.cs` (CreateCustomerPortalSession, CreateCheckoutSession)
- `frontend/src/pages/settings/SettingsBillingTab.tsx`

**Plan limits enforced in handlers:** MaxEventsPerMonth, MaxAttendeesPerEvent, features JSONB

**Acceptance criteria:**
- [ ] GET `/api/v1/plans` returns all plans with pricing and feature list (public endpoint)
- [ ] `CreatePlanCheckoutCommand` creates Stripe Checkout Session for subscription
- [ ] `checkout.session.completed` Stripe webhook updates `Organization.plan_id` and `stripe_subscription_id`
- [ ] `PlanLimitExceededException` thrown in `CreateEventCommand` when monthly event limit reached
- [ ] Billing portal link generated via `IPaymentGateway.CreateCustomerPortalSessionAsync`

---

## 14.3 Implement discount code validation, group pricing, and ticket sale window enforcement

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement discount code management and advanced ticket pricing per Data Model `DiscountCode` entity and User Stories.

**Key files:**
- `src/EventFlow.Application/Events/Commands/CreateDiscountCodeCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Events/Commands/UpdateDiscountCodeCommand.cs`
- `src/EventFlow.Application/Events/Queries/GetDiscountCodesQuery.cs`
- `src/EventFlow.Api/Controllers/EventsController.cs` (discount code endpoints)
- Public endpoint: POST `/public/events/{orgSlug}/{eventSlug}/validate-discount`

**Acceptance criteria:**
- [ ] `ValidateDiscountCommand` returns discount amount (percentage or fixed) without creating registration
- [ ] Discount code `uses_count` incremented atomically with registration creation
- [ ] `max_uses` limit enforced with distributed lock to prevent race conditions
- [ ] `valid_from` and `valid_until` windows enforced in validator
- [ ] `applicable_tier_ids` JSONB restricts discount to specific ticket tiers

---

## 14.4 Implement super admin portal: tenant management, impersonation, platform metrics

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement the platform admin features per User Stories (Super Admin persona) and API Specification Admin section.

**Key files:**
- `src/EventFlow.Application/Admin/Queries/GetAllTenantsQuery.cs` (IgnoreQueryFilters, platform_admin only)
- `src/EventFlow.Application/Admin/Queries/GetPlatformMetricsQuery.cs`
- `src/EventFlow.Application/Admin/Commands/ImpersonateTenantCommand.cs` (audit logged)
- `src/EventFlow.Application/Admin/Commands/UpdateTenantPlanCommand.cs`
- `src/EventFlow.Api/Controllers/AdminController.cs`
- `frontend/src/pages/admin/AdminDashboardPage.tsx`
- `frontend/src/pages/admin/AdminTenantsPage.tsx`

**Acceptance criteria:**
- [ ] All admin endpoints require `platform_admin` Keycloak realm role
- [ ] `ImpersonateTenantCommand` generates short-lived impersonation token (15min), audit logged
- [ ] `GetPlatformMetricsQuery` returns MRR, total events, total registrations, GTV
- [ ] `GetAllTenantsQuery` uses `IgnoreQueryFilters()` with `[AdminOperation]` attribute
- [ ] Architecture test asserts only `[AdminOperation]` handlers use `IgnoreQueryFilters()`

---

## 14.5 Implement white-label tenant theming: registration pages, emails, custom domain

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:l`, `agent:frontend`
**Status:** Pending

Implement full white-label theming capability per 01-UX-RESEARCH.md section 11 (White-Label Theming) and User Story (Organization Admin: email branding).

**Key files:**
- `src/EventFlow.Application/Identity/Tenants/Commands/UpdateTenantThemeCommand.cs` (BrandColor validation)
- `src/EventFlow.Domain/ValueObjects/BrandColor.cs` (validates hex, checks WCAG contrast ratio)
- `frontend/src/app/providers/TenantThemeProvider.tsx` (CSS vars from API)
- `frontend/src/pages/settings/SettingsWhiteLabelTab.tsx` (real-time preview)

**Scope:** Registration pages, email templates, public schedule page, check-in kiosk (NOT organizer admin)

**Acceptance criteria:**
- [ ] `BrandColor` value object validates hex format and WCAG AA contrast ratio (4.5:1) at save time
- [ ] CSS custom properties (`--brand-primary`, `--brand-logo`, `--brand-font`) applied to all public routes
- [ ] Custom domain support: `custom_domain` field validated via DNS CNAME check
- [ ] Email templates use `email_sender_name` and `email_sender_address` from organization
- [ ] White-label settings locked behind `white-label` Unleash feature flag (plan-gated)

---

