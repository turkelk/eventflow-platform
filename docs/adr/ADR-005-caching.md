# ADR-005: Caching Strategy — Redis with CacheBehavior

**Status**: Accepted  
**Date**: 2025-01-01  
**Deciders**: Architecture Team  
**Consulted**: Engineering Leads  

---

## Context

EventFlow serves multiple tenants with read-heavy workloads. The most frequent queries include:

- **Tenant configuration / theme**: Fetched on every page load to apply white-label CSS variables. Without caching, this adds a DB round-trip to every initial page render.
- **Event list for tenant**: The dashboard fetches upcoming events on every load. With 200 events per tenant and 50 concurrent users, this becomes high-read traffic.
- **Attendee list for check-in**: On event day, all check-in staff query the attendee list. For a 500-person event with 10 check-in devices, this is 10 concurrent queries for the same data.
- **Session schedule**: Multiple organizers and all attendees view the schedule frequently.
- **Venue lookup**: Venue data changes rarely — no need to hit the DB on every registration page load.
- **Feature flags**: Unleash flag evaluation happens per-request. The Unleash SDK caches locally, but tenant-specific flag overrides benefit from an additional cache layer.

**NOT cached** (explicit exclusions):
- Write operations and their responses
- Check-in status (real-time — must always reflect true state)
- User-specific sensitive data (payment details, private attendee fields)
- Analytics queries with time-range parameters (too many cache key permutations)
- Any data subject to race conditions where stale reads cause correctness issues

---

## Decision

**Redis (StackExchange.Redis) as the distributed cache. Implemented as a MediatR `CacheBehavior` pipeline behavior applied to queries implementing `ICacheableQuery`. Key format: `{tenant}:{entity}:{id}`. Default TTL: 5 minutes. Command handlers invalidate affected cache keys after successful mutations.**

---

## Cache Key Format

```
{tenantId}:{entity}:{qualifier}

Examples:
ten_01HXYZ:events:list                        ← All events for tenant
ten_01HXYZ:events:evt_01HXYZ                  ← Specific event by ID
ten_01HXYZ:events:evt_01HXYZ:sessions         ← Sessions for specific event
ten_01HXYZ:events:evt_01HXYZ:attendees:page:1 ← Paginated attendees (page 1)
ten_01HXYZ:venues:all                         ← All venues for tenant
ten_01HXYZ:tenant:theme                       ← Tenant white-label theme
ten_01HXYZ:tenant:settings                    ← Tenant configuration
ten_01HXYZ:speakers:spk_01HXYZ               ← Speaker profile
ten_01HXYZ:search:q:react+conference          ← Search result (short TTL)
```

**Why tenant-prefix first?** Enables bulk invalidation of all cache entries for a tenant with `SCAN {tenantId}:*` pattern. Useful for tenant data migrations or tenant deletion cleanup.

---

## ICacheableQuery Interface

```csharp
// Application/Common/Interfaces/ICacheableQuery.cs
public interface ICacheableQuery
{
    /// <summary>
    /// Cache key for this query result.
    /// Must be unique per query parameter combination.
    /// Format: {tenantId}:{entity}:{qualifier}
    /// </summary>
    string CacheKey { get; }

    /// <summary>
    /// Time-to-live for this cache entry.
    /// Null = use default TTL (5 minutes).
    /// </summary>
    TimeSpan? CacheDuration => null;

    /// <summary>
    /// If true, bypass cache and always fetch from DB.
    /// Used for: user explicitly requesting refresh, admin views.
    /// </summary>
    bool BypassCache => false;
}
```

### Query Implementation Example

