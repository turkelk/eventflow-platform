# Wave 13: Attendee & Speaker Experience

## 13.1 Implement attendee self-service portal: personal agenda, session feedback, profile update

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:l`, `agent:backend`
**Status:** Pending

Implement the authenticated attendee experience per User Stories: Attendee persona and API Specification `/api/v1/attendee`.

**Key files:**
- `src/EventFlow.Application/Registrations/Commands/RegisterForSessionCommand.cs` (record + handler)
- `src/EventFlow.Application/Registrations/Queries/GetRegistrationByIdQuery.cs` (attendee's own registration)
- `src/EventFlow.Application/Common/Commands/UpdateAttendeeProfileCommand.cs`
- `src/EventFlow.Api/Controllers/AttendeeController.cs` (attendee-scoped, JWT required)
- `frontend/src/pages/public/RegistrationPage.tsx` (post-registration attendee portal link)

**Acceptance criteria:**
- [ ] Attendee can view own registration with QR code via GET `/api/v1/attendee/events/{id}/registration`
- [ ] Personal agenda: add/remove sessions, check for conflicts before adding
- [ ] Session feedback: POST `/api/v1/attendee/events/{id}/sessions/{sessionId}/feedback` with rating + comment
- [ ] Profile update includes `networking_opt_in`, `interests`, `goals` for matchmaking
- [ ] Attendee portal accessible without full organizer account (attendee Keycloak role)

---

## 13.2 Implement attendee registration cancellation, transfer, and self-service refund flow

**Labels:** `type:feature`, `layer:backend`, `priority:medium`, `size:m`, `agent:backend`
**Status:** Pending

Implement attendee self-service registration management per User Stories and API Specification public endpoints.

**Key files:**
- `src/EventFlow.Application/Registrations/Commands/CancelRegistrationCommand.cs` (attendee-initiated path)
- `src/EventFlow.Application/Registrations/Commands/TransferRegistrationCommand.cs`
- `src/EventFlow.Api/Controllers/PublicRegistrationsController.cs` (public endpoints with token validation)
- `frontend/src/pages/public/RegistrationPage.tsx` (cancel/transfer links in confirmation email)

**Acceptance criteria:**
- [ ] POST `/public/registrations/{registrationNumber}/cancel` validates cancellation token from email
- [ ] Cancellation policy enforced: refund eligibility based on event `registration_closes_at`
- [ ] Transfer changes attendee name/email, generates new QR code, sends confirmation to new attendee
- [ ] `RegistrationCancelledEvent` published → email consumer notifies original attendee
- [ ] Waitlist promotion triggered automatically when capacity freed by cancellation

---

## 13.3 Build attendee networking match UI and speaker portal frontend

**Labels:** `type:feature`, `layer:frontend`, `priority:medium`, `size:m`, `agent:frontend`
**Status:** Pending

Implement attendee-facing networking and speaker self-service UX per User Stories.

**Key files:**
- `frontend/src/pages/public/SpeakerPortalPage.tsx` (magic link access, bio/headshot update, session view)
- `frontend/src/features/ai/components/MatchSuggestionCard.tsx` (accept/dismiss actions)
- `frontend/src/pages/attendees/AttendeeNetworkingPage.tsx` (attendee matchmaking view)

**Acceptance criteria:**
- [ ] Speaker portal shows assigned sessions, room, time slot, and allows bio/headshot update
- [ ] Speaker invitation acceptance via magic link token (no Keycloak account required for portal)
- [ ] Attendee match suggestions show `matchReasons` array in plain English
- [ ] Accept match sends notification email to both attendees
- [ ] Match suggestions only visible to attendees with `networking_opt_in = true`

---

