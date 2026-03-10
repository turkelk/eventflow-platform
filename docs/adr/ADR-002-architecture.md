# ADR-002: Architecture — Modular Monolith with Vertical Slice Architecture

**Status**: Accepted  
**Date**: 2025-01-01  
**Deciders**: Architecture Team  
**Consulted**: Product, Engineering Leads  

---

## Context

EventFlow is a greenfield SaaS platform targeting mid-market organizations. The team is building from scratch with a target of reaching beta within 4–6 months. We must choose between:

1. **Microservices**: Independent deployable services per bounded context.
2. **Modular Monolith**: Single deployable unit with clearly delineated internal modules, each with its own feature folders.

### Evaluation Criteria

**Against Microservices at this stage:**
- **Team size**: Small founding engineering team (estimated 4–8 engineers). Microservices require distributed systems expertise, independent CI/CD per service, service mesh configuration, and inter-service contract management. The overhead is disproportionate.
- **Domain maturity**: The bounded contexts are not yet proven. Splitting prematurely along the wrong boundaries creates distributed monolith anti-patterns — tightly coupled services that deploy together anyway. The right time to extract a service is when you feel the pain of the boundary in a monolith.
- **Operational complexity**: Microservices require Kubernetes service mesh (Istio/Linkerd), distributed tracing, inter-service auth, API gateway routing, and independent database schemas. This is significant infrastructure investment before a single line of product code.
- **No independent scaling justification yet**: The only candidate for independent scaling would be the real-time check-in feature (event-day load spike). This is better handled by horizontal pod scaling of the monolith than extracting a microservice.
- **No different deployment cadence required**: All features (registration, sessions, AI, analytics) ship together in the same release cycle at this stage.

**For Modular Monolith:**
- **Speed of development**: Feature development crosses module boundaries frequently (e.g., creating a Registration triggers Email, affects Analytics, updates Check-in capacity). In a monolith, this is a single transaction. In microservices, it's distributed saga choreography.
- **Refactoring to microservices is possible later**: A well-structured modular monolith with clear interface boundaries can be extracted into services when genuinely warranted. The modules become service candidates when the time comes.
- **Vertical slice architecture**: Each feature is self-contained (Command + Handler + Validator + DTO in one folder). This enables team members to work on independent features without merge conflicts and makes the codebase navigable without knowing the entire system.

### Bounded Context Identification

For future microservice extraction planning, the domain has these candidate bounded contexts:

| Context | Current Approach | Future Extraction Trigger |
|---|---|---|
| **Identity & Tenancy** | Delegated to Keycloak | Already external — Keycloak IS the identity service |
| **Event Management** | Core module | Extract if event creation pipeline needs independent scaling |
| **Attendee & Registration** | Core module | Extract if registration load (event-day spikes) requires independent scaling |
| **AI & Recommendations** | Module backed by external AI API | Extract if AI processing becomes async-heavy and needs independent worker scaling |
| **Communications** | Module | Extract if email/SMS volume requires independent queue processing at scale |
| **Analytics** | Module | Extract if analytics queries impact OLTP performance (consider read replica first) |
| **Check-In** | Module with PWA offline support | Extract if real-time check-in load spikes require isolation |

**Decision**: Modular monolith now. Clear extraction plan documented for future team.

---

## Decision

**Modular Monolith with Vertical Slice Architecture (VSA).**

MediatR CQRS is the backbone. All business logic lives in MediatR handlers. Controllers are strictly thin: parse HTTP request → `mediator.Send()` → return HTTP response. Zero business logic in controllers, zero business logic in infrastructure adapters.

---

## Solution Folder Structure

