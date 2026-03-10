# Wave 6: Feature Implementation

## 6.1 Implement onboarding flow: workspace setup, team invite, first event creation

**Labels:** `type:feature`, `layer:frontend`, `priority:critical`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the 3-step onboarding flow targeting < 10 minutes to published event per 01-UX-RESEARCH.md section 7.

**Key files:**
- `frontend/src/pages/onboarding/OnboardingWorkspacePage.tsx` (org name, industry, events/year)
- `frontend/src/pages/onboarding/OnboardingInvitePage.tsx` (up to 5 email invites, skippable)
- `frontend/src/pages/onboarding/OnboardingFirstEventPage.tsx` (AI-assisted, manual, or template)
- `frontend/src/app/guards/OnboardingGuard.tsx` (redirect to onboarding if !tenant.onboardingComplete)

**Acceptance criteria:**
- [ ] Onboarding completes in < 10 minutes from account creation to published event (E2E validated)
- [ ] Step 2 (invite) is skippable without penalty
- [ ] AI-assisted option calls POST `/api/v1/ai/events/{id}/agenda` with streaming response
- [ ] Template gallery offers 5 event types: Annual Conference, Team Offsite, Product Launch, Webinar, Workshop
- [ ] `OnboardingGuard` redirects new users to `/onboarding/workspace` before any other route

---

## 6.2 Build Dashboard page with role-aware home view and real-time activity feed

**Labels:** `type:feature`, `layer:frontend`, `priority:critical`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the role-aware home dashboard per 01-UX-RESEARCH.md section 8 and 04-FRONTEND-ARCHITECTURE.md.

**Key files:**
- `frontend/src/pages/dashboard/DashboardPage.tsx`
- `frontend/src/features/analytics/hooks/useAnalytics.ts`
- `frontend/src/features/events/components/EventCard.tsx`
- `frontend/src/shared/components/PresenceIndicator.tsx`

**Role variants:** EventManager (stat cards + upcoming events + activity feed), Executive (revenue chart + cross-event table), CheckInStaff (full-screen counter + search)

**Acceptance criteria:**
- [ ] GET `/api/v1/analytics/dashboard` drives the home view based on JWT role claim
- [ ] 4 stat cards: Upcoming Events, Registered This Month, Open Tasks, Revenue MTD
- [ ] Live activity feed subscribes to WebSocket `registration.created` and `checkin.completed` events
- [ ] AI insights banner dismissable, links to recommended action URL
- [ ] All stat cards render skeleton screens while loading (no layout shift)

---

## 6.3 Build Events list page, Event detail shell with tab navigation, and setup checklist

**Labels:** `type:feature`, `layer:frontend`, `priority:critical`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the Events list and detail pages per 04-FRONTEND-ARCHITECTURE.md page tree.

**Key files:**
- `frontend/src/pages/events/EventsPage.tsx` (grid/list view, filters, cursor pagination)
- `frontend/src/pages/events/EventDetailPage.tsx` (shell with context nav tabs)
- `frontend/src/features/events/components/CreateEventDrawer.tsx` (right Sheet, 800px)
- `frontend/src/features/events/components/EventSetupChecklist.tsx` (progress bar, inline action buttons)
- `frontend/src/features/events/components/EventContextNav.tsx` (tab strip: Overview|Registration|Sessions...)
- `frontend/src/features/events/hooks/useEvents.ts`
- `frontend/src/features/events/hooks/useCreateEvent.ts`

**Acceptance criteria:**
- [ ] `CreateEventDrawer` slides in from right without losing page context
- [ ] Setup checklist shows % complete progress bar, collapses after 3 tasks done
- [ ] Event status badges use correct semantic colors (Draft=slate, Published=emerald, Live=red)
- [ ] Event list supports grid/list/calendar views with URL-persisted view preference
- [ ] `EventCapacityBar` shows fill percentage with warning color at 90%+

---

## 6.4 Build Session Scheduler with drag-and-drop timeline and conflict detection

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the session scheduling UI per 01-UX-RESEARCH.md section 9.3 and 04-FRONTEND-ARCHITECTURE.md.

**Key files:**
- `frontend/src/pages/events/EventSessionsTab.tsx`
- `frontend/src/features/sessions/components/SessionScheduler.tsx` (@dnd-kit/core timeline grid)
- `frontend/src/features/sessions/components/CreateSessionDrawer.tsx`
- `frontend/src/features/sessions/components/SessionConflictAlert.tsx`
- `frontend/src/features/sessions/hooks/useSessionScheduler.ts`
- `frontend/src/features/sessions/hooks/useSessions.ts`

