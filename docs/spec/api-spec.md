# API Specification

> 106 endpoints. All endpoints under `/api/v1/` require JWT authentication unless under `/api/v1/public/`.

## Events

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/events` | `GetEventsQuery` | List events for the organization. Supports cursor pagination, filtering by status/date/type. Requires organizer role. |
| `POST` | `/api/v1/events` | `CreateEventCommand` | Create a new event. Dispatches CreateEventCommand. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}` | `GetEventByIdQuery` | Get full event details including ticket tiers and session count. Requires organizer role. |
| `PUT` | `/api/v1/events/{eventId}` | `UpdateEventCommand` | Update event details. Dispatches UpdateEventCommand. Requires organizer role. |
| `DELETE` | `/api/v1/events/{eventId}` | `DeleteEventCommand` | Soft-delete a draft event. Cannot delete events with confirmed registrations. Requires admin role. |
| `POST` | `/api/v1/events/{eventId}/publish` | `CreateEventCommand` | Publish event (make registration page live). Validates all required fields. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/cancel` | `CreateEventCommand` | Cancel event and trigger refund processing for all confirmed registrations. Requires admin role. |
| `POST` | `/api/v1/events/{eventId}/clone` | `CreateEventCommand` | Clone event as a new draft, copying sessions, ticket tiers, and settings. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/analytics` | `GetEventByIdQuery` | Event-level analytics: registrations over time, revenue, check-in rate, session attendance. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/export/attendees` | `GetEventByIdQuery` | Export attendee list as CSV or PDF. Async job with S3 pre-signed URL response. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/ticket-tiers` | `GetEventByIdQuery` | List ticket tiers for an event. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/ticket-tiers` | `CreateEventCommand` | Create a new ticket tier. Requires organizer role. |
| `PUT` | `/api/v1/events/{eventId}/ticket-tiers/{tierId}` | `UpdateEventCommand` | Update ticket tier details (price, quantity, sale window). Requires organizer role. |
| `DELETE` | `/api/v1/events/{eventId}/ticket-tiers/{tierId}` | `DeleteEventCommand` | Delete a ticket tier (only if no tickets sold). Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/discount-codes` | `GetEventByIdQuery` | List discount codes for an event. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/discount-codes` | `CreateEventCommand` | Create a discount code. Requires organizer role. |
| `PUT` | `/api/v1/events/{eventId}/discount-codes/{codeId}` | `UpdateEventCommand` | Update or deactivate a discount code. Requires organizer role. |
| `DELETE` | `/api/v1/events/{eventId}/discount-codes/{codeId}` | `DeleteEventCommand` | Delete a discount code. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/sessions` | `GetEventByIdQuery` | List all sessions for an event with speaker assignments. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/sessions` | `CreateEventCommand` | Create a new session. Requires organizer role. |
| `PUT` | `/api/v1/events/{eventId}/sessions/{sessionId}` | `UpdateEventCommand` | Update session details (title, time, room, track). Requires organizer role. |
| `DELETE` | `/api/v1/events/{eventId}/sessions/{sessionId}` | `DeleteEventCommand` | Delete a session. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/sessions/{sessionId}/speakers` | `CreateEventCommand` | Assign a speaker to a session. Requires organizer role. |
| `DELETE` | `/api/v1/events/{eventId}/sessions/{sessionId}/speakers/{speakerId}` | `DeleteEventCommand` | Remove a speaker from a session. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/speakers` | `GetEventByIdQuery` | List speakers for an event. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/speakers` | `CreateEventCommand` | Invite a speaker by email. Sends invitation email with portal link. Requires organizer role. |
| `PUT` | `/api/v1/events/{eventId}/speakers/{speakerId}` | `UpdateEventCommand` | Update speaker details. Requires organizer role. |
| `DELETE` | `/api/v1/events/{eventId}/speakers/{speakerId}` | `DeleteEventCommand` | Remove a speaker from the event. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/sponsors` | `GetEventByIdQuery` | List sponsors for an event. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/sponsors` | `CreateEventCommand` | Add a sponsor to an event with package details. Requires organizer role. |
| `PUT` | `/api/v1/events/{eventId}/sponsors/{sponsorId}` | `UpdateEventCommand` | Update sponsor details or tier. Requires organizer role. |
| `DELETE` | `/api/v1/events/{eventId}/sponsors/{sponsorId}` | `DeleteEventCommand` | Remove a sponsor from the event. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/registrations` | `GetEventByIdQuery` | List registrations with attendee details. Cursor pagination, filter by status/ticket tier. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/registrations/{registrationId}` | `GetEventByIdQuery` | Get full registration details including tickets and custom field responses. Requires organizer role. |
| `PUT` | `/api/v1/events/{eventId}/registrations/{registrationId}/status` | `UpdateEventCommand` | Manually update registration status (e.g., confirm a waitlisted attendee). Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/registrations/{registrationId}/refund` | `CreateEventCommand` | Issue full or partial refund via Stripe. Requires admin role. |
| `GET` | `/api/v1/events/{eventId}/waitlist` | `GetEventByIdQuery` | List waitlisted registrations in order. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/waitlist/promote` | `CreateEventCommand` | Manually promote next N waitlisted attendees to confirmed. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/check-in/stats` | `GetEventByIdQuery` | Real-time check-in statistics (total checked in, by tier, by hour). Requires check_in_staff role. |
| `POST` | `/api/v1/events/{eventId}/check-in/scan` | `CreateEventCommand` | Process QR code scan. Returns attendee info and check-in result. Dispatches CheckInAttendeeCommand. Requires check_in_staff role. |
| `POST` | `/api/v1/events/{eventId}/check-in/manual` | `CreateEventCommand` | Manual check-in by registration number or attendee name search. Requires check_in_staff role. |
| `POST` | `/api/v1/events/{eventId}/check-in/{registrationId}/undo` | `CreateEventCommand` | Undo a check-in (mark as not checked in). Requires check_in_staff role. |
| `GET` | `/api/v1/events/{eventId}/check-in/attendees` | `GetEventByIdQuery` | Get attendee list for offline PWA cache. Returns compact attendee+QR token list. Requires check_in_staff role. |
| `GET` | `/api/v1/events/{eventId}/campaigns` | `GetEventByIdQuery` | List email campaigns for an event. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/campaigns` | `CreateEventCommand` | Create a new email campaign (draft). Requires organizer role. |
| `PUT` | `/api/v1/events/{eventId}/campaigns/{campaignId}` | `UpdateEventCommand` | Update campaign content and recipient filter. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/campaigns/{campaignId}/schedule` | `CreateEventCommand` | Schedule campaign for future send. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/campaigns/{campaignId}/send-now` | `CreateEventCommand` | Send campaign immediately. Enqueues to Redis Streams for worker processing. Requires organizer role. |
| `POST` | `/api/v1/events/{eventId}/campaigns/{campaignId}/preview` | `CreateEventCommand` | Send a preview email to the requesting user. Requires organizer role. |
| `GET` | `/api/v1/events/{eventId}/campaigns/{campaignId}/stats` | `GetEventByIdQuery` | Get campaign delivery stats (sent, opened, clicked, bounced). Requires organizer role. |

