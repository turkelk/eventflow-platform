# ADR-004: Event-Driven Architecture — Redis Streams

**Status**: Accepted  
**Date**: 2025-01-01  
**Deciders**: Architecture Team  
**Consulted**: Engineering Leads  

---

## Context

EventFlow requires asynchronous event processing for several critical workflows:

- **Registration confirmed** → send confirmation email, update analytics, trigger AI matchmaking
- **Attendee checked in** → update real-time counter, notify organizer dashboard, sync capacity
- **Campaign sent** → track delivery, update engagement metrics
- **Event published** → notify subscribers, generate AI insights baseline
- **Speaker invited** → send invitation email via async queue

These workflows must be decoupled from the synchronous HTTP request path. A user registering for an event should not wait for an email to be sent before receiving their HTTP 201 response.

### Technology Evaluation

| Option | Verdict | Reason |
|---|---|---|
| **Redis Streams** | ✅ Chosen | Already in infrastructure for caching and distributed locks. Consumer groups provide reliable delivery. Appropriate throughput for mid-market SaaS. Zero additional infrastructure. |
| RabbitMQ | Rejected | Additional infrastructure component. Adds operational complexity without meaningful benefit at current scale. Dead lettering is more complex to configure. |
| Apache Kafka | Rejected | Severely over-engineered for current requirements. Requires ZooKeeper/KRaft. Operational burden is massive. Kafka is appropriate at 1M+ msg/sec — we're targeting thousands/sec at most. |
| Azure Service Bus | Rejected | Cloud vendor lock-in. Not suitable for local development without emulation. |
| In-process MediatR notifications | Rejected | Loses messages on application restart. Not suitable for guaranteed delivery. No fan-out to multiple consumers. |

**Redis Streams chosen**: Keeps infrastructure simple (Redis already required), provides consumer groups for reliable delivery, supports fan-out via multiple consumer groups reading the same stream, and is operationally straightforward.

---

## Decision

**Redis Streams as the integration event bus. Consumer groups for reliable at-least-once delivery. Dead letter stream for failed events. Outbox pattern for publish-after-commit guarantee.**

---

## Event Schema

All integration events follow this envelope schema:

```json
{
  "type": "attendee.registered",
  "source": "eventflow.registrations",
  "specversion": "1.0",
  "id": "01HXYZ...",
  "time": "2025-03-15T09:30:00Z",
  "datacontenttype": "application/json",
  "correlationid": "req_01HXYZ...",
  "tenantid": "ten_01HXYZ...",
  "data": {
    "registrationId": "reg_01HXYZ...",
    "attendeeId": "att_01HXYZ...",
    "eventId": "evt_01HXYZ...",
    "attendeeEmail": "maya@acme.com",
    "attendeeName": "Maya Chen",
    "ticketTypeId": "tkt_01HXYZ...",
    "registeredAt": "2025-03-15T09:30:00Z"
  }
}
```

**Schema design principles**:
- `type`: dot-separated noun.verb format (`attendee.registered`, `event.published`, `checkin.completed`)
- `source`: service/module that emitted the event
- `id`: ULID (lexicographically sortable, collision-resistant)
- `correlationid`: HTTP request ID for distributed tracing (propagated from `X-Correlation-ID` header)
- `tenantid`: Always present — consumer group processors enforce tenant context
- `data`: Event-specific payload, typed and documented per event type

---

## Domain Events Catalog

### Event Management Events

| Event Type | Stream Key | Payload Fields | Consumers |
|---|---|---|---|
| `event.created` | `eventflow:events` | eventId, tenantId, createdBy, eventName | analytics-consumer |
| `event.published` | `eventflow:events` | eventId, tenantId, publishedAt, eventName, registrationUrl | notifications-consumer, analytics-consumer |
| `event.cancelled` | `eventflow:events` | eventId, tenantId, cancelledAt, reason | notifications-consumer (bulk email to registrants), analytics-consumer |
| `event.live` | `eventflow:events` | eventId, tenantId, startedAt | checkin-consumer (opens check-in), realtime-consumer |
| `event.completed` | `eventflow:events` | eventId, tenantId, completedAt, totalAttendees | analytics-consumer (trigger post-event report), ai-consumer (generate insights) |