```
EventFlow.sln
│
├── src/
│   ├── EventFlow.Api/                          ← ASP.NET Core Web API host
│   │   ├── Controllers/
│   │   │   ├── EventsController.cs             ← THIN: mediator.Send() only
│   │   │   ├── RegistrationsController.cs
│   │   │   ├── SessionsController.cs
│   │   │   ├── AttendeesController.cs
│   │   │   ├── SpeakersController.cs
│   │   │   ├── SponsorsController.cs
│   │   │   ├── VenuesController.cs
│   │   │   ├── CampaignsController.cs
│   │   │   ├── AnalyticsController.cs
│   │   │   ├── CheckInController.cs
│   │   │   ├── AiController.cs
│   │   │   ├── TenantsController.cs
│   │   │   └── SearchController.cs
│   │   ├── Middleware/
│   │   │   ├── TenantResolutionMiddleware.cs   ← Extracts org_id from JWT → ITenantContext
│   │   │   ├── ExceptionHandlingMiddleware.cs  ← Global error → ProblemDetails
│   │   │   └── RequestLoggingMiddleware.cs
│   │   ├── Extensions/
│   │   │   ├── ServiceCollectionExtensions.cs
│   │   │   └── WebApplicationExtensions.cs
│   │   ├── Hubs/
│   │   │   ├── CheckInHub.cs                  ← SignalR hub for real-time check-in
│   │   │   └── EventActivityHub.cs            ← Real-time dashboard activity feed
│   │   ├── Program.cs
│   │   ├── appsettings.json
│   │   ├── appsettings.Development.json
│   │   └── appsettings.Production.json
│   │
│   ├── EventFlow.Application/                  ← All business logic. Feature folders (VSA).
│   │   ├── Behaviors/                          ← MediatR pipeline behaviors
│   │   │   ├── LoggingBehavior.cs              ← Outermost: logs request/response/duration
│   │   │   ├── FeatureFlagBehavior.cs          ← Checks Unleash flags; throws if disabled
│   │   │   ├── ValidationBehavior.cs           ← Runs FluentValidation; throws ValidationException
│   │   │   ├── CacheBehavior.cs                ← Implements ICacheableQuery pattern
│   │   │   ├── CostTrackingBehavior.cs         ← Tracks AI token costs and query costs
│   │   │   └── TransactionBehavior.cs          ← Wraps commands in DB transaction
│   │   │
│   │   ├── Common/
│   │   │   ├── Interfaces/
│   │   │   │   ├── ICurrentUser.cs
│   │   │   │   ├── ITenantContext.cs
│   │   │   │   ├── IDateTimeProvider.cs
│   │   │   │   ├── ICacheService.cs
│   │   │   │   ├── IEventBus.cs
│   │   │   │   ├── IFileStorage.cs
│   │   │   │   ├── IEmailService.cs
│   │   │   │   ├── ISmsService.cs
│   │   │   │   ├── IAiService.cs
│   │   │   │   └── IFeatureFlagService.cs
│   │   │   ├── DTOs/
│   │   │   │   ├── PagedResult.cs
│   │   │   │   ├── PaginationQuery.cs
│   │   │   │   └── Result.cs
│   │   │   └── Exceptions/
│   │   │       ├── ValidationException.cs
│   │   │       ├── NotFoundException.cs
│   │   │       ├── ForbiddenException.cs
│   │   │       ├── ConflictException.cs
│   │   │       └── FeatureDisabledException.cs
│   │   │
│   │   ├── Features/
│   │   │   ├── Events/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── CreateEvent/
│   │   │   │   │   │   ├── CreateEventCommand.cs
│   │   │   │   │   │   ├── CreateEventHandler.cs
│   │   │   │   │   │   └── CreateEventValidator.cs
│   │   │   │   │   ├── UpdateEvent/
│   │   │   │   │   │   ├── UpdateEventCommand.cs
│   │   │   │   │   │   ├── UpdateEventHandler.cs
│   │   │   │   │   │   └── UpdateEventValidator.cs
│   │   │   │   │   ├── PublishEvent/
│   │   │   │   │   │   ├── PublishEventCommand.cs
│   │   │   │   │   │   ├── PublishEventHandler.cs
│   │   │   │   │   │   └── PublishEventValidator.cs
│   │   │   │   │   ├── CancelEvent/
│   │   │   │   │   ├── DuplicateEvent/
│   │   │   │   │   └── AiGenerateEvent/         ← AI-assisted event creation
│   │   │   │   │       ├── AiGenerateEventCommand.cs
│   │   │   │   │       ├── AiGenerateEventHandler.cs
│   │   │   │   │       └── AiGenerateEventValidator.cs
│   │   │   │   ├── Queries/
│   │   │   │   │   ├── GetEventById/
│   │   │   │   │   │   ├── GetEventByIdQuery.cs
│   │   │   │   │   │   └── GetEventByIdHandler.cs
│   │   │   │   │   ├── GetEvents/
│   │   │   │   │   │   ├── GetEventsQuery.cs
│   │   │   │   │   │   └── GetEventsHandler.cs
│   │   │   │   │   └── GetEventDashboard/
│   │   │   │   │       ├── GetEventDashboardQuery.cs
│   │   │   │   │       └── GetEventDashboardHandler.cs
│   │   │   │   └── DTOs/
│   │   │   │       ├── EventDto.cs
│   │   │   │       ├── EventSummaryDto.cs
│   │   │   │       └── EventDashboardDto.cs
│   │   │   │
│   │   │   ├── Sessions/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── CreateSession/
│   │   │   │   │   ├── UpdateSession/
│   │   │   │   │   ├── ReorderSessions/         ← Drag-and-drop agenda reorder
│   │   │   │   │   ├── AiGenerateAgenda/        ← AI agenda builder
│   │   │   │   │   │   ├── AiGenerateAgendaCommand.cs
│   │   │   │   │   │   ├── AiGenerateAgendaHandler.cs
│   │   │   │   │   │   └── AiGenerateAgendaValidator.cs
│   │   │   │   │   └── AssignSpeakerToSession/
│   │   │   │   ├── Queries/
│   │   │   │   │   ├── GetSessionsByEvent/
│   │   │   │   │   └── GetSessionConflicts/     ← Conflict detection for scheduler
│   │   │   │   └── DTOs/
│   │   │   │
│   │   │   ├── Attendees/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── ImportAttendees/
│   │   │   │   │   ├── UpdateAttendee/
│   │   │   │   │   ├── BulkTagAttendees/
│   │   │   │   │   └── GenerateAttendeeEmbedding/ ← AI: generate profile vector
│   │   │   │   ├── Queries/
│   │   │   │   │   ├── GetAttendees/
│   │   │   │   │   ├── GetAttendeeById/
│   │   │   │   │   └── GetAiMatchmaking/        ← AI attendee matchmaking
│   │   │   │   └── DTOs/
│   │   │   │
│   │   │   ├── Registrations/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── CreateRegistration/
│   │   │   │   │   ├── CancelRegistration/
│   │   │   │   │   ├── CheckInAttendee/         ← Day-of check-in
│   │   │   │   │   ├── BulkCheckIn/
│   │   │   │   │   └── SyncOfflineCheckIns/     ← PWA offline sync endpoint
│   │   │   │   ├── Queries/
│   │   │   │   │   ├── GetRegistrations/
│   │   │   │   │   ├── GetCheckInStatus/
│   │   │   │   │   └── GetRegistrationQrCode/
│   │   │   │   └── DTOs/
│   │   │   │
│   │   │   ├── Speakers/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── InviteSpeaker/           ← Magic link invitation
│   │   │   │   │   ├── AcceptSpeakerInvite/
│   │   │   │   │   └── UpdateSpeaker/
│   │   │   │   ├── Queries/
│   │   │   │   └── DTOs/
│   │   │   │
│   │   │   ├── Sponsors/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── CreateSponsorTier/
│   │   │   │   │   ├── AddSponsor/
│   │   │   │   │   ├── ReorderSponsorTier/      ← Drag-drop tier reorder
│   │   │   │   │   └── UpdateSponsorBenefits/
│   │   │   │   ├── Queries/
│   │   │   │   └── DTOs/
│   │   │   │
│   │   │   ├── Venues/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── CreateVenue/
│   │   │   │   │   └── UpdateVenue/
│   │   │   │   ├── Queries/
│   │   │   │   │   ├── SearchVenues/            ← Venue discovery
│   │   │   │   │   └── GetVenueById/
│   │   │   │   └── DTOs/
│   │   │   │
│   │   │   ├── Campaigns/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── CreateCampaign/
│   │   │   │   │   ├── SendCampaign/            ← Triggers email/SMS dispatch
│   │   │   │   │   ├── ScheduleCampaign/
│   │   │   │   │   └── AiGenerateCampaignCopy/  ← AI email copy generation
│   │   │   │   ├── Queries/
│   │   │   │   │   ├── GetCampaigns/
│   │   │   │   │   └── GetCampaignAnalytics/
│   │   │   │   └── DTOs/
│   │   │   │
│   │   │   ├── Analytics/
│   │   │   │   ├── Queries/
│   │   │   │   │   ├── GetPortfolioDashboard/   ← Cross-event metrics (David persona)
│   │   │   │   │   ├── GetEventAnalytics/
│   │   │   │   │   ├── GetSessionAnalytics/
│   │   │   │   │   ├── GetRevenueReport/
│   │   │   │   │   ├── GetAttendeeJourney/
│   │   │   │   │   └── GetNpsReport/
│   │   │   │   └── DTOs/
│   │   │   │
│   │   │   ├── Ai/
│   │   │   │   ├── Commands/
│   │   │   │   │   ├── GenerateEventFromDescription/
│   │   │   │   │   ├── GenerateAgendaSuggestions/
│   │   │   │   │   ├── GenerateMatchmakingSuggestions/
│   │   │   │   │   ├── GenerateCampaignCopy/
│   │   │   │   │   └── GenerateInsightsBanner/  ← Homepage AI insights
│   │   │   │   └── DTOs/
│   │   │   │       ├── AiSuggestionDto.cs
│   │   │   │       └── AiReasoningDto.cs        ← Transparency: "Why?" explanation
│   │   │   │
│   │   │   ├── Search/
│   │   │   │   └── Queries/
│   │   │   │       └── UnifiedSearch/           ← Powers Cmd+K command palette
│   │   │   │           ├── UnifiedSearchQuery.cs
│   │   │   │           └── UnifiedSearchHandler.cs
│   │   │   │
│   │   │   └── Tenants/
│   │   │       ├── Commands/
│   │   │       │   ├── ProvisionTenant/
│   │   │       │   ├── UpdateTenantTheme/       ← White-label brand settings
│   │   │       │   └── UpdateTenantSettings/
│   │   │       ├── Queries/
│   │   │       │   ├── GetTenantById/
│   │   │       │   └── GetTenantTheme/          ← Served before page render
│   │   │       └── DTOs/
│   │   │           ├── TenantDto.cs
│   │   │           └── TenantThemeDto.cs
│   │
│   ├── EventFlow.Domain/                        ← Entities, value objects, domain events. ZERO external deps.
│   │   ├── Entities/
│   │   │   ├── Event.cs
│   │   │   ├── Session.cs
│   │   │   ├── Attendee.cs
│   │   │   ├── Registration.cs
│   │   │   ├── Speaker.cs
│   │   │   ├── Sponsor.cs
│   │   │   ├── SponsorTier.cs
│   │   │   ├── Venue.cs
│   │   │   ├── Campaign.cs
│   │   │   ├── Ticket.cs
│   │   │   ├── TicketType.cs
│   │   │   ├── CheckIn.cs
│   │   │   ├── Tenant.cs
│   │   │   ├── TenantTheme.cs
│   │   │   └── AuditLog.cs
│   │   ├── Enums/
│   │   │   ├── EventStatus.cs                  ← Draft, Published, Live, Completed, Cancelled
│   │   │   ├── EventFormat.cs                  ← InPerson, Virtual, Hybrid
│   │   │   ├── SessionType.cs                  ← Keynote, Workshop, Panel, Networking, Break
│   │   │   ├── RegistrationStatus.cs           ← Pending, Confirmed, Cancelled, Waitlisted
│   │   │   ├── TicketStatus.cs
│   │   │   ├── CampaignStatus.cs
│   │   │   ├── CampaignType.cs                 ← Email, Sms
│   │   │   ├── SponsorTierLevel.cs             ← Platinum, Gold, Silver, Bronze, Custom
│   │   │   └── UserRole.cs                     ← Admin, EventManager, Coordinator, Viewer
│   │   ├── ValueObjects/
│   │   │   ├── EmailAddress.cs
│   │   │   ├── PhoneNumber.cs
│   │   │   ├── DateTimeRange.cs
│   │   │   ├── Money.cs
│   │   │   ├── Address.cs
│   │   │   ├── BrandColor.cs                   ← Validated hex color for white-label
│   │   │   └── Capacity.cs                     ← Min/Max capacity with validation
│   │   ├── Events/                             ← Domain events (not integration events)
│   │   │   ├── EventCreated.cs
│   │   │   ├── EventPublished.cs
│   │   │   ├── EventCancelled.cs
│   │   │   ├── AttendeeRegistered.cs
│   │   │   ├── RegistrationCancelled.cs
│   │   │   ├── AttendeeCheckedIn.cs
│   │   │   ├── SessionCreated.cs
│   │   │   ├── CampaignSent.cs
│   │   │   ├── SpeakerInvited.cs
│   │   │   ├── SpeakerAccepted.cs
│   │   │   └── TenantProvisioned.cs
│   │   └── Exceptions/
│   │       ├── EventCapacityExceededException.cs
│   │       ├── InvalidEventStatusTransitionException.cs
│   │       └── DuplicateRegistrationException.cs
│   │
│   └── EventFlow.Infrastructure/               ← EF Core, external adapters. Implements Application interfaces.
│       ├── Persistence/
│       │   ├── EventFlowDbContext.cs
│       │   ├── Configurations/                 ← IEntityTypeConfiguration<T> per entity
│       │   │   ├── EventConfiguration.cs
│       │   │   ├── SessionConfiguration.cs
│       │   │   ├── AttendeeConfiguration.cs
│       │   │   ├── RegistrationConfiguration.cs
│       │   │   └── TenantConfiguration.cs
│       │   ├── Migrations/
│       │   └── Repositories/
│       │       └── (Optional: only if complex queries justify repo abstraction)
│       ├── Adapters/                           ← ONE external call per method. No orchestration.
│       │   ├── CacheService.cs                 ← Implements ICacheService via StackExchange.Redis
│       │   ├── EventBusService.cs              ← Implements IEventBus via Redis Streams
│       │   ├── FileStorageService.cs           ← Implements IFileStorage via AWSSDK.S3 / MinIO
│       │   ├── EmailService.cs                 ← Implements IEmailService via SendGrid/SMTP
│       │   ├── SmsService.cs                   ← Implements ISmsService via Twilio
│       │   ├── AiService.cs                    ← Implements IAiService via OpenAI SDK
│       │   ├── FeatureFlagService.cs           ← Implements IFeatureFlagService via Unleash SDK
│       │   └── KeycloakAdminService.cs         ← Admin API calls for tenant user management
│       ├── BackgroundServices/
│       │   ├── CampaignSchedulerService.cs     ← Processes scheduled campaigns
│       │   ├── EventStreamConsumer.cs          ← Redis Streams consumer group processor
│       │   └── AiInsightsGeneratorService.cs   ← Periodic AI insights generation
│       └── Extensions/
│           └── InfrastructureServiceExtensions.cs
│
├── tests/
│   ├── EventFlow.UnitTests/
│   │   ├── Features/
│   │   │   ├── Events/
│   │   │   │   ├── CreateEventHandlerTests.cs
│   │   │   │   ├── CreateEventValidatorTests.cs
│   │   │   │   ├── PublishEventHandlerTests.cs
│   │   │   │   └── AiGenerateEventHandlerTests.cs
│   │   │   ├── Registrations/
│   │   │   │   ├── CreateRegistrationHandlerTests.cs
│   │   │   │   └── CheckInAttendeeHandlerTests.cs
│   │   │   └── Analytics/
│   │   │       └── GetPortfolioDashboardHandlerTests.cs
│   │   ├── Domain/
│   │   │   ├── EventTests.cs
│   │   │   ├── RegistrationTests.cs
│   │   │   └── ValueObjects/
│   │   │       ├── MoneyTests.cs
│   │   │       ├── EmailAddressTests.cs
│   │   │       └── BrandColorTests.cs
│   │   └── Behaviors/
│   │       ├── ValidationBehaviorTests.cs
│   │       ├── CacheBehaviorTests.cs
│   │       └── FeatureFlagBehaviorTests.cs
│   │
│   ├── EventFlow.IntegrationTests/
│   │   ├── Infrastructure/
│   │   │   ├── TestWebApplicationFactory.cs    ← TestContainers: PostgreSQL + Redis
│   │   │   ├── DatabaseFixture.cs
│   │   │   └── SeedData.cs
│   │   ├── Api/
│   │   │   ├── EventsApiTests.cs
│   │   │   ├── RegistrationsApiTests.cs
│   │   │   ├── CheckInApiTests.cs
│   │   │   └── SearchApiTests.cs
│   │   └── Features/
│   │       ├── RegistrationFlowTests.cs        ← Full pipeline: register → email → check-in
│   │       └── TenantIsolationTests.cs         ← Verify cross-tenant data leakage is impossible
│   │
│   └── EventFlow.E2ETests/                     ← Playwright
│       ├── Flows/
│       │   ├── OnboardingFlowTests.cs
│       │   ├── EventCreationFlowTests.cs
│       │   └── CheckInFlowTests.cs
│       └── playwright.config.ts
│
├── docker-compose.yml
├── docker-compose.override.yml
├── .editorconfig
├── .prettierrc
├── Directory.Build.props
└── Directory.Packages.props
```

