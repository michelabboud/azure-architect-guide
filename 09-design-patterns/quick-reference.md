# Design Patterns Quick Reference

## Pattern Decision Matrix

```
QUICK PATTERN SELECTOR:
───────────────────────

RELIABILITY ISSUES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Problem                           │ Pattern                │ Azure Service │
├───────────────────────────────────┼────────────────────────┼───────────────┤
│ Service calls failing randomly    │ Retry                  │ Polly/SDK     │
│ Dependent service is down         │ Circuit Breaker        │ Polly         │
│ One failure crashes everything    │ Bulkhead               │ Service Fabric│
│ Need to know if service healthy   │ Health Endpoint        │ App Service   │
│ Distributed operation can fail    │ Compensating Txn       │ Durable Func  │
│ Need to coordinate instances      │ Leader Election        │ Blob Lease    │
└───────────────────────────────────┴────────────────────────┴───────────────┘

PERFORMANCE ISSUES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Database is slow                  │ Cache-Aside            │ Redis Cache   │
│ Reads bottleneck writes           │ CQRS                   │ SQL + Cosmos  │
│ Need query flexibility            │ Materialized View      │ Synapse       │
│ Data too large for one DB         │ Sharding               │ Cosmos DB     │
│ Serving static files is expensive │ Static Content Hosting │ Blob + CDN    │
└───────────────────────────────────┴────────────────────────┴───────────────┘

MESSAGING ISSUES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Traffic spikes overwhelm service  │ Queue-Based Load Level │ Service Bus   │
│ Need to scale message processing  │ Competing Consumers    │ Functions     │
│ Multiple services need same event │ Publisher/Subscriber   │ Event Grid    │
│ Need ordered processing           │ Sequential Convoy      │ Service Bus   │
│ Messages are too large            │ Claim Check            │ Blob + SB     │
│ Need complex workflow             │ Saga                   │ Durable Func  │
└───────────────────────────────────┴────────────────────────┴───────────────┘

API/GATEWAY ISSUES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Multiple backends, one API        │ Gateway Routing        │ API Mgmt      │
│ Reduce client calls               │ Gateway Aggregation    │ API Mgmt      │
│ Different clients, different needs│ Backends for Frontends │ API Mgmt      │
│ Need cross-cutting concerns       │ Gateway Offloading     │ API Mgmt      │
│ Protect backend from direct access│ Gatekeeper             │ API Mgmt      │
└───────────────────────────────────┴────────────────────────┴───────────────┘

MIGRATION ISSUES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Migrating monolith gradually      │ Strangler Fig          │ API Mgmt      │
│ New system talks to legacy        │ Anti-Corruption Layer  │ Functions     │
│ Need to extend without modifying  │ Sidecar                │ AKS           │
│ Need helper for network calls     │ Ambassador             │ AKS           │
└───────────────────────────────────┴────────────────────────┴───────────────┘
```

## Pattern Code Examples

### Retry Pattern

```csharp
// Using Polly in .NET
using Polly;
using Polly.Retry;

// Simple retry
var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)),
        onRetry: (exception, timeSpan, retryCount, context) =>
        {
            _logger.LogWarning($"Retry {retryCount} after {timeSpan.TotalSeconds}s due to {exception.Message}");
        });

// Usage
var result = await retryPolicy.ExecuteAsync(async () =>
{
    return await _httpClient.GetAsync("https://api.example.com/data");
});
```

### Circuit Breaker Pattern

```csharp
// Circuit breaker with Polly
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (exception, duration) =>
        {
            _logger.LogWarning($"Circuit opened for {duration.TotalSeconds}s");
        },
        onReset: () =>
        {
            _logger.LogInformation("Circuit closed");
        },
        onHalfOpen: () =>
        {
            _logger.LogInformation("Circuit half-open, testing...");
        });

// Combine retry + circuit breaker
var combinedPolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy);

var result = await combinedPolicy.ExecuteAsync(async () =>
{
    return await _httpClient.GetAsync("https://api.example.com/data");
});
```

### Cache-Aside Pattern