```csharp
// Application/Features/Events/Queries/GetEvents/GetEventsQuery.cs
public record GetEventsQuery(
    EventStatus? StatusFilter,
    int Page,
    int PageSize
) : IRequest<PagedResult<EventSummaryDto>>, ICacheableQuery
{
    // ITenantContext.TenantId is injected by CacheBehavior — not in the query record
    // to avoid exposing tenant context as a constructor parameter
    public string CacheKey => $"events:list:status:{StatusFilter}:page:{Page}:size:{PageSize}";
    public TimeSpan? CacheDuration => TimeSpan.FromMinutes(3); // Events list: shorter TTL (changes frequently)
}

// Application/Features/Tenants/Queries/GetTenantTheme/GetTenantThemeQuery.cs
public record GetTenantThemeQuery(Guid TenantId)
    : IRequest<TenantThemeDto>, ICacheableQuery
{
    public string CacheKey => $"tenant:theme"; // TenantId prefixed by CacheBehavior
    public TimeSpan? CacheDuration => TimeSpan.FromMinutes(60); // Theme changes rarely
}

// Application/Features/Sessions/Queries/GetSessionsByEvent/GetSessionsByEventQuery.cs
public record GetSessionsByEventQuery(Guid EventId)
    : IRequest<IReadOnlyList<SessionDto>>, ICacheableQuery
{
    public string CacheKey => $"events:{EventId}:sessions";
    public TimeSpan? CacheDuration => TimeSpan.FromMinutes(10); // Session schedule: medium TTL
}
```

---

## CacheBehavior Implementation

```csharp
// Application/Behaviors/CacheBehavior.cs
public class CacheBehavior<TRequest, TResponse>
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
{
    private readonly ICacheService _cache;
    private readonly ITenantContext _tenantContext;
    private readonly ILogger<CacheBehavior<TRequest, TResponse>> _logger;

    public async Task<TResponse> Handle(
        TRequest request,
        RequestHandlerDelegate<TResponse> next,
        CancellationToken ct)
    {
        // Only cache queries implementing ICacheableQuery
        if (request is not ICacheableQuery cacheableQuery)
            return await next();

        if (cacheableQuery.BypassCache)
        {
            _logger.LogDebug("Cache bypass for {RequestType}", typeof(TRequest).Name);
            return await next();
        }

        // Tenant-scoped full cache key
        var fullKey = $"{_tenantContext.TenantId}:{cacheableQuery.CacheKey}";
        var ttl = cacheableQuery.CacheDuration ?? TimeSpan.FromMinutes(5);

        // Try cache first
        var cached = await _cache.GetAsync<TResponse>(fullKey, ct);
        if (cached is not null)
        {
            _logger.LogDebug("Cache HIT: {CacheKey}", fullKey);
            return cached;
        }

        _logger.LogDebug("Cache MISS: {CacheKey}", fullKey);

        // Execute handler
        var result = await next();

        // Store in cache
        if (result is not null)
        {
            await _cache.SetAsync(fullKey, result, ttl, ct);
        }

        return result;
    }
}
```

---

## Cache Invalidation Strategy

Command handlers explicitly invalidate affected cache keys after successful mutations. This is **write-through invalidation** (not write-through update — we invalidate and let the next read repopulate).

```csharp
// Application/Features/Events/Commands/UpdateEvent/UpdateEventHandler.cs
public class UpdateEventHandler : IRequestHandler<UpdateEventCommand, EventDto>
{
    private readonly EventFlowDbContext _db;
    private readonly ICacheService _cache;
    private readonly ITenantContext _tenantContext;

    public async Task<EventDto> Handle(
        UpdateEventCommand command,
        CancellationToken ct)
    {
        var @event = await _db.Events
            .FirstOrDefaultAsync(e => e.Id == command.EventId, ct)
            ?? throw new NotFoundException($"Event {command.EventId} not found.");

        @event.Update(command.Name, command.Description, command.StartDateUtc, command.EndDateUtc);

        await _db.SaveChangesAsync(ct);

        // Invalidate affected cache keys
        var tenantId = _tenantContext.TenantId;
        await Task.WhenAll(
            _cache.RemoveAsync($"{tenantId}:events:{command.EventId}", ct),    // Specific event
            _cache.RemoveByPatternAsync($"{tenantId}:events:list:*", ct)       // All event list pages
        );

        return @event.ToDto();
    }
}

// Application/Features/Tenants/Commands/UpdateTenantTheme/UpdateTenantThemeHandler.cs
public class UpdateTenantThemeHandler : IRequestHandler<UpdateTenantThemeCommand, TenantThemeDto>
{
    public async Task<TenantThemeDto> Handle(
        UpdateTenantThemeCommand command,
        CancellationToken ct)
    {
        // ... update theme in DB ...

        // Invalidate theme cache — all pages using this tenant's theme will refetch
        await _cache.RemoveAsync(
            $"{_tenantContext.TenantId}:tenant:theme", ct);

        // Also publish integration event so other pods invalidate their local memory caches
        await _eventBus.PublishAsync(
            new TenantThemeUpdated(_tenantContext.TenantId), ct);

        return updatedTheme.ToDto();
    }
}
```

