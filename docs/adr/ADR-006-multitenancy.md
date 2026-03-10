# ADR-006: Multi-Tenancy Strategy — Shared Database with Row-Level Security

**Status**: Accepted  
**Date**: 2025-01-01  
**Deciders**: Architecture Team  
**Consulted**: Security, Engineering Leads, Product  

---

## Context

EventFlow is a multi-tenant SaaS platform. Every tenant (organization) must have complete data isolation — one tenant must never be able to see, modify, or delete another tenant's data. This is a hard security requirement, not a best-effort concern.

We evaluated three isolation strategies:

### Option A: Database-Per-Tenant

Each tenant has a dedicated PostgreSQL database (or separate RDS instance).

**Pros**: Maximum isolation; independent backup/restore; tenant-specific tuning; compliance-friendly (tenant data is physically separate).

**Cons**: Operational overhead scales linearly with tenant count. 100 tenants = 100 databases to provision, migrate, monitor, and maintain. EF Core migrations must run against every tenant database independently. Connection pooling complexity (PgBouncer must route to correct database). Extremely expensive in cloud (100 RDS instances). **Rejected for MVP and initial growth stage.**

### Option B: Schema-Per-Tenant

All tenants share one PostgreSQL instance but each has a dedicated schema (`tenant_01hxyz.events`, `tenant_01hxyz.attendees`, etc.).

**Pros**: Better isolation than RLS; independent schema migrations theoretically possible; shared instance economics.

**Cons**: PostgreSQL has no native schema-per-tenant optimization. EF Core does not natively support dynamic schema selection (requires complex DbContext factory). Schema migrations must still run per-tenant (N migrations per deployment). Performance degrades with large numbers of schemas. **Rejected — complexity without meaningful benefit over RLS.**

### Option C: Shared Database with EF Core Global Query Filters + Row-Level Security (Defense in Depth)

All tenants share tables. `tenant_id` column on every tenant-scoped table. EF Core global query filters automatically append `WHERE tenant_id = @currentTenantId` to every query. PostgreSQL RLS policies enforce isolation at the database layer as an additional safeguard.

**Pros**: Operationally simple (one database, one migration run). EF Core global filters are well-supported. RLS provides defense-in-depth — even if the application layer has a bug, RLS prevents data leakage at the DB level. Cost-efficient.

**Cons**: Tenants share infrastructure resources. Large tenant queries can impact smaller tenants (noisy neighbor). **Mitigated** by connection pooling limits per tenant and query timeout policies.

**Decision**: Option C — Shared database with EF Core global query filters + PostgreSQL RLS.

---

## Decision

**Shared database. EF Core global query filters as the primary isolation mechanism. PostgreSQL Row-Level Security (RLS) as defense-in-depth. `org_id` JWT claim → `ITenantContext` → EF Core filter → RLS policy.**

---

## Data Flow: org_id → TenantContext → Database

```
HTTP Request
    │
    ▼
[JWT Bearer Token]
  Contains: org_id = "ten_01HXYZ..."
    │
    ▼
[TenantResolutionMiddleware]   (Api/Middleware/TenantResolutionMiddleware.cs)
  Extracts org_id from JWT claim
  Validates: org_id is a valid UUID
  Populates: ITenantContext.TenantId = Guid.Parse(orgId)
    │
    ▼
[ITenantContext]               (Application/Common/Interfaces/ITenantContext.cs)
  Scoped service — lives for the duration of the HTTP request
  Properties: TenantId, UserId, UserEmail, Roles
    │
    ▼
[MediatR Handler]
  Injects ITenantContext
  NEVER passes TenantId as a query parameter — ITenantContext provides it
    │
    ▼
[EventFlowDbContext]
  Global query filters reference ITenantContext.TenantId
  EF Core appends: AND events.tenant_id = '01HXYZ...' to every query
    │
    ▼
[EF Core → Npgsql → PostgreSQL]
  Before executing SQL: DbConnection interceptor runs:
  SET LOCAL app.current_tenant_id = '01HXYZ...'
  PostgreSQL RLS policy reads this setting
  RLS adds another WHERE tenant_id = current_setting('app.current_tenant_id')
  ↑ SECOND LAYER — independent of application code
    │
    ▼
[PostgreSQL Table Data]
  Returns ONLY rows matching tenant_id
  Two independent enforcement layers — belt AND suspenders
```