```csharp
// Cache-aside with Redis
public class ProductService
{
    private readonly IDatabase _cache;
    private readonly ProductDbContext _db;
    private readonly TimeSpan _cacheExpiry = TimeSpan.FromMinutes(10);

    public async Task<Product> GetProductAsync(string productId)
    {
        // Try cache first
        var cached = await _cache.StringGetAsync($"product:{productId}");
        if (cached.HasValue)
        {
            return JsonSerializer.Deserialize<Product>(cached);
        }

        // Cache miss - get from database
        var product = await _db.Products.FindAsync(productId);
        if (product != null)
        {
            // Store in cache
            await _cache.StringSetAsync(
                $"product:{productId}",
                JsonSerializer.Serialize(product),
                _cacheExpiry);
        }

        return product;
    }

    public async Task UpdateProductAsync(Product product)
    {
        // Update database
        _db.Products.Update(product);
        await _db.SaveChangesAsync();

        // Invalidate cache
        await _cache.KeyDeleteAsync($"product:{product.Id}");
    }
}
```

### Queue-Based Load Leveling

```csharp
// Azure Service Bus producer
public class OrderService
{
    private readonly ServiceBusSender _sender;

    public async Task SubmitOrderAsync(Order order)
    {
        var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
        {
            MessageId = order.Id,
            ContentType = "application/json",
            Subject = "NewOrder"
        };

        await _sender.SendMessageAsync(message);
    }
}

// Azure Function consumer
public class OrderProcessor
{
    [FunctionName("ProcessOrder")]
    public async Task Run(
        [ServiceBusTrigger("orders", Connection = "ServiceBusConnection")]
        ServiceBusReceivedMessage message,
        ILogger log)
    {
        var order = JsonSerializer.Deserialize<Order>(message.Body);

        // Process order (at our own pace)
        await ProcessOrderAsync(order);

        log.LogInformation($"Processed order {order.Id}");
    }
}
```

### Saga Pattern (Durable Functions)

```csharp
// Saga orchestration with Durable Functions
[FunctionName("OrderSaga")]
public static async Task<OrderResult> RunOrderSaga(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var order = context.GetInput<Order>();
    var result = new OrderResult { OrderId = order.Id };

    try
    {
        // Step 1: Reserve inventory
        var inventoryReserved = await context.CallActivityAsync<bool>(
            "ReserveInventory", order);

        if (!inventoryReserved)
        {
            result.Status = "Failed - No inventory";
            return result;
        }

        // Step 2: Process payment
        var paymentResult = await context.CallActivityAsync<PaymentResult>(
            "ProcessPayment", order);

        if (!paymentResult.Success)
        {
            // Compensate: Release inventory
            await context.CallActivityAsync("ReleaseInventory", order);
            result.Status = "Failed - Payment declined";
            return result;
        }

        // Step 3: Ship order
        var shippingResult = await context.CallActivityAsync<ShippingResult>(
            "ShipOrder", order);

        if (!shippingResult.Success)
        {
            // Compensate: Refund payment, release inventory
            await context.CallActivityAsync("RefundPayment", paymentResult);
            await context.CallActivityAsync("ReleaseInventory", order);
            result.Status = "Failed - Shipping unavailable";
            return result;
        }

        result.Status = "Completed";
        result.TrackingNumber = shippingResult.TrackingNumber;
        return result;
    }
    catch (Exception ex)
    {
        // Handle unexpected failures with compensation
        await context.CallActivityAsync("HandleSagaFailure", new { order, ex.Message });
        throw;
    }
}
```

### Publisher/Subscriber (Event Grid)

```csharp
// Publisher
public class OrderPublisher
{
    private readonly EventGridPublisherClient _client;

    public async Task PublishOrderCreatedAsync(Order order)
    {
        var @event = new EventGridEvent(
            subject: $"orders/{order.Id}",
            eventType: "Order.Created",
            dataVersion: "1.0",
            data: new OrderCreatedEvent
            {
                OrderId = order.Id,
                CustomerId = order.CustomerId,
                Total = order.Total,
                CreatedAt = DateTime.UtcNow
            });

        await _client.SendEventAsync(@event);
    }
}

// Subscriber (Azure Function)
[FunctionName("HandleOrderCreated")]
public async Task HandleOrderCreated(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    ILogger log)
{
    var orderEvent = eventGridEvent.Data.ToObjectFromJson<OrderCreatedEvent>();

    // React to event (e.g., send confirmation email)
    await _emailService.SendOrderConfirmationAsync(orderEvent);

    log.LogInformation($"Processed Order.Created for {orderEvent.OrderId}");
}
```

### Valet Key (SAS Token)