## Organizations

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/organizations/me` | `GetOrganizationsQuery` | Get current user's organization profile and plan details. Requires auth. |
| `PUT` | `/api/v1/organizations/me` | `UpdateOrganizationCommand` | Update organization profile (name, branding, timezone). Requires admin role. |
| `GET` | `/api/v1/organizations/me/members` | `GetOrganizationsQuery` | List organization members with roles. Cursor-based pagination. Requires admin role. |
| `POST` | `/api/v1/organizations/me/members/invite` | `CreateOrganizationCommand` | Invite a new member by email with a specified role. Requires admin role. |
| `PUT` | `/api/v1/organizations/me/members/{memberId}/role` | `UpdateOrganizationCommand` | Update a member's role. Requires admin role. |
| `DELETE` | `/api/v1/organizations/me/members/{memberId}` | `DeleteOrganizationCommand` | Remove a member from the organization. Requires admin role. |
| `GET` | `/api/v1/organizations/me/billing` | `GetOrganizationsQuery` | Get Stripe subscription status, current plan, and invoice history. Requires billing_admin role. |
| `POST` | `/api/v1/organizations/me/billing/portal` | `CreateOrganizationCommand` | Create Stripe Customer Portal session URL for self-service billing management. Requires billing_admin role. |
| `GET` | `/api/v1/organizations/me/analytics` | `GetOrganizationsQuery` | Aggregate analytics across all org events (total registrations, revenue, events). Requires admin role. |
| `GET` | `/api/v1/organizations/me/webhooks` | `GetOrganizationsQuery` | List configured webhook endpoints. Requires admin role. |
| `POST` | `/api/v1/organizations/me/webhooks` | `CreateOrganizationCommand` | Create a new webhook endpoint with event subscriptions. Requires admin role. |
| `PUT` | `/api/v1/organizations/me/webhooks/{webhookId}` | `UpdateOrganizationCommand` | Update webhook endpoint URL or subscribed events. Requires admin role. |
| `DELETE` | `/api/v1/organizations/me/webhooks/{webhookId}` | `DeleteOrganizationCommand` | Delete a webhook endpoint. Requires admin role. |
| `POST` | `/api/v1/organizations/me/webhooks/{webhookId}/test` | `CreateOrganizationCommand` | Send a test payload to the webhook endpoint. Requires admin role. |

## Attendee

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/attendee/events` | `GetAttendeesQuery` | Authenticated attendee: List events the attendee is registered for. Requires attendee auth. |
| `GET` | `/api/v1/attendee/events/{eventId}/registration` | `GetAttendeeByIdQuery` | Authenticated attendee: Get own registration details and QR code. Requires attendee auth. |
| `GET` | `/api/v1/attendee/events/{eventId}/sessions` | `GetAttendeeByIdQuery` | Authenticated attendee: Get session list with personal agenda selections. Requires attendee auth. |
| `POST` | `/api/v1/attendee/events/{eventId}/sessions/{sessionId}/register` | `CreateAttendeeCommand` | Authenticated attendee: Add session to personal agenda. Requires attendee auth. |
| `DELETE` | `/api/v1/attendee/events/{eventId}/sessions/{sessionId}/register` | `DeleteAttendeeCommand` | Authenticated attendee: Remove session from personal agenda. Requires attendee auth. |
| `GET` | `/api/v1/attendee/events/{eventId}/matches` | `GetAttendeeByIdQuery` | Authenticated attendee: Get AI-powered attendee match suggestions. Requires attendee auth + feature flag. |
| `POST` | `/api/v1/attendee/events/{eventId}/matches/{matchId}/accept` | `CreateAttendeeCommand` | Authenticated attendee: Accept a networking match suggestion. Requires attendee auth. |
| `POST` | `/api/v1/attendee/events/{eventId}/matches/{matchId}/dismiss` | `CreateAttendeeCommand` | Authenticated attendee: Dismiss a networking match suggestion. Requires attendee auth. |
| `POST` | `/api/v1/attendee/events/{eventId}/sessions/{sessionId}/feedback` | `CreateAttendeeCommand` | Authenticated attendee: Submit session feedback rating and comment. Requires attendee auth. |
| `PUT` | `/api/v1/attendee/profile` | `UpdateAttendeeCommand` | Authenticated attendee: Update profile, interests, and networking preferences. Requires attendee auth. |

