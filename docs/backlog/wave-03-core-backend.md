# Wave 3: Core Backend

## 3.1 Implement Organization (Tenant) CRUD: commands, handlers, validators, controller

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:m`, `agent:backend`
**Status:** Pending

Implement the full Organization management feature slice per the API Specification `/api/v1/organizations`.

**Key files:**
- `src/EventFlow.Application/Identity/Tenants/Commands/CreateTenantCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Identity/Tenants/Commands/UpdateTenantSettingsCommand.cs`
- `src/EventFlow.Application/Identity/Tenants/Commands/UpdateTenantThemeCommand.cs`
- `src/EventFlow.Application/Identity/Tenants/Commands/UploadTenantLogoCommand.cs`
- `src/EventFlow.Application/Identity/Tenants/Queries/GetTenantByIdQuery.cs`
- `src/EventFlow.Application/Identity/Tenants/Queries/GetTenantThemeQuery.cs` (cached 60min)
- `src/EventFlow.Api/Controllers/TenantsController.cs` (thin, mediator.Send only)

**Acceptance criteria:**
- [ ] POST `/api/v1/tenants` creates tenant + Keycloak user, publishes `TenantProvisioned` event
- [ ] GET `/api/v1/tenants/{id}/theme` is public (no auth) and Redis-cached 60 minutes
- [ ] `UpdateTenantThemeCommand` invalidates `{tenantId}:tenant:theme` cache key
- [ ] `CreateTenantValidator` enforces slug format (lowercase, hyphens only, 3-100 chars)
- [ ] Controller has zero business logic — only `mediator.Send()` calls

---

## 3.2 Implement Team Member management: invite, accept, role update, remove

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement the full member management feature slice per API Specification `/api/v1/organizations/me/members`.

**Key files:**
- `src/EventFlow.Application/Identity/Members/Commands/InviteMemberCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Identity/Members/Commands/AcceptInvitationCommand.cs`
- `src/EventFlow.Application/Identity/Members/Commands/UpdateMemberRoleCommand.cs`
- `src/EventFlow.Application/Identity/Members/Commands/RemoveMemberCommand.cs`
- `src/EventFlow.Application/Identity/Members/Commands/RevokeInvitationCommand.cs`
- `src/EventFlow.Application/Identity/Members/Queries/GetTenantMembersQuery.cs`
- `src/EventFlow.Api/Controllers/MembersController.cs`

**Acceptance criteria:**
- [ ] `InviteMemberCommand` generates secure token, sends email via `IEmailService`, creates `Invitation` record
- [ ] `AcceptInvitationCommand` validates token not expired, creates Keycloak user, sets `org_id` attribute
- [ ] `UpdateMemberRoleCommand` restricted to `RequireOwner` policy
- [ ] Invitation token is 32 bytes base64url, expires in 7 days
- [ ] `MemberInvitedEvent` and `MemberJoinedEvent` domain events published

---

## 3.3 Implement Event CRUD and lifecycle: create, update, publish, cancel, duplicate

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:l`, `agent:backend`
**Status:** Pending

Implement the core Event management feature slice per API Specification `/api/v1/events`.

**Key files:**
- `src/EventFlow.Application/Events/Commands/CreateEventCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Events/Commands/UpdateEventCommand.cs`
- `src/EventFlow.Application/Events/Commands/PublishEventCommand.cs`
- `src/EventFlow.Application/Events/Commands/CancelEventCommand.cs`
- `src/EventFlow.Application/Events/Commands/DuplicateEventCommand.cs`
- `src/EventFlow.Application/Events/Queries/GetEventsQuery.cs` (cached 3min, cursor pagination)
- `src/EventFlow.Application/Events/Queries/GetEventByIdQuery.cs`
- `src/EventFlow.Api/Controllers/EventsController.cs`

**Acceptance criteria:**
- [ ] `PublishEventCommand` validates: ≥1 TicketType, valid dates, name present; returns 409 if already published
- [ ] `DuplicateEventCommand` copies sessions, sponsor tiers, ticket types based on boolean flags
- [ ] `CancelEventCommand` with `NotifyAttendees=true` publishes `EventCancelledEvent` to trigger bulk email
- [ ] `GetEventsQuery` supports all filters from API spec (status, type, format, date range, tags, search)
- [ ] Plan limit check in `CreateEventCommand`: throws `PlanLimitExceededException` if exceeded

---

## 3.4 Implement Session, Track, and TicketType management commands, queries, and controllers

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

Implement session scheduling, track management, and ticket tier CRUD per API Specification sections 7, 8, 9.

