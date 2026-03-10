# Add Feature

## Branch
Create a feature branch: `feature/<short-description>`

## Steps
1. **Command/Query** — Create in `Application/` with a MediatR `IRequest<T>`.
2. **Validator** — Add a FluentValidation validator for the command.
3. **Handler** — Implement `IRequestHandler<TRequest, TResponse>` with all business logic.
4. **DTO** — Add response DTO if returning data.
5. **Controller endpoint** — Add a thin endpoint: parse request → `_mediator.Send(command)` → return result. **No business logic in controllers.**

## Tests
- Add at least one unit test for the core logic.
- Add an integration test if the feature touches the database or external APIs.
- Run the full test suite before committing.

## Rules
- **Controllers MUST be thin** — parse request → `_mediator.Send()` → return response. No business logic, no DB queries, no service calls.
- **No business service classes** — do NOT create `IXxxService` with business logic. All business logic lives in MediatR handlers.
- **Infrastructure adapters are thin** — if you need to call an external API, create an interface in `Application/` (e.g. `IPaymentGateway`) and implement it in `Infrastructure/` with one external call per method. The MediatR handler orchestrates, the adapter just does I/O.
- Follow existing naming conventions and directory structure.
- Never commit secrets, tokens, or credentials.
- Keep the PR focused on one feature — avoid unrelated changes.

## PR Checklist
- [ ] Feature works as described
- [ ] Tests pass
- [ ] No lint/build errors
- [ ] No hardcoded secrets
- [ ] Thin controllers — logic lives in handlers/services