### Registration Events

| Event Type | Stream Key | Payload Fields | Consumers |
|---|---|---|---|
| `attendee.registered` | `eventflow:registrations` | registrationId, attendeeId, eventId, attendeeEmail, ticketTypeId, tenantId | email-consumer (confirmation email), analytics-consumer, ai-consumer (matchmaking refresh) |
| `registration.cancelled` | `eventflow:registrations` | registrationId, attendeeId, eventId, cancelledAt, reason, tenantId | email-consumer (cancellation notification), capacity-consumer (release seat), analytics-consumer |
| `registration.waitlisted` | `eventflow:registrations` | registrationId, attendeeId, eventId, position, tenantId | email-consumer (waitlist confirmation) |
| `waitlist.promoted` | `eventflow:registrations` | registrationId, attendeeId, eventId, tenantId | email-consumer (promotion notification) |

### Check-In Events

| Event Type | Stream Key | Payload Fields | Consumers |
|---|---|---|---|
| `checkin.completed` | `eventflow:checkins` | checkinId, attendeeId, eventId, sessionId?, checkedInAt, method (qr/manual), tenantId | realtime-consumer (push to dashboard WebSocket), analytics-consumer |
| `checkin.batch_synced` | `eventflow:checkins` | eventId, checkins[], syncedAt, tenantId | realtime-consumer, analytics-consumer |

### Communication Events

| Event Type | Stream Key | Payload Fields | Consumers |
|---|---|---|---|
| `campaign.sent` | `eventflow:campaigns` | campaignId, eventId, recipientCount, channel (email/sms), sentAt, tenantId | analytics-consumer |
| `campaign.delivery_updated` | `eventflow:campaigns` | campaignId, delivered, bounced, opened, clicked, tenantId | analytics-consumer |
| `speaker.invited` | `eventflow:speakers` | speakerId, eventId, email, magicLinkToken, expiresAt, tenantId | email-consumer |
| `speaker.accepted` | `eventflow:speakers` | speakerId, eventId, acceptedAt, tenantId | notifications-consumer (notify organizer), analytics-consumer |

### AI Events

| Event Type | Stream Key | Payload Fields | Consumers |
|---|---|---|---|
| `ai.agenda_generated` | `eventflow:ai` | eventId, sessionCount, confidence, tenantId | analytics-consumer (track AI usage), cost-tracking-consumer |
| `ai.matchmaking_run` | `eventflow:ai` | eventId, matchCount, tokensUsed, tenantId | cost-tracking-consumer |

### Tenant Events

| Event Type | Stream Key | Payload Fields | Consumers |
|---|---|---|---|
| `tenant.provisioned` | `eventflow:tenants` | tenantId, orgName, adminEmail, plan | email-consumer (welcome email), analytics-consumer |
| `tenant.theme_updated` | `eventflow:tenants` | tenantId, primaryColor, logoUrl | cache-invalidation-consumer |

---

## Redis Streams Architecture

### Stream Key Naming Convention

```
eventflow:{domain}     → e.g., eventflow:registrations, eventflow:checkins
```

NOTE: Tenant ID is embedded in the message payload, not the stream key. All tenants share streams — consumer groups enforce tenant isolation within message processing.

### Consumer Groups

Each downstream consumer is a separate consumer group on the relevant stream(s):

```
Stream: eventflow:registrations
├── Consumer Group: email-processor      ← Sends confirmation/cancellation emails
├── Consumer Group: analytics-processor  ← Updates registration analytics
└── Consumer Group: ai-processor         ← Refreshes matchmaking recommendations

Stream: eventflow:checkins
├── Consumer Group: realtime-processor   ← Pushes to SignalR hubs (dashboard counter)
└── Consumer Group: analytics-processor  ← Updates check-in analytics

Stream: eventflow:events
├── Consumer Group: notifications-processor  ← Organizer/attendee notifications
└── Consumer Group: analytics-processor
```