---

## MediatR Pipeline Behavior Order

Behaviors execute outermost-first (decorator pattern). The pipeline for every request:

```
HTTP Request
    ↓
Controller (thin: parse → mediator.Send())
    ↓
[1] LoggingBehavior          ← Logs request type, user, tenant, duration. Always outermost.
    ↓
[2] FeatureFlagBehavior      ← Checks Unleash for [FeatureFlag("feature-name")] attribute on command.
    ↓                           Throws FeatureDisabledException if flag is off.
[3] ValidationBehavior       ← Runs all IValidator<TRequest> for this request type.
    ↓                           Throws ValidationException with all failures if invalid.
[4] CacheBehavior            ← For ICacheableQuery: returns cached result if hit, continues to handler if miss.
    ↓
[5] CostTrackingBehavior     ← For AI commands: tracks token consumption per tenant for billing/quotas.
    ↓
[6] Handler                  ← The actual business logic.
    ↓
HTTP Response
```

```csharp
// Example: Thin controller — ZERO business logic
[ApiController]
[Route("api/events")]
public class EventsController : ControllerBase
{
    private readonly IMediator _mediator;

    public EventsController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<IActionResult> CreateEvent(
        [FromBody] CreateEventRequest request,
        CancellationToken ct)
    {
        var command = new CreateEventCommand(
            Name: request.Name,
            Description: request.Description,
            StartDateUtc: request.StartDateUtc,
            EndDateUtc: request.EndDateUtc,
            Format: request.Format,
            MaxCapacity: request.MaxCapacity,
            VenueId: request.VenueId
        );
        var result = await _mediator.Send(command, ct);
        return CreatedAtAction(nameof(GetEvent), new { id = result.EventId }, result);
    }

    [HttpGet("{id:guid}")]
    public async Task<IActionResult> GetEvent(Guid id, CancellationToken ct)
    {
        var query = new GetEventByIdQuery(id);
        var result = await _mediator.Send(query, ct);
        return Ok(result);
    }
}
```

