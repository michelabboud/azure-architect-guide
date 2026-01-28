# Quick Reference: Performance Anti-Patterns

## Anti-Pattern Decision Matrix

```
QUICK ANTI-PATTERN SELECTOR:
─────────────────────────────

PERFORMANCE ISSUES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Symptom                           │ Anti-Pattern            │ Azure Service │
├───────────────────────────────────┼────────────────────────┼───────────────┤
│ Database CPU constantly high      │ Busy Database          │ Azure Monitor │
│ Too many small network calls      │ Chatty I/O             │ Application   │
│ Retrieving unused data            │ Extraneous Fetching    │ Insights      │
│ Repeated identical queries        │ No Caching             │ Redis Cache   │
│ Blocking threads during I/O       │ Synchronous I/O        │ Async APIs    │
│ All requests hit one store        │ Monolithic Persistence │ Cosmos DB     │
│ Object creation/destruction spikes│ Improper Instantiation │ DI Container  │
│ Service cascading failures        │ Retry Storm            │ Polly/App Svc │
└────────────────────────────────────┴────────────────────────┴───────────────┘

COST ISSUES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Problem                           │ Anti-Pattern            │ Cost Impact   │
├───────────────────────────────────┼────────────────────────┼───────────────┤
│ Large unnecessary data transfers  │ Extraneous Fetching    │ High (▲▲▲)    │
│ Repeated DB queries for same data │ No Caching             │ High (▲▲▲)    │
│ Database overutilization          │ Busy Database          │ High (▲▲▲)    │
│ Excessive retry attempts          │ Retry Storm            │ Medium (▲▲)   │
└────────────────────────────────────┴────────────────────────┴───────────────┘

RELIABILITY ISSUES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Problem                           │ Anti-Pattern            │ Failure Mode  │
├───────────────────────────────────┼────────────────────────┼───────────────┤
│ One misbehaving tenant affects all│ Busy Database          │ Cascading     │
│ Service failure causes overload    │ Retry Storm            │ Cascading     │
│ Single data store fails            │ Monolithic Persistence │ Total outage  │
│ Thread exhaustion under load       │ Synchronous I/O        │ Timeouts      │
└────────────────────────────────────┴────────────────────────┴───────────────┘
```

## Anti-Pattern Quick Reference Table