### Publisher Implementation

```csharp
// Application/Common/Interfaces/IEventBus.cs
public interface IEventBus
{
    Task PublishAsync<T>(
        T integrationEvent,
        CancellationToken ct = default)
        where T : IntegrationEvent;
}

// Infrastructure/Adapters/EventBusService.cs
public class EventBusService : IEventBus
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<EventBusService> _logger;

    public async Task PublishAsync<T>(
        T integrationEvent,
        CancellationToken ct = default)
        where T : IntegrationEvent
    {
        var db = _redis.GetDatabase();
        var streamKey = ResolveStreamKey(integrationEvent.Type);

        // Redis Streams XADD — appends to stream
        await db.StreamAddAsync(
            streamKey,
            new NameValueEntry[]
            {
                new("type",          integrationEvent.Type),
                new("source",        integrationEvent.Source),
                new("id",            integrationEvent.Id),
                new("time",          integrationEvent.Time.ToString("O")),
                new("correlationid", integrationEvent.CorrelationId),
                new("tenantid",      integrationEvent.TenantId.ToString()),
                new("data",          JsonSerializer.Serialize(integrationEvent.Data))
            },
            maxLength: 10_000,      // Cap stream length — old messages auto-trimmed
            useApproximateMaxLength: true
        );

        _logger.LogInformation(
            "Published {EventType} for tenant {TenantId} to stream {StreamKey}",
            integrationEvent.Type, integrationEvent.TenantId, streamKey);
    }

    private static string ResolveStreamKey(string eventType) => eventType.Split('.')[0] switch
    {
        "attendee" or "registration" or "waitlist" => "eventflow:registrations",
        "checkin"                                  => "eventflow:checkins",
        "event"                                    => "eventflow:events",
        "campaign" or "speaker"                    => "eventflow:campaigns",
        "ai"                                       => "eventflow:ai",
        "tenant"                                   => "eventflow:tenants",
        _                                          => "eventflow:general"
    };
}
```

### Consumer Implementation

```csharp
// Infrastructure/BackgroundServices/EventStreamConsumer.cs
public class EventStreamConsumer : BackgroundService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<EventStreamConsumer> _logger;

    private const string ConsumerGroup = "email-processor";
    private const string ConsumerName = "eventflow-api-1"; // Pod name in K8s
    private static readonly string[] Streams =
    [
        "eventflow:registrations",
        "eventflow:speakers",
        "eventflow:events",
    ];

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var db = _redis.GetDatabase();

        // Create consumer groups if they don't exist
        foreach (var stream in Streams)
        {
            try
            {
                await db.StreamCreateConsumerGroupAsync(
                    stream, ConsumerGroup, "$",
                    createStream: true);
            }
            catch (RedisException ex) when (ex.Message.Contains("BUSYGROUP"))
            {
                // Group already exists — expected on restart
            }
        }

        while (!ct.IsCancellationRequested)
        {
            var streamPositions = Streams
                .Select(s => new StreamPosition(s, ">")) // ">" = only unread messages
                .ToArray();

            var results = await db.StreamReadGroupAsync(
                streamPositions,
                ConsumerGroup,
                ConsumerName,
                count: 10);

            foreach (var result in results)
            {
                foreach (var message in result.Entries)
                {
                    await ProcessMessageWithRetryAsync(db, result.Key, message, ct);
                }
            }

            // Process pending (unacknowledged) messages from previous crashes
            await ProcessPendingMessagesAsync(db, ct);

            await Task.Delay(100, ct); // Polling interval — adjust based on load
        }
    }

    private async Task ProcessMessageWithRetryAsync(
        IDatabase db,
        RedisKey streamKey,
        StreamEntry message,
        CancellationToken ct)
    {
        var retryCount = GetRetryCount(message);

        try
        {
            using var scope = _scopeFactory.CreateScope();
            var dispatcher = scope.ServiceProvider
                .GetRequiredService<IIntegrationEventDispatcher>();

            await dispatcher.DispatchAsync(message, ct);

            // Acknowledge successful processing
            await db.StreamAcknowledgeAsync(streamKey, ConsumerGroup, message.Id);

            _logger.LogInformation(
                "Processed message {MessageId} from stream {StreamKey}",
                message.Id, streamKey);
        }
        catch (Exception ex) when (retryCount < 3)
        {
            _logger.LogWarning(ex,
                "Message {MessageId} processing failed (attempt {Attempt}/3). Will retry.",
                message.Id, retryCount + 1);
            // Leave unacknowledged — XAUTOCLAIM will reassign after idle timeout
        }
        catch (Exception ex)
        {
            _logger.LogError(ex,
                "Message {MessageId} exceeded retry limit. Moving to dead letter stream.",
                message.Id);

            await MoveToDeadLetterAsync(db, streamKey, message, ex, ct);

            // Acknowledge original to prevent infinite retry
            await db.StreamAcknowledgeAsync(streamKey, ConsumerGroup, message.Id);
        }
    }
}
```