```csharp
// Generate SAS token for direct blob upload
public string GenerateUploadSasToken(string containerName, string blobName)
{
    var blobClient = _blobServiceClient
        .GetBlobContainerClient(containerName)
        .GetBlobClient(blobName);

    var sasBuilder = new BlobSasBuilder
    {
        BlobContainerName = containerName,
        BlobName = blobName,
        Resource = "b",
        ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(15)
    };

    sasBuilder.SetPermissions(BlobSasPermissions.Write | BlobSasPermissions.Create);

    var sasToken = blobClient.GenerateSasUri(sasBuilder);
    return sasToken.ToString();
}

// Client can now upload directly to blob storage
// without going through your API
```

### Claim Check Pattern

```csharp
// Store large payload, send reference
public class LargeMessageService
{
    private readonly BlobContainerClient _blobClient;
    private readonly ServiceBusSender _sender;

    public async Task SendLargeMessageAsync(byte[] largePayload, string metadata)
    {
        // Store payload in blob
        var blobName = $"messages/{Guid.NewGuid()}";
        var blob = _blobClient.GetBlobClient(blobName);
        await blob.UploadAsync(new BinaryData(largePayload));

        // Send claim check (reference) via message queue
        var message = new ServiceBusMessage(JsonSerializer.Serialize(new
        {
            ClaimCheck = blobName,
            Metadata = metadata
        }));

        await _sender.SendMessageAsync(message);
    }

    public async Task<byte[]> RetrieveLargeMessageAsync(string claimCheck)
    {
        var blob = _blobClient.GetBlobClient(claimCheck);
        var response = await blob.DownloadContentAsync();
        return response.Value.Content.ToArray();
    }
}
```

---

## Pattern Anti-Patterns

```
COMMON MISTAKES:
────────────────

RETRY PATTERN:
❌ Retry on non-transient errors (400 Bad Request)
❌ No exponential backoff (overwhelms failing service)
❌ Infinite retries (never gives up)
❌ Same retry config for all operations
✅ Retry only transient errors (5xx, timeouts)
✅ Exponential backoff with jitter
✅ Finite retry count with circuit breaker

CIRCUIT BREAKER:
❌ Breaking on single failure
❌ Too long break duration
❌ No monitoring of circuit state
✅ Break after threshold of failures
✅ Reasonable break duration (30s-5m)
✅ Log and alert on state changes

CACHE-ASIDE:
❌ Caching everything
❌ No cache invalidation strategy
❌ Cache stampede (all requests hit DB on expiry)
✅ Cache hot data only
✅ Event-driven invalidation or TTL
✅ Use locking or stale-while-revalidate

QUEUE-BASED LOAD LEVELING:
❌ No dead letter queue
❌ No message retry/poison handling
❌ Synchronous processing expectation
✅ Configure DLQ for failed messages
✅ Implement retry with backoff
✅ Design for async, eventual consistency
```

---

## Azure Service Mapping

```
PATTERN → AZURE SERVICES:
─────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│ Retry, Circuit Breaker, Bulkhead                                          │
│ → Polly library (.NET)                                                     │
│ → resilience4j (Java)                                                      │
│ → Built into Azure SDKs                                                    │
│ → Application Insights (monitoring)                                        │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Queue-Based Load Leveling, Competing Consumers                            │
│ → Azure Service Bus                                                        │
│ → Azure Queue Storage                                                      │
│ → Azure Functions (consumer)                                               │
│ → Azure Container Apps (consumer)                                          │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Publisher/Subscriber                                                       │
│ → Azure Event Grid (events)                                                │
│ → Azure Service Bus Topics (messages)                                      │
│ → Azure Event Hubs (streaming)                                             │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Saga, Choreography                                                        │
│ → Durable Functions                                                        │
│ → Azure Logic Apps                                                         │
│ → Dapr workflows                                                           │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Cache-Aside, CQRS                                                         │
│ → Azure Cache for Redis                                                    │
│ → Azure Cosmos DB (multi-model)                                           │
│ → Azure SQL Database (read replicas)                                       │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Gateway Patterns (Routing, Aggregation, BFF)                              │
│ → Azure API Management                                                     │
│ → Azure Front Door                                                         │
│ → Azure Application Gateway                                                │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Sidecar, Ambassador                                                        │
│ → Azure Kubernetes Service                                                 │
│ → Azure Container Apps                                                     │
│ → Dapr (sidecar runtime)                                                   │
└────────────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Reliability Patterns](01-reliability-patterns.md)* | *Back to [Chapter Overview](README.md)*