```
┌────────────────────────────────────────────────────────────────────────────┐
│                         ANTI-PATTERN QUICK REFERENCE                       │
├──────────────────┬──────────────────────┬──────────────────┬──────────────┤
│ Anti-Pattern     │ Key Symptoms         │ Root Cause       │ Quick Fix    │
├──────────────────┼──────────────────────┼──────────────────┼──────────────┤
│ BUSY DATABASE    │ • DB CPU always high │ • Complex stored │ • Move logic │
│                  │ • Cannot scale       │   procedures     │   to app tier│
│                  │ • Queries timeout    │ • Data transform │ • Simplify   │
│                  │                      │   in DB          │   queries    │
├──────────────────┼──────────────────────┼──────────────────┼──────────────┤
│ CHATTY I/O       │ • High latency       │ • Many small     │ • Batch      │
│                  │ • Network underused  │   requests       │   requests   │
│                  │ • Slow operations    │ • No aggregation │ • Use bulk   │
│                  │                      │ • N+1 queries    │   APIs       │
├──────────────────┼──────────────────────┼──────────────────┼──────────────┤
│ EXTRANEOUS       │ • Large responses    │ • SELECT *       │ • Project    │
│ FETCHING         │ • Memory pressure    │ • Unused fields  │   columns    │
│                  │ • Slow serialization │ • Full objects   │ • Paginate   │
│                  │                      │                  │ • Use GraphQL│
├──────────────────┼──────────────────────┼──────────────────┼──────────────┤
│ IMPROPER         │ • GC pressure spikes │ • Creating many  │ • Use DI     │
│ INSTANTIATION    │ • Memory exhaustion  │   short-lived    │   lifetimes  │
│                  │ • Socket exhaustion  │   objects        │ • Object     │
│                  │                      │ • No connection  │   pooling    │
│                  │                      │   reuse          │ • Singleton  │
├──────────────────┼──────────────────────┼──────────────────┼──────────────┤
│ MONOLITHIC       │ • All queries slow   │ • Single data    │ • Polyglot   │
│ PERSISTENCE      │ • Cannot optimize    │   store for all  │   persistence│
│                  │ • Single point of    │ • Mixed OLTP/    │ • Separate   │
│                  │   failure            │   OLAP           │   stores     │
├──────────────────┼──────────────────────┼──────────────────┼──────────────┤
│ NO CACHING       │ • Repeated identical │ • No cache layer │ • Add Redis  │
│                  │   queries            │ • Every request  │   cache      │
│                  │ • Database exhausted │   hits DB        │ • Cache at   │
│                  │ • Slow responses     │                  │   multiple   │
│                  │                      │                  │   layers     │
├──────────────────┼──────────────────────┼──────────────────┼──────────────┤
│ RETRY STORM      │ • Cascading failures │ • Aggressive     │ • Exponential│
│                  │ • Service overload   │   retries        │   backoff    │
│                  │ • Cannot recover     │ • No jitter      │ • Add jitter │
│                  │ • Thundering herd    │ • No circuit      │ • Circuit    │
│                  │                      │   breaker        │   breaker    │
├──────────────────┼──────────────────────┼──────────────────┼──────────────┤
│ SYNCHRONOUS      │ • Thread pool full   │ • Blocking       │ • Use        │
│ I/O              │ • Timeouts increase  │   during I/O     │   async/await│
│                  │ • Poor scalability   │ • .Result, .Wait │ • Avoid      │
│                  │                      │ • Sync APIs      │   blocking   │
│                  │                      │                  │   calls      │
└──────────────────┴──────────────────────┴──────────────────┴──────────────┘
```

## Performance Anti-Patterns

### Busy Database

```csharp
// ❌ ANTI-PATTERN: Complex stored procedure
var result = await _db.Customers
    .FromSqlInterpolated($@"
        EXEC CalculateCustomerAnalytics @StartDate={startDate}, @EndDate={endDate}")
    .ToListAsync();

// ✅ SOLUTION: Move to application tier
public async Task<List<CustomerAnalytics>> CalculateAnalyticsAsync(DateTime startDate, DateTime endDate)
{
    var customers = await _db.Customers
        .Include(c => c.Orders)
        .Where(c => c.Orders.Any(o => o.OrderDate >= startDate && o.OrderDate <= endDate))
        .Select(c => new { c.CustomerId, c.CustomerName, c.Orders })
        .ToListAsync();

    // Process in application tier
    return customers.Select(c => new CustomerAnalytics
    {
        CustomerId = c.CustomerId,
        CustomerName = c.CustomerName,
        Revenue = c.Orders.Sum(o => o.Total),
        OrderCount = c.Orders.Count,
        CustomerTier = CalculateTier(c.Orders.Sum(o => o.Total))
    }).ToList();
}

private string CalculateTier(decimal revenue)
{
    return revenue switch
    {
        >= 100000 => "Platinum",
        >= 50000 => "Gold",
        >= 10000 => "Silver",
        _ => "Bronze"
    };
}
```

### Chatty I/O

```csharp
// ❌ ANTI-PATTERN: N+1 queries
var customers = await _db.Customers.ToListAsync();
foreach (var customer in customers)
{
    customer.Orders = await _db.Orders
        .Where(o => o.CustomerId == customer.CustomerId)
        .ToListAsync(); // N additional queries!
}

// ✅ SOLUTION: Batch with Include
var customersWithOrders = await _db.Customers
    .Include(c => c.Orders)
    .ToListAsync(); // Single query with JOIN

// ✅ SOLUTION: Bulk API
var customerIds = new[] { 1, 2, 3 };
var result = await _cosmosClient.BulkReadAsync(customerIds); // Batch operation
```

### Extraneous Fetching