---

## Outbox Pattern — Publish After Database Commit

A critical correctness concern: if we publish to Redis Streams inside a MediatR handler before `SaveChangesAsync()` completes, and the DB write subsequently fails, we've published a ghost event. Conversely, if we publish after `SaveChangesAsync()` but the process crashes before publishing, we lose the event.

**Solution**: Transactional Outbox Pattern.

```csharp
// Domain: Entities append domain events to a collection
// Domain/Entities/Event.cs
public class Event : AggregateRoot
{
    private readonly List<DomainEvent> _domainEvents = [];
    public IReadOnlyList<DomainEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void ClearDomainEvents() => _domainEvents.Clear();

    protected void AddDomainEvent(DomainEvent domainEvent)
        => _domainEvents.Add(domainEvent);

    public void Publish()
    {
        if (Status != EventStatus.Draft)
            throw new InvalidEventStatusTransitionException(
                Status, EventStatus.Published);

        Status = EventStatus.Published;
        PublishedAt = DateTime.UtcNow;

        // Domain event collected — will be dispatched after DB commit
        AddDomainEvent(new EventPublished(Id, TenantId, Name, PublishedAt.Value));
    }
}

// Infrastructure: SaveChangesAsync dispatches domain events AFTER commit
// Infrastructure/Persistence/EventFlowDbContext.cs
public override async Task<int> SaveChangesAsync(CancellationToken ct = default)
{
    // Collect domain events from all tracked aggregates
    var aggregates = ChangeTracker
        .Entries<AggregateRoot>()
        .Where(e => e.Entity.DomainEvents.Any())
        .Select(e => e.Entity)
        .ToList();

    var domainEvents = aggregates
        .SelectMany(a => a.DomainEvents)
        .ToList();

    aggregates.ForEach(a => a.ClearDomainEvents());

    // Step 1: Save outbox records and entity changes atomically
    var outboxMessages = domainEvents
        .Select(e => new OutboxMessage
        {
            Id = Guid.NewGuid(),
            Type = e.GetType().Name,
            Payload = JsonSerializer.Serialize(e, e.GetType()),
            CreatedAt = DateTime.UtcNow,
            ProcessedAt = null
        })
        .ToList();

    OutboxMessages.AddRange(outboxMessages);

    // Step 2: Commit DB transaction (entities + outbox records atomically)
    var result = await base.SaveChangesAsync(ct);

    // Step 3: Outbox processor (separate BackgroundService) reads unprocessed
    //         outbox records and publishes to Redis Streams
    //         This runs independently — tolerates application restart

    return result;
}
```

### Outbox Processor