---

## Consequences

### Positive
- **Velocity**: Small team ships features fast. No distributed systems overhead.
- **Debuggability**: Single process; step through the entire request in a debugger.
- **Transactional integrity**: Cross-feature operations (register → send email → update analytics) in a single DB transaction.
- **Vertical slice clarity**: Each feature folder is self-contained. New engineers find all related code in one place.
- **Extract later**: Modules have clean interfaces. Extraction to microservices is possible without rewriting business logic.

### Negative
- **Single deployment unit**: All features deploy together. A bug in AI generation can block a check-in hotfix release. Mitigation: Feature flags isolate risky features; comprehensive test suite catches regressions.
- **Shared database**: All modules share one PostgreSQL instance. Schema changes require care. Mitigation: EF Core migrations + CI migration safety checks.
- **Scaling granularity**: Cannot scale only the check-in feature during event-day spikes. Mitigation: Horizontal pod autoscaling scales the entire application, which is acceptable at current scale.

---

## Future Microservice Extraction Criteria

Extract a module into a microservice when **all three** of these conditions are met:
1. The module has a clearly different scaling profile from the rest of the monolith.
2. The module has a different deployment cadence (e.g., a team owns it independently).
3. The module has a clean interface boundary that can become a network contract without major refactoring.

Candidates in order of extraction likelihood: AI Service → Communications Service → Analytics Service.

---

## References
- [Vertical Slice Architecture — Jimmy Bogard](https://jimmybogard.com/vertical-slice-architecture/)
- [MediatR documentation](https://github.com/jbogard/MediatR)
- [Modular Monolith — Sam Newman](https://samnewman.io/blog/2019/02/19/modular-monolith/)