```csharp
// ❌ ANTI-PATTERN: Fetch entire object when only a few fields needed
var products = await _db.Products.ToListAsync(); // Loads all columns
var names = products.Select(p => p.Name).ToList(); // Only needed this

// ✅ SOLUTION: Project only needed columns
var productNames = await _db.Products
    .Select(p => new { p.ProductId, p.Name })
    .ToListAsync();

// ✅ SOLUTION: Pagination for large result sets
var page = await _db.Products
    .OrderBy(p => p.ProductId)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .Select(p => new { p.ProductId, p.Name, p.Price })
    .ToListAsync();
```

### Improper Instantiation

```csharp
// ❌ ANTI-PATTERN: Creating HttpClient for each request
public class OrderService
{
    public async Task<Order> GetOrderAsync(int id)
    {
        using (var client = new HttpClient()) // Created and destroyed each call
        {
            var response = await client.GetAsync($"https://api.example.com/orders/{id}");
            return await response.Content.ReadAsAsync<Order>();
        }
    }
}

// ✅ SOLUTION: Dependency injection with IHttpClientFactory
public class OrderService
{
    private readonly IHttpClientFactory _clientFactory;

    public OrderService(IHttpClientFactory clientFactory)
    {
        _clientFactory = clientFactory; // Reused instances
    }

    public async Task<Order> GetOrderAsync(int id)
    {
        var client = _clientFactory.CreateClient();
        var response = await client.GetAsync($"https://api.example.com/orders/{id}");
        return await response.Content.ReadAsAsync<Order>();
    }
}

// ✅ SOLUTION: Proper DI lifetimes
services.AddScoped<IRepository, Repository>();        // One per request
services.AddTransient<IValidator, Validator>();       // Each time
services.AddSingleton<ICache, MemoryCache>();        // One for entire app
```

### Monolithic Persistence

```csharp
// ❌ ANTI-PATTERN: Everything in one database
// OLTP queries compete with OLAP workloads
var order = await _sqlDb.Orders
    .Where(o => o.OrderId == 123)
    .FirstOrDefaultAsync(); // Fast, but...

// ...same database handles large reports
var report = await _sqlDb.Orders
    .Where(o => o.OrderDate.Year == 2024)
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Total = g.Sum(o => o.Total) })
    .ToListAsync(); // Slow! Locks needed

// ✅ SOLUTION: Polyglot persistence
// Transactional: Azure SQL
var order = await _sqlDb.Orders.FindAsync(123);

// Analytical: Azure Synapse / Data Lake
var report = await _synapse.QueryAsync(@"
    SELECT CustomerId, SUM(Total) as Total
    FROM Orders
    WHERE OrderDate >= '2024-01-01'
    GROUP BY CustomerId");

// Real-time caching: Redis
var cachedCustomer = await _redis.GetAsync<Customer>("customer:123");
if (cachedCustomer == null)
{
    cachedCustomer = await _sqlDb.Customers.FindAsync(123);
    await _redis.SetAsync("customer:123", cachedCustomer, TimeSpan.FromHours(1));
}
```

### No Caching

```csharp
// ❌ ANTI-PATTERN: Repeated database queries
public async Task<Product> GetProductAsync(int productId)
{
    // Every request hits the database
    return await _db.Products.FindAsync(productId);
}

// ✅ SOLUTION: Cache-aside pattern with Redis
public class ProductService
{
    private readonly IDatabase _redis;
    private readonly DbContext _db;
    private const string CacheKeyPrefix = "product:";
    private readonly TimeSpan _cacheExpiry = TimeSpan.FromHours(1);

    public async Task<Product> GetProductAsync(int productId)
    {
        var cacheKey = $"{CacheKeyPrefix}{productId}";

        // Try cache first
        var cached = await _redis.StringGetAsync(cacheKey);
        if (cached.HasValue)
        {
            return JsonSerializer.Deserialize<Product>(cached);
        }

        // Cache miss - get from DB
        var product = await _db.Products.FindAsync(productId);
        if (product != null)
        {
            // Store in cache
            await _redis.StringSetAsync(
                cacheKey,
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
        var cacheKey = $"{CacheKeyPrefix}{product.ProductId}";
        await _redis.KeyDeleteAsync(cacheKey);
    }
}
```