### Cache Invalidation Matrix

| Command | Keys Invalidated |
|---|---|
| `UpdateEventCommand` | `{t}:events:{id}`, `{t}:events:list:*` |
| `PublishEventCommand` | `{t}:events:{id}`, `{t}:events:list:*` |
| `CancelEventCommand` | `{t}:events:{id}`, `{t}:events:list:*` |
| `CreateSessionCommand` | `{t}:events:{eventId}:sessions` |
| `UpdateSessionCommand` | `{t}:events:{eventId}:sessions` |
| `ReorderSessionsCommand` | `{t}:events:{eventId}:sessions` |
| `CreateRegistrationCommand` | `{t}:events:{eventId}:attendees:*` |
| `BulkTagAttendeesCommand` | `{t}:events:{eventId}:attendees:*` |
| `UpdateTenantThemeCommand` | `{t}:tenant:theme` |
| `UpdateTenantSettingsCommand` | `{t}:tenant:settings` |
| `ImportAttendeesCommand` | `{t}:events:{eventId}:attendees:*` |

---

## ICacheService Implementation

```csharp
// Infrastructure/Adapters/CacheService.cs
public class CacheService : ICacheService
{
    private readonly IConnectionMultiplexer _redis;

    public async Task<T?> GetAsync<T>(string key, CancellationToken ct = default)
    {
        var db = _redis.GetDatabase();
        var value = await db.StringGetAsync(key);
        if (!value.HasValue) return default;
        return JsonSerializer.Deserialize<T>(value!);
    }

    public async Task SetAsync<T>(
        string key,
        T value,
        TimeSpan ttl,
        CancellationToken ct = default)
    {
        var db = _redis.GetDatabase();
        var serialized = JsonSerializer.Serialize(value);
        await db.StringSetAsync(key, serialized, ttl);
    }

    public async Task RemoveAsync(string key, CancellationToken ct = default)
    {
        var db = _redis.GetDatabase();
        await db.KeyDeleteAsync(key);
    }

    public async Task RemoveByPatternAsync(
        string pattern,
        CancellationToken ct = default)
    {
        // Use SCAN to avoid blocking with KEYS command in production
        var server = _redis.GetServer(_redis.GetEndPoints().First());
        var keys = server
            .Keys(pattern: pattern)
            .ToArray();

        if (keys.Length > 0)
        {
            var db = _redis.GetDatabase();
            await db.KeyDeleteAsync(keys);
        }
    }
}
```

---

## Distributed Lock — RedLock Pattern

Distributed locks prevent race conditions in concurrent operations:
- **Check-in capacity**: Two devices simultaneously checking in the last attendee must not exceed capacity.
- **Outbox processor**: Only one pod should process outbox records at a time.
- **Campaign send**: Prevent duplicate sends from scheduled campaigns.
- **AI generation**: Prevent concurrent AI requests for the same event (cost protection).

```csharp
// Application/Common/Interfaces/IDistributedLockService.cs
public interface IDistributedLockService
{
    /// <summary>
    /// Acquires a distributed lock. Returns null if lock is not available.
    /// Dispose the returned handle to release the lock.
    /// </summary>
    Task<IDistributedLockHandle?> AcquireAsync(
        string resource,
        TimeSpan expiry,
        CancellationToken ct = default);
}

// Infrastructure/Adapters/DistributedLockService.cs
// Implements RedLock pattern via RedLock.net (multi-node Redis support)
public class DistributedLockService : IDistributedLockService
{
    private readonly IRedLockFactory _redLockFactory;

    public async Task<IDistributedLockHandle?> AcquireAsync(
        string resource,
        TimeSpan expiry,
        CancellationToken ct = default)
    {
        var redLock = await _redLockFactory.CreateLockAsync(
            resource: $"eventflow:lock:{resource}",
            expiryTime: expiry,
            waitTime: TimeSpan.FromSeconds(5),
            retryTime: TimeSpan.FromMilliseconds(200)
        );

        return redLock.IsAcquired
            ? new RedLockHandle(redLock)
            : null;
    }
}

// Usage in CheckInAttendeeHandler:
// await using var @lock = await _lockService.AcquireAsync(
//     $"checkin:{command.EventId}:{command.AttendeeId}",
//     TimeSpan.FromSeconds(10), ct);
// if (@lock == null) throw new ConflictException("Check-in already in progress for this attendee.");
```

