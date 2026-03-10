# API Module

## Architecture
This project uses **MediatR CQRS**. All business logic lives in command/query handlers — never in controllers.

- **Commands** — mutations (create, update, delete). Each has a handler and a FluentValidation validator.
- **Queries** — reads. Each has a handler. Use `[CachedQuery]` for frequently accessed data.
- **Controllers** — thin entry points. Parse request → `_mediator.Send()` → return response. No logic.

## Directory Structure
- `Controllers/` — API endpoints (thin, no logic). One controller per feature area.
- `Application/` — Feature folders, each containing:
  - `Commands/` — Command records + handlers + validators (one file per command, handler co-located)
  - `Queries/` — Query records + handlers (one file per query)
  - `DTOs/` — Response DTOs
  - `Interfaces/` — Contracts for infrastructure adapters (e.g. `IEmailService`, `IStorageService`)
- `Domain/` — Entities, Enums, Value Objects. ZERO dependencies on other layers.
- `Infrastructure/` — DbContext, external service adapters. Each adapter method = ONE external call, no orchestration. Implements Application-layer interfaces.

## CQRS Rules
- **Every write operation** = one `XxxCommand : IRequest<TResponse>` record + one `XxxHandler : IRequestHandler` + one `XxxValidator : AbstractValidator`
- **Every read operation** = one `XxxQuery : IRequest<TResponse>` record + one `XxxHandler : IRequestHandler`
- **No business service classes.** Do NOT create `IProjectService.CreateProject()`. Create `CreateProjectCommand` + `CreateProjectHandler` instead.
- **Infrastructure adapters are thin.** `IEmailService.SendAsync(to, subject, body)` = one SMTP/API call. `IStorageService.UploadAsync(key, bytes)` = one S3 call. No multi-step logic.

## MediatR Pipeline Behaviors
Request flows through (outermost first): LoggingBehavior → ValidationBehavior → CacheBehavior → FeatureFlagBehavior → Handler

## Naming Conventions
- Classes: PascalCase (`UserService`, `CreateEventCommand`)
- Methods: PascalCase (`GetUserById`)
- Variables: camelCase (`userId`, `scanResult`)
- Interfaces: `I` prefix (`IUserService`, `ICacheProvider`)
- Files: one class per file, filename matches class name

## Security Rules
- Never log tokens, passwords, or API keys.
- Validate all input at the API boundary.
- Extract user identity from JWT claims — never trust client-sent user IDs.
- Check authorization on every endpoint.

## Error Handling
- Return appropriate HTTP status codes (400, 401, 403, 404, 500).
- Log errors with context (correlation ID, user ID, operation).
- Never expose stack traces or internal details in API responses.