**Key files:**
- `src/EventFlow.Application/Events/Sessions/Commands/CreateSessionCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Events/Sessions/Commands/UpdateSessionCommand.cs`
- `src/EventFlow.Application/Events/Sessions/Commands/ReorderSessionsCommand.cs`
- `src/EventFlow.Application/Events/Sessions/Queries/GetAgendaByEventQuery.cs`
- `src/EventFlow.Application/Events/Tracks/Commands/CreateTrackCommand.cs`
- `src/EventFlow.Application/Events/TicketTypes/Commands/CreateTicketTypeCommand.cs`
- `src/EventFlow.Api/Controllers/SessionsController.cs`
- `src/EventFlow.Api/Controllers/TicketTypesController.cs`

**Acceptance criteria:**
- [ ] `CreateSessionCommand` validates session within event date range and detects track time conflicts
- [ ] `ReorderSessionsCommand` updates `SortOrder` for all sessions in a single DB transaction
- [ ] `DeleteTicketTypeCommand` returns 409 if registrations exist for the ticket type
- [ ] `GetAgendaByEventQuery` returns sessions grouped by track, cached 10 minutes
- [ ] Session cache key `{tenantId}:events:{eventId}:sessions` invalidated on any session mutation

---

## 3.5 Implement Registration creation, confirmation, cancellation, transfer, and bulk import

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:l`, `agent:backend`
**Status:** Pending

Implement the full Registration feature slice per API Specification section 11.

**Key files:**
- `src/EventFlow.Application/Registrations/Commands/CreateRegistrationCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Registrations/Commands/ConfirmRegistrationCommand.cs`
- `src/EventFlow.Application/Registrations/Commands/CancelRegistrationCommand.cs`
- `src/EventFlow.Application/Registrations/Commands/TransferRegistrationCommand.cs`
- `src/EventFlow.Application/Registrations/Commands/BulkImportRegistrationsCommand.cs`
- `src/EventFlow.Application/Registrations/Queries/GetRegistrationsQuery.cs` (cursor pagination, filters)
- `src/EventFlow.Application/Registrations/Queries/ExportRegistrationsQuery.cs`
- `src/EventFlow.Api/Controllers/RegistrationsController.cs`

**Acceptance criteria:**
- [ ] `CreateRegistrationCommand` acquires distributed lock on `{eventId}:{ticketTypeId}` to prevent oversell
- [ ] `RegistrationCreatedEvent` published post-commit for email confirmation via outbox
- [ ] `BulkImportRegistrationsCommand` returns `BulkImportResultDto` with success/failure counts
- [ ] `ExportRegistrationsQuery` generates async S3 CSV with pre-signed download URL
- [ ] `GetRegistrationsQuery` returns summary stats (totalConfirmed, totalCancelled, totalWaitlisted)

---

## 3.6 Implement Check-in: QR scan, manual check-in, undo, offline sync, and real-time status

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:l`, `agent:backend`
**Status:** Pending

Implement the check-in feature slice critical for event-day operations per API Specification section 12.

**Key files:**
- `src/EventFlow.Application/CheckIn/Commands/CheckInAttendeeCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/CheckIn/Commands/UndoCheckInCommand.cs`
- `src/EventFlow.Application/CheckIn/Commands/SyncOfflineCheckInsCommand.cs`
- `src/EventFlow.Application/Registrations/Queries/GetCheckInStatusQuery.cs` (5s TTL cache)
- `src/EventFlow.Application/Registrations/Queries/GetRegistrationByQrCodeQuery.cs`
- `src/EventFlow.Api/Controllers/CheckInController.cs`
- `src/EventFlow.Api/Hubs/CheckInHub.cs` (SignalR)

**Acceptance criteria:**
- [ ] `CheckInAttendeeCommand` acquires distributed lock `checkin:{eventId}:{attendeeId}` to prevent double check-in
- [ ] Returns typed error codes: `AlreadyCheckedIn`, `RegistrationNotFound`, `RegistrationNotConfirmed`
- [ ] `SyncOfflineCheckInsCommand` is idempotent: same `DeviceIdentifier`+`QrCodeData` = no duplicate
- [ ] `AttendeeCheckedInEvent` published to Redis Streams → `RealtimeEventConsumer` → SignalR push
- [ ] Check-in endpoint rate limit: 300/min per user (highest of all endpoints)

---

## 3.7 Implement Venue management and inquiry system commands, queries, and controller

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement the Venue feature slice per API Specification section 14 and Domain Model section 2.4.