### Retry Storm

```csharp
// ❌ ANTI-PATTERN: Aggressive retries without backoff
var policy = Policy
    .Handle<HttpRequestException>()
    .RetryAsync(10); // Retry 10 times immediately - causes storm!

// ✅ SOLUTION: Exponential backoff with jitter
using Polly;
using Polly.CircuitBreaker;

var retryPolicy = Policy
    .Handle<HttpRequestException>()
    .Or<TimeoutException>()
    .WaitAndRetryAsync(
        retryCount: 3,
        sleepDurationProvider: attempt =>
            TimeSpan.FromSeconds(Math.Pow(2, attempt)) // 2s, 4s, 8s
            .Add(TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000))), // jitter
        onRetry: (exception, timeSpan, retryCount, context) =>
        {
            _logger.LogWarning(
                $"Retry {retryCount} after {timeSpan.TotalSeconds}s due to {exception.GetType().Name}");
        });

// ✅ SOLUTION: Circuit breaker prevents retry storms
var circuitBreakerPolicy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30),
        onBreak: (exception, duration) =>
        {
            _logger.LogCritical($"Circuit opened for {duration.TotalSeconds}s");
        },
        onReset: () =>
        {
            _logger.LogInformation("Circuit reset, service recovered");
        });

// Combine: retry with circuit breaker
var combinedPolicy = Policy.WrapAsync(retryPolicy, circuitBreakerPolicy);

var result = await combinedPolicy.ExecuteAsync(async () =>
{
    return await _httpClient.GetAsync("https://api.example.com/data");
});
```

### Synchronous I/O

```csharp
// ❌ ANTI-PATTERN: Blocking calls
public ActionResult GetUser(int userId)
{
    var user = _userService.GetUser(userId); // Blocks thread!
    return Ok(user);
}

// ❌ ANTI-PATTERN: Using .Result or .Wait()
public async Task<List<Order>> GetOrdersAsync(int customerId)
{
    var orders = _db.Orders.ToListAsync().Result; // Deadlock risk!
    return orders;
}

// ✅ SOLUTION: Async all the way
public async Task<ActionResult> GetUserAsync(int userId)
{
    var user = await _userService.GetUserAsync(userId); // Non-blocking
    return Ok(user);
}

public async Task<List<Order>> GetOrdersAsync(int customerId)
{
    var orders = await _db.Orders
        .Where(o => o.CustomerId == customerId)
        .ToListAsync(); // Properly async
    return orders;
}

// ✅ SOLUTION: Async middleware pipeline
public class OrderController : ControllerBase
{
    private readonly IOrderService _service;

    [HttpGet("{orderId}")]
    public async Task<IActionResult> GetOrderAsync(int orderId)
    {
        try
        {
            var order = await _service.GetOrderAsync(orderId);
            return Ok(order);
        }
        catch (Exception ex)
        {
            return StatusCode(500, ex.Message);
        }
    }
}
```

## Azure-Specific Anti-Patterns

### Anti-Pattern: Ignoring Blob Replication Latency

```csharp
// ❌ ANTI-PATTERN: Assuming immediate read after write
var blob = new BlobClient(new Uri($"https://myaccount.blob.core.windows.net/container/file"));
await blob.UploadAsync(stream, overwrite: true);

// Immediately read - might get old version due to replication!
var downloaded = await blob.DownloadAsync();

// ✅ SOLUTION: Use Blob Lease or implement retry with delay
var blob = new BlobClient(new Uri(blobUri));
await blob.UploadAsync(stream, overwrite: true);

// Option 1: Wait for replication
await Task.Delay(TimeSpan.FromSeconds(2));
var downloaded = await blob.DownloadAsync();

// Option 2: Use conditional request with ETag
BlobProperties properties = await blob.GetPropertiesAsync();
var eTag = properties.ETag;

BlobDownloadInfo download = await blob.DownloadAsync(
    conditions: new BlobRequestConditions { IfMatch = eTag });
```

