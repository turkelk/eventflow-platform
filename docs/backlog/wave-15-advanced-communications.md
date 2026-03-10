# Wave 15: Advanced Communications

## 15.1 Implement SMS campaign support via Twilio and push notification infrastructure

**Labels:** `type:feature`, `layer:backend`, `priority:low`, `size:m`, `agent:backend`
**Status:** Pending

Implement SMS and push notification channels per Domain Model `Campaign.CampaignType` enum (Email, SMS, Push).

**Key files:**
- `src/EventFlow.Application/Common/Interfaces/ISmsService.cs`
- `src/EventFlow.Infrastructure/Adapters/SmsService.cs` (Twilio SDK, thin wrapper — one SDK call per method)
- `src/EventFlow.Application/Campaigns/Commands/SendCampaignImmediatelyCommand.cs` (SMS branch)
- `src/EventFlow.Infrastructure/BackgroundServices/EventStreamConsumer.cs` (SMS consumer group)

**Acceptance criteria:**
- [ ] `ISmsService.SendSmsAsync(to, body)` — one Twilio API call, no orchestration logic
- [ ] SMS campaigns use `sms_body` field from `Campaign` entity
- [ ] SMS delivery status tracked via Twilio status callback webhook
- [ ] `CampaignType.SMS` restricted to plans with SMS feature flag enabled
- [ ] Phone number format validated (E.164) in `CreateCampaignValidator`

---

## 15.2 Build venue discovery UI with search, filters, map view, and inquiry flow

**Labels:** `type:feature`, `layer:frontend`, `priority:medium`, `size:l`, `agent:frontend`
**Status:** Pending

Implement the venue management frontend per API Specification section 14 and User Stories (Venue Manager persona).

**Key files:**
- `frontend/src/pages/venues/VenuesPage.tsx` (search, filters, card grid)
- `frontend/src/pages/venues/VenueDetailPage.tsx` (detail with rooms, photos, availability calendar)
- `frontend/src/features/venues/components/VenueCard.tsx`
- `frontend/src/features/venues/components/VenueSearchFilters.tsx`
- `frontend/src/features/venues/components/VenueMapView.tsx` (lazy loaded)
- `frontend/src/features/venues/hooks/useVenues.ts`

**Acceptance criteria:**
- [ ] Venue search filters: type, city, capacity range, amenities (multi-select), keyword
- [ ] Availability calendar shows dates booked by events using the venue
- [ ] Inquiry form sends POST `/api/v1/venues/{id}/inquiries` with event date and attendee count
- [ ] `VenueMapView` lazy-loaded (heavy dependency), shows venue pin on map
- [ ] Venue reviews shown with star ratings and per-dimension scores (AV, Catering, Staff)

---

## 15.3 Implement real-time collaboration multiplayer presence and live field updates

**Labels:** `type:feature`, `layer:frontend`, `priority:medium`, `size:l`, `agent:frontend`
**Status:** Pending

Implement multi-user real-time collaboration per 01-UX-RESEARCH.md competitive gap analysis and v2 must-have list.

**Key files:**
- `src/EventFlow.Api/Hubs/EventActivityHub.cs` (presence join/leave, field update broadcast)
- `frontend/src/shared/components/PresenceIndicator.tsx` (multiplayer avatar stack)
- `frontend/src/shared/hooks/useInlineEdit.ts` (conflict detection on concurrent edits)
- `frontend/src/app/providers/WebSocketProvider.tsx` (presence subscription)

**WS messages:** `presence.joined`, `presence.left`, `field.updated`, `field.conflict`

**Acceptance criteria:**
- [ ] `PresenceIndicator` shows colored avatar circles for all users currently viewing the event
- [ ] Field edit by User A broadcasts `field.updated` to all other connected clients within 50ms
- [ ] Conflict detection: if User B has modified the same field since User A loaded it, show conflict modal
- [ ] Presence group scoped to `eventId` — no cross-event presence leakage
- [ ] Presence updates disconnection after 30s idle or browser close

---

## 15.4 Implement session registration by attendees and capacity-controlled breakout sessions

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement session-level registration for capacity-controlled breakout sessions per User Stories (Attendee: personal agenda).

**Key files:**
- `src/EventFlow.Application/Registrations/Commands/RegisterForSessionCommand.cs` (record + handler + validator)
- `src/EventFlow.Application/Registrations/Queries/GetCheckInStatusQuery.cs` (session-level breakdown)
- `src/EventFlow.Api/Controllers/AttendeeController.cs` (session register/deregister endpoints)

**Acceptance criteria:**
- [ ] `RegisterForSessionCommand` respects `Session.max_capacity` with distributed lock
- [ ] Session with `max_capacity = null` allows unlimited session registrations
- [ ] `GetCheckInStatusQuery` includes `sessionBreakdown` with per-session checked-in count
- [ ] Session attendance shown in event analytics `GetSessionAttendanceQuery`
- [ ] `SessionRegistration.feedback_rating` and `feedback_comment` stored post-event

---

