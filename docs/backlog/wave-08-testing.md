# Wave 8: Testing

## 8.1 Setup test infrastructure: TestWebApplicationFactory, Testcontainers, MSW, test data builders

**Labels:** `type:chore`, `layer:testing`, `priority:high`, `size:m`, `agent:test-backend`
**Status:** Pending

Create the shared test infrastructure used by all backend and frontend test suites.

**Key files:**
- `tests/EventFlow.IntegrationTests/Infrastructure/TestWebApplicationFactory.cs` (Testcontainers: PostgreSQL + Redis)
- `tests/EventFlow.IntegrationTests/Infrastructure/DatabaseFixture.cs`
- `tests/EventFlow.IntegrationTests/Infrastructure/SeedData.cs`
- `tests/EventFlow.UnitTests/Builders/` (EventBuilder, RegistrationBuilder, TenantBuilder test data)
- `frontend/src/tests/mocks/handlers.ts` (MSW handlers for all API endpoints)
- `frontend/src/tests/setup.ts` (MSW server setup, vi.setup)

**Acceptance criteria:**
- [ ] `TestWebApplicationFactory` spins up real PostgreSQL and Redis via Testcontainers
- [ ] Each test class gets fresh DB via transaction rollback strategy
- [ ] Test data builders use fluent API: `new EventBuilder().WithPublishedStatus().WithCapacity(100).Build()`
- [ ] MSW handlers mock all 106 API endpoints with realistic response shapes
- [ ] Testcontainers reuse strategy enabled (parallel test classes share infrastructure)

---

## 8.2 Write unit tests for all MediatR handlers and pipeline behaviors

**Labels:** `type:chore`, `layer:testing`, `priority:high`, `size:l`, `agent:test-backend`
**Status:** Pending

Implement unit tests for all command/query handlers and behaviors per NFR section 6.1.

**Key files:**
- `tests/EventFlow.UnitTests/Features/Events/CreateEventHandlerTests.cs`
- `tests/EventFlow.UnitTests/Features/Events/PublishEventHandlerTests.cs`
- `tests/EventFlow.UnitTests/Features/Registrations/CreateRegistrationHandlerTests.cs`
- `tests/EventFlow.UnitTests/Features/CheckIn/CheckInAttendeeHandlerTests.cs`
- `tests/EventFlow.UnitTests/Behaviors/ValidationBehaviorTests.cs`
- `tests/EventFlow.UnitTests/Behaviors/CacheBehaviorTests.cs`
- `tests/EventFlow.UnitTests/Behaviors/FeatureFlagBehaviorTests.cs`

**Coverage:** Happy path, validation failure, auth/ownership failure, tenant isolation, domain event publication

**Acceptance criteria:**
- [ ] 90% line coverage, 85% branch coverage enforced in CI
- [ ] Every handler test covers: success, `NotFoundException`, `ForbiddenException`, `ValidationException`
- [ ] `TenantIsolationTests`: cross-tenant access returns 404, never returns other tenant's data
- [ ] `CacheBehavior` tests: cache hit skips handler, cache miss calls handler and populates cache
- [ ] All tests use Moq for dependencies — no real DB or Redis connections

---

## 8.3 Write integration tests for all API controllers and middleware chain

**Labels:** `type:chore`, `layer:testing`, `priority:high`, `size:l`, `agent:test-backend`
**Status:** Pending

Implement integration tests for all 106 API endpoints using TestWebApplicationFactory.

**Key files:**
- `tests/EventFlow.IntegrationTests/Api/EventsApiTests.cs`
- `tests/EventFlow.IntegrationTests/Api/RegistrationsApiTests.cs`
- `tests/EventFlow.IntegrationTests/Api/CheckInApiTests.cs`
- `tests/EventFlow.IntegrationTests/Api/SearchApiTests.cs`
- `tests/EventFlow.IntegrationTests/Features/RegistrationFlowTests.cs` (register→email→check-in pipeline)
- `tests/EventFlow.IntegrationTests/Features/TenantIsolationTests.cs`

