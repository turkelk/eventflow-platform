# Wave 2: Domain & Data

## 2.1 Implement core Domain entities: Organization, OrganizationMember, SubscriptionPlan

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:m`, `agent:backend`
**Status:** Pending

Create the Identity & Tenancy domain entities following the Data Model and 02-DOMAIN-MODEL.md.

**Key files:**
- `src/EventFlow.Domain/Common/AggregateRoot.cs`
- `src/EventFlow.Domain/Common/Entity.cs`
- `src/EventFlow.Domain/Common/IDomainEvent.cs`
- `src/EventFlow.Domain/Identity/Tenant.cs` (maps to Organization entity)
- `src/EventFlow.Domain/Identity/TenantMember.cs` (maps to OrganizationMember)
- `src/EventFlow.Infrastructure/Persistence/Configurations/TenantConfiguration.cs`
- `src/EventFlow.Infrastructure/Persistence/Configurations/TenantMemberConfiguration.cs`

**Acceptance criteria:**
- [ ] `Tenant` aggregate root enforces slug uniqueness and plan limits as domain invariants
- [ ] All entities use `Guid` PKs with UUID v7 or `uuid-ossp` generation
- [ ] EF Core configurations define all FK relationships and indexes from Data Model
- [ ] `AggregateRoot<T>` base class collects domain events via `AddDomainEvent()`
- [ ] Value objects (`TenantTheme`, `BillingInfo`) are immutable records

---

## 2.2 Implement Event aggregate and related entities: Session, Track, TicketType, Speaker

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:m`, `agent:backend`
**Status:** Pending

Create the Event Management domain entities from the Data Model and 02-DOMAIN-MODEL.md section 2.2.

**Key files:**
- `src/EventFlow.Domain/Events/Event.cs` (aggregate root with domain event collection)
- `src/EventFlow.Domain/Events/Session.cs`
- `src/EventFlow.Domain/Events/Track.cs`
- `src/EventFlow.Domain/Events/TicketType.cs`
- `src/EventFlow.Domain/Speakers/Speaker.cs`
- `src/EventFlow.Domain/Events/EventSpeaker.cs`
- `src/EventFlow.Infrastructure/Persistence/Configurations/EventConfiguration.cs`
- `src/EventFlow.Infrastructure/Persistence/Configurations/SessionConfiguration.cs`

**Acceptance criteria:**
- [ ] `Event` enforces: EndDate > StartDate, Capacity > 0, status transition rules
- [ ] `TicketType` invariant: `QuantitySold` cannot exceed `Quantity`
- [ ] `EventStatus` enum: Draft, Published, Live, Completed, Cancelled
- [ ] All JSONB columns (`CustomFields`, `AiMetadata`) mapped with `HasColumnType("jsonb")`
- [ ] `EventPublished` domain event raised on `Event.Publish()` call

---

## 2.3 Implement Registration, CheckInRecord, Attendee, and Waitlist entities

**Labels:** `type:feature`, `layer:backend`, `priority:critical`, `size:m`, `agent:backend`
**Status:** Pending

Create Attendee & Registration context entities from the Data Model and 02-DOMAIN-MODEL.md section 2.3.

**Key files:**
- `src/EventFlow.Domain/Registrations/Registration.cs` (aggregate root)
- `src/EventFlow.Domain/Registrations/CheckInRecord.cs`
- `src/EventFlow.Domain/Registrations/SessionRegistration.cs`
- `src/EventFlow.Domain/Registrations/WaitlistEntry.cs`
- `src/EventFlow.Domain/ValueObjects/MatchingProfile.cs`
- `src/EventFlow.Infrastructure/Persistence/Configurations/RegistrationConfiguration.cs`
- `src/EventFlow.Infrastructure/Persistence/Configurations/AttendeeConfiguration.cs`

