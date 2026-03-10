# Add API Endpoint

## Steps
1. **Command or Query** — Create in `Application/`:
   - Commands for mutations (`IRequest<TResponse>`)
   - Queries for reads (`IRequest<TResponse>`)
2. **Validator** — Add FluentValidation validator for the request.
3. **Handler** — Implement `IRequestHandler` with all business logic.
4. **Controller method** — Add to the appropriate controller:
   ```csharp
   [HttpPost]
   public async Task<ActionResult<ResponseDto>> Create(CreateRequest request)
       => Ok(await _mediator.Send(new CreateCommand(request)));
   ```
   **Controller is THIN — parse request, send to MediatR, return response. That's it.**
5. **Infrastructure adapter** (if needed) — If the handler needs external I/O (email, storage, payment):
   - Define interface in `Application/Interfaces/` (e.g. `IEmailService`)
   - Implement in `Infrastructure/Services/` — one external call per method, no orchestration
   - Handler injects the interface, not the implementation

## Authentication
- Add `[Authorize]` attribute to protected endpoints.
- Use claims from the JWT for user identification — never trust client-sent user IDs.

## Validation
- Validate ALL external input at the boundary.
- Return 400 with descriptive error messages for invalid input.
- Never trust client data — validate types, ranges, and formats.

## Tests
- **Unit test** the handler/service logic.
- **Integration test** the full endpoint (request → response).
- Test auth: verify 401 for unauthenticated, 403 for unauthorized.
- Test validation: verify 400 for bad input.