## Venues

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/venues` | `GetVenuesQuery` | Search venue listings. Query params: city, capacity, date, amenities. Cursor pagination. Requires auth. |
| `POST` | `/api/v1/venues` | `CreateVenueCommand` | Create a venue listing (venue manager or admin). Requires auth. |
| `GET` | `/api/v1/venues/{venueId}` | `GetVenueByIdQuery` | Get venue details with spaces and availability. Requires auth. |
| `PUT` | `/api/v1/venues/{venueId}` | `UpdateVenueCommand` | Update venue details. Requires venue manager or admin role. |
| `POST` | `/api/v1/venues/{venueId}/inquiries` | `CreateVenueCommand` | Submit a venue inquiry for an event date. Requires organizer auth. |
| `GET` | `/api/v1/venues/{venueId}/inquiries` | `GetVenueByIdQuery` | List inquiries for a venue (venue manager view). Requires venue manager auth. |
| `PUT` | `/api/v1/venues/{venueId}/inquiries/{inquiryId}/respond` | `UpdateVenueCommand` | Venue manager responds to an inquiry. Requires venue manager auth. |

## Admin

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/admin/tenants` | `GetAdminsQuery` | Super admin: List all organizations. Cursor pagination. Requires super_admin role. |
| `GET` | `/api/v1/admin/tenants/{orgId}` | `GetAdminByIdQuery` | Super admin: Get organization details and usage metrics. Requires super_admin role. |
| `POST` | `/api/v1/admin/tenants/{orgId}/impersonate` | `CreateAdminCommand` | Super admin: Generate impersonation token for support. Requires super_admin role. Audit logged. |
| `PUT` | `/api/v1/admin/tenants/{orgId}/plan` | `UpdateAdminCommand` | Super admin: Override organization plan. Requires super_admin role. |
| `GET` | `/api/v1/admin/platform/metrics` | `GetAdminsQuery` | Super admin: Platform-wide metrics (MRR, total events, registrations, GTV). Requires super_admin role. |
| `GET` | `/api/v1/admin/feature-flags` | `GetAdminsQuery` | Super admin: List Unleash feature flags and their states. Requires super_admin role. |

