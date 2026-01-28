# Data Patterns

## Cache-Aside Pattern

### Problem

Repeated database queries for the same data waste resources and slow down your application.

### Solution

```
CACHE-ASIDE FLOW:
─────────────────

READ PATH:
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  1. Application checks cache                                          │
│      │                                                                 │
│      ├─▶ CACHE HIT: Return data immediately                          │
│      │                                                                 │
│      └─▶ CACHE MISS:                                                  │
│           2. Query database                                           │
│           3. Store result in cache (with TTL)                        │
│           4. Return data                                              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

WRITE PATH:
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  1. Update database                                                   │
│  2. Invalidate cache entry (delete, not update)                      │
│                                                                        │
│  Why invalidate instead of update?                                    │
│  • Avoids race conditions                                             │
│  • Next read will populate cache with fresh data                     │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
public class ProductRepository : IProductRepository
{
    private readonly IDatabase _cache;
    private readonly ProductDbContext _db;
    private readonly ILogger<ProductRepository> _logger;
    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(10);

    public ProductRepository(
        IConnectionMultiplexer redis,
        ProductDbContext db,
        ILogger<ProductRepository> logger)
    {
        _cache = redis.GetDatabase();
        _db = db;
        _logger = logger;
    }

    public async Task<Product?> GetByIdAsync(string productId)
    {
        var cacheKey = $"product:{productId}";

        // 1. Try cache first
        var cached = await _cache.StringGetAsync(cacheKey);
        if (cached.HasValue)
        {
            _logger.LogDebug("Cache HIT for {ProductId}", productId);
            return JsonSerializer.Deserialize<Product>(cached!);
        }

        _logger.LogDebug("Cache MISS for {ProductId}", productId);

        // 2. Query database
        var product = await _db.Products
            .AsNoTracking()
            .FirstOrDefaultAsync(p => p.Id == productId);

        if (product != null)
        {
            // 3. Populate cache
            await _cache.StringSetAsync(
                cacheKey,
                JsonSerializer.Serialize(product),
                CacheDuration);
        }

        return product;
    }

    public async Task UpdateAsync(Product product)
    {
        // 1. Update database
        _db.Products.Update(product);
        await _db.SaveChangesAsync();

        // 2. Invalidate cache
        var cacheKey = $"product:{product.Id}";
        await _cache.KeyDeleteAsync(cacheKey);

        _logger.LogDebug("Invalidated cache for {ProductId}", product.Id);
    }

    // Handle cache stampede with locking
    public async Task<Product?> GetByIdWithLockAsync(string productId)
    {
        var cacheKey = $"product:{productId}";
        var lockKey = $"lock:product:{productId}";

        var cached = await _cache.StringGetAsync(cacheKey);
        if (cached.HasValue)
        {
            return JsonSerializer.Deserialize<Product>(cached!);
        }

        // Try to acquire lock
        var lockAcquired = await _cache.StringSetAsync(
            lockKey, "1",
            TimeSpan.FromSeconds(10),
            When.NotExists);

        if (lockAcquired)
        {
            try
            {
                // Double-check cache after acquiring lock
                cached = await _cache.StringGetAsync(cacheKey);
                if (cached.HasValue)
                {
                    return JsonSerializer.Deserialize<Product>(cached!);
                }

                // Load from database
                var product = await _db.Products.FindAsync(productId);
                if (product != null)
                {
                    await _cache.StringSetAsync(
                        cacheKey,
                        JsonSerializer.Serialize(product),
                        CacheDuration);
                }
                return product;
            }
            finally
            {
                await _cache.KeyDeleteAsync(lockKey);
            }
        }
        else
        {
            // Wait for other request to populate cache
            await Task.Delay(100);
            return await GetByIdAsync(productId);
        }
    }
}
```

---

## CQRS (Command Query Responsibility Segregation)

### Problem

Read and write operations have different requirements. Optimizing for one hurts the other.

### Solution

