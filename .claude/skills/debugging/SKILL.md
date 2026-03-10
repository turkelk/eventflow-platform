# Debugging Guide

## Server-Side Errors
1. Check the application logs (Serilog structured logs in console/Seq).
2. Look for the exception type and stack trace.
3. Reproduce with a minimal request using `curl` or the Swagger UI.
4. Add breakpoints or temporary logging — remove after fixing.
5. Check middleware order if the request isn't reaching the handler.

### .NET Specific
- Check `Program.cs` service registration — missing DI registrations cause runtime errors.
- Use `dotnet watch run` for hot reload during debugging.
- Check MediatR pipeline behaviors — they can swallow or transform exceptions.

## Database Issues
- Connect with `psql` to run queries directly.
- Check connection string in environment variables.
- Look for migration issues: `SELECT * FROM "__EFMigrationsHistory";`
- Slow queries: use `EXPLAIN ANALYZE` to check query plans.
- Connection pool exhaustion: check for unclosed connections or missing `using` statements.

## Frontend Issues
- Check browser console for errors.
- Check Network tab for failed API calls.
- Verify auth token is present in request headers.
- Check React Query devtools for cache state.

## Common Patterns
| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| 500 Internal Server Error | Unhandled exception | Check logs for stack trace |
| 404 Not Found | Wrong route or missing registration | Verify route registration |
| 401 Unauthorized | Missing or invalid auth token | Check auth configuration |
| 403 Forbidden | Insufficient permissions | Check authorization policies |
| Timeout | Slow query or external API | Check DB queries, add timeouts |
| Empty response | Serialization issue | Check DTO mapping and nulls |