---

## Implementation

### 1. ITenantContext

```csharp
// Application/Common/Interfaces/ITenantContext.cs
public interface ITenantContext
{
    Guid TenantId { get; set; }
    Guid UserId { get; set; }
    string? UserEmail { get; set; }
    IReadOnlyList<string> Roles { get; set; }
    bool IsAuthenticated { get; }
}

// Implementation registered as Scoped
// Application/Common/Services/TenantContext.cs
public class TenantContext : ITenantContext
{
    public Guid TenantId { get; set; }
    public Guid UserId { get; set; }
    public string? UserEmail { get; set; }
    public IReadOnlyList<string> Roles { get; set; } = [];
    public bool IsAuthenticated => TenantId != Guid.Empty;
}
```

### 2. TenantResolutionMiddleware

```csharp
// Api/Middleware/TenantResolutionMiddleware.cs
public class TenantResolutionMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<TenantResolutionMiddleware> _logger;

    public TenantResolutionMiddleware(
        RequestDelegate next,
        ILogger<TenantResolutionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(
        HttpContext context,
        ITenantContext tenantContext)
    {
        if (context.User.Identity?.IsAuthenticated != true)
        {
            await _next(context);
            return;
        }

        var orgIdClaim = context.User.FindFirst("org_id")?.Value;

        if (string.IsNullOrWhiteSpace(orgIdClaim) ||
            !Guid.TryParse(orgIdClaim, out var tenantId))
        {
            _logger.LogWarning(
                "Authenticated request missing valid org_id claim. User: {UserId}",
                context.User.FindFirst("sub")?.Value);

            context.Response.StatusCode = StatusCodes.Status403Forbidden;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Invalid or missing org_id claim."
            });
            return;
        }

        tenantContext.TenantId = tenantId;
        tenantContext.UserId = Guid.Parse(
            context.User.FindFirst("sub")!.Value);
        tenantContext.UserEmail = context.User.FindFirst("email")?.Value;
        tenantContext.Roles = context.User
            .FindAll("roles")
            .Select(c => c.Value)
            .ToList();

        using (_logger.BeginScope(new Dictionary<string, object>
        {
            ["TenantId"] = tenantId,
            ["UserId"] = tenantContext.UserId
        }))
        {
            await _next(context);
        }
    }
}
```

### 3. EF Core Global Query Filters

```csharp
// Infrastructure/Persistence/EventFlowDbContext.cs
public class EventFlowDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    public EventFlowDbContext(
        DbContextOptions<EventFlowDbContext> options,
        ITenantContext tenantContext)
        : base(options)
    {
        _tenantContext = tenantContext;
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply global query filters to ALL tenant-scoped entities
        // These filters are automatically appended to EVERY query
        // including those in handlers — no handler needs to manually add WHERE tenant_id

        var tenantScopedEntities = new[]
        {
            typeof(Event), typeof(Session), typeof(Attendee),
            typeof(Registration), typeof(Speaker), typeof(Sponsor),
            typeof(SponsorTier), typeof(Campaign), typeof(Ticket),
            typeof(TicketType), typeof(CheckIn), typeof(Venue)
        };

        foreach (var entityType in tenantScopedEntities)
        {
            modelBuilder.Entity(entityType)
                .HasQueryFilter(BuildTenantFilter(entityType));
        }

        // Tenant entity itself is NOT filtered (fetched by ID in provisioning)
        // AuditLog is NOT filtered — admin can see all tenant audit logs

        ApplyEntityConfigurations(modelBuilder);
        base.OnModelCreating(modelBuilder);
    }

    private LambdaExpression BuildTenantFilter(Type entityType)
    {
        // Builds: e => e.TenantId == _tenantContext.TenantId
        var parameter = Expression.Parameter(entityType, "e");
        var tenantIdProperty = Expression.Property(parameter, "TenantId");
        var tenantIdValue = Expression.Property(
            Expression.Constant(_tenantContext),
            nameof(ITenantContext.TenantId));
        var equality = Expression.Equal(tenantIdProperty, tenantIdValue);
        return Expression.Lambda(equality, parameter);
    }

    public override Task<int> SaveChangesAsync(CancellationToken ct = default)
    {
        // Auto-populate TenantId on new entities
        foreach (var entry in ChangeTracker.Entries<ITenantScoped>()
            .Where(e => e.State == EntityState.Added))
        {
            entry.Entity.TenantId = _tenantContext.TenantId;
        }

        // Auto-populate audit fields
        foreach (var entry in ChangeTracker.Entries<AuditableEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedAt = DateTime.UtcNow;
                entry.Entity.CreatedBy = _tenantContext.UserId;
            }
            else if (entry.State == EntityState.Modified)
            {
                entry.Entity.UpdatedAt = DateTime.UtcNow;
                entry.Entity.UpdatedBy = _tenantContext.UserId;
            }
        }

        return base.SaveChangesAsync(ct);
    }
}
```