### Anti-Pattern: Service Bus Message Lock Timeout

```csharp
// ❌ ANTI-PATTERN: Processing takes longer than lock duration
[FunctionName("ProcessMessage")]
public async Task ProcessLongRunningOperation(
    [ServiceBusTrigger("myqueue")] ServiceBusReceivedMessage message)
{
    // Default lock timeout is 30 seconds
    // If processing takes > 30s, message is released and reprocessed!
    await LongRunningProcessAsync(message); // Takes 2 minutes
}

// ✅ SOLUTION: Renew lock or adjust timeout
public async Task ProcessWithLockRenewal(
    [ServiceBusTrigger("myqueue")] ServiceBusReceivedMessage message,
    ServiceBusClient client)
{
    var processor = client.CreateProcessor("myqueue");
    var lockRenewalTask = RenewLockPeriodicallyAsync(
        message.LockToken,
        TimeSpan.FromSeconds(10));

    try
    {
        await LongRunningProcessAsync(message);
        await processor.CompleteMessageAsync(message);
    }
    finally
    {
        // Cancel lock renewal
        lockRenewalTask.Cancel();
    }
}
```

### Anti-Pattern: Cosmos DB Hot Partition

```csharp
// ❌ ANTI-PATTERN: Using non-selective partition key
public class Order
{
    public string id { get; set; }
    public string PartitionKey { get; set; } = "Orders"; // ❌ All items in one partition!
    public string CustomerId { get; set; }
    public decimal Total { get; set; }
}

// ✅ SOLUTION: Use CustomerId as partition key
public class Order
{
    public string id { get; set; }
    public string CustomerId { get; set; } // ✅ Distributes load
    public decimal Total { get; set; }
}

// ✅ SOLUTION: Hierarchical partition key for hot data
var containerProperties = new ContainerProperties
{
    Id = "Orders",
    PartitionKeyPath = "/CustomerId/Region" // Composite key
};
```

## Cost Anti-Patterns

### Anti-Pattern: No Data Pagination

```csharp
// ❌ ANTI-PATTERN: Fetches all data (expensive!)
var allCustomers = await _db.Customers.ToListAsync(); // Could be millions!

// ✅ SOLUTION: Paginate with limit
public async Task<PagedResult<Customer>> GetCustomersAsync(
    int pageNumber,
    int pageSize = 100)
{
    var total = await _db.Customers.CountAsync();
    var items = await _db.Customers
        .OrderBy(c => c.CustomerId)
        .Skip((pageNumber - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    return new PagedResult<Customer>
    {
        Items = items,
        Total = total,
        PageNumber = pageNumber,
        PageSize = pageSize
    };
}
```

### Anti-Pattern: Unoptimized Cosmos DB Queries

```csharp
// ❌ ANTI-PATTERN: Inefficient query (high RU cost)
var items = container.GetItemLinqQueryable<Order>()
    .Where(o => o.Total > 100) // Filter happens server-side
    .Select(o => new { o.Id, o.CustomerId, o.Total })
    .ToList(); // But processes all items first!

// ✅ SOLUTION: Use partition key, limit results
var itemIterator = container
    .GetItemQueryIterator<Order>(
        queryText: "SELECT c.id, c.CustomerId, c.Total FROM c WHERE c.CustomerId = @customerId AND c.Total > @total",
        queryDefinition: new QueryDefinition(
            "SELECT c.id, c.CustomerId, c.Total FROM c WHERE c.CustomerId = @customerId AND c.Total > @total")
            .WithParameter("@customerId", customerId)
            .WithParameter("@total", 100),
        requestOptions: new QueryRequestOptions { PartitionKey = new PartitionKey(customerId) });

var results = new List<Order>();
while (itemIterator.HasMoreResults)
{
    foreach (var item in await itemIterator.ReadNextAsync())
    {
        results.Add(item);
    }
}
```