**Test per endpoint:** correct status code, auth required (401 without JWT), role required (403 wrong role), response shape matches DTO

**Acceptance criteria:**
- [ ] `TenantIsolationTests`: GET event from wrong tenant JWT returns 404
- [ ] `RegistrationFlowTests`: full pipeline (create→confirm→check-in→analytics updated) in single test
- [ ] All POST endpoints with duplicate `Idempotency-Key` return cached response without side effects
- [ ] Rate limiting middleware tested: 429 returned when limit exceeded
- [ ] WebSocket hub integration test: check-in triggers SignalR message to subscribed clients

---

## 8.4 Write frontend component and hook tests with Vitest and React Testing Library

**Labels:** `type:chore`, `layer:testing`, `priority:high`, `size:l`, `agent:test-frontend`
**Status:** Pending

Implement frontend unit and component tests per 04-FRONTEND-ARCHITECTURE.md section 15.

**Key files:**
- `frontend/src/features/events/components/__tests__/EventCard.test.tsx`
- `frontend/src/features/checkin/hooks/__tests__/useOfflineCheckIn.test.ts`
- `frontend/src/shared/components/__tests__/InlineEditField.test.tsx`
- `frontend/src/features/sessions/components/__tests__/SessionScheduler.test.tsx`
- `frontend/src/shared/components/__tests__/CommandPalette.test.tsx`
- `frontend/src/features/ai/components/__tests__/AISuggestionCard.test.tsx`

**Acceptance criteria:**
- [ ] 85% line coverage enforced in CI via Vitest `--coverage`
- [ ] All form submissions tested with `userEvent` (not `fireEvent`)
- [ ] `InlineEditField` test: click → edit mode, blur → save called, Escape → cancel, no save
- [ ] `useOfflineCheckIn` test: offline storage, reconnect sync, deduplication
- [ ] Accessibility queries used (`getByRole`, `getByLabelText`) — no `getByTestId` in component tests

---

## 8.5 Write Playwright E2E tests for all critical user journeys

**Labels:** `type:chore`, `layer:testing`, `priority:high`, `size:l`, `agent:test-e2e`
**Status:** Pending

Implement E2E tests per 04-FRONTEND-ARCHITECTURE.md section 15 covering all 8 critical journeys.

**Key files:**
- `e2e/onboarding.spec.ts` (sign up → workspace → first event published, < 10 min)
- `e2e/events.spec.ts` (create → configure → publish → check-in → analytics)
- `e2e/checkin.spec.ts` (QR scan, offline mode via `route.abort()`, reconnect sync)
- `e2e/ai-agenda.spec.ts` (prompt → stream → apply → undo)
- `e2e/registration.spec.ts` (public page → register → confirmation email)
- `e2e/team-invite.spec.ts` (invite → accept → role-gated access verified)
- `e2e/campaign.spec.ts` (create → preview → send → metrics)
- `e2e/playwright.config.ts` (Chromium + Firefox + WebKit, mobile viewports, retry:2)

**Acceptance criteria:**
- [ ] All 8 critical journeys pass on Chromium, Firefox, and WebKit
- [ ] Mobile tests run on iPhone 13 (375px) and Pixel 5 (393px) viewports
- [ ] Offline check-in test uses `page.route('**/api/**', route => route.abort())`
- [ ] Screenshot captured on every test failure as CI artifact
- [ ] All E2E tests complete in under 20 minutes with 4 parallel workers

---

## 8.6 Configure SAST, SCA, and DAST security scanning in CI pipeline

**Labels:** `type:security`, `layer:testing`, `priority:high`, `size:m`, `agent:test-e2e`
**Status:** Pending

Implement security scanning pipeline per NFR section 6.3.