**Acceptance criteria:**
- [ ] Drag sessions onto timeline grid, resize by dragging bottom edge (duration change)
- [ ] Overlapping sessions in same track highlighted in red (conflict detection)
- [ ] `ReorderSessionsCommand` called on drop with new sort order payload
- [ ] Keyboard equivalent: focus session → arrow keys → Enter to confirm placement
- [ ] All drag-drop operations have `aria-label` and work keyboard-only for WCAG compliance

---

## 6.5 Build Attendee management table with bulk actions and import/export

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the attendee management UI per 04-FRONTEND-ARCHITECTURE.md and API Specification.

**Key files:**
- `frontend/src/pages/attendees/AttendeesPage.tsx`
- `frontend/src/features/attendees/components/AttendeeTable.tsx` (TanStack Table)
- `frontend/src/features/attendees/components/AttendeeDrawer.tsx` (right drawer detail view)
- `frontend/src/features/attendees/components/BulkActionBar.tsx` (fixed bottom bar)
- `frontend/src/features/attendees/components/AttendeeImportModal.tsx`
- `frontend/src/features/attendees/hooks/useAttendees.ts`
- `frontend/src/features/attendees/hooks/useBulkActions.ts`

**Acceptance criteria:**
- [ ] Checkbox multi-select with floating action bar: Email, Export, Cancel Ticket, Add Tag, Remove
- [ ] Bulk action bar shows count of selected items and appears fixed at viewport bottom
- [ ] `AttendeeImportModal` accepts CSV, maps columns, previews first 5 rows before import
- [ ] Export triggers GET `/api/v1/events/{id}/registrations/export` and polls for download URL
- [ ] Inline edit on attendee notes field with 800ms debounce auto-save

---

## 6.6 Build PWA Check-In page with QR scanner, offline mode, and real-time counter

**Labels:** `type:feature`, `layer:frontend`, `priority:critical`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the event-day check-in PWA per 01-UX-RESEARCH.md section 10, Persona 3 (Sarah), and 04-FRONTEND-ARCHITECTURE.md section 13.

**Key files:**
- `frontend/src/pages/checkin/CheckInPage.tsx` (CheckInLayout, full-screen, high contrast)
- `frontend/src/features/checkin/components/QRScanner.tsx` (camera API, no extra app)
- `frontend/src/features/checkin/components/CheckInCounter.tsx` (live via WebSocket)
- `frontend/src/features/checkin/hooks/useCheckIn.ts`
- `frontend/src/features/checkin/hooks/useOfflineCheckIn.ts` (IndexedDB, sync on reconnect)
- `frontend/vite.config.ts` (vite-plugin-pwa, NetworkFirst attendee cache, standalone manifest)

**Acceptance criteria:**
- [ ] QR scan → check-in confirmation in under 2 seconds p95
- [ ] Offline mode stores check-ins in IndexedDB, syncs via POST `.../checkin/sync` on reconnect
- [ ] `OfflineBanner` appears immediately when `navigator.onLine = false`
- [ ] Counter updates via WebSocket `checkin.updated` event without page refresh
- [ ] PWA installs as standalone app (no browser chrome) with `theme_color: #6366F1`

---

## 6.7 Build Sponsor tier builder with drag-and-drop and sponsor management pages

**Labels:** `type:feature`, `layer:frontend`, `priority:medium`, `size:m`, `agent:frontend`
**Status:** Pending

Implement the sponsor management UI per 01-UX-RESEARCH.md section 9.3 (sponsor tier builder) and API Specification section 15.

**Key files:**
- `frontend/src/pages/sponsors/SponsorsPage.tsx`
- `frontend/src/pages/events/EventSponsorsTab.tsx`
- `frontend/src/features/sponsors/components/SponsorTierBuilder.tsx` (@dnd-kit/sortable)
- `frontend/src/features/sponsors/components/SponsorCard.tsx`
- `frontend/src/features/sponsors/components/SponsorInviteModal.tsx`
- `frontend/src/features/sponsors/hooks/useSponsors.ts`

**Acceptance criteria:**
- [ ] Drag sponsor logos between tier rows (Platinum/Gold/Silver/Bronze)
- [ ] Reorder within a tier updates `sort_order` via `ReorderSponsorTiersCommand`
- [ ] Sponsor portal token displayed with copy button for sharing with sponsor
- [ ] Sponsor lead capture count shown on each sponsor card
- [ ] `SponsorInviteModal` creates sponsor and optionally sends portal access email

