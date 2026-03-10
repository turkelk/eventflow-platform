# ADR-001: Database Selection — PostgreSQL

**Status**: Accepted  
**Date**: 2025-01-01  
**Deciders**: Architecture Team  
**Consulted**: Product, Engineering Leads  

---

## Context

EventFlow is a multi-tenant SaaS event management platform serving mid-market organizations running 10–200 events per year. The data model is rich and relational: an Event has many Sessions, Speakers, Attendees, Sponsors, Tickets, and Communications. Attendees register for specific Sessions. Sponsors belong to Tiers within Events. Analytics aggregate across Events within a Tenant organization.

We evaluated two primary candidates:

1. **PostgreSQL** — battle-tested relational RDBMS with JSONB support, Row-Level Security, strong ACID guarantees, and a mature .NET ecosystem (Npgsql, EF Core).
2. **MongoDB (Document DB)** — flexible schema, native JSON storage, horizontal sharding.

Key domain characteristics that inform this decision:

- **Highly relational data**: Events → Sessions → Speakers, Attendees → Registrations → Tickets, Sponsors → Tiers → Events. These are normalized relationships requiring JOIN-efficient queries. A document store would force us to denormalize aggressively or perform expensive application-side joins.
- **Multi-tenancy with Row-Level Security**: PostgreSQL's native RLS policies provide a defense-in-depth tenant isolation layer that is difficult to replicate in MongoDB without application-layer enforcement only.
- **Complex analytics queries**: Post-event analytics require aggregations across Attendees, Sessions, Check-ins, and Revenue. PostgreSQL's window functions, CTEs, and aggregation pipeline are first-class. MongoDB's aggregation pipeline is capable but less ergonomic for relational aggregations.
- **ACID transactions**: Operations like "register an attendee" must atomically create a Registration, decrement available Tickets, and publish a domain event. PostgreSQL's multi-statement transactions are essential.
- **AI/Vector capabilities**: EventFlow's AI matchmaking and recommendation features will eventually require vector similarity search. PostgreSQL with the `pgvector` extension is the simplest path — no additional infrastructure required. MongoDB Atlas Vector Search would require a separate Atlas deployment.
- **Semi-structured data**: Event metadata, speaker bios, sponsor benefits, and custom registration form fields benefit from flexible schema. PostgreSQL's `JSONB` column type handles this without sacrificing relational integrity — no need to move to a document store for this alone.
- **Team expertise**: The engineering team has strong PostgreSQL and EF Core expertise. MongoDB expertise is limited, introducing a ramp-up risk.

---

## Decision

**PostgreSQL 16+ is the primary and sole database for EventFlow.**

MongoDB and other document stores are explicitly rejected for the reasons detailed in the Context section.

---

## Implementation Details

### EF Core Configuration

```csharp
// Infrastructure/Persistence/EventFlowDbContext.cs
public class EventFlowDbContext : DbContext
{
    private readonly ITenantContext _tenantContext;

    public DbSet<Event> Events => Set<Event>();
    public DbSet<Session> Sessions => Set<Session>();
    public DbSet<Attendee> Attendees => Set<Attendee>();
    public DbSet<Registration> Registrations => Set<Registration>();
    public DbSet<Speaker> Speakers => Set<Speaker>();
    public DbSet<Sponsor> Sponsors => Set<Sponsor>();
    public DbSet<Venue> Venues => Set<Venue>();
    public DbSet<Campaign> Campaigns => Set<Campaign>();
    public DbSet<Ticket> Tickets => Set<Ticket>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Global query filter — enforces tenant isolation at the ORM layer
        // Every query automatically includes WHERE tenant_id = @tenantId
        modelBuilder.Entity<Event>()
            .HasQueryFilter(e => e.TenantId == _tenantContext.TenantId);
        // Applied to ALL tenant-scoped entities

        // JSONB columns for flexible metadata
        modelBuilder.Entity<Event>()
            .Property(e => e.CustomFields)
            .HasColumnType("jsonb");

        modelBuilder.Entity<Session>()
            .Property(s => s.AiMetadata)
            .HasColumnType("jsonb");

        base.OnModelCreating(modelBuilder);
    }
}
```

### PostgreSQL Extensions

```sql
-- Applied via EF Core migration
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";    -- UUID primary keys
CREATE EXTENSION IF NOT EXISTS "pgcrypto";     -- Cryptographic functions
CREATE EXTENSION IF NOT EXISTS "vector";       -- pgvector for AI embeddings
CREATE EXTENSION IF NOT EXISTS "pg_trgm";      -- Trigram indexes for full-text search
```

### Index Strategy