```
CQRS ARCHITECTURE:
──────────────────

TRADITIONAL (Single Model):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Commands ──┐                                                         │
│             ├──▶ Single Model ◀──▶ Single Database                   │
│  Queries  ──┘                                                         │
│                                                                        │
│  Problems:                                                             │
│  • Complex queries slow down writes                                   │
│  • Write-optimized schema hurts read performance                      │
│  • Scaling reads and writes together                                  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

CQRS (Separate Models):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Commands ──▶ Write Model ──▶ Write DB (normalized)                  │
│                    │                                                   │
│                    │ sync (events)                                    │
│                    ▼                                                   │
│  Queries ──▶  Read Model  ◀── Read DB (denormalized, optimized)      │
│                                                                        │
│  Benefits:                                                             │
│  • Independent scaling                                                │
│  • Optimized schemas for each                                         │
│  • Different storage technologies                                      │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Write Side (Commands)
public class CreateOrderCommandHandler : ICommandHandler<CreateOrderCommand>
{
    private readonly IOrderRepository _repository;
    private readonly IEventPublisher _eventPublisher;

    public async Task<OrderId> HandleAsync(CreateOrderCommand command)
    {
        // Business logic and validation
        var order = Order.Create(
            command.CustomerId,
            command.Items,
            command.ShippingAddress);

        // Persist to write store
        await _repository.SaveAsync(order);

        // Publish events for read model sync
        await _eventPublisher.PublishAsync(new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId,
            Items = order.Items,
            Total = order.Total,
            CreatedAt = order.CreatedAt
        });

        return order.Id;
    }
}

// Read Side (Queries)
public class OrderQueryService : IOrderQueryService
{
    private readonly CosmosClient _cosmosClient;
    private readonly Container _container;

    public async Task<OrderSummaryDto?> GetOrderSummaryAsync(string orderId)
    {
        // Query denormalized read model (optimized for this query)
        var response = await _container.ReadItemAsync<OrderSummaryDto>(
            orderId,
            new PartitionKey(orderId));

        return response.Resource;
    }

    public async Task<IEnumerable<OrderSummaryDto>> GetCustomerOrdersAsync(
        string customerId, int limit = 10)
    {
        var query = _container.GetItemQueryIterator<OrderSummaryDto>(
            new QueryDefinition(
                "SELECT * FROM c WHERE c.customerId = @customerId ORDER BY c.createdAt DESC")
                .WithParameter("@customerId", customerId),
            requestOptions: new QueryRequestOptions { MaxItemCount = limit });

        var results = new List<OrderSummaryDto>();
        while (query.HasMoreResults)
        {
            var response = await query.ReadNextAsync();
            results.AddRange(response);
        }

        return results;
    }
}

// Event Handler (Syncs Read Model)
public class OrderCreatedEventHandler
{
    private readonly Container _readModelContainer;

    [FunctionName("SyncOrderReadModel")]
    public async Task HandleOrderCreated(
        [EventGridTrigger] EventGridEvent eventGridEvent)
    {
        var orderEvent = eventGridEvent.Data.ToObjectFromJson<OrderCreatedEvent>();

        // Build denormalized read model
        var orderSummary = new OrderSummaryDto
        {
            id = orderEvent.OrderId,  // Cosmos DB requires lowercase 'id'
            OrderId = orderEvent.OrderId,
            CustomerId = orderEvent.CustomerId,
            CustomerName = await GetCustomerNameAsync(orderEvent.CustomerId),
            ItemCount = orderEvent.Items.Count,
            Total = orderEvent.Total,
            Status = "Created",
            CreatedAt = orderEvent.CreatedAt
        };

        await _readModelContainer.UpsertItemAsync(
            orderSummary,
            new PartitionKey(orderSummary.OrderId));
    }
}
```

---

## Event Sourcing Pattern

### Problem

You need a complete audit trail and ability to reconstruct state at any point in time.

### Solution