## Anti-Pattern Symptom Flow Chart

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ANTI-PATTERN DIAGNOSIS FLOWCHART                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Application Performance Degradation                                       │
│   │                                                                          │
│   ├─► Is database CPU/memory high?                                          │
│   │   └─► YES ──────────────────────────────► BUSY DATABASE                │
│   │   └─► NO ──► Continue                                                   │
│   │                                                                          │
│   ├─► Are queries executing many times for same data?                       │
│   │   └─► YES ──────────────────────────────► NO CACHING                   │
│   │   └─► NO ──► Continue                                                   │
│   │                                                                          │
│   ├─► Many small network requests (N+1 pattern)?                            │
│   │   └─► YES ──────────────────────────────► CHATTY I/O                   │
│   │   └─► NO ──► Continue                                                   │
│   │                                                                          │
│   ├─► Response payloads unusually large?                                    │
│   │   └─► YES ──────────────────────────────► EXTRANEOUS FETCHING          │
│   │   └─► NO ──► Continue                                                   │
│   │                                                                          │
│   ├─► Thread pool exhaustion errors?                                        │
│   │   └─► YES ──────────────────────────────► SYNCHRONOUS I/O              │
│   │   └─► NO ──► Continue                                                   │
│   │                                                                          │
│   ├─► Service cascading failures after error?                               │
│   │   └─► YES ──────────────────────────────► RETRY STORM                  │
│   │   └─► NO ──► Continue                                                   │
│   │                                                                          │
│   ├─► High memory allocation/GC pressure?                                   │
│   │   └─► YES ──────────────────────────────► IMPROPER INSTANTIATION       │
│   │   └─► NO ──► Continue                                                   │
│   │                                                                          │
│   └─► Different workload types on one store?                                │
│       └─► YES ──────────────────────────────► MONOLITHIC PERSISTENCE       │
│       └─► NO ──► Application-specific issue                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Monitoring Anti-Patterns in Azure

### Key Metrics to Watch

```
BUSY DATABASE:
├─ Azure Monitor: DatabaseCpuPercent > 80%
├─ Query Insight: Long-running queries
└─ Alert: Sustained high CPU for > 5 minutes

CHATTY I/O:
├─ Application Insights: High request count with low payload
├─ Network: High latency despite low bandwidth
└─ Alert: Request rate > expected threshold

NO CACHING:
├─ Application Insights: Identical queries in time window
├─ Redis: Low hit rate (< 70%)
└─ Database: Spike in identical queries

SYNCHRONOUS I/O:
├─ Application Insights: Increasing request duration
├─ Performance Counters: High thread pool utilization
└─ Alert: Worker thread depletion warnings

RETRY STORM:
├─ Application Insights: Spike in retry exceptions
├─ Failed request rate increases during outage
└─ Alert: Exponential growth in request count
```

## Best Practices Summary

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    ANTI-PATTERN PREVENTION CHECKLIST                       │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│ DESIGN PHASE:                                                             │
│ □ Choose appropriate data store for access patterns                        │
│ □ Design partition strategy (avoid hot partitions)                         │
│ □ Plan caching strategy for read-heavy data                               │
│ □ Design for async I/O from the start                                     │
│ □ Implement circuit breaker pattern for resilience                         │
│                                                                            │
│ DEVELOPMENT PHASE:                                                        │
│ □ Use async/await consistently (no .Result or .Wait)                      │
│ □ Implement batching for network calls                                     │
│ □ Add caching with proper invalidation strategy                            │
│ □ Use DI with appropriate lifetimes                                        │
│ □ Project only needed columns in queries                                   │
│                                                                            │
│ TESTING PHASE:                                                            │
│ □ Load test to identify performance bottlenecks                            │
│ □ Monitor for N+1 query patterns                                           │
│ □ Verify cache hit rates                                                   │
│ □ Test failure scenarios (retry, circuit breaker)                          │
│ □ Validate partition distribution in Cosmos DB                             │
│                                                                            │
│ PRODUCTION MONITORING:                                                    │
│ □ Set up alerts for CPU/memory thresholds                                  │
│ □ Monitor request latency trends                                           │
│ □ Track cache hit rates and adjust TTLs                                    │
│ □ Monitor retry rates and adjust policies                                  │
│ □ Set up cost budgets and review RU consumption                            │
│                                                                            │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Azure Service Mapping for Anti-Pattern Solutions

