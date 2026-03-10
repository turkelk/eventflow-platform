# Infrastructure Module

## Database Access
- All database access goes through Entity Framework Core DbContext.
- Use `async` methods (`ToListAsync`, `FirstOrDefaultAsync`, etc.).
- Never use raw SQL unless absolutely necessary — use LINQ queries.
- Dispose connections properly (use `using` or DI scoped lifetime).

## Migrations
- **Always create a migration** when changing the data model.
- Migrations must be reversible (implement both Up and Down).
- Never modify an existing migration that has been applied — create a new one.
- Test migrations against a fresh database before committing.
- Review generated SQL before applying to production.

## Query Guidelines
- Use indexes for frequently queried columns.
- Avoid N+1 queries — use eager loading or batch fetching.
- Use `EXPLAIN ANALYZE` to verify query plans for complex queries.
- Use `jsonb` columns sparingly — prefer normalized tables for queryable data.
- Paginate large result sets — never return unbounded collections.

## Connection Management
- Use connection pooling (built into EF Core).
- Set appropriate pool sizes for the deployment environment.
- Handle connection timeouts gracefully.

## Row-Level Security
- RLS policies are enforced at the database level.
- The application sets the tenant context on each connection before queries.
- Never bypass RLS except in system-level background jobs.
- Always test that cross-tenant data access is blocked.

## Security
- Never log full query results containing user data.
- Use parameterized queries — never concatenate user input into SQL.
- Encrypt sensitive columns (tokens, credentials) at the application level.
- Audit log destructive operations (DELETE, UPDATE on sensitive tables).