**Acceptance criteria:**
- [ ] `Registration` aggregate enforces status transitions (Pending→Confirmed, no re-confirm after Cancel)
- [ ] `ConfirmationCode` generated as `EVF-{year}-{random5}` format unique per event
- [ ] `QrCodeToken` generated as UUID, unique constraint in DB
- [ ] `CheckInRecord.DeviceIdentifier` field supports offline deduplication
- [ ] All domain events defined: `RegistrationCreatedEvent`, `AttendeeCheckedInEvent`

---

## 2.4 Implement Venue, Sponsor, Campaign, and supporting entities

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Create remaining domain entities from Data Model sections: Venue, Sponsorship, Communications, Analytics, AI.

**Key files:**
- `src/EventFlow.Domain/Venues/Venue.cs` (aggregate root)
- `src/EventFlow.Domain/Venues/VenueRoom.cs`
- `src/EventFlow.Domain/Sponsors/Sponsor.cs`
- `src/EventFlow.Domain/Sponsors/EventSponsor.cs`
- `src/EventFlow.Domain/Campaigns/Campaign.cs`
- `src/EventFlow.Domain/Campaigns/EmailTemplate.cs`
- `src/EventFlow.Domain/AI/AIAgendaSuggestion.cs`
- `src/EventFlow.Domain/AI/MatchSuggestion.cs`

**Acceptance criteria:**
- [ ] `Campaign` enforces: cannot update a Sent campaign
- [ ] `SponsorTier` entity linked to Event with sort order for drag-and-drop
- [ ] `AIAgendaSuggestion` tracks `TokensUsed` for cost tracking
- [ ] `AuditLog` entity has no tenant-scoped global filter (admin visibility)
- [ ] All EF Core `IEntityTypeConfiguration<T>` files created for each entity

---

## 2.5 Create EventFlowDbContext with multi-tenant global query filters and EF Core migrations

**Labels:** `type:feature`, `layer:database`, `priority:critical`, `size:l`, `agent:backend`
**Status:** Pending

Implement the DbContext with tenant isolation following ADR-001 and ADR-006.

**Key files:**
- `src/EventFlow.Infrastructure/Persistence/EventFlowDbContext.cs`
- `src/EventFlow.Application/Common/Interfaces/ITenantContext.cs`
- `src/EventFlow.Infrastructure/Persistence/TenantConnectionInterceptor.cs`
- `src/EventFlow.Infrastructure/Persistence/Migrations/` (initial migration)
- `src/EventFlow.Infrastructure/Extensions/InfrastructureServiceExtensions.cs`

**Acceptance criteria:**
- [ ] Global query filter `WHERE tenant_id = @currentTenantId` on all 20 tenant-scoped entities
- [ ] `SaveChangesAsync` auto-populates `TenantId`, `CreatedAt`, `UpdatedBy` on entity add/modify
- [ ] `TenantConnectionInterceptor` sets `SET LOCAL app.current_tenant_id` on each connection open
- [ ] PostgreSQL extensions enabled: `uuid-ossp`, `pgcrypto`, `vector`, `pg_trgm`
- [ ] All composite indexes from ADR-001 created in migration (tenant+status+date, name trgm, etc.)

---

## 2.6 Setup PostgreSQL Row-Level Security policies for defense-in-depth tenant isolation

**Labels:** `type:security`, `layer:database`, `priority:critical`, `size:m`, `agent:backend`
**Status:** Pending

Implement RLS as the third layer of tenant isolation per ADR-006 security audit.

**Key files:**
- `src/EventFlow.Infrastructure/Persistence/Migrations/EnableRowLevelSecurity.sql`
- `infra/postgres/init/02-rls-policies.sql`
- `infra/postgres/init/03-app-user.sql` (create `eventflow_app_user` without BYPASSRLS)

**SQL per table:**
```sql
ALTER TABLE events ENABLE ROW LEVEL SECURITY;
ALTER TABLE events FORCE ROW LEVEL SECURITY;
CREATE POLICY eventflow_tenant_isolation ON events
  USING (org_id = current_setting('app.current_tenant_id', true)::uuid);
```
(Repeated for all 20 tenant-scoped tables)