```
EVENT SOURCING:
───────────────

TRADITIONAL (State Storage):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Store current state only:                                            │
│  { "balance": 500, "lastUpdated": "2024-01-15" }                     │
│                                                                        │
│  Problem: Lost history - how did we get to $500?                     │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

EVENT SOURCING (Event Storage):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Store all events:                                                    │
│  1. AccountOpened { amount: 0 }                                      │
│  2. MoneyDeposited { amount: 1000 }                                  │
│  3. MoneyWithdrawn { amount: 200 }                                   │
│  4. MoneyDeposited { amount: 50 }                                    │
│  5. MoneyWithdrawn { amount: 350 }                                   │
│                                                                        │
│  Current state = replay all events: $500                             │
│  State at event 2 = replay events 1-2: $1000                         │
│                                                                        │
│  Benefits:                                                            │
│  • Complete audit trail                                               │
│  • Temporal queries ("what was balance on Jan 10?")                  │
│  • Debug by replaying events                                         │
│  • Build new read models from event history                          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Event Store using Cosmos DB
public class EventStore
{
    private readonly Container _container;

    public async Task AppendEventsAsync(
        string streamId,
        IEnumerable<IEvent> events,
        long expectedVersion)
    {
        var batch = _container.CreateTransactionalBatch(
            new PartitionKey(streamId));

        long version = expectedVersion;
        foreach (var @event in events)
        {
            version++;
            var envelope = new EventEnvelope
            {
                id = $"{streamId}:{version}",
                StreamId = streamId,
                Version = version,
                EventType = @event.GetType().Name,
                Data = JsonSerializer.Serialize(@event, @event.GetType()),
                Timestamp = DateTimeOffset.UtcNow
            };
            batch.CreateItem(envelope);
        }

        var response = await batch.ExecuteAsync();
        if (!response.IsSuccessStatusCode)
        {
            throw new ConcurrencyException(
                $"Stream {streamId} was modified. Expected version {expectedVersion}");
        }
    }

    public async Task<IEnumerable<IEvent>> GetEventsAsync(
        string streamId,
        long fromVersion = 0)
    {
        var query = _container.GetItemQueryIterator<EventEnvelope>(
            new QueryDefinition(
                "SELECT * FROM c WHERE c.streamId = @streamId AND c.version > @fromVersion ORDER BY c.version")
                .WithParameter("@streamId", streamId)
                .WithParameter("@fromVersion", fromVersion));

        var events = new List<IEvent>();
        while (query.HasMoreResults)
        {
            var response = await query.ReadNextAsync();
            foreach (var envelope in response)
            {
                var eventType = Type.GetType($"MyApp.Events.{envelope.EventType}");
                var @event = (IEvent)JsonSerializer.Deserialize(envelope.Data, eventType!);
                events.Add(@event);
            }
        }

        return events;
    }
}

// Aggregate with Event Sourcing
public class BankAccount
{
    public string Id { get; private set; }
    public decimal Balance { get; private set; }
    public long Version { get; private set; }

    private readonly List<IEvent> _uncommittedEvents = new();

    // Reconstruct from events
    public static BankAccount FromEvents(IEnumerable<IEvent> events)
    {
        var account = new BankAccount();
        foreach (var @event in events)
        {
            account.Apply(@event);
            account.Version++;
        }
        return account;
    }

    // Commands produce events
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new InvalidOperationException("Amount must be positive");

        var @event = new MoneyDeposited
        {
            AccountId = Id,
            Amount = amount,
            Timestamp = DateTimeOffset.UtcNow
        };

        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new InvalidOperationException("Amount must be positive");
        if (Balance < amount)
            throw new InvalidOperationException("Insufficient funds");

        var @event = new MoneyWithdrawn
        {
            AccountId = Id,
            Amount = amount,
            Timestamp = DateTimeOffset.UtcNow
        };

        Apply(@event);
        _uncommittedEvents.Add(@event);
    }

    // Apply events to update state
    private void Apply(IEvent @event)
    {
        switch (@event)
        {
            case AccountOpened e:
                Id = e.AccountId;
                Balance = 0;
                break;
            case MoneyDeposited e:
                Balance += e.Amount;
                break;
            case MoneyWithdrawn e:
                Balance -= e.Amount;
                break;
        }
    }

    public IEnumerable<IEvent> GetUncommittedEvents() => _uncommittedEvents;
    public void ClearUncommittedEvents() => _uncommittedEvents.Clear();
}
```