```
┌────────────────────────────────────────────────────────────────────────────┐
│ Busy Database                                                              │
│ → Azure Monitor (diagnostics)                                              │
│ → Azure App Service (scale application tier)                               │
│ → Azure SQL Database (read replicas for OLAP)                             │
│ → Azure Synapse (separate analytics workload)                             │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Chatty I/O                                                                 │
│ → Azure API Management (batch operations)                                  │
│ → Azure Service Bus (message batching)                                     │
│ → Azure Functions (aggregation logic)                                      │
│ → Graph API (reduced roundtrips)                                          │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Extraneous Fetching                                                        │
│ → GraphQL (flexible querying)                                              │
│ → OData (field selection)                                                  │
│ → Azure SQL (column projection)                                            │
│ → Cosmos DB (SELECT specific properties)                                   │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Improper Instantiation                                                     │
│ → .NET Dependency Injection                                                │
│ → HttpClientFactory                                                        │
│ → Connection pooling (SQL, Redis)                                          │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Monolithic Persistence                                                     │
│ → Azure SQL Database (OLTP)                                                │
│ → Azure Cosmos DB (multi-model, high scale)                               │
│ → Azure Synapse (OLAP/Analytics)                                          │
│ → Azure Cache for Redis (caching)                                          │
│ → Azure Data Lake (big data)                                               │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ No Caching                                                                 │
│ → Azure Cache for Redis (distributed cache)                               │
│ → Azure Content Delivery Network (static content)                          │
│ → Azure Front Door (edge caching)                                          │
│ → Application-level caching (IDistributedCache)                           │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Retry Storm                                                                │
│ → Polly library (retry policies)                                           │
│ → Azure Service Bus (dead-letter queues)                                  │
│ → Application Insights (monitoring retries)                                │
│ → Azure Functions (durable functions for orchestration)                    │
└────────────────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────────────────┐
│ Synchronous I/O                                                            │
│ → Async/await patterns                                                     │
│ → Azure App Service (async middleware)                                     │
│ → Azure Functions (native async)                                           │
│ → Azure Service Bus (async consumers)                                      │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Code Patterns

### Implement Caching Pattern

```csharp
public interface ICacheService
{
    Task<T> GetAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiry = null);
    Task InvalidateAsync(string key);
}

public class RedisCacheService : ICacheService
{
    private readonly IConnectionMultiplexer _redis;
    private readonly ILogger<RedisCacheService> _logger;

    public async Task<T> GetAsync<T>(string key, Func<Task<T>> factory, TimeSpan? expiry = null)
    {
        var db = _redis.GetDatabase();

        // Try to get from cache
        var cached = await db.StringGetAsync(key);
        if (cached.HasValue)
        {
            _logger.LogInformation($"Cache hit: {key}");
            return JsonSerializer.Deserialize<T>(cached.ToString());
        }

        _logger.LogInformation($"Cache miss: {key}");

        // Get from factory
        var value = await factory();

        // Store in cache
        if (value != null)
        {
            await db.StringSetAsync(
                key,
                JsonSerializer.Serialize(value),
                expiry ?? TimeSpan.FromHours(1));
        }

        return value;
    }

    public async Task InvalidateAsync(string key)
    {
        var db = _redis.GetDatabase();
        await db.KeyDeleteAsync(key);
    }
}
```

### Implement Retry Policy

```csharp
public static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .WaitAndRetryAsync(new[]
        {
            TimeSpan.FromSeconds(1),
            TimeSpan.FromSeconds(2),
            TimeSpan.FromSeconds(4)
        });
}

public static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 3,
            durationOfBreak: TimeSpan.FromSeconds(30));
}

// Register in DI
services.AddHttpClient<IOrderService, OrderService>()
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());
```

---

*Next: [Busy Database](01-busy-database.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