### 4. PostgreSQL RLS — Defense in Depth

```sql
-- Migration: EnableRowLevelSecurity.sql
-- Applied to ALL tenant-scoped tables

-- Example for events table (repeated for all tenant tables)
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
ALTER TABLE events FORCE ROW LEVEL SECURITY;  -- Applies to table owner too

CREATE POLICY eventflow_tenant_isolation ON events
    AS PERMISSIVE
    FOR ALL
    TO eventflow_app_user  -- The application DB user (not superuser)
    USING (
        tenant_id = current_setting('app.current_tenant_id', true)::uuid
    );

-- Repeat for all tenant-scoped tables:
-- sessions, attendees, registrations, speakers, sponsors, sponsor_tiers,
-- campaigns, tickets, ticket_types, check_ins, venues

-- Create application DB user with limited privileges (not superuser)
CREATE ROLE eventflow_app_user LOGIN PASSWORD '${APP_DB_PASSWORD}';
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public
    TO eventflow_app_user;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public
    TO eventflow_app_user;
-- eventflow_app_user does NOT have BYPASSRLS privilege
```

### 5. DbConnection Interceptor — Set RLS Context Variable

```csharp
// Infrastructure/Persistence/TenantConnectionInterceptor.cs
// Sets the PostgreSQL session variable that RLS policies read
public class TenantConnectionInterceptor : DbConnectionInterceptor
{
    private readonly ITenantContext _tenantContext;

    public TenantConnectionInterceptor(ITenantContext tenantContext)
    {
        _tenantContext = tenantContext;
    }

    public override async Task ConnectionOpenedAsync(
        DbConnection connection,
        ConnectionEndEventData eventData,
        CancellationToken ct)
    {
        if (_tenantContext.IsAuthenticated)
        {
            // Set RLS context variable for this connection
            // Using SET LOCAL (transaction-scoped, not session-scoped)
            // This ensures the variable is cleared after the transaction
            await using var cmd = connection.CreateCommand();
            cmd.CommandText =
                $"SET LOCAL app.current_tenant_id = '{_tenantContext.TenantId}'";
            await cmd.ExecuteNonQueryAsync(ct);
        }
    }
}

// Registration in InfrastructureServiceExtensions.cs:
// services.AddDbContext<EventFlowDbContext>(options =>
// {
//     options.UseNpgsql(connectionString);
//     options.AddInterceptors(
//         serviceProvider.GetRequiredService<TenantConnectionInterceptor>());
// });
// services.AddScoped<TenantConnectionInterceptor>();
```

### 6. Bypassing Global Filters (Admin Operations Only)

Certain platform-level operations (tenant provisioning, admin audit, billing) must bypass tenant filters:

```csharp
// ONLY used in admin/platform-level handlers
// NEVER in tenant-scoped request handlers
public class GetAllTenantsForBillingHandler
    : IRequestHandler<GetAllTenantsForBillingQuery, IReadOnlyList<TenantBillingDto>>
{
    private readonly EventFlowDbContext _db;

    public async Task<IReadOnlyList<TenantBillingDto>> Handle(
        GetAllTenantsForBillingQuery query,
        CancellationToken ct)
    {
        // IgnoreQueryFilters() is EXPLICITLY documented and code-reviewed
        // Any use of IgnoreQueryFilters() outside admin handlers is a security finding
        var tenants = await _db.Tenants
            .IgnoreQueryFilters()  // Admin operation — intentional bypass
            .Select(t => t.ToBillingDto())
            .ToListAsync(ct);

        return tenants;
    }
}

// Enforced via architecture test (ArchUnit equivalent for .NET):
// [Fact]
// public void OnlyAdminHandlers_ShouldUseIgnoreQueryFilters()
// {
//     // Scan all handler types
//     // Assert: any method containing IgnoreQueryFilters()
//     //         must be in a class decorated with [AdminOperation]
// }
```

---

## Tenant Entity

```csharp
// Domain/Entities/Tenant.cs
public class Tenant
{
    public Guid Id { get; private set; }
    public string Name { get; private set; } = string.Empty;
    public string Slug { get; private set; } = string.Empty;  // URL-safe identifier
    public TenantPlan Plan { get; private set; }               // Free, Starter, Growth, Enterprise
    public TenantStatus Status { get; private set; }           // Active, Suspended, Cancelled
    public DateTime ProvisionedAt { get; private set; }
    public DateTime? SuspendedAt { get; private set; }
    public TenantTheme? Theme { get; private set; }            // White-label configuration
    public TenantSettings Settings { get; private set; } = new();
    public int MaxEventsPerYear { get; private set; }          // Plan-based limit
    public int MaxAttendeesPerEvent { get; private set; }      // Plan-based limit

    // NOT tenant-scoped (no TenantId FK) — this IS the top-level entity
}

// Domain/Entities/TenantTheme.cs — White-label settings
public class TenantTheme
{
    public Guid TenantId { get; private set; }
    public string PrimaryColor { get; private set; } = "#6366F1";    // Default brand indigo
    public string SecondaryColor { get; private set; } = "#8B5CF6";
    public string? LogoUrl { get; private set; }
    public string? FaviconUrl { get; private set; }
    public string? CustomDomain { get; private set; }                  // e.g., events.acme.com
    public string? CustomFont { get; private set; }
    public bool DarkModeEnabled { get; private set; } = true;
}
```

---

## Tenant Provisioning Flow

```csharp
// Application/Features/Tenants/Commands/ProvisionTenant/ProvisionTenantHandler.cs
// Called when a new organization signs up
public class ProvisionTenantHandler
    : IRequestHandler<ProvisionTenantCommand, TenantDto>
{
    public async Task<TenantDto> Handle(
        ProvisionTenantCommand command,
        CancellationToken ct)
    {
        // 1. Create tenant record in PostgreSQL
        var tenant = Tenant.Create(
            command.OrganizationName,
            command.AdminEmail,
            TenantPlan.Starter);

        _db.Tenants.Add(tenant);

        // 2. Create Keycloak user in the eventflow realm
        //    Assign org_id attribute = tenant.Id
        await _keycloakAdminService.CreateTenantAdminUserAsync(
            tenantId: tenant.Id,
            email: command.AdminEmail,
            name: command.AdminName,
            ct: ct);

        // 3. Commit (atomic: tenant record + audit log)
        await _db.SaveChangesAsync(ct);

        // 4. Publish TenantProvisioned event
        //    → email-consumer sends welcome email
        //    → analytics-consumer initializes tenant metrics
        await _eventBus.PublishAsync(
            new TenantProvisioned(tenant.Id, tenant.Name, command.AdminEmail), ct);

        return tenant.ToDto();
    }
}
```

---

## Plan-Based Feature Limits

Multi-tenancy includes plan enforcement. Limits are checked in MediatR handlers:

```csharp
// Application/Features/Events/Commands/CreateEvent/CreateEventHandler.cs
public async Task<EventDto> Handle(
    CreateEventCommand command,
    CancellationToken ct)
{
    // Check plan limits before creating
    var tenant = await _db.Tenants
        .FirstOrDefaultAsync(t => t.Id == _tenantContext.TenantId, ct)
        ?? throw new NotFoundException("Tenant not found.");

    var currentEventCount = await _db.Events
        .CountAsync(e => e.CreatedAt.Year == DateTime.UtcNow.Year, ct);

    if (currentEventCount >= tenant.MaxEventsPerYear)
        throw new PlanLimitExceededException(
            $"Your {tenant.Plan} plan allows {tenant.MaxEventsPerYear} events per year. "
          + "Upgrade to create more events.");

    // Proceed with event creation...
}
```