---

## 6.8 Build Campaign builder, email template editor, and campaign analytics

**Labels:** `type:feature`, `layer:frontend`, `priority:medium`, `size:m`, `agent:frontend`
**Status:** Pending

Implement the Communications feature pages per 04-FRONTEND-ARCHITECTURE.md and API Specification section 16.

**Key files:**
- `frontend/src/pages/campaigns/CampaignsPage.tsx`
- `frontend/src/pages/campaigns/CampaignBuilderPage.tsx`
- `frontend/src/features/communications/components/CampaignBuilder.tsx`
- `frontend/src/features/communications/components/EmailTemplateEditor.tsx`
- `frontend/src/features/communications/components/CampaignMetricsCard.tsx`
- `frontend/src/features/communications/hooks/useCampaigns.ts`

**Acceptance criteria:**
- [ ] Recipients preview shows count + 3 sample emails (GET `.../recipients-preview`)
- [ ] Campaign status badge: Draft (slate), Scheduled (amber), Sending (blue), Sent (emerald)
- [ ] Send test email button triggers POST `.../test` with current user's email
- [ ] Campaign analytics shows open rate, click rate, bounces as cards with sparklines
- [ ] Template editor supports Handlebars merge tags with `{{attendeeFirstName}}` syntax

---

## 6.9 Build Settings pages: team, billing, white-label, SSO/SCIM, integrations

**Labels:** `type:feature`, `layer:frontend`, `priority:high`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the organization settings pages per 04-FRONTEND-ARCHITECTURE.md page tree.

**Key files:**
- `frontend/src/pages/settings/SettingsPage.tsx` (shell with tab navigation)
- `frontend/src/pages/settings/SettingsTeamTab.tsx` (member list, invite, role change)
- `frontend/src/pages/settings/SettingsBillingTab.tsx` (plan, Stripe portal link)
- `frontend/src/pages/settings/SettingsWhiteLabelTab.tsx` (logo, colors, custom domain, font)
- `frontend/src/pages/settings/SettingsSecurityTab.tsx` (SSO wizard, SCIM endpoint, audit logs)
- `frontend/src/pages/settings/SettingsIntegrationsTab.tsx` (webhooks, CRM, Slack)

**Acceptance criteria:**
- [ ] White-label preview renders registration page with current brand colors in real-time
- [ ] SSO setup wizard: guided SAML/OIDC config with metadata URL or XML upload
- [ ] Billing tab renders Stripe Customer Portal link (POST `.../billing/portal`)
- [ ] Webhook endpoint form validates HTTPS URL and tests delivery
- [ ] Team tab shows pending invitations with revoke option

---

## 6.10 Build public registration page with white-label theming and Stripe checkout

**Labels:** `type:feature`, `layer:frontend`, `priority:critical`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the attendee-facing public registration page per 01-UX-RESEARCH.md and API Specification public endpoints.

**Key files:**
- `frontend/src/pages/public/RegistrationPage.tsx`
- `frontend/src/pages/public/SchedulePage.tsx`
- `frontend/src/app/layouts/PublicLayout.tsx`
- `frontend/src/app/providers/TenantThemeProvider.tsx` (CSS vars from GET `/api/v1/tenants/{id}/theme`)
- `frontend/src/pages/public/SpeakerPortalPage.tsx`

**Acceptance criteria:**
- [ ] `TenantThemeProvider` applies `--brand-primary`, `--brand-secondary`, `--brand-logo` CSS vars
- [ ] WCAG contrast ratio ≥ 4.5:1 validated at theme load; auto-adjust text color if needed
- [ ] POST `/public/events/{orgSlug}/{eventSlug}/register` initiates Stripe Checkout redirect
- [ ] Confirmation page shows QR code, calendar invite download, and event details
- [ ] Registration form renders custom fields from `event.custom_registration_schema` JSONB

---