**Key files:**
- `.semgrep/rules.yml` (no SQL interpolation, no shell exec, no localStorage tokens, no dangerouslySetInnerHTML, SSRF patterns)
- `.github/workflows/security.yml` (Semgrep, Snyk .NET, Snyk npm, OWASP ZAP)
- `.zap/rules.tsv` (ZAP alert filter rules)
- `frontend/.eslintrc.json` (no-danger rule, no localStorage for tokens)

**Thresholds:** Semgrep HIGH → block PR; Snyk HIGH/CRITICAL with fix → block PR; ZAP HIGH → block nightly

**Acceptance criteria:**
- [ ] Semgrep runs on every PR with custom EventFlow rules (SQL injection, SSRF, secret detection)
- [ ] Snyk SCA scans .NET NuGet + npm packages; blocks PR on HIGH+ CVE with available fix
- [ ] OWASP ZAP active scan runs nightly against staging environment
- [ ] SARIF reports uploaded to GitHub Security tab for all scanners
- [ ] Container image scan (Snyk) runs on every Docker build in CI

---

## 8.7 Build attendee post-event session feedback and event ratings UI

**Labels:** `type:feature`, `layer:fullstack`, `priority:high`, `size:l`, `agent:fullstack`
**Status:** Pending

## Goal
Allow attendees to submit session feedback and overall event ratings after an event completes.

## Acceptance Criteria
- Attendee portal shows feedback form after event status = `Completed`
- Per-session rating (1–5 stars) + optional comment
- Overall event NPS score (0–10)
- Submitted feedback stored and visible in organizer analytics
- `POST /api/v1/attendee/events/{eventId}/sessions/{sessionId}/feedback` endpoint

## Technical Notes
- New `SessionFeedback` entity: `registrationId`, `sessionId`, `rating`, `comment`, `submittedAt`
- Frontend: feedback card list in attendee portal, one submission per session per registration
- Aggregate ratings exposed via `GetSessionAttendanceQuery`

---

## 8.8 Implement AttendeeMatch entity and attendee networking match UI

**Labels:** `type:feature`, `layer:fullstack`, `priority:medium`, `size:l`, `agent:fullstack`
**Status:** Pending

## Goal
Persist AI-generated attendee match suggestions and allow attendees to accept or dismiss matches via the attendee portal.

## Acceptance Criteria
- `AttendeeMatch` entity maps to existing `MatchSuggestion` aggregate; attendee-facing status: `Suggested`, `Accepted`, `Dismissed`
- `GET /api/v1/attendee/events/{eventId}/matches` returns matches for logged-in attendee
- `POST /api/v1/attendee/events/{eventId}/matches/{matchId}/accept|dismiss` updates status
- Frontend: match card with name, company, match reasons, accept/dismiss actions
- Accepted match triggers introduction email to both parties

## Technical Notes
- Attendee auth via Keycloak (attendee role or registration confirmation token)
- Email introduction sent via `IEmailService` on accept
- Extends attendee self-service portal page

---

## 8.9 Implement attendee-facing API endpoints for session registration and profile update

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

## Goal
Authenticated attendee portal API: view registered events, manage session sign-ups, and update profile.

## Acceptance Criteria
- `GET /api/v1/attendee/events` lists events the attendee is registered for
- `GET /api/v1/attendee/events/{eventId}/registration` returns registration detail
- `GET /api/v1/attendee/events/{eventId}/sessions` returns schedule with registration status
- `POST /api/v1/attendee/events/{eventId}/sessions/{sessionId}/register` registers for a session
- `DELETE /api/v1/attendee/events/{eventId}/sessions/{sessionId}/register` cancels session registration
- `PUT /api/v1/attendee/profile` updates name, company, job title, matching profile

## Technical Notes
- Attendee identity resolved from JWT `sub` claim matched to `Registration.AttendeeEmail`
- Session registration checks `CapacityOverride` and creates `SessionRegistration` record
- Profile update propagates to all active registrations for this attendee

---

