# API Design — EventFlow

> **Version**: 1.0 | **Status**: Approved for Implementation
> All controllers are THIN: parse HTTP request → `mediator.Send()` → return result. Zero business logic in controllers.

---

## Table of Contents

1. [API Conventions](#1-api-conventions)
2. [Authentication & Authorization](#2-authentication--authorization)
3. [MediatR Pipeline](#3-mediatr-pipeline)
4. [Tenants API](#4-tenants-api)
5. [Team Members API](#5-team-members-api)
6. [Events API](#6-events-api)
7. [Sessions API](#7-sessions-api)
8. [Tracks API](#8-tracks-api)
9. [Ticket Types API](#9-ticket-types-api)
10. [Speakers API](#10-speakers-api)
11. [Registrations API](#11-registrations-api)
12. [Check-In API](#12-check-in-api)
13. [Waitlist API](#13-waitlist-api)
14. [Venues API](#14-venues-api)
15. [Sponsors API](#15-sponsors-api)
16. [Campaigns API](#16-campaigns-api)
17. [Analytics API](#17-analytics-api)
18. [AI API](#18-ai-api)
19. [Search API](#19-search-api)
20. [Real-Time API (SSE / WebSocket)](#20-real-time-api)
21. [Error Response Standard](#21-error-response-standard)
22. [Rate Limiting Rules](#22-rate-limiting-rules)

---

## 1. API Conventions

### Base URL
```
Production:  https://api.eventflow.app/api/v1
Local:       http://localhost:5000/api/v1
```

### Versioning
URL path versioning: `/api/v1/...`. New major versions (`/api/v2/...`) are introduced only for breaking changes. Non-breaking additions (new fields, new endpoints) are additive within `v1`.

### Multi-Tenancy Header
Every authenticated request must include the tenant context. The tenant is resolved from the Keycloak JWT `tenant_id` claim. For public endpoints (registration pages, speaker portals), the tenant is resolved from the URL slug.

```http
Authorization: Bearer <keycloak_jwt>
```
The `tenant_id` claim is embedded in the JWT by Keycloak's realm configuration. Controllers extract it via `ICurrentUserService.TenantId`.

### Pagination
All list endpoints use **cursor-based pagination**:
```json
{
  "data": [...],
  "pagination": {
    "pageSize": 25,
    "hasNextPage": true,
    "nextCursor": "eyJpZCI6IjEyMyIsImNyZWF0ZWRBdCI6IjIwMjUtMDMtMTUifQ==",
    "totalCount": 142
  }
}
```
Query params: `?pageSize=25&cursor=<base64_cursor>`

### Filtering & Sorting
```
?status=Published
?type=Conference
?sortBy=startDate&sortDesc=true
?search=<term>         (URL-encoded search string)
?tags=ai,product      (comma-separated)
```

### Standard Success Responses
```
200 OK          — Successful GET / PUT / PATCH
201 Created     — Successful POST creating a new resource
202 Accepted    — Async operation started (AI generation, bulk import)
204 No Content  — Successful DELETE
```

### Standard Headers
```http
Content-Type: application/json
X-Request-Id: <uuid>           # Echoed from Serilog correlation ID
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1710000000
```

### Thin Controller Pattern (enforced)
```csharp
// CORRECT — every controller method looks exactly like this:
[HttpPost]
public async Task<ActionResult<EventDto>> Create(
    [FromBody] CreateEventRequest request,
    CancellationToken ct)
    => CreatedAtAction(nameof(GetById),
        new { eventId = result.Id },
        await _mediator.Send(new CreateEventCommand(
            TenantId, request.Name, request.Type, request.Format,
            request.StartDateUtc, request.EndDateUtc, request.TimeZoneId,
            request.Capacity, request.IsPublic, request.Description,
            request.VenueId, request.VirtualMeetingUrl, UserId), ct));

// WRONG — no business logic in controllers:
// ❌ if (event.Status == EventStatus.Draft) ...
// ❌ var sessions = await _sessionRepository.GetByEvent(eventId);
// ❌ event.Capacity = request.Capacity;
```

All controllers inherit from `EventFlowControllerBase` which exposes:
```csharp
public abstract class EventFlowControllerBase : ControllerBase
{
    protected IMediator _mediator;
    protected Guid TenantId => /* from JWT claim */;
    protected string UserId => /* from JWT sub claim */;
    protected TenantRole UserRole => /* from JWT role claim */;
}
```

---

## 2. Authentication & Authorization

### Auth Mechanism
- **OIDC / JWT** via Keycloak
- Token validated on every request by ASP.NET Core `AddJwtBearer`
- Keycloak public keys fetched from `/.well-known/openid-configuration`

### Roles
```
owner          — Full access including billing and tenant deletion
admin          — Full access except billing
event_manager  — Create/edit events, manage registrations, send campaigns
viewer         — Read-only on all event data
checkin_staff  — Check-in operations only (no event creation)
public         — Unauthenticated (registration pages, public event pages)
```

### Authorization Policy Matrix
```
Policy Name        → Minimum Role Required
─────────────────────────────────────────
RequireOwner       → owner
RequireAdmin       → admin
RequireManager     → event_manager
RequireViewer      → viewer (any authenticated member)
RequireCheckIn     → checkin_staff
Public             → No auth required
```

Policies are registered in `Program.cs` and applied via `[Authorize(Policy = "...")]` on controller actions.

---

## 3. MediatR Pipeline

Every request (Command and Query) flows through this ordered pipeline:

```
HTTP Request
    │
    ▼
[Controller] — thin: maps request → MediatR command/query
    │
    ▼
[LoggingBehavior]       — Logs request name, tenant, user, duration
    │
    ▼
[FeatureFlagBehavior]   — Checks Unleash feature flag for command type;
                          returns 404 FeatureNotAvailableError if flag is OFF
    │
    ▼
[ValidationBehavior]    — Runs FluentValidation; returns 422 with field errors
    │
    ▼
[CacheBehavior]         — Queries only: checks Redis; returns cached response
                          if valid. Commands: invalidates relevant cache keys.
    │
    ▼
[CostTrackingBehavior]  — AI commands only: records token usage to tenant ledger
    │
    ▼
[Handler]               — ALL business logic lives here
    │
    ▼
Response / DomainEvent → Redis Streams
```

---

## 4. Tenants API

### Controller: `TenantsController`
**Route prefix**: `/api/v1/tenants`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | POST | `/api/v1/tenants` | `CreateTenantCommand` | Public (initial signup) | 5/min/IP |
| 2 | GET | `/api/v1/tenants/{tenantId}` | `GetTenantByIdQuery` | RequireViewer | 120/min |
| 3 | PUT | `/api/v1/tenants/{tenantId}` | `UpdateTenantSettingsCommand` | RequireAdmin | 30/min |
| 4 | GET | `/api/v1/tenants/{tenantId}/theme` | `GetTenantThemeQuery` | Public | 300/min |
| 5 | PUT | `/api/v1/tenants/{tenantId}/theme` | `UpdateTenantThemeCommand` | RequireAdmin | 30/min |
| 6 | POST | `/api/v1/tenants/{tenantId}/logo` | `UploadTenantLogoCommand` | RequireAdmin | 10/min |

---

#### POST `/api/v1/tenants`
**Command**: `CreateTenantCommand`

**Request Body**:
```json
{
  "name": "Acme Corp",
  "slug": "acme-corp",
  "ownerEmail": "maya@acmecorp.com",
  "ownerFirstName": "Maya",
  "ownerLastName": "Chen",
  "plan": "Growth",
  "industry": "Technology"
}
```

**Response** `201 Created`:
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "name": "Acme Corp",
  "slug": "acme-corp",
  "plan": "Growth",
  "status": "Trial",
  "trialEndsAt": "2025-04-15T00:00:00Z",
  "maxEventsPerYear": 50,
  "maxAttendeesPerEvent": 1000,
  "createdAt": "2025-03-15T10:00:00Z"
}
```

**Controller**:
```csharp
[HttpPost]
[AllowAnonymous]
public async Task<ActionResult<TenantDto>> Create(
    [FromBody] CreateTenantRequest request, CancellationToken ct)
    => CreatedAtAction(nameof(GetById),
        new { tenantId = result.Id },
        await _mediator.Send(new CreateTenantCommand(
            request.Name, request.Slug, request.OwnerKeycloakUserId,
            request.OwnerEmail, request.OwnerFirstName, request.OwnerLastName,
            Enum.Parse<TenantPlan>(request.Plan), request.Industry), ct));
```

---

#### GET `/api/v1/tenants/{tenantId}/theme`
**Query**: `GetTenantThemeQuery`

**Response** `200 OK`:
```json
{
  "primaryColor": "#6366F1",
  "secondaryColor": "#8B5CF6",
  "logoUrl": "https://cdn.eventflow.app/tenants/acme/logo.png",
  "faviconUrl": "https://cdn.eventflow.app/tenants/acme/favicon.ico",
  "customDomain": "events.acmecorp.com",
  "fontFamily": "Inter"
}
```

**Controller**:
```csharp
[HttpGet("{tenantId}/theme")]
[AllowAnonymous]
public async Task<ActionResult<TenantThemeDto>> GetTheme(
    Guid tenantId, CancellationToken ct)
    => Ok(await _mediator.Send(new GetTenantThemeQuery(tenantId), ct));
```

---

## 5. Team Members API

### Controller: `MembersController`
**Route prefix**: `/api/v1/tenants/{tenantId}/members`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/members` | `GetTenantMembersQuery` | RequireViewer | 60/min |
| 2 | GET | `/members/{memberId}` | `GetMemberByIdQuery` | RequireViewer | 120/min |
| 3 | POST | `/members/invite` | `InviteMemberCommand` | RequireAdmin | 20/min |
| 4 | POST | `/members/accept-invite` | `AcceptInvitationCommand` | Public (with token) | 10/min |
| 5 | PATCH | `/members/{memberId}/role` | `UpdateMemberRoleCommand` | RequireOwner | 30/min |
| 6 | DELETE | `/members/{memberId}` | `RemoveMemberCommand` | RequireAdmin | 20/min |
| 7 | GET | `/members/invitations` | `GetPendingInvitationsQuery` | RequireAdmin | 60/min |
| 8 | DELETE | `/members/invitations/{invitationId}` | `RevokeInvitationCommand` | RequireAdmin | 20/min |
| 9 | GET | `/members/me` | `GetCurrentUserProfileQuery` | RequireViewer | 120/min |

---

#### POST `/api/v1/tenants/{tenantId}/members/invite`
**Command**: `InviteMemberCommand`

**Request Body**:
```json
{
  "email": "david@acmecorp.com",
  "role": "EventManager"
}
```

**Response** `201 Created`:
```json
{
  "id": "inv_abc123",
  "email": "david@acmecorp.com",
  "role": "EventManager",
  "status": "Pending",
  "expiresAt": "2025-03-22T10:00:00Z",
  "createdAt": "2025-03-15T10:00:00Z"
}
```

**Controller**:
```csharp
[HttpPost("invite")]
[Authorize(Policy = "RequireAdmin")]
public async Task<ActionResult<InvitationDto>> Invite(
    Guid tenantId, [FromBody] InviteMemberRequest request, CancellationToken ct)
    => CreatedAtAction(nameof(GetInvitations), null,
        await _mediator.Send(new InviteMemberCommand(
            tenantId, request.Email,
            Enum.Parse<TenantRole>(request.Role), UserId), ct));
```

---

#### PATCH `/api/v1/tenants/{tenantId}/members/{memberId}/role`
**Command**: `UpdateMemberRoleCommand`

**Request Body**:
```json
{
  "role": "Admin"
}
```

**Response** `200 OK`: Returns updated `TenantMemberDto`

**Controller**:
```csharp
[HttpPatch("{memberId}/role")]
[Authorize(Policy = "RequireOwner")]
public async Task<ActionResult<TenantMemberDto>> UpdateRole(
    Guid tenantId, Guid memberId,
    [FromBody] UpdateMemberRoleRequest request, CancellationToken ct)
    => Ok(await _mediator.Send(new UpdateMemberRoleCommand(
        tenantId, memberId, Enum.Parse<TenantRole>(request.Role)), ct));
```

---

## 6. Events API

### Controller: `EventsController`
**Route prefix**: `/api/v1/events`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/events` | `GetEventsQuery` | RequireViewer | 60/min |
| 2 | POST | `/events` | `CreateEventCommand` | RequireManager | 30/min |
| 3 | GET | `/events/{eventId}` | `GetEventByIdQuery` | RequireViewer | 120/min |
| 4 | PUT | `/events/{eventId}` | `UpdateEventCommand` | RequireManager | 60/min |
| 5 | DELETE | `/events/{eventId}` | `CancelEventCommand` | RequireManager | 10/min |
| 6 | POST | `/events/{eventId}/publish` | `PublishEventCommand` | RequireManager | 20/min |
| 7 | POST | `/events/{eventId}/duplicate` | `DuplicateEventCommand` | RequireManager | 10/min |
| 8 | POST | `/events/{eventId}/cover-image` | `UploadEventCoverImageCommand` | RequireManager | 20/min |
| 9 | PUT | `/events/{eventId}/settings` | `UpdateEventSettingsCommand` | RequireManager | 30/min |
| 10 | PUT | `/events/{eventId}/tags` | `UpdateEventTagsCommand` | RequireManager | 30/min |
| 11 | GET | `/events/{eventId}/checklist` | `GetEventSetupChecklistQuery` | RequireViewer | 120/min |
| 12 | GET | `/events/{eventId}/dashboard` | `GetEventDashboardQuery` | RequireViewer | 60/min |
| 13 | GET | `/public/events/{tenantSlug}/{eventSlug}` | `GetEventBySlugQuery` | Public | 300/min |

---

#### GET `/api/v1/events`
**Query**: `GetEventsQuery`

**Query Parameters**:
```
?status=Published
?type=Conference
?format=InPerson
?startAfter=2025-03-01T00:00:00Z
?startBefore=2025-12-31T00:00:00Z
?search=sales+kickoff
?tags=enterprise,product
?sortBy=startDate
?sortDesc=false
?pageSize=25
?cursor=<base64>
```

**Response** `200 OK`:
```json
{
  "data": [
    {
      "id": "evt_abc123",
      "name": "Q4 Sales Kickoff 2025",
      "slug": "q4-sales-kickoff-2025",
      "type": "Conference",
      "format": "InPerson",
      "status": "Published",
      "startDateUtc": "2025-03-15T09:00:00Z",
      "endDateUtc": "2025-03-16T18:00:00Z",
      "timeZoneId": "America/Chicago",
      "capacity": 350,
      "registrationCount": 280,
      "checkedInCount": 0,
      "coverImageUrl": "https://cdn.eventflow.app/events/evt_abc123/cover.jpg",
      "venueName": "Austin Convention Center",
      "venueCity": "Austin",
      "createdAt": "2025-01-10T08:00:00Z"
    }
  ],
  "pagination": {
    "pageSize": 25,
    "hasNextPage": true,
    "nextCursor": "eyJpZCI6ImV2dF9hYmMxMjMifQ==",
    "totalCount": 87
  }
}
```

**Controller**:
```csharp
[HttpGet]
[Authorize(Policy = "RequireViewer")]
public async Task<ActionResult<PagedResult<EventSummaryDto>>> GetAll(
    [FromQuery] GetEventsRequest request, CancellationToken ct)
    => Ok(await _mediator.Send(new GetEventsQuery(
        TenantId, request.Status, request.Type, request.Format,
        request.StartAfter, request.StartBefore, request.Search,
        request.Tags, request.SortBy ?? "startDate", request.SortDesc,
        request.PageSize ?? 25, request.Cursor), ct));
```

---

#### POST `/api/v1/events`
**Command**: `CreateEventCommand`

**Request Body**:
```json
{
  "name": "Q4 Sales Kickoff 2025",
  "type": "Conference",
  "format": "InPerson",
  "startDateUtc": "2025-03-15T09:00:00Z",
  "endDateUtc": "2025-03-16T18:00:00Z",
  "timeZoneId": "America/Chicago",
  "capacity": 350,
  "isPublic": true,
  "description": "Annual sales team kickoff conference.",
  "venueId": "ven_xyz789",
  "virtualMeetingUrl": null
}
```

**Response** `201 Created`:
```json
{
  "id": "evt_abc123",
  "name": "Q4 Sales Kickoff 2025",
  "slug": "q4-sales-kickoff-2025",
  "status": "Draft",
  "type": "Conference",
  "format": "InPerson",
  "startDateUtc": "2025-03-15T09:00:00Z",
  "endDateUtc": "2025-03-16T18:00:00Z",
  "timeZoneId": "America/Chicago",
  "capacity": 350,
  "registrationCount": 0,
  "isPublic": true,
  "createdAt": "2025-01-10T08:00:00Z",
  "updatedAt": "2025-01-10T08:00:00Z"
}
```

**Controller**:
```csharp
[HttpPost]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<EventDto>> Create(
    [FromBody] CreateEventRequest request, CancellationToken ct)
{
    var result = await _mediator.Send(new CreateEventCommand(
        TenantId, request.Name, Enum.Parse<EventType>(request.Type),
        Enum.Parse<EventFormat>(request.Format), request.StartDateUtc,
        request.EndDateUtc, request.TimeZoneId, request.Capacity,
        request.IsPublic, request.Description, request.VenueId,
        request.VirtualMeetingUrl, UserId), ct);
    return CreatedAtAction(nameof(GetById), new { eventId = result.Id }, result);
}
```

---

#### POST `/api/v1/events/{eventId}/publish`
**Command**: `PublishEventCommand`

**Request Body**: `{}` (empty — event ID from route)

**Response** `200 OK`:
```json
{
  "id": "evt_abc123",
  "status": "Published",
  "publishedAt": "2025-01-15T14:30:00Z",
  "registrationUrl": "https://events.acmecorp.com/events/q4-sales-kickoff-2025"
}
```

**Status Codes**: `409 Conflict` if event has no ticket types or is already published.

**Controller**:
```csharp
[HttpPost("{eventId}/publish")]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<EventDto>> Publish(
    Guid eventId, CancellationToken ct)
    => Ok(await _mediator.Send(new PublishEventCommand(eventId, TenantId), ct));
```

---

#### POST `/api/v1/events/{eventId}/duplicate`
**Command**: `DuplicateEventCommand`

**Request Body**:
```json
{
  "newName": "Q1 Sales Kickoff 2026",
  "newStartDate": "2026-01-15T09:00:00Z",
  "newEndDate": "2026-01-16T18:00:00Z",
  "copySessions": true,
  "copySponsorTiers": true,
  "copyTicketTypes": true
}
```

**Response** `201 Created`: Returns new `EventDto` with `status: Draft`

**Controller**:
```csharp
[HttpPost("{eventId}/duplicate")]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<EventDto>> Duplicate(
    Guid eventId, [FromBody] DuplicateEventRequest request, CancellationToken ct)
{
    var result = await _mediator.Send(new DuplicateEventCommand(
        eventId, TenantId, request.NewName, request.NewStartDate,
        request.NewEndDate, request.CopySessions, request.CopySponsorTiers,
        request.CopyTicketTypes), ct);
    return CreatedAtAction(nameof(GetById), new { eventId = result.Id }, result);
}
```

---

#### GET `/api/v1/events/{eventId}/checklist`
**Query**: `GetEventSetupChecklistQuery`

**Response** `200 OK`:
```json
{
  "eventId": "evt_abc123",
  "overallProgress": 40,
  "items": [
    { "key": "eventCreated", "label": "Event created", "completed": true, "actionUrl": null },
    { "key": "registrationCustomized", "label": "Customize registration page", "completed": false, "actionUrl": "/events/evt_abc123/registration" },
    { "key": "sessionsAdded", "label": "Add your first session", "completed": false, "actionUrl": "/events/evt_abc123/sessions/new" },
    { "key": "ticketsConfigured", "label": "Set ticket types & pricing", "completed": true, "actionUrl": "/events/evt_abc123/tickets" },
    { "key": "speakersInvited", "label": "Invite speakers", "completed": false, "actionUrl": "/events/evt_abc123/speakers" },
    { "key": "published", "label": "Publish your event", "completed": false, "actionUrl": null }
  ]
}
```

---

## 7. Sessions API

### Controller: `SessionsController`
**Route prefix**: `/api/v1/events/{eventId}/sessions`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/sessions` | `GetSessionsByEventQuery` | RequireViewer | 120/min |
| 2 | POST | `/sessions` | `CreateSessionCommand` | RequireManager | 60/min |
| 3 | GET | `/sessions/{sessionId}` | `GetSessionByIdQuery` | RequireViewer | 120/min |
| 4 | PUT | `/sessions/{sessionId}` | `UpdateSessionCommand` | RequireManager | 60/min |
| 5 | DELETE | `/sessions/{sessionId}` | `DeleteSessionCommand` | RequireManager | 30/min |
| 6 | POST | `/sessions/reorder` | `ReorderSessionsCommand` | RequireManager | 30/min |
| 7 | GET | `/agenda` | `GetAgendaByEventQuery` | RequireViewer | 120/min |
| 8 | POST | `/sessions/{sessionId}/speakers` | `AssignSpeakerToSessionCommand` | RequireManager | 30/min |
| 9 | DELETE | `/sessions/{sessionId}/speakers/{speakerId}` | `RemoveSpeakerFromSessionCommand` | RequireManager | 30/min |

---

#### POST `/api/v1/events/{eventId}/sessions`
**Command**: `CreateSessionCommand`

**Request Body**:
```json
{
  "title": "Opening Keynote: The Future of SaaS",
  "description": "An overview of where enterprise SaaS is heading in 2025.",
  "type": "Keynote",
  "startTimeUtc": "2025-03-15T09:00:00Z",
  "endTimeUtc": "2025-03-15T10:00:00Z",
  "trackId": "trk_xyz",
  "venueRoomId": "room_abc",
  "capacityOverride": null,
  "isPublished": true,
  "sortOrder": 1
}
```

**Response** `201 Created`:
```json
{
  "id": "ses_abc123",
  "eventId": "evt_abc123",
  "title": "Opening Keynote: The Future of SaaS",
  "type": "Keynote",
  "startTimeUtc": "2025-03-15T09:00:00Z",
  "endTimeUtc": "2025-03-15T10:00:00Z",
  "durationMinutes": 60,
  "trackId": "trk_xyz",
  "trackName": "Main Stage",
  "venueRoomId": "room_abc",
  "venueRoomName": "Ballroom A",
  "isPublished": true,
  "speakers": [],
  "sortOrder": 1
}
```

**Controller**:
```csharp
[HttpPost]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<SessionDto>> Create(
    Guid eventId, [FromBody] CreateSessionRequest request, CancellationToken ct)
{
    var result = await _mediator.Send(new CreateSessionCommand(
        eventId, TenantId, request.Title, request.Description,
        Enum.Parse<SessionType>(request.Type), request.StartTimeUtc,
        request.EndTimeUtc, request.TrackId, request.VenueRoomId,
        request.CapacityOverride, request.IsPublished, request.SortOrder), ct);
    return CreatedAtAction(nameof(GetById), new { eventId, sessionId = result.Id }, result);
}
```

---

#### POST `/api/v1/events/{eventId}/sessions/reorder`
**Command**: `ReorderSessionsCommand`

**Request Body**:
```json
{
  "orderedSessions": [
    { "sessionId": "ses_abc123", "sortOrder": 1 },
    { "sessionId": "ses_def456", "sortOrder": 2 },
    { "sessionId": "ses_ghi789", "sortOrder": 3 }
  ]
}
```

**Response** `204 No Content`

**Controller**:
```csharp
[HttpPost("reorder")]
[Authorize(Policy = "RequireManager")]
public async Task<IActionResult> Reorder(
    Guid eventId, [FromBody] ReorderSessionsRequest request, CancellationToken ct)
{
    await _mediator.Send(new ReorderSessionsCommand(
        eventId, TenantId, request.OrderedSessions
            .Select(x => new SessionOrderItem(x.SessionId, x.SortOrder)).ToList()), ct);
    return NoContent();
}
```

---

#### GET `/api/v1/events/{eventId}/agenda`
**Query**: `GetAgendaByEventQuery`

**Response** `200 OK`:
```json
{
  "eventId": "evt_abc123",
  "tracks": [
    {
      "id": "trk_xyz",
      "name": "Main Stage",
      "color": "#6366F1"
    }
  ],
  "sessions": [
    {
      "id": "ses_abc123",
      "title": "Opening Keynote",
      "type": "Keynote",
      "startTimeUtc": "2025-03-15T09:00:00Z",
      "endTimeUtc": "2025-03-15T10:00:00Z",
      "trackId": "trk_xyz",
      "venueRoomName": "Ballroom A",
      "speakers": [
        {
          "speakerId": "spk_abc",
          "name": "Dr. Sarah Kim",
          "photoUrl": "https://cdn.eventflow.app/speakers/spk_abc.jpg",
          "role": "Keynote Speaker"
        }
      ],
      "isPublished": true,
      "sortOrder": 1
    }
  ]
}
```

---

## 8. Tracks API

### Controller: `TracksController`
**Route prefix**: `/api/v1/events/{eventId}/tracks`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/tracks` | `GetSessionsByEventQuery` (filter=tracks) | RequireViewer | 120/min |
| 2 | POST | `/tracks` | `CreateTrackCommand` | RequireManager | 30/min |
| 3 | PUT | `/tracks/{trackId}` | `UpdateTrackCommand` | RequireManager | 30/min |
| 4 | DELETE | `/tracks/{trackId}` | `DeleteTrackCommand` | RequireManager | 20/min |

**Controller**:
```csharp
[HttpPost]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<TrackDto>> Create(
    Guid eventId, [FromBody] CreateTrackRequest request, CancellationToken ct)
    => CreatedAtAction(nameof(GetAll), null,
        await _mediator.Send(new CreateTrackCommand(
            eventId, TenantId, request.Name, request.Color,
            request.Description, request.SortOrder), ct));
```

---

## 9. Ticket Types API

### Controller: `TicketTypesController`
**Route prefix**: `/api/v1/events/{eventId}/ticket-types`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/ticket-types` | `GetEventByIdQuery` (tickets included) | RequireViewer | 120/min |
| 2 | POST | `/ticket-types` | `CreateTicketTypeCommand` | RequireManager | 30/min |
| 3 | PUT | `/ticket-types/{ticketTypeId}` | `UpdateTicketTypeCommand` | RequireManager | 60/min |
| 4 | DELETE | `/ticket-types/{ticketTypeId}` | `DeleteTicketTypeCommand` | RequireManager | 20/min |

---

#### POST `/api/v1/events/{eventId}/ticket-types`
**Command**: `CreateTicketTypeCommand`

**Request Body**:
```json
{
  "name": "General Admission",
  "description": "Full conference access",
  "price": 299.00,
  "currency": "USD",
  "quantity": 300,
  "saleStartsAt": "2025-01-01T00:00:00Z",
  "saleEndsAt": "2025-03-14T23:59:59Z",
  "isPublic": true,
  "transferAllowed": true,
  "maxPerOrder": 5,
  "sortOrder": 1
}
```

**Response** `201 Created`:
```json
{
  "id": "tkt_abc123",
  "eventId": "evt_abc123",
  "name": "General Admission",
  "price": 299.00,
  "currency": "USD",
  "quantity": 300,
  "quantitySold": 0,
  "quantityRemaining": 300,
  "saleStartsAt": "2025-01-01T00:00:00Z",
  "saleEndsAt": "2025-03-14T23:59:59Z",
  "isPublic": true,
  "sortOrder": 1
}
```

**Controller**:
```csharp
[HttpPost]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<TicketTypeDto>> Create(
    Guid eventId, [FromBody] CreateTicketTypeRequest request, CancellationToken ct)
    => CreatedAtAction(nameof(GetAll), null,
        await _mediator.Send(new CreateTicketTypeCommand(
            eventId, TenantId, request.Name, request.Description,
            request.Price, request.Currency, request.Quantity,
            request.SaleStartsAt, request.SaleEndsAt, request.IsPublic,
            request.TransferAllowed, request.MaxPerOrder, request.SortOrder), ct));
```

---

## 10. Speakers API

### Controller: `SpeakersController`
**Route prefix**: `/api/v1/speakers`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/speakers` | `GetSpeakersQuery` | RequireViewer | 60/min |
| 2 | POST | `/speakers` | `CreateSpeakerCommand` | RequireManager | 30/min |
| 3 | GET | `/speakers/{speakerId}` | `GetSpeakerByIdQuery` | RequireViewer | 120/min |
| 4 | PUT | `/speakers/{speakerId}` | `UpdateSpeakerCommand` | RequireManager | 30/min |
| 5 | POST | `/speakers/{speakerId}/photo` | `UploadSpeakerPhotoCommand` | RequireManager | 10/min |
| 6 | GET | `/events/{eventId}/speakers` | `GetSpeakersByEventQuery` | RequireViewer | 120/min |
| 7 | POST | `/events/{eventId}/speakers/{speakerId}/invite` | `InviteSpeakerToEventCommand` | RequireManager | 20/min |
| 8 | DELETE | `/events/{eventId}/speakers/{speakerId}` | `RemoveSpeakerFromEventCommand` | RequireManager | 20/min |
| 9 | POST | `/speakers/confirm-invite` | `ConfirmSpeakerInviteCommand` | Public (with token) | 10/min |
| 10 | GET | `/speakers/portal` | `GetSpeakerPortalDataQuery` | Public (with token) | 60/min |

---

#### POST `/api/v1/events/{eventId}/speakers/{speakerId}/invite`
**Command**: `InviteSpeakerToEventCommand`

**Request Body**:
```json
{
  "role": "Keynote Speaker"
}
```

**Response** `201 Created`:
```json
{
  "speakerId": "spk_abc",
  "eventId": "evt_abc123",
  "email": "dr.kim@university.edu",
  "status": "Invited",
  "role": "Keynote Speaker",
  "magicLinkSentAt": "2025-01-15T10:00:00Z"
}
```

**Controller**:
```csharp
[HttpPost("/api/v1/events/{eventId}/speakers/{speakerId}/invite")]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<EventSpeakerDto>> Invite(
    Guid eventId, Guid speakerId,
    [FromBody] InviteSpeakerRequest request, CancellationToken ct)
    => CreatedAtAction(nameof(GetBySpeakerId), new { speakerId },
        await _mediator.Send(new InviteSpeakerToEventCommand(
            speakerId, eventId, TenantId, request.Role), ct));
```

---

#### GET `/api/v1/speakers/portal?token={token}`
**Query**: `GetSpeakerPortalDataQuery`

**Response** `200 OK`:
```json
{
  "speaker": {
    "id": "spk_abc",
    "firstName": "Sarah",
    "lastName": "Kim",
    "bio": "Dr. Kim is VP of Engineering at...",
    "photoUrl": "https://cdn.eventflow.app/speakers/spk_abc.jpg"
  },
  "events": [
    {
      "eventId": "evt_abc123",
      "eventName": "Q4 Sales Kickoff 2025",
      "eventDate": "2025-03-15",
      "sessions": [
        {
          "title": "Opening Keynote",
          "startTime": "2025-03-15T09:00:00Z",
          "room": "Ballroom A"
        }
      ]
    }
  ]
}
```

**Controller**:
```csharp
[HttpGet("portal")]
[AllowAnonymous]
public async Task<ActionResult<SpeakerPortalDto>> GetPortal(
    [FromQuery] string token, CancellationToken ct)
    => Ok(await _mediator.Send(new GetSpeakerPortalDataQuery(token), ct));
```

---

## 11. Registrations API

### Controller: `RegistrationsController`
**Route prefix**: `/api/v1/events/{eventId}/registrations`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/registrations` | `GetRegistrationsQuery` | RequireViewer | 60/min |
| 2 | POST | `/registrations` | `CreateRegistrationCommand` | Public | 20/min/IP |
| 3 | GET | `/registrations/{registrationId}` | `GetRegistrationByIdQuery` | RequireViewer | 120/min |
| 4 | PATCH | `/registrations/{registrationId}` | `UpdateRegistrationCommand` | RequireManager | 30/min |
| 5 | POST | `/registrations/{registrationId}/confirm` | `ConfirmRegistrationCommand` | RequireManager | 30/min |
| 6 | POST | `/registrations/{registrationId}/cancel` | `CancelRegistrationCommand` | RequireManager | 30/min |
| 7 | POST | `/registrations/{registrationId}/transfer` | `TransferRegistrationCommand` | RequireManager | 20/min |
| 8 | POST | `/registrations/bulk-import` | `BulkImportRegistrationsCommand` | RequireManager | 5/min |
| 9 | GET | `/registrations/export` | `ExportRegistrationsQuery` | RequireManager | 10/min |
| 10 | POST | `/registrations/{registrationId}/sessions/{sessionId}` | `RegisterForSessionCommand` | Public | 30/min/IP |
| 11 | GET | `/registrations/lookup?confirmationCode={code}` | `GetRegistrationByConfirmationCodeQuery` | RequireCheckIn | 300/min |

---

#### POST `/api/v1/events/{eventId}/registrations`
**Command**: `CreateRegistrationCommand`

**Request Body**:
```json
{
  "ticketTypeId": "tkt_abc123",
  "attendeeEmail": "john.smith@techcorp.com",
  "attendeeFirstName": "John",
  "attendeeLastName": "Smith",
  "attendeePhone": "+1-512-555-0100",
  "attendeeCompany": "TechCorp",
  "attendeeJobTitle": "Senior Engineer",
  "dietaryRestrictions": "Vegetarian",
  "specialRequirements": null,
  "customFields": {
    "referralSource": "LinkedIn",
    "shirtSize": "L"
  },
  "matchingProfile": {
    "industry": "SaaS",
    "seniority": "Senior",
    "interests": ["AI", "Platform Engineering"],
    "goals": ["Networking", "Learning"],
    "optedInToMatching": true
  }
}
```

**Response** `201 Created`:
```json
{
  "id": "reg_abc123",
  "eventId": "evt_abc123",
  "confirmationCode": "EVF-2025-A7X3K",
  "ticketType": {
    "id": "tkt_abc123",
    "name": "General Admission",
    "price": 299.00
  },
  "attendeeEmail": "john.smith@techcorp.com",
  "attendeeFirstName": "John",
  "attendeeLastName": "Smith",
  "status": "Confirmed",
  "paymentStatus": "Paid",
  "amountPaid": 299.00,
  "qrCodeUrl": "https://cdn.eventflow.app/qr/reg_abc123.png",
  "registeredAt": "2025-02-01T14:22:00Z"
}
```

**Status Codes**:
- `409 Conflict`: Event at capacity (triggers waitlist offer if enabled)
- `422 Unprocessable Entity`: Validation errors (field-level)
- `410 Gone`: Ticket sale window has closed

**Controller**:
```csharp
[HttpPost]
[AllowAnonymous]
public async Task<ActionResult<RegistrationDto>> Register(
    Guid eventId, [FromBody] CreateRegistrationRequest request, CancellationToken ct)
{
    var result = await _mediator.Send(new CreateRegistrationCommand(
        eventId, TenantId, request.TicketTypeId,
        request.AttendeeEmail, request.AttendeeFirstName, request.AttendeeLastName,
        request.AttendeePhone, request.AttendeeCompany, request.AttendeeJobTitle,
        request.DietaryRestrictions, request.SpecialRequirements,
        request.CustomFields, request.MatchingProfile), ct);
    return CreatedAtAction(nameof(GetById), new { eventId, registrationId = result.Id }, result);
}
```

---

#### GET `/api/v1/events/{eventId}/registrations`
**Query**: `GetRegistrationsQuery`

**Query Parameters**:
```
?status=Confirmed
?ticketTypeId=tkt_abc123
?search=john+smith
?checkedInOnly=false
?sortBy=registeredAt
?sortDesc=true
?pageSize=50
?cursor=<base64>
```

**Response** `200 OK`:
```json
{
  "data": [
    {
      "id": "reg_abc123",
      "confirmationCode": "EVF-2025-A7X3K",
      "attendeeEmail": "john.smith@techcorp.com",
      "attendeeFirstName": "John",
      "attendeeLastName": "Smith",
      "attendeeCompany": "TechCorp",
      "ticketTypeName": "General Admission",
      "status": "Confirmed",
      "isCheckedIn": false,
      "checkedInAt": null,
      "registeredAt": "2025-02-01T14:22:00Z"
    }
  ],
  "pagination": {
    "pageSize": 50,
    "hasNextPage": true,
    "nextCursor": "eyJpZCI6InJlZ19hYmMxMjMifQ==",
    "totalCount": 280
  },
  "summary": {
    "totalConfirmed": 280,
    "totalCancelled": 12,
    "totalWaitlisted": 34,
    "totalCheckedIn": 0
  }
}
```

---

#### GET `/api/v1/events/{eventId}/registrations/export`
**Query**: `ExportRegistrationsQuery`

**Query Parameters**: `?format=csv&columns=email,firstName,lastName,company,status,checkedIn`

**Response** `200 OK`:
```json
{
  "downloadUrl": "https://cdn.eventflow.app/exports/reg_export_abc123.csv",
  "expiresAt": "2025-03-15T11:00:00Z",
  "recordCount": 280
}
```

---

## 12. Check-In API

### Controller: `CheckInController`
**Route prefix**: `/api/v1/events/{eventId}/checkin`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | POST | `/checkin` | `CheckInAttendeeCommand` | RequireCheckIn | 300/min |
| 2 | DELETE | `/checkin/{checkInRecordId}` | `UndoCheckInCommand` | RequireCheckIn | 60/min |
| 3 | POST | `/checkin/sync` | `SyncOfflineCheckInsCommand` | RequireCheckIn | 10/min |
| 4 | GET | `/checkin/status` | `GetCheckInStatusQuery` | RequireCheckIn | 120/min |
| 5 | GET | `/checkin/lookup` | `GetRegistrationByQrCodeQuery` | RequireCheckIn | 300/min |

---

#### POST `/api/v1/events/{eventId}/checkin`
**Command**: `CheckInAttendeeCommand`

**Request Body**:
```json
{
  "qrCodeData": "EVF-2025-A7X3K",
  "method": "QrScan",
  "sessionId": null,
  "deviceIdentifier": "device_ipad_001"
}
```

**Response** `200 OK`:
```json
{
  "success": true,
  "checkInRecordId": "chk_abc123",
  "attendee": {
    "firstName": "John",
    "lastName": "Smith",
    "company": "TechCorp",
    "ticketType": "General Admission",
    "photoUrl": null
  },
  "checkedInAt": "2025-03-15T09:02:15Z",
  "eventCheckinCount": 143
}
```

**Error Responses**:
```json
// 409 Already checked in
{ "success": false, "errorCode": "AlreadyCheckedIn", "message": "John Smith already checked in at 09:01 AM", "originalCheckInAt": "2025-03-15T09:01:00Z" }

// 404 Not found
{ "success": false, "errorCode": "RegistrationNotFound", "message": "No registration found for this code" }

// 403 Not confirmed
{ "success": false, "errorCode": "RegistrationNotConfirmed", "message": "Registration is pending approval" }
```

**Controller**:
```csharp
[HttpPost]
[Authorize(Policy = "RequireCheckIn")]
public async Task<ActionResult<CheckInResultDto>> CheckIn(
    Guid eventId, [FromBody] CheckInRequest request, CancellationToken ct)
    => Ok(await _mediator.Send(new CheckInAttendeeCommand(
        eventId, TenantId, request.QrCodeData,
        Enum.Parse<CheckInMethod>(request.Method),
        request.SessionId, UserId, request.DeviceIdentifier), ct));
```

---

#### POST `/api/v1/events/{eventId}/checkin/sync`
**Command**: `SyncOfflineCheckInsCommand`

**Request Body**:
```json
{
  "records": [
    {
      "qrCodeData": "EVF-2025-A7X3K",
      "checkedInAt": "2025-03-15T09:02:15Z",
      "deviceIdentifier": "device_ipad_001",
      "sessionId": null
    },
    {
      "qrCodeData": "EVF-2025-B8Y4L",
      "checkedInAt": "2025-03-15T09:02:47Z",
      "deviceIdentifier": "device_ipad_001",
      "sessionId": null
    }
  ]
}
```

**Response** `200 OK`:
```json
{
  "synced": 38,
  "duplicates": 2,
  "errors": 0,
  "errorDetails": []
}
```

---

#### GET `/api/v1/events/{eventId}/checkin/status`
**Query**: `GetCheckInStatusQuery`

**Response** `200 OK`:
```json
{
  "eventId": "evt_abc123",
  "totalRegistrations": 280,
  "checkedInCount": 143,
  "checkedInPercent": 51.1,
  "lastUpdatedAt": "2025-03-15T10:14:00Z",
  "recentCheckIns": [
    {
      "registrationId": "reg_abc123",
      "firstName": "John",
      "lastName": "Smith",
      "company": "TechCorp",
      "checkedInAt": "2025-03-15T10:13:55Z"
    }
  ],
  "sessionBreakdown": [
    {
      "sessionId": "ses_abc123",
      "sessionTitle": "Opening Keynote",
      "checkedInCount": 89
    }
  ]
}
```

---

## 13. Waitlist API

### Controller: `WaitlistController`
**Route prefix**: `/api/v1/events/{eventId}/waitlist`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/waitlist` | `GetWaitlistQuery` | RequireViewer | 60/min |
| 2 | POST | `/waitlist` | `JoinWaitlistCommand` | Public | 10/min/IP |
| 3 | POST | `/waitlist/{entryId}/offer` | `OfferWaitlistSpotCommand` | RequireManager | 30/min |
| 4 | POST | `/waitlist/convert` | `ConvertWaitlistToRegistrationCommand` | Public (with token) | 5/min/IP |

---

#### POST `/api/v1/events/{eventId}/waitlist`
**Command**: `JoinWaitlistCommand`

**Request Body**:
```json
{
  "ticketTypeId": "tkt_abc123",
  "email": "jane.doe@example.com",
  "firstName": "Jane",
  "lastName": "Doe"
}
```

**Response** `201 Created`:
```json
{
  "id": "wl_abc123",
  "position": 7,
  "status": "Waiting",
  "message": "You're #7 on the waitlist. We'll notify you if a spot opens.",
  "joinedAt": "2025-02-15T10:00:00Z"
}
```

---

## 14. Venues API

### Controller: `VenuesController`
**Route prefix**: `/api/v1/venues`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/venues` | `GetVenuesQuery` | RequireViewer | 60/min |
| 2 | POST | `/venues` | `CreateVenueCommand` | RequireManager | 20/min |
| 3 | GET | `/venues/{venueId}` | `GetVenueByIdQuery` | RequireViewer | 120/min |
| 4 | PUT | `/venues/{venueId}` | `UpdateVenueCommand` | RequireManager | 30/min |
| 5 | POST | `/venues/{venueId}/rooms` | `AddVenueRoomCommand` | RequireManager | 20/min |
| 6 | PUT | `/venues/{venueId}/rooms/{roomId}` | `UpdateVenueRoomCommand` | RequireManager | 30/min |
| 7 | POST | `/venues/{venueId}/reviews` | `SubmitVenueReviewCommand` | RequireManager | 10/min |
| 8 | POST | `/venues/{venueId}/photos` | `UploadVenuePhotoCommand` | RequireManager | 10/min |
| 9 | GET | `/venues/{venueId}/availability` | `GetVenueAvailabilityQuery` | RequireViewer | 60/min |

---

#### GET `/api/v1/venues`
**Query**: `GetVenuesQuery`

**Query Parameters**:
```
?type=ConferenceCenter
?city=Austin
?countryCode=US
?minCapacity=100
?maxCapacity=500
?amenities=WiFi,AV,Parking
?search=downtown+austin
?pageSize=25
?cursor=<base64>
```

**Response** `200 OK`:
```json
{
  "data": [
    {
      "id": "ven_xyz789",
      "name": "Austin Convention Center",
      "type": "ConferenceCenter",
      "city": "Austin",
      "stateProvince": "TX",
      "countryCode": "US",
      "totalCapacity": 5000,
      "amenities": ["WiFi", "AV", "Parking", "Catering"],
      "averageRating": 4.6,
      "reviewCount": 12,
      "coverPhotoUrl": "https://cdn.eventflow.app/venues/ven_xyz789/cover.jpg",
      "roomCount": 15
    }
  ],
  "pagination": {
    "pageSize": 25,
    "hasNextPage": false,
    "nextCursor": null,
    "totalCount": 8
  }
}
```

**Controller**:
```csharp
[HttpGet]
[Authorize(Policy = "RequireViewer")]
public async Task<ActionResult<PagedResult<VenueSummaryDto>>> GetAll(
    [FromQuery] GetVenuesRequest request, CancellationToken ct)
    => Ok(await _mediator.Send(new GetVenuesQuery(
        TenantId, request.Type, request.City, request.CountryCode,
        request.MinCapacity, request.MaxCapacity, request.Amenities,
        request.Search, request.PageSize ?? 25, request.Cursor), ct));
```

---

## 15. Sponsors API

### Controller: `SponsorsController`
**Route prefix**: `/api/v1/sponsors` and `/api/v1/events/{eventId}/sponsors`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/sponsors` | `GetSponsorsQuery` | RequireViewer | 60/min |
| 2 | POST | `/sponsors` | `CreateSponsorCommand` | RequireManager | 20/min |
| 3 | GET | `/sponsors/{sponsorId}` | `GetSponsorByIdQuery` | RequireViewer | 120/min |
| 4 | PUT | `/sponsors/{sponsorId}` | `UpdateSponsorCommand` | RequireManager | 30/min |
| 5 | POST | `/sponsors/{sponsorId}/logo` | `UploadSponsorLogoCommand` | RequireManager | 10/min |
| 6 | GET | `/events/{eventId}/sponsor-tiers` | `GetEventSponsorsQuery` | RequireViewer | 120/min |
| 7 | POST | `/events/{eventId}/sponsor-tiers` | `CreateSponsorTierCommand` | RequireManager | 20/min |
| 8 | PUT | `/events/{eventId}/sponsor-tiers/{tierId}` | `UpdateSponsorTierCommand` | RequireManager | 30/min |
| 9 | POST | `/events/{eventId}/sponsor-tiers/reorder` | `ReorderSponsorTiersCommand` | RequireManager | 20/min |
| 10 | POST | `/events/{eventId}/sponsors` | `AddEventSponsorCommand` | RequireManager | 20/min |
| 11 | PUT | `/events/{eventId}/sponsors/{eventSponsorId}` | `UpdateEventSponsorCommand` | RequireManager | 30/min |
| 12 | POST | `/events/{eventId}/sponsors/{eventSponsorId}/leads` | `CaptureSponsorLeadCommand` | RequireViewer | 60/min |
| 13 | GET | `/events/{eventId}/sponsors/{eventSponsorId}/leads` | `GetSponsorLeadsQuery` | RequireViewer | 60/min |

---

#### POST `/api/v1/events/{eventId}/sponsor-tiers/reorder`
**Command**: `ReorderSponsorTiersCommand`

**Request Body**:
```json
{
  "orderedTierIds": [
    "tier_platinum",
    "tier_gold",
    "tier_silver",
    "tier_bronze"
  ]
}
```

**Response** `204 No Content`

**Controller**:
```csharp
[HttpPost("/api/v1/events/{eventId}/sponsor-tiers/reorder")]
[Authorize(Policy = "RequireManager")]
public async Task<IActionResult> Reorder(
    Guid eventId, [FromBody] ReorderSponsorTiersRequest request, CancellationToken ct)
{
    await _mediator.Send(new ReorderSponsorTiersCommand(
        eventId, TenantId, request.OrderedTierIds), ct);
    return NoContent();
}
```

---

## 16. Campaigns API

### Controller: `CampaignsController`
**Route prefix**: `/api/v1/campaigns`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/campaigns` | `GetCampaignsQuery` | RequireViewer | 60/min |
| 2 | POST | `/campaigns` | `CreateCampaignCommand` | RequireManager | 20/min |
| 3 | GET | `/campaigns/{campaignId}` | `GetCampaignByIdQuery` | RequireViewer | 120/min |
| 4 | PUT | `/campaigns/{campaignId}` | `UpdateCampaignCommand` | RequireManager | 30/min |
| 5 | POST | `/campaigns/{campaignId}/schedule` | `ScheduleCampaignCommand` | RequireManager | 20/min |
| 6 | POST | `/campaigns/{campaignId}/send` | `SendCampaignImmediatelyCommand` | RequireManager | 5/min |
| 7 | DELETE | `/campaigns/{campaignId}/schedule` | `CancelScheduledCampaignCommand` | RequireManager | 20/min |
| 8 | POST | `/campaigns/{campaignId}/test` | `SendTestEmailCommand` | RequireManager | 10/min |
| 9 | GET | `/campaigns/recipients-preview` | `GetCampaignRecipientsPreviewQuery` | RequireManager | 30/min |
| 10 | GET | `/campaigns/{campaignId}/analytics` | `GetCampaignAnalyticsQuery` | RequireViewer | 60/min |
| 11 | GET | `/email-templates` | `GetEmailTemplatesQuery` | RequireViewer | 60/min |
| 12 | POST | `/email-templates` | `CreateEmailTemplateCommand` | RequireManager | 10/min |
| 13 | PUT | `/email-templates/{templateId}` | `UpdateEmailTemplateCommand` | RequireManager | 30/min |

---

#### POST `/api/v1/campaigns`
**Command**: `CreateCampaignCommand`

**Request Body**:
```json
{
  "eventId": "evt_abc123",
  "name": "Q4 Kickoff — Last Chance Reminder",
  "subject": "Last chance to register — Q4 Sales Kickoff",
  "type": "Email",
  "previewText": "Only 20 spots remaining...",
  "bodyHtml": "<h1>Don't miss out!</h1><p>{{attendeeFirstName}}, registration closes in 48 hours.</p>",
  "bodyText": "Don't miss out! Registration closes in 48 hours.",
  "filter": {
    "allRegistrants": false,
    "confirmedOnly": false,
    "specificTicketTypeIds": null,
    "tags": null
  }
}
```

**Response** `201 Created`:
```json
{
  "id": "cam_abc123",
  "name": "Q4 Kickoff — Last Chance Reminder",
  "subject": "Last chance to register — Q4 Sales Kickoff",
  "status": "Draft",
  "type": "Email",
  "estimatedRecipientCount": 450,
  "createdAt": "2025-03-01T10:00:00Z"
}
```

**Controller**:
```csharp
[HttpPost]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<CampaignDto>> Create(
    [FromBody] CreateCampaignRequest request, CancellationToken ct)
{
    var result = await _mediator.Send(new CreateCampaignCommand(
        TenantId, request.EventId, request.Name, request.Subject,
        Enum.Parse<CampaignType>(request.Type), request.BodyHtml,
        request.BodyText, request.SmsBody, request.PreviewText,
        request.Filter.ToRecipientFilter(), UserId), ct);
    return CreatedAtAction(nameof(GetById), new { campaignId = result.Id }, result);
}
```

---

#### GET `/api/v1/campaigns/recipients-preview`
**Query**: `GetCampaignRecipientsPreviewQuery`

**Query Parameters**: `?eventId=evt_abc123&confirmedOnly=true`

**Response** `200 OK`:
```json
{
  "estimatedCount": 280,
  "sampleEmails": [
    "john.smith@techcorp.com",
    "jane.doe@startup.io",
    "bob.jones@enterprise.com"
  ]
}
```

---

#### GET `/api/v1/campaigns/{campaignId}/analytics`
**Query**: `GetCampaignAnalyticsQuery`

**Response** `200 OK`:
```json
{
  "campaignId": "cam_abc123",
  "sentAt": "2025-03-10T09:00:00Z",
  "metrics": {
    "totalSent": 450,
    "delivered": 444,
    "opens": 201,
    "uniqueOpens": 189,
    "clicks": 87,
    "uniqueClicks": 74,
    "bounces": 6,
    "unsubscribes": 3,
    "openRate": 42.6,
    "clickRate": 16.7
  }
}
```

---

## 17. Analytics API

### Controller: `AnalyticsController`
**Route prefix**: `/api/v1/analytics`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/analytics/dashboard` | `GetTenantDashboardQuery` | RequireViewer | 30/min |
| 2 | GET | `/analytics/events/{eventId}` | `GetEventAnalyticsQuery` | RequireViewer | 60/min |
| 3 | GET | `/analytics/events/{eventId}/registration-trend` | `GetRegistrationTrendQuery` | RequireViewer | 60/min |
| 4 | GET | `/analytics/events/{eventId}/sessions` | `GetSessionAttendanceQuery` | RequireViewer | 60/min |
| 5 | GET | `/analytics/cross-event` | `GetCrossEventReportQuery` | RequireAdmin | 20/min |
| 6 | GET | `/analytics/events/{eventId}/roi` | `GetROIReportQuery` | RequireAdmin | 20/min |

---

#### GET `/api/v1/analytics/dashboard`
**Query**: `GetTenantDashboardQuery`

**Query Parameters**: `?from=2025-01-01&to=2025-12-31`

**Response** `200 OK` (EventManager role):
```json
{
  "role": "EventManager",
  "period": { "from": "2025-01-01T00:00:00Z", "to": "2025-12-31T00:00:00Z" },
  "stats": {
    "upcomingEventsCount": 7,
    "totalRegistrationsThisMonth": 1240,
    "registrationGrowthPercent": 18.2,
    "openTasksCount": 12,
    "overdueTasksCount": 3,
    "revenueMtd": 24500.00,
    "revenueMtdGrowthPercent": 34.1
  },
  "upcomingEvents": [
    {
      "id": "evt_abc123",
      "name": "Q4 Sales Kickoff 2025",
      "startDate": "2025-03-15T09:00:00Z",
      "city": "Austin, TX",
      "registrationCount": 280,
      "capacity": 350,
      "capacityPercent": 80.0
    }
  ],
  "liveEvents": [
    {
      "id": "evt_live123",
      "name": "DevSummit 2025",
      "checkedInCount": 47,
      "totalRegistrations": 120
    }
  ],
  "recentActivity": [
    {
      "type": "Registration",
      "description": "Sarah K. registered for Q4 Kickoff",
      "occurredAt": "2025-03-15T10:08:00Z"
    }
  ],
  "aiInsights": [
    {
      "id": "ins_abc123",
      "type": "RegistrationLagging",
      "text": "Registration for Q4 Sales Kickoff is tracking 23% below this time last year.",
      "actionText": "Consider sending a reminder campaign",
      "actionUrl": "/campaigns/new?eventId=evt_abc123"
    }
  ]
}
```

**Controller**:
```csharp
[HttpGet("dashboard")]
[Authorize(Policy = "RequireViewer")]
public async Task<ActionResult<TenantDashboardDto>> GetDashboard(
    [FromQuery] DateTimeOffset? from,
    [FromQuery] DateTimeOffset? to,
    CancellationToken ct)
    => Ok(await _mediator.Send(new GetTenantDashboardQuery(
        TenantId, UserRole, from, to), ct));
```

---

#### GET `/api/v1/analytics/events/{eventId}/registration-trend`
**Query**: `GetRegistrationTrendQuery`

**Query Parameters**: `?granularity=Daily`

**Response** `200 OK`:
```json
{
  "eventId": "evt_abc123",
  "granularity": "Daily",
  "dataPoints": [
    { "date": "2025-02-01", "registrations": 12, "cumulative": 12 },
    { "date": "2025-02-02", "registrations": 8, "cumulative": 20 },
    { "date": "2025-02-03", "registrations": 23, "cumulative": 43 }
  ]
}
```

---

#### GET `/api/v1/analytics/cross-event`
**Query**: `GetCrossEventReportQuery`

**Query Parameters**: `?from=2025-01-01&to=2025-12-31&eventIds=evt_abc,evt_def`

**Response** `200 OK`:
```json
{
  "period": { "from": "2025-01-01", "to": "2025-12-31" },
  "totals": {
    "totalEvents": 14,
    "totalRegistrations": 4280,
    "totalRevenue": 187500.00,
    "totalAttendees": 3940,
    "averageAttendanceRate": 92.1,
    "averageNps": 72
  },
  "events": [
    {
      "id": "evt_abc123",
      "name": "Q4 Sales Kickoff 2025",
      "date": "2025-03-15",
      "registrations": 280,
      "attended": 258,
      "revenue": 83720.00,
      "nps": 74,
      "roi": 3.2
    }
  ]
}
```

---

## 18. AI API

### Controller: `AIController`
**Route prefix**: `/api/v1/ai`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | POST | `/ai/events/{eventId}/agenda` | `GenerateAgendaSuggestionCommand` | RequireManager | 10/min (cost-controlled) |
| 2 | GET | `/ai/events/{eventId}/agenda` | `GetAgendaSuggestionsQuery` | RequireViewer | 60/min |
| 3 | POST | `/ai/agenda/{suggestionId}/apply` | `ApplyAgendaSuggestionCommand` | RequireManager | 20/min |
| 4 | DELETE | `/ai/agenda/{suggestionId}` | `DismissAgendaSuggestionCommand` | RequireManager | 30/min |
| 5 | POST | `/ai/events/{eventId}/matchmaking/generate` | `GenerateMatchSuggestionsCommand` | RequireManager | 5/min |
| 6 | GET | `/ai/events/{eventId}/matchmaking` | `GetMatchSuggestionsQuery` | RequireManager | 60/min |
| 7 | POST | `/ai/events/{eventId}/matchmaking/notify` | `SendMatchNotificationsCommand` | RequireManager | 10/min |
| 8 | GET | `/ai/insights` | `GetActiveInsightsQuery` | RequireViewer | 60/min |
| 9 | DELETE | `/ai/insights/{insightId}` | `DismissInsightCommand` | RequireViewer | 30/min |

---

#### POST `/api/v1/ai/events/{eventId}/agenda`
**Command**: `GenerateAgendaSuggestionCommand`

**Request Body**:
```json
{
  "promptText": "Two-day SaaS product conference for 300 enterprise software professionals in Austin, TX. Mix of keynotes, technical workshops, and networking. Focus on AI, platform engineering, and go-to-market."
}
```

**Response** `202 Accepted` (async — streams result via SSE or polls):
```json
{
  "suggestionId": "sug_abc123",
  "status": "Generating",
  "estimatedSeconds": 15,
  "pollUrl": "/api/v1/ai/agenda/sug_abc123"
}
```

**Polling Response** (GET `/api/v1/ai/agenda/{suggestionId}`) `200 OK` when complete:
```json
{
  "id": "sug_abc123",
  "status": "Generated",
  "confidence": "High",
  "reasoningText": "Based on 847 similar two-day SaaS conferences, I structured Day 1 around thought leadership and Day 2 around hands-on workshops. I included a mid-morning networking break after reviewing attendee satisfaction data from similar events.",
  "suggestedSessions": [
    {
      "title": "Opening Keynote: The AI-Native Enterprise",
      "type": "Keynote",
      "suggestedStartTime": "2025-03-15T09:00:00Z",
      "suggestedEndTime": "2025-03-15T10:00:00Z",
      "suggestedTrack": "Main Stage",
      "aiRationale": "High-energy opening to set conference tone"
    },
    {
      "title": "Networking Break",
      "type": "Networking",
      "suggestedStartTime": "2025-03-15T10:00:00Z",
      "suggestedEndTime": "2025-03-15T10:30:00Z",
      "suggestedTrack": "All",
      "aiRationale": "Mid-morning breaks improve satisfaction scores by 23% in similar events"
    }
  ],
  "generatedAt": "2025-01-10T08:00:15Z",
  "tokensUsed": 1247
}
```

**Controller**:
```csharp
[HttpPost("events/{eventId}/agenda")]
[Authorize(Policy = "RequireManager")]
public async Task<ActionResult<AISuggestionInitiatedDto>> GenerateAgenda(
    Guid eventId, [FromBody] GenerateAgendaRequest request, CancellationToken ct)
    => Accepted(await _mediator.Send(new GenerateAgendaSuggestionCommand(
        eventId, TenantId, request.PromptText, UserId), ct));
```

---

#### POST `/api/v1/ai/agenda/{suggestionId}/apply`
**Command**: `ApplyAgendaSuggestionCommand`

**Request Body**:
```json
{
  "replaceExistingSessions": false
}
```

**Response** `200 OK`:
```json
{
  "sessionsCreated": 12,
  "sessionsSkipped": 0,
  "message": "12 sessions have been added to your agenda. Review and adjust as needed."
}
```

---

#### POST `/api/v1/ai/events/{eventId}/matchmaking/generate`
**Command**: `GenerateMatchSuggestionsCommand`

**Request Body**:
```json
{
  "maxSuggestionsPerAttendee": 5
}
```

**Response** `202 Accepted`:
```json
{
  "jobId": "match_job_abc123",
  "status": "Processing",
  "attendeesAnalyzed": 280,
  "estimatedSeconds": 45
}
```

---

#### GET `/api/v1/ai/insights`
**Query**: `GetActiveInsightsQuery`

**Query Parameters**: `?eventId=evt_abc123&type=RegistrationLagging`

**Response** `200 OK`:
```json
{
  "insights": [
    {
      "id": "ins_abc123",
      "type": "RegistrationLagging",
      "eventId": "evt_abc123",
      "eventName": "Q4 Sales Kickoff 2025",
      "text": "Registration for Q4 Sales Kickoff is tracking 23% below this time last year.",
      "actionText": "Consider sending a reminder campaign",
      "actionUrl": "/campaigns/new?eventId=evt_abc123",
      "confidence": "High",
      "generatedAt": "2025-03-01T08:00:00Z",
      "expiresAt": "2025-03-15T00:00:00Z"
    }
  ]
}
```

**Controller**:
```csharp
[HttpGet("insights")]
[Authorize(Policy = "RequireViewer")]
public async Task<ActionResult<ActiveInsightsDto>> GetInsights(
    [FromQuery] Guid? eventId,
    [FromQuery] string? type,
    CancellationToken ct)
    => Ok(await _mediator.Send(new GetActiveInsightsQuery(
        TenantId, eventId,
        type != null ? Enum.Parse<InsightType>(type) : null), ct));
```

---

## 19. Search API

### Controller: `SearchController`
**Route prefix**: `/api/v1/search`

| # | Method | Route | Command/Query | Auth | Rate Limit |
|---|--------|-------|---------------|------|------------|
| 1 | GET | `/search` | `GlobalSearchQuery` | RequireViewer | 120/min |

---

#### GET `/api/v1/search`
**Query**: `GlobalSearchQuery`

**Query Parameters**: `?q=sarah+kim&maxResults=20`

**Response** `200 OK`:
```json
{
  "query": "sarah kim",
  "results": {
    "events": [
      {
        "id": "evt_abc123",
        "name": "Q4 Sales Kickoff 2025",
        "status": "Published",
        "startDate": "2025-03-15",
        "url": "/events/evt_abc123"
      }
    ],
    "attendees": [
      {
        "id": "reg_abc123",
        "name": "Sarah Kim",
        "email": "sarah.kim@techcorp.com",
        "eventName": "Q4 Sales Kickoff 2025",
        "url": "/events/evt_abc123/registrations/reg_abc123"
      }
    ],
    "speakers": [
      {
        "id": "spk_abc",
        "name": "Dr. Sarah Kim",
        "company": "Stanford University",
        "photoUrl": "https://cdn.eventflow.app/speakers/spk_abc.jpg",
        "url": "/speakers/spk_abc"
      }
    ],
    "venues": []
  },
  "totalCount": 3
}
```

**Controller**:
```csharp
[HttpGet]
[Authorize(Policy = "RequireViewer")]
public async Task<ActionResult<GlobalSearchResultDto>> Search(
    [FromQuery] string q,
    [FromQuery] int maxResults = 20,
    CancellationToken ct = default)
    => Ok(await _mediator.Send(new GlobalSearchQuery(TenantId, q, maxResults), ct));
```

---

## 20. Real-Time API

### 20.1 Server-Sent Events (SSE)

Used for: live check-in counter, activity feed, AI generation progress.

```
GET /api/v1/events/{eventId}/stream
    Authorization: Bearer <jwt>
    Accept: text/event-stream
```

**Events Emitted**:
```
event: checkin.updated
data: {"checkedInCount": 143, "totalRegistrations": 280, "lastCheckIn": {"name": "John Smith", "checkedInAt": "2025-03-15T10:14:00Z"}}

event: registration.created
data: {"registrationId": "reg_xyz", "attendeeName": "Jane Doe", "ticketType": "General Admission"}

event: ai.suggestion.ready
data: {"suggestionId": "sug_abc123", "status": "Generated", "confidence": "High"}

event: heartbeat
data: {"timestamp": "2025-03-15T10:14:30Z"}
```

**Implementation**:
- ASP.NET Core `Response.ContentType = "text/event-stream"` endpoint
- Backed by Redis Streams subscriber — each tenant has its own stream key
- Client reconnects automatically on disconnect (SSE native behavior)
- Heartbeat every 30 seconds to keep connections alive

### 20.2 WebSocket (Real-Time Collaboration)

Used for: concurrent multi-user editing (v2 feature), presence indicators.

```
WS /api/v1/events/{eventId}/collaborate
    Authorization: Bearer <jwt> (in query param for WS: ?token=<jwt>)
```

**Message Types**:
```json
// Client → Server: field update
{"type": "field.update", "entity": "session", "entityId": "ses_abc123", "field": "title", "value": "New Title", "clientId": "user_abc_tab_1"}

// Server → Client: broadcast update to other connected clients
{"type": "field.updated", "entity": "session", "entityId": "ses_abc123", "field": "title", "value": "New Title", "updatedBy": {"userId": "usr_abc", "name": "Maya Chen"}}

// Server → Client: presence
{"type": "presence.update", "activeUsers": [{"userId": "usr_abc", "name": "Maya Chen", "color": "#6366F1", "currentView": "/events/evt_abc123/sessions"}]}

// Server → Client: conflict
{"type": "field.conflict", "entity": "session", "entityId": "ses_abc123", "field": "title", "yourValue": "My Title", "serverValue": "Their Title", "updatedBy": "David"}
```

---

## 21. Error Response Standard

All error responses follow RFC 7807 (Problem Details for HTTP APIs):

```json
// 400 Bad Request — malformed input
{
  "type": "https://api.eventflow.app/errors/bad-request",
  "title": "Bad Request",
  "status": 400,
  "traceId": "00-abc123-def456-00"
}

// 401 Unauthorized
{
  "type": "https://api.eventflow.app/errors/unauthorized",
  "title": "Authentication required",
  "status": 401,
  "traceId": "00-abc123-def456-00"
}

// 403 Forbidden
{
  "type": "https://api.eventflow.app/errors/forbidden",
  "title": "Insufficient permissions",
  "status": 403,
  "detail": "Role 'Viewer' cannot perform 'PublishEvent'. Required: 'EventManager'.",
  "traceId": "00-abc123-def456-00"
}

// 404 Not Found
{
  "type": "https://api.eventflow.app/errors/not-found",
  "title": "Resource not found",
  "status": 404,
  "detail": "Event 'evt_abc123' was not found or you do not have access to it.",
  "traceId": "00-abc123-def456-00"
}

// 409 Conflict
{
  "type": "https://api.eventflow.app/errors/conflict",
  "title": "Conflict",
  "status": 409,
  "detail": "This event is already published.",
  "traceId": "00-abc123-def456-00"
}

// 422 Unprocessable Entity — FluentValidation failures
{
  "type": "https://api.eventflow.app/errors/validation",
  "title": "Validation failed",
  "status": 422,
  "errors": {
    "name": ["Event name is required", "Event name cannot exceed 200 characters"],
    "startDateUtc": ["Start date must be in the future"],
    "endDateUtc": ["End date must be after start date"]
  },
  "traceId": "00-abc123-def456-00"
}

// 429 Too Many Requests
{
  "type": "https://api.eventflow.app/errors/rate-limited",
  "title": "Rate limit exceeded",
  "status": 429,
  "detail": "You have exceeded 20 requests per minute for this endpoint.",
  "retryAfter": 47,
  "traceId": "00-abc123-def456-00"
}

// 503 Service Unavailable (feature flag off)
{
  "type": "https://api.eventflow.app/errors/feature-unavailable",
  "title": "Feature not available",
  "status": 503,
  "detail": "AI Agenda Generation is not enabled for your plan. Upgrade to Growth to access this feature.",
  "featureFlag": "ai-agenda-generation",
  "traceId": "00-abc123-def456-00"
}
```

**Exception → HTTP Mapping** (in `ExceptionHandlingMiddleware`):
```csharp
NotFoundException         → 404
ForbiddenException       → 403
DomainException          → 409
ValidationException      → 422 (with field-level errors)
FeatureDisabledException → 503
UnauthorizedException    → 401
Default (unhandled)      → 500 (traceId only, no internal details exposed)
```

---

## 22. Rate Limiting Rules

Implemented via ASP.NET Core `RateLimiter` middleware with Redis backing (for distributed enforcement across pod replicas).

### Rate Limit Policies

| Policy Name | Limit | Window | Scope | Applied To |
|---|---|---|---|---|
| `PublicStrict` | 5 | per minute | per IP | Tenant creation, signup |
| `PublicModerate` | 20 | per minute | per IP | Public registration, waitlist join |
| `PublicRelaxed` | 300 | per minute | per IP | Public event pages, theme endpoint |
| `AuthenticatedDefault` | 120 | per minute | per user+tenant | Standard read endpoints |
| `AuthenticatedWrite` | 60 | per minute | per user+tenant | Standard write endpoints |
| `WriteHeavy` | 30 | per minute | per user+tenant | Event create, update operations |
| `BulkOperations` | 5 | per minute | per tenant | Bulk import, campaign send |
| `AIGeneration` | 10 | per minute | per tenant | AI agenda, matchmaking (cost-limited) |
| `FileUpload` | 10 | per minute | per user+tenant | All file upload endpoints |
| `Export` | 10 | per minute | per user+tenant | Export endpoints |
| `CheckIn` | 300 | per minute | per user+tenant | Check-in operations (high volume at event) |

### Rate Limit Headers
Every response includes:
```http
X-RateLimit-Policy: AuthenticatedDefault
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1710000000
Retry-After: 13   (only on 429 responses)
```

### Special Cases

**AI Endpoints** — additionally gated by `CostTrackingBehavior`:
- Each tenant has a monthly AI token budget based on their plan
- Budget enforced in `CostTrackingBehavior` before calling the AI service
- If budget exceeded: `429` with `detail: "Monthly AI token budget exhausted. Resets on [date]."`

**Check-In Endpoints** — elevated limits:
- Check-in is time-sensitive (500 attendees checking in simultaneously)
- `RequireCheckIn` role gets 300/min per user — each check-in staff member has independent limit
- QR scan endpoint additionally has per-device-identifier deduplication for offline sync

**Webhook Ingress** (for Stripe, email provider callbacks):
- IP allowlist validation before rate limiter
- No user-based rate limiting (source IP must be in allowlist)