**Key files:**
- `src/EventFlow.Application/Venues/Commands/CreateVenueCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Venues/Commands/UpdateVenueCommand.cs`
- `src/EventFlow.Application/Venues/Commands/AddVenueRoomCommand.cs`
- `src/EventFlow.Application/Venues/Commands/SubmitVenueReviewCommand.cs`
- `src/EventFlow.Application/Venues/Commands/UploadVenuePhotoCommand.cs`
- `src/EventFlow.Application/Venues/Queries/GetVenuesQuery.cs` (cached 30min, filters)
- `src/EventFlow.Application/Venues/Queries/GetVenueAvailabilityQuery.cs`
- `src/EventFlow.Api/Controllers/VenuesController.cs`

**Acceptance criteria:**
- [ ] `GetVenuesQuery` supports all API spec filters: type, city, countryCode, minCapacity, amenities, search
- [ ] `GetVenueAvailabilityQuery` cross-checks events using venue in requested date window
- [ ] Venue inquiry endpoints: POST `/api/v1/venues/{id}/inquiries` and PUT respond
- [ ] `UploadVenuePhotoCommand` uses `IFileStorage` to store in `venues/{venantId}/{venueId}/` path
- [ ] Venues with `org_id = null` treated as platform-wide listings accessible to all tenants

---

## 3.8 Implement Sponsor management: tiers, event sponsors, lead capture, portal token

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement the Sponsorship feature slice per API Specification section 15 and Domain Model section 2.5.

**Key files:**
- `src/EventFlow.Application/Sponsors/Commands/CreateSponsorCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Sponsors/Commands/CreateSponsorTierCommand.cs`
- `src/EventFlow.Application/Sponsors/Commands/ReorderSponsorTiersCommand.cs`
- `src/EventFlow.Application/Sponsors/Commands/AddEventSponsorCommand.cs`
- `src/EventFlow.Application/Sponsors/Commands/CaptureSponsorLeadCommand.cs`
- `src/EventFlow.Application/Sponsors/Queries/GetEventSponsorsQuery.cs`
- `src/EventFlow.Application/Sponsors/Queries/GetSponsorLeadsQuery.cs`
- `src/EventFlow.Api/Controllers/SponsorsController.cs`

**Acceptance criteria:**
- [ ] `ReorderSponsorTiersCommand` updates `SortOrder` in single transaction
- [ ] `CaptureSponsorLeadCommand` requires `attendee.sponsor_contact_opt_in = true`
- [ ] Sponsor portal token generated as UUID, stored in `portal_access_token` field
- [ ] GET `/api/v1/sponsor-portal/{portalToken}` and `/leads` endpoints (public, token-gated)
- [ ] `UploadSponsorLogoCommand` stores logo via `IFileStorage`

---

## 3.9 Implement Campaign and EmailTemplate management with scheduling and send

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement the Communications feature slice per API Specification section 16 and Domain Model section 2.6.

**Key files:**
- `src/EventFlow.Application/Campaigns/Commands/CreateCampaignCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Campaigns/Commands/ScheduleCampaignCommand.cs`
- `src/EventFlow.Application/Campaigns/Commands/SendCampaignImmediatelyCommand.cs`
- `src/EventFlow.Application/Campaigns/Commands/SendTestEmailCommand.cs`
- `src/EventFlow.Application/Campaigns/Commands/CreateEmailTemplateCommand.cs`
- `src/EventFlow.Application/Campaigns/Queries/GetCampaignRecipientsPreviewQuery.cs`
- `src/EventFlow.Infrastructure/BackgroundServices/CampaignSchedulerService.cs`
- `src/EventFlow.Api/Controllers/CampaignsController.cs`

**Acceptance criteria:**
- [ ] `UpdateCampaignCommand` returns 409 if campaign status is `Sent`
- [ ] `SendCampaignImmediatelyCommand` enqueues to Redis Streams for worker processing (not inline send)
- [ ] `CampaignSchedulerService` processes campaigns scheduled for current minute
- [ ] `GetCampaignRecipientsPreviewQuery` resolves `RecipientFilter` to count + 3 sample emails
- [ ] `CampaignSentEvent` domain event published with `RecipientCount`

---

## 3.10 Implement Speaker management: invite, confirm via magic link, portal, session assignment

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement the Speaker feature slice per API Specification section 10 and Domain Model section 2.2.

**Key files:**
- `src/EventFlow.Application/Speakers/Commands/CreateSpeakerCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Speakers/Commands/InviteSpeakerToEventCommand.cs`
- `src/EventFlow.Application/Speakers/Commands/ConfirmSpeakerInviteCommand.cs`
- `src/EventFlow.Application/Speakers/Commands/AssignSpeakerToSessionCommand.cs`
- `src/EventFlow.Application/Speakers/Queries/GetSpeakerPortalDataQuery.cs`
- `src/EventFlow.Application/Speakers/Queries/GetSpeakersByEventQuery.cs`
- `src/EventFlow.Api/Controllers/SpeakersController.cs`