```sql
-- Composite index for tenant-scoped event queries (most frequent query pattern)
CREATE INDEX CONCURRENTLY idx_events_tenant_status_date
    ON events (tenant_id, status, start_date_utc)
    WHERE deleted_at IS NULL;

-- Trigram index for attendee name search (powers command palette search)
CREATE INDEX CONCURRENTLY idx_attendees_name_trgm
    ON attendees USING gin (full_name gin_trgm_ops)
    WHERE tenant_id IS NOT NULL;

-- Vector index for AI attendee matchmaking
CREATE INDEX CONCURRENTLY idx_attendees_embedding
    ON attendees USING ivfflat (profile_embedding vector_cosine_ops)
    WITH (lists = 100);

-- Partial index for active registrations
CREATE INDEX CONCURRENTLY idx_registrations_active
    ON registrations (event_id, attendee_id)
    WHERE status = 'confirmed' AND cancelled_at IS NULL;
```

### Row-Level Security (Defense in Depth)

```sql
-- RLS as a second layer of tenant isolation behind EF Core global filters
-- Protects against application-layer bugs that bypass the ORM
ALTER TABLE events ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON events
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Set tenant context at connection time in Infrastructure layer
-- SET LOCAL app.current_tenant_id = '<tenant-id>';
```

### JSONB Usage Policy

JSONB columns are used **only** for genuinely schema-variable data:
- `events.custom_fields` — organizer-defined custom registration form fields
- `sessions.ai_metadata` — AI-generated session scoring and tags (evolves frequently)
- `attendees.profile_data` — enriched profile data from third-party integrations
- `campaigns.template_variables` — dynamic email/SMS merge variables

JSONB is **not** used as a workaround for lazy schema design. All known, stable fields are proper typed columns.

### Connection Pooling

PgBouncer in transaction pooling mode sits between the application and PostgreSQL. This is critical for a multi-tenant SaaS where connection counts scale with tenant count, not just request rate.

```yaml
# docker-compose.yml — pgbouncer service
pgbouncer:
  image: pgbouncer/pgbouncer:1.22
  environment:
    DATABASES_HOST: postgres
    DATABASES_PORT: 5432
    DATABASES_DBNAME: eventflow
    PGBOUNCER_POOL_MODE: transaction
    PGBOUNCER_MAX_CLIENT_CONN: 1000
    PGBOUNCER_DEFAULT_POOL_SIZE: 25
```

### Migration Strategy

EF Core Migrations with a custom migration runner that:
1. Acquires a distributed lock (Redis) before running migrations
2. Runs migrations in a transaction where possible
3. Records migration history in `__ef_migrations_history`
4. Emits structured logs for each migration step

---

## Consequences

### Positive
- **Strong relational integrity**: Foreign keys, constraints, and transactions enforce data consistency without application-layer workarounds.
- **Native RLS**: Provides defense-in-depth tenant isolation at the database layer — a security guarantee no document store offers without significant custom code.
- **pgvector**: AI matchmaking and agenda recommendation features can use vector similarity search without additional infrastructure.
- **JSONB flexibility**: Semi-structured data (custom fields, AI metadata) is handled natively without abandoning relational benefits.
- **EF Core maturity**: Excellent migration tooling, LINQ provider, and multi-tenant filter support.
- **Operational simplicity**: One database technology to operate, monitor, and tune.
- **Full-text search via pg_trgm**: Powers the command palette search without Elasticsearch overhead.

### Negative
- **Horizontal write scaling**: PostgreSQL scales reads easily (replicas) but write scaling requires sharding (Citus) or application-level partitioning. Mitigation: At mid-market SaaS scale (initial target), a well-tuned primary PostgreSQL instance handles 10,000+ TPS. Partitioning by `tenant_id` deferred until needed.
- **Schema migrations at scale**: Large table migrations (e.g., adding a column to `registrations` with millions of rows) require `CONCURRENTLY` operations and careful timing. Mitigation: Automated migration safety checks in CI pipeline.
- **No native multi-document transactions across collections**: N/A — this is a PostgreSQL strength, not a limitation.

### Neutral
- JSONB query performance is excellent for read-heavy document-style access but slower than dedicated columns for write-heavy JSONB updates. This is acceptable given our JSONB usage policy above.

---

## Alternatives Considered

| Option | Verdict | Reason Rejected |
|---|---|---|
| MongoDB | Rejected | Highly relational domain; no native RLS; application-side JOINs; no pgvector equivalent without Atlas |
| CockroachDB | Rejected | PostgreSQL-compatible but distributed complexity unwarranted at current scale; higher operational overhead |
| MySQL/MariaDB | Rejected | Inferior JSONB support; no RLS; weaker window function support; no pgvector |
| DynamoDB | Rejected | No relational queries; complex data modeling for our domain; vendor lock-in |
| Hybrid (PostgreSQL + MongoDB) | Rejected | Operational complexity outweighs marginal flexibility benefit; JSONB handles our semi-structured needs |

---

## References
- [pgvector extension](https://github.com/pgvector/pgvector)
- [PostgreSQL Row-Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [Npgsql EF Core provider](https://www.npgsql.org/efcore/)
- [PgBouncer documentation](https://www.pgbouncer.org/)