---

## Sharding Pattern

### Problem

A single database can't handle your data volume or throughput requirements.

### Solution

```
SHARDING STRATEGIES:
────────────────────

HASH-BASED (Even distribution):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Key: customer_123 → hash(123) % 4 = Shard 2                         │
│                                                                        │
│  Shard 0    Shard 1    Shard 2    Shard 3                            │
│  ████       ████       ████       ████                                │
│                                                                        │
│  Pro: Even distribution                                               │
│  Con: Range queries require hitting all shards                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

RANGE-BASED (Ordered data):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  2024-01 → Shard 1, 2024-02 → Shard 2, 2024-03 → Shard 3            │
│                                                                        │
│  Shard 1      Shard 2      Shard 3                                    │
│  Jan data     Feb data     Mar data                                   │
│                                                                        │
│  Pro: Range queries efficient within shard                            │
│  Con: Hot spots (current month shard is busiest)                     │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

DIRECTORY-BASED (Lookup table):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Lookup: customer_123 → Shard 2                                      │
│          customer_456 → Shard 1                                      │
│                                                                        │
│  Pro: Flexible placement, easy rebalancing                           │
│  Con: Lookup table becomes bottleneck/SPOF                           │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Cosmos DB Partitioning

```csharp
// Cosmos DB uses partition keys for automatic sharding
public class OrderRepository
{
    private readonly Container _container;

    public async Task CreateOrderAsync(Order order)
    {
        // CustomerId is partition key - all customer orders in same partition
        await _container.CreateItemAsync(
            order,
            new PartitionKey(order.CustomerId));
    }

    // Efficient: Query within single partition
    public async Task<IEnumerable<Order>> GetCustomerOrdersAsync(string customerId)
    {
        var query = _container.GetItemQueryIterator<Order>(
            new QueryDefinition("SELECT * FROM c WHERE c.customerId = @customerId")
                .WithParameter("@customerId", customerId),
            requestOptions: new QueryRequestOptions
            {
                PartitionKey = new PartitionKey(customerId)  // Single partition
            });

        var orders = new List<Order>();
        while (query.HasMoreResults)
        {
            orders.AddRange(await query.ReadNextAsync());
        }
        return orders;
    }

    // Expensive: Cross-partition query (fan-out)
    public async Task<IEnumerable<Order>> GetOrdersByDateAsync(DateTime date)
    {
        // This queries ALL partitions - use sparingly
        var query = _container.GetItemQueryIterator<Order>(
            new QueryDefinition("SELECT * FROM c WHERE c.orderDate >= @date")
                .WithParameter("@date", date));

        var orders = new List<Order>();
        while (query.HasMoreResults)
        {
            orders.AddRange(await query.ReadNextAsync());
        }
        return orders;
    }
}
```

### Partition Key Selection

```
PARTITION KEY BEST PRACTICES:
─────────────────────────────

GOOD PARTITION KEYS:
✅ High cardinality (many distinct values)
✅ Even distribution of data and requests
✅ Frequently used in WHERE clauses
✅ CustomerId for customer-centric apps
✅ TenantId for multi-tenant apps
✅ DeviceId for IoT scenarios

BAD PARTITION KEYS:
❌ Low cardinality (status, country)
❌ Timestamp alone (hot partition for "now")
❌ Monotonically increasing (all writes to one partition)
❌ Rarely used in queries

COMPOSITE KEYS:
For time-series data, combine:
  tenantId + yyyy-MM → "tenant123-2024-01"
  • Spreads load across partitions
  • Enables efficient range queries within month
```

---

*Next: [Gateway Patterns](04-gateway-patterns.md)* | *Back to [Messaging Patterns](02-messaging-patterns.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