**Acceptance criteria:**
- [ ] RLS enabled and forced on all tenant-scoped tables
- [ ] `eventflow_app_user` has no `BYPASSRLS` privilege
- [ ] Cross-tenant query returns 0 rows when `app.current_tenant_id` is set to another tenant
- [ ] Admin operations using `IgnoreQueryFilters()` documented and tested
- [ ] Architecture test asserts `IgnoreQueryFilters()` only in admin handlers

---

## 2.7 Setup Redis connection, distributed cache service, and distributed lock (RedLock)

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement Redis infrastructure services for caching and distributed locking per ADR-005.

**Key files:**
- `src/EventFlow.Application/Common/Interfaces/ICacheService.cs`
- `src/EventFlow.Application/Common/Interfaces/IDistributedLockService.cs`
- `src/EventFlow.Infrastructure/Adapters/CacheService.cs` (StackExchange.Redis)
- `src/EventFlow.Infrastructure/Adapters/DistributedLockService.cs` (RedLock.net)
- `src/EventFlow.Application/Common/Interfaces/ICacheableQuery.cs`

**Acceptance criteria:**
- [ ] `CacheService.GetAsync<T>` / `SetAsync` / `RemoveAsync` / `RemoveByPatternAsync` implemented
- [ ] Cache keys follow format `{tenantId}:{entity}:{qualifier}`
- [ ] `DistributedLockService.AcquireAsync` uses RedLock pattern with 5s expiry
- [ ] `ICacheableQuery` interface defines `CacheKey`, `CacheDuration`, `BypassCache`
- [ ] Redis connection string supports auth password from env var

---

## 2.8 Setup Redis Streams event bus and outbox pattern

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:l`, `agent:backend`
**Status:** Pending

Implement the integration event bus using Redis Streams with the transactional outbox pattern from ADR-004.

**Key files:**
- `src/EventFlow.Application/Common/Interfaces/IEventBus.cs`
- `src/EventFlow.Infrastructure/Adapters/EventBusService.cs` (Redis Streams XADD)
- `src/EventFlow.Infrastructure/BackgroundServices/EventStreamConsumer.cs` (consumer groups)
- `src/EventFlow.Infrastructure/BackgroundServices/OutboxProcessorService.cs`
- `src/EventFlow.Domain/Common/OutboxMessage.cs`
- `src/EventFlow.Infrastructure/BackgroundServices/RealtimeEventConsumer.cs`

**Acceptance criteria:**
- [ ] `EventBusService.PublishAsync` uses XADD with CloudEvents envelope schema
- [ ] `OutboxProcessorService` polls every 1s, acquires distributed lock before processing
- [ ] Consumer groups created with `BUSYGROUP` error handling on restart
- [ ] Dead letter stream `eventflow:dlq` receives messages after 3 failed attempts
- [ ] `SaveChangesAsync` in DbContext writes outbox records atomically with entity changes

---

## 2.9 Setup MinIO S3-compatible file storage service

**Labels:** `type:feature`, `layer:backend`, `priority:high`, `size:m`, `agent:backend`
**Status:** Pending

Implement file upload/download/delete via MinIO with pre-signed URLs per the infrastructure design.

**Key files:**
- `src/EventFlow.Application/Common/Interfaces/IFileStorage.cs`
- `src/EventFlow.Infrastructure/Adapters/FileStorageService.cs` (AWSSDK.S3 or Minio SDK)
- `src/EventFlow.Api/Controllers/FilesController.cs` (GET /api/v1/files/upload-url, POST /api/v1/files/confirm-upload)

**Acceptance criteria:**
- [ ] `GetPresignedUploadUrlAsync(fileName, contentType, context)` returns signed URL with 15min expiry
- [ ] `ConfirmUploadAsync` verifies SHA-256 checksum after upload
- [ ] File type validated by magic bytes (not just extension)
- [ ] Max file size 50MB enforced at URL generation
- [ ] Bucket path structure: `{context}/{tenantId}/{entityId}/{filename}`

---