---

## TTL Reference Table

| Cache Key Pattern | TTL | Rationale |
|---|---|---|
| `{t}:tenant:theme` | 60 min | Theme changes rarely; fetched on every page load |
| `{t}:tenant:settings` | 30 min | Settings change infrequently |
| `{t}:events:list:*` | 3 min | Event lists change moderately often |
| `{t}:events:{id}` | 5 min | Individual event detail |
| `{t}:events:{id}:sessions` | 10 min | Session schedule, changed by organizers only |
| `{t}:events:{id}:attendees:*` | 2 min | Attendees change on registrations; shorter TTL |
| `{t}:venues:all` | 30 min | Venues change rarely |
| `{t}:speakers:{id}` | 15 min | Speaker profiles rarely change |
| `{t}:search:*` | 30 sec | Search results — very short TTL; data changes frequently |
| `{t}:analytics:portfolio` | 5 min | Dashboard analytics — acceptable staleness |

---

## Cache Warming Strategy

For event-day performance (check-in staff use case — Sarah persona):

```csharp
// Triggered when event.live integration event is received
// Pre-warms the attendee list cache before check-in staff connect
public class EventLiveConsumer
{
    public async Task HandleAsync(EventLive @event, CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var mediator = scope.ServiceProvider.GetRequiredService<IMediator>();

        // Pre-warm attendee cache for all pages
        // This ensures check-in staff get sub-100ms responses
        await mediator.Send(new GetAttendeesForCheckInQuery(@event.EventId)
        {
            BypassCache = false // This warms the cache
        }, ct);

        _logger.LogInformation(
            "Pre-warmed attendee cache for live event {EventId}",
            @event.EventId);
    }
}
```

---

## Consequences

### Positive
- **Reduced DB load**: Tenant theme, event lists, and session schedules served from Redis — the most frequent reads never hit PostgreSQL.
- **Sub-10ms read latency**: Redis responses are typically <1ms on the same network. Stale-while-revalidate pattern on the frontend makes the UI feel instant.
- **Predictable DB load**: Cache absorbs read spikes (event-day attendee list queries). DB load is smooth and predictable.
- **MediatR pipeline integration**: `CacheBehavior` is transparent to handler authors — they don't write caching code, they implement `ICacheableQuery`.
- **Multi-tenant safe**: All keys prefixed with `{tenantId}` — cross-tenant cache hits are structurally impossible.

### Negative
- **Cache invalidation complexity**: Every command handler must know which cache keys to invalidate. Mitigation: Cache invalidation matrix document (above) + code review checklist item.
- **Pattern-scan for invalidation**: `RemoveByPatternAsync` uses Redis `SCAN` which is O(N). Mitigation: Limit pattern scans to low-frequency operations (bulk imports). Use specific key deletes for common operations.
- **Memory pressure**: Redis memory must be monitored. Default `maxmemory-policy: allkeys-lru` ensures cache auto-evicts under pressure. This is acceptable — the pattern degrades gracefully (cache miss → DB query).
- **Eventual consistency**: Cached data may be up to TTL seconds stale. This is acceptable for read-heavy queries (event lists, session schedules). It is NOT acceptable for check-in status — explicitly excluded from caching.

---

## References
- [StackExchange.Redis documentation](https://stackexchange.github.io/StackExchange.Redis/)
- [RedLock.net](https://github.com/samcook/RedLock.net)
- [Cache-aside pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/cache-aside)
- [MediatR pipeline behaviors](https://github.com/jbogard/MediatR/wiki/Behaviors)