## {orgSlug}

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/public/events/{orgSlug}/{eventSlug}` | `Get{orgSlug}ByIdQuery` | Public endpoint: Get published event details for registration page rendering. No auth required. |
| `GET` | `/public/events/{orgSlug}/{eventSlug}/sessions` | `Get{orgSlug}ByIdQuery` | Public endpoint: Get published event session schedule. No auth required. |
| `POST` | `/public/events/{orgSlug}/{eventSlug}/validate-discount` | `Create{orgSlug}Command` | Public endpoint: Validate a discount code and return discount amount. No auth required. |
| `POST` | `/public/events/{orgSlug}/{eventSlug}/register` | `Create{orgSlug}Command` | Public endpoint: Create registration and initiate Stripe Checkout Session. Returns checkout URL. No auth required. |

## Speaker-portal

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/speaker-portal/{invitationToken}` | `GetSpeaker-portalByIdQuery` | Public endpoint: Get speaker portal data using invitation token. No auth required. |
| `PUT` | `/api/v1/speaker-portal/{invitationToken}` | `UpdateSpeaker-portalCommand` | Public endpoint: Speaker submits/updates their profile and accepts invitation. No auth required. |

## Sponsor-portal

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/sponsor-portal/{portalToken}` | `GetSponsor-portalByIdQuery` | Public endpoint: Get sponsor portal data (leads, event info). No auth required. |
| `GET` | `/api/v1/sponsor-portal/{portalToken}/leads` | `GetSponsor-portalByIdQuery` | Public endpoint: Download sponsor leads as CSV. No auth required (token-gated). |

## {registrationNumber}

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/public/registrations/{registrationNumber}` | `Get{registrationNumber}ByIdQuery` | Public endpoint: Get registration confirmation details by registration number. No auth required (used in confirmation email link). |
| `POST` | `/public/registrations/{registrationNumber}/cancel` | `Create{registrationNumber}Command` | Public endpoint: Attendee self-cancels registration per event cancellation policy. No auth required (token-validated). |

## Stripe

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `POST` | `/api/v1/stripe/webhooks` | `CreateStripeCommand` | Stripe webhook receiver. Validates Stripe signature, enqueues to Redis Streams. No auth (signature-validated). |
| `POST` | `/api/v1/stripe/checkout/success` | `CreateStripeCommand` | Handle Stripe Checkout success redirect. Confirms registration. No auth required. |

## Plans

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/plans` | `GetPlansQuery` | Public: List available subscription plans and features. No auth required. |
| `POST` | `/api/v1/plans/checkout` | `CreatePlanCommand` | Create Stripe Checkout Session for plan subscription. Requires auth. |

## Files

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/files/upload-url` | `GetFilesQuery` | Get S3 pre-signed upload URL for a given file type and context (event cover, speaker headshot). Requires auth. |
| `POST` | `/api/v1/files/confirm-upload` | `CreateFileCommand` | Confirm file upload completion and associate with entity. Requires auth. |

## Health

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/health` | `GetHealthsQuery` | Health check endpoint for load balancer and Kubernetes liveness probe. No auth required. |

## Ready

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/ready` | `GetReadysQuery` | Readiness probe checking DB and Redis connectivity. No auth required. |

## Metrics

| Method | Path | Command/Query | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/v1/metrics` | `GetMetricsQuery` | Prometheus metrics endpoint. Internal network only. |