---

## Security Audit: Three Layers of Tenant Isolation

| Layer | Mechanism | Enforced By | Bypass Risk |
|---|---|---|---|
| **Layer 1** | JWT `org_id` claim validation | `TenantResolutionMiddleware` | JWT forgery (mitigated by RS256 signature) |
| **Layer 2** | EF Core global query filters | `EventFlowDbContext.OnModelCreating()` | Developer accidentally calling `IgnoreQueryFilters()` (mitigated by architecture tests) |
| **Layer 3** | PostgreSQL RLS policies | Database engine (independent of application) | Requires DB superuser credentials (never given to app) |

For a cross-tenant data breach to occur, **all three layers** must be simultaneously compromised. This is the defense-in-depth principle applied to multi-tenancy.

### Security Testing Requirements

```csharp
// IntegrationTests/Features/TenantIsolationTests.cs
// These tests must NEVER be removed or skipped

[Fact]
public async Task GetEvents_WithTenantAJwt_ShouldNotReturnTenantBEvents()
{
    // Arrange: Create events for Tenant A and Tenant B
    // Act: Call GET /api/events with Tenant A's JWT
    // Assert: Response contains ONLY Tenant A's events
    //         Response does NOT contain any Tenant B event IDs
}

[Fact]
public async Task GetEventById_WithWrongTenantJwt_ShouldReturn404NotFound()
{
    // Arrange: Event belongs to Tenant B
    // Act: Call GET /api/events/{tenantBEventId} with Tenant A's JWT
    // Assert: 404 Not Found (not 403 — we don't leak the existence of the resource)
}

[Fact]
public async Task UpdateEvent_WithWrongTenantJwt_ShouldReturn404NotFound()
{
    // Cross-tenant write attempt must fail
}

[Fact]
public async Task CheckIn_ForAnotherTenantsAttendee_ShouldReturn404()
{
    // Cross-tenant check-in attempt must fail
}
```

---

## Consequences

### Positive
- **Operational simplicity**: One database, one migration run per deployment. Adding a tenant requires no infrastructure provisioning.
- **Cost efficiency**: All tenants share one PostgreSQL instance. Infrastructure cost does not scale linearly with tenant count.
- **Defense in depth**: Three independent isolation layers make cross-tenant data leakage practically impossible.
- **EF Core native**: Global query filters are a first-class EF Core feature — well-tested, well-documented, no custom hacks.
- **Upgrade path**: If a specific enterprise customer requires dedicated infrastructure (data residency compliance), database-per-tenant can be offered as a premium tier using the same codebase with a different `DbContextOptions` configuration.

### Negative
- **Noisy neighbor risk**: A tenant running a very large report query consumes PostgreSQL resources that affect all tenants. Mitigation: Per-tenant query statement timeout (`SET statement_timeout`), connection pool limits per tenant, dedicated read replica for analytics.
- **Schema migration coupling**: Migrations affect all tenants simultaneously. A failed migration blocks all tenants. Mitigation: Blue-green deployment, backward-compatible migrations (additive only), feature flags to gate new schema-dependent features.
- **RLS complexity**: RLS policies must be carefully maintained when schema changes occur. Mitigation: Migration tests validate RLS policies are still applied after each migration.
- **`IgnoreQueryFilters()` misuse risk**: A developer could accidentally bypass tenant filters. Mitigation: Architecture unit tests scan the codebase for `IgnoreQueryFilters()` usage and assert it only appears in admin-scoped handlers.

---

## References
- [EF Core Global Query Filters](https://learn.microsoft.com/en-us/ef/core/querying/filters)
- [PostgreSQL Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Multi-tenant SaaS patterns — Martin Fowler](https://martinfowler.com/articles/patterns-of-distributed-systems/)
- [SET LOCAL command — PostgreSQL](https://www.postgresql.org/docs/current/sql-set.html)