```csharp
// Infrastructure/BackgroundServices/OutboxProcessorService.cs
// Polls outbox table every 1 second, publishes unprocessed events to Redis Streams
public class OutboxProcessorService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var db = scope.ServiceProvider.GetRequiredService<EventFlowDbContext>();
            var eventBus = scope.ServiceProvider.GetRequiredService<IEventBus>();

            // Use distributed lock to prevent duplicate processing in multi-pod setup
            await using var distributedLock = await _lockService
                .AcquireAsync("outbox:processor", TimeSpan.FromSeconds(30), ct);

            if (distributedLock == null) continue; // Another pod has the lock

            var pendingMessages = await db.OutboxMessages
                .Where(m => m.ProcessedAt == null)
                .OrderBy(m => m.CreatedAt)
                .Take(50)
                .ToListAsync(ct);

            foreach (var message in pendingMessages)
            {
                await eventBus.PublishAsync(message, ct);
                message.ProcessedAt = DateTime.UtcNow;
            }

            await db.SaveChangesAsync(ct);
            await Task.Delay(1_000, ct);
        }
    }
}
```

---

## Dead Letter Handling

```csharp
// Dead letter stream: eventflow:dlq
// Messages moved here after 3 failed processing attempts

private async Task MoveToDeadLetterAsync(
    IDatabase db,
    RedisKey originalStream,
    StreamEntry message,
    Exception exception,
    CancellationToken ct)
{
    await db.StreamAddAsync(
        "eventflow:dlq",
        new NameValueEntry[]
        {
            new("original_stream",  originalStream),
            new("original_id",      message.Id.ToString()),
            new("type",             message["type"]!),
            new("tenantid",         message["tenantid"]!),
            new("data",             message["data"]!),
            new("error",            exception.Message),
            new("stack_trace",      exception.StackTrace ?? string.Empty),
            new("failed_at",        DateTime.UtcNow.ToString("O")),
            new("consumer_group",   ConsumerGroup)
        }
    );
    // DLQ monitored by Seq alerts — engineering team notified on DLQ growth
    // Manual replay tool: POST /api/admin/dlq/replay/{messageId}
}
```

---

## Real-Time Dashboard via SignalR + Redis Streams

The real-time check-in counter and activity feed use SignalR hubs backed by Redis Streams:

```csharp
// Infrastructure/BackgroundServices/RealtimeEventConsumer.cs
// Consumes eventflow:checkins and pushes to SignalR hub
public class RealtimeEventConsumer : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        // Reads checkin.completed events
        // Calls IHubContext<CheckInHub>.Clients.Group(eventId)
        //       .SendAsync("CheckInUpdated", new { count, attendeeName }, ct)
        // Dashboard React component subscribes to this hub event
        // Implements the live check-in counter (Sarah persona requirement)
    }
}
```

---

## Consequences

### Positive
- **No additional infrastructure**: Redis is already required for caching. Reusing it for events keeps the infrastructure footprint minimal.
- **Consumer groups**: At-least-once delivery with automatic consumer failover. If a pod dies, pending messages are reassigned after an idle timeout (`XAUTOCLAIM`).
- **Outbox pattern**: Guarantees events are published after DB commit — no ghost events, no lost events on process crash.
- **Scalability**: Redis Streams handle millions of messages at our scale. If we exceed Redis capacity, migrating to Kafka is possible with interface abstraction (`IEventBus`).
- **Observability**: All events visible in Redis CLI for debugging. Seq structured logs provide correlation tracking.

### Negative
- **At-least-once (not exactly-once)**: Consumers must be idempotent — processing the same message twice must be safe. Mitigation: All consumer operations check for existence before creating (e.g., "has this checkin already been recorded?"). Idempotency keys in outbox messages.
- **Message ordering**: Redis Streams guarantee per-stream ordering but not cross-stream ordering. Mitigation: Our consumers do not require cross-stream ordering.
- **Redis persistence**: If Redis goes down, unprocessed stream messages may be lost if AOF/RDB persistence is not configured. Mitigation: Production Redis uses AOF persistence with `appendfsync everysec`; outbox pattern provides durability guarantee from PostgreSQL.

---

## References
- [Redis Streams documentation](https://redis.io/docs/data-types/streams/)
- [Transactional Outbox Pattern](https://microservices.io/patterns/data/transactional-outbox.html)
- [XAUTOCLAIM command](https://redis.io/commands/xautoclaim/)