## 6.11 Implement Stripe payment integration: checkout, webhooks, refunds, billing portal

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:l`, `agent:backend`
**Status:** Pending

Implement end-to-end Stripe payment processing per API Specification sections for Stripe and Plans.

**Key files:**
- `src/EventFlow.Application/Registrations/Commands/CreateRegistrationCommand.cs` (Stripe Checkout Session creation)
- `src/EventFlow.Infrastructure/Adapters/StripePaymentAdapter.cs` (implements `IPaymentGateway`, thin SDK wrapper)
- `src/EventFlow.Application/Common/Interfaces/IPaymentGateway.cs`
- `src/EventFlow.Api/Controllers/StripeController.cs` (POST /api/v1/stripe/webhooks, /checkout/success)
- `src/EventFlow.Application/Plans/Commands/CreatePlanCheckoutCommand.cs`
- `src/EventFlow.Application/Registrations/Commands/RefundRegistrationCommand.cs`

**Acceptance criteria:**
- [ ] Stripe webhook signature validated using `Stripe-Signature` header before processing
- [ ] Webhook events enqueued to Redis Streams (not processed inline)
- [ ] `RefundRegistrationCommand` issues full or partial refund, updates `PaymentStatus`
- [ ] Stripe Customer Portal session created for tenant billing self-service
- [ ] Idempotency keys used for all Stripe API calls to prevent duplicate charges

---

## 6.12 Send confirmation email with QR code and calendar invite attachment

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
When an attendee registers, send a confirmation email containing the QR code image and an `.ics` calendar invite attachment.

## Acceptance Criteria
- `RegistrationConfirmedEvent` consumer triggers email with QR code PNG attached
- `.ics` file generated for event start/end time and attached to email
- Email uses the tenant's `RegistrationConfirmation` email template
- Works for both free and paid registrations
- SendGrid/SMTP adapter supports file attachments

## Technical Notes
- Generate `.ics` via `Ical.Net` NuGet package
- QR code stored in S3; fetch and attach before send
- Calendar invite `SUMMARY`, `DTSTART`, `DTEND`, `LOCATION` populated from event data

---

## 6.13 Implement EmailDelivery entity and email delivery tracking

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
Track individual email delivery status (sent, delivered, opened, bounced) for transactional and campaign emails.

## Acceptance Criteria
- `EmailDelivery` entity: `id`, `tenantId`, `campaignId?`, `registrationId?`, `toEmail`, `subject`, `status`, `sentAt`, `deliveredAt`, `openedAt`, `bouncedAt`, `errorMessage`
- SendGrid/SMTP webhook handler at `POST /api/v1/stripe/webhooks` pattern (dedicated email webhook endpoint)
- Delivery status updates cascade to `CampaignMetrics` denormalized fields
- EF Core migration added

## Technical Notes
- Webhook signature verified (SendGrid event webhook secret)
- Idempotent processing: check `EmailDelivery.id` before insert
- Expose aggregate delivery stats via `GetCampaignAnalyticsQuery`

---

## 6.14 Implement check-in attendees list and campaign preview/stats endpoints

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:s`, `agent:backend`
**Status:** Pending

## Goal
Fill missing endpoints: pre-event check-in attendee roster and campaign preview/stats APIs.

## Acceptance Criteria
- `GET /api/v1/events/{eventId}/check-in/attendees` returns full confirmed attendee list optimized for check-in (id, name, qrCode, ticketType, isCheckedIn)
- `POST /api/v1/events/{eventId}/campaigns/{campaignId}/preview` renders email template with sample data and returns HTML
- `GET /api/v1/events/{eventId}/campaigns/{campaignId}/stats` returns campaign delivery metrics scoped to event
- Check-in attendees endpoint cached in Redis at event `Live` status (cache warming)

## Technical Notes
- Check-in list pre-warmed by `EventLiveConsumer` background handler
- Campaign preview uses `BodyHtml` with Handlebars/Liquid render for sample attendee data
- Stats endpoint aggregates `EmailDelivery` records for the campaign

---

## 6.15 Implement speaker portal GET and PUT endpoints via invitation token

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:s`, `agent:backend`
**Status:** Pending

## Goal
Public speaker portal endpoints allowing speakers to view their sessions and update their profile via magic link token.

## Acceptance Criteria
- `GET /api/v1/speaker-portal/{invitationToken}` returns speaker profile, assigned sessions, and event details
- `PUT /api/v1/speaker-portal/{invitationToken}` allows speaker to update bio, photo URL, social links
- Token validated: not expired, not revoked, matches speaker record
- Session assignment details shown: session title, time, room, co-speakers

## Technical Notes
- Maps to existing `GetSpeakerPortalDataQuery` and new `UpdateSpeakerViaPortalCommand`
- Token single-use for updates (re-generates on each save) or time-limited (48h)
- No JWT required — token IS the authentication for this public endpoint

---