**Acceptance criteria:**
- [ ] `InviteSpeakerToEventCommand` generates 32-byte base64url magic link token, 48hr expiry
- [ ] `SpeakerInvitedToEventEvent` published → email consumer sends invitation email with portal URL
- [ ] `GetSpeakerPortalDataQuery` resolves token without requiring JWT auth
- [ ] `ConfirmSpeakerInviteCommand` marks token as used (single-use enforcement)
- [ ] Speaker portal PUT endpoint allows speaker to update bio, headshot, accept invitation

---

## 3.11 Implement Global Search query powering Cmd+K command palette

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:s`, `agent:backend`
**Status:** Pending

Implement the unified search endpoint that powers the `Cmd+K` command palette per API Specification section 19.

**Key files:**
- `src/EventFlow.Application/Search/Queries/GlobalSearchQuery.cs` (record + handler)
- `src/EventFlow.Api/Controllers/SearchController.cs`

**Search scope:** Events (by name), Attendees (by name/email), Venues (by name), Speakers (by name)

**SQL approach:** `pg_trgm` trigram index with `%` operator or `tsvector` full-text search, tenant-scoped

**Acceptance criteria:**
- [ ] GET `/api/v1/search?q=&maxResults=20` returns results grouped by entity type
- [ ] Search is tenant-scoped (global filter applies)
- [ ] Results include `id`, `name`/`title`, entity type, and deep-link URL
- [ ] Response cached in Redis with 30s TTL under key `{tenantId}:search:{queryHash}`
- [ ] p95 response time under 200ms with trigram index on `events.title`, `attendees.full_name`

---

## 3.12 Implement Waitlist: join, offer spot, convert to registration

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement the Waitlist feature slice per API Specification section 13 and Domain Model section 2.3.

**Key files:**
- `src/EventFlow.Application/Registrations/Waitlist/Commands/JoinWaitlistCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Registrations/Waitlist/Commands/OfferWaitlistSpotCommand.cs`
- `src/EventFlow.Application/Registrations/Waitlist/Commands/ConvertWaitlistToRegistrationCommand.cs`
- `src/EventFlow.Application/Registrations/Waitlist/Queries/GetWaitlistQuery.cs`
- `src/EventFlow.Api/Controllers/WaitlistController.cs`

**Acceptance criteria:**
- [ ] `JoinWaitlistCommand` returns position number and prevents duplicate waitlist entries per email/event
- [ ] `OfferWaitlistSpotCommand` sets `OfferExpiresAt` and sends notification email
- [ ] `ConvertWaitlistToRegistrationCommand` validates offer not expired, creates registration atomically
- [ ] POST `/api/v1/events/{id}/waitlist/promote` allows organizer to manually promote next N attendees
- [ ] `WaitlistEntryAddedEvent` and `WaitlistOfferMadeEvent` domain events published

---

## 3.13 Implement DELETE /api/v1/events/{eventId} soft-delete endpoint

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:s`, `agent:backend`
**Status:** Pending

## Goal
Allow organizers to delete draft events, with soft-delete to preserve audit trail.

## Acceptance Criteria
- `DELETE /api/v1/events/{eventId}` soft-deletes a draft event (`deleted_at` timestamp set)
- Returns `204 No Content` on success
- Returns `409 Conflict` if event is `Published` or `Live` (must cancel first)
- Soft-deleted events excluded from all queries via EF Core global query filter
- `EventDeletedDomainEvent` published for analytics/audit

## Technical Notes
- `DeleteEventCommand` handler: validates status = `Draft` or `Cancelled`, sets `DeletedAt`
- EF Core global filter already handles `deleted_at IS NULL` via `HasQueryFilter`
- Cascades: sessions, ticket types, sponsor tiers also soft-deleted

---

## 3.14 Implement session speaker assignment and removal API endpoints

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:s`, `agent:backend`
**Status:** Pending

## Goal
Fill missing session-speaker management endpoints for assigning and removing speakers from sessions.

## Acceptance Criteria
- `POST /api/v1/events/{eventId}/sessions/{sessionId}/speakers` assigns a speaker to a session with optional role
- `DELETE /api/v1/events/{eventId}/sessions/{sessionId}/speakers/{speakerId}` removes speaker from session
- Speaker must already be an `EventSpeaker` for the event
- `SessionSpeaker` join record created/deleted
- Session detail response includes updated speaker list

## Technical Notes
- Maps to existing `AssignSpeakerToSessionCommand` and `RemoveSpeakerFromSessionCommand`
- Validates: speaker exists in `EventSpeaker` for this event before assigning to session
- Invalidates `sessions:{eventId}` cache key on mutation

---

