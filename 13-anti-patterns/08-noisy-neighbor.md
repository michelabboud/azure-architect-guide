# Anti-Pattern: Noisy Neighbor

## Overview

The Noisy Neighbor anti-pattern occurs in multi-tenant systems when one tenant consumes a disproportionate share of shared resources, degrading performance for other tenants. This is a critical concern in cloud architectures where resources are often shared for cost efficiency.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       NOISY NEIGHBOR ANTI-PATTERN                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Multi-Tenant Application (Shared Resources)                               │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │  Tenant A            Tenant B (Noisy)          Tenant C             │   │
│   │  ┌─────────┐         ┌─────────────────┐       ┌─────────┐          │   │
│   │  │  Normal │         │ ███████████████ │       │  Normal │          │   │
│   │  │   Load  │         │ ███████████████ │       │   Load  │          │   │
│   │  │   5%    │         │ ███  90%  ████ │       │   5%    │          │   │
│   │  └────┬────┘         │ ███████████████ │       └────┬────┘          │   │
│   │       │              └────────┬────────┘            │               │   │
│   │       │                       │                     │               │   │
│   │       ▼                       ▼                     ▼               │   │
│   │  ┌─────────────────────────────────────────────────────────────┐   │   │
│   │  │                   SHARED DATABASE                            │   │   │
│   │  │                                                              │   │   │
│   │  │   CPU: ████████████████████████████████████████████ 100%    │   │   │
│   │  │                                                              │   │   │
│   │  │   Tenant B's heavy queries consume all resources             │   │   │
│   │  │   Tenants A and C experience timeouts!                       │   │   │
│   │  │                                                              │   │   │
│   │  └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Problems:                                                                  │
│   • One tenant affects all others                                           │
│   • Unpredictable performance                                               │
│   • SLA violations                                                          │
│   • Customer complaints                                                     │
│   • Difficult to identify culprit                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Shared Database Without Limits

```csharp
// ANTI-PATTERN: All tenants share same database without resource limits
public class OrderRepository
{
    public async Task<List<Order>> GetAllOrdersAsync(int tenantId)
    {
        // No limits - large tenant can run expensive query
        return await _db.Orders
            .Where(o => o.TenantId == tenantId)
            .Include(o => o.LineItems)
            .Include(o => o.Customer)
            .ToListAsync(); // Could return millions of rows!
    }

    public async Task<Report> GenerateReportAsync(int tenantId)
    {
        // Expensive aggregation blocks database for all tenants
        return await _db.Orders
            .Where(o => o.TenantId == tenantId)
            .GroupBy(o => o.OrderDate.Month)
            .Select(g => new MonthlyStats
            {
                Month = g.Key,
                Total = g.Sum(o => o.Total),
                Count = g.Count()
            })
            .ToListAsync();
    }
}
```

### 2. Shared Storage Without Quotas

```csharp
// ANTI-PATTERN: No storage limits per tenant
public class DocumentService
{
    public async Task UploadDocumentAsync(int tenantId, Stream content)
    {
        // No size or count limits
        var blobClient = _container.GetBlobClient($"{tenantId}/{Guid.NewGuid()}");
        await blobClient.UploadAsync(content);
        // One tenant could upload terabytes, filling shared storage!
    }
}
```

### 3. Shared API Without Rate Limiting

```csharp
// ANTI-PATTERN: No rate limiting per tenant
[HttpGet("search")]
public async Task<ActionResult<SearchResults>> Search([FromQuery] string query)
{
    var tenantId = HttpContext.GetTenantId();

    // No rate limits - one tenant can flood the API
    return await _searchService.SearchAsync(tenantId, query);
}
```

### 4. Shared Queue Without Prioritization

```csharp
// ANTI-PATTERN: All tenants use same queue
public class NotificationService
{
    public async Task SendNotificationAsync(int tenantId, Notification notification)
    {
        // Same queue for all tenants
        await _queueClient.SendMessageAsync(
            JsonSerializer.Serialize(notification));
    }
}
// High-volume tenant floods queue, delaying messages for others
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| Variable latency | Same query varies 10ms to 10s |
| Periodic slowdowns | Issues at specific times (batch jobs) |
| Tenant complaints | Some tenants report issues, others don't |
| Resource spikes | CPU/memory spikes correlate with tenant activity |
| Timeout increases | Connection timeouts during peak usage |

### Detection Queries

```sql
-- Find tenants consuming most resources
SELECT TOP 10
    TenantId,
    COUNT(*) AS QueryCount,
    SUM(total_worker_time) / 1000 AS TotalCPU_ms,
    AVG(total_elapsed_time) / 1000 AS AvgDuration_ms
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
CROSS APPLY (
    SELECT TenantId = -- extract tenant from query
        CASE WHEN st.text LIKE '%TenantId = %'
        THEN SUBSTRING(st.text, CHARINDEX('TenantId = ', st.text) + 11, 10)
        ELSE 'Unknown' END
) t
GROUP BY TenantId
ORDER BY TotalCPU_ms DESC;
```

```kusto
// Application Insights: Find noisy tenants
requests
| where timestamp > ago(1h)
| extend TenantId = tostring(customDimensions.TenantId)
| summarize
    RequestCount = count(),
    AvgDuration = avg(duration),
    P95Duration = percentile(duration, 95),
    ErrorCount = countif(success == false)
by TenantId
| order by RequestCount desc
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Multi-Tenant Application (Resource Isolation)                             │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │  Tenant A            Tenant B (Heavy)          Tenant C             │   │
│   │  ┌─────────┐         ┌─────────────────┐       ┌─────────┐          │   │
│   │  │  Normal │         │ Rate Limited    │       │  Normal │          │   │
│   │  │   Load  │         │ Query Timeouts  │       │   Load  │          │   │
│   │  │         │         │ Resource Quotas │       │         │          │   │
│   │  └────┬────┘         └────────┬────────┘       └────┬────┘          │   │
│   │       │                       │                     │               │   │
│   │       │    ┌─────────────────┼─────────────────┐   │               │   │
│   │       │    │   RESOURCE GOVERNOR / QUOTAS      │   │               │   │
│   │       │    │   • CPU limits per tenant         │   │               │   │
│   │       │    │   • Memory limits per tenant      │   │               │   │
│   │       │    │   • IOPS limits per tenant        │   │               │   │
│   │       │    │   • Rate limiting per tenant      │   │               │   │
│   │       │    └─────────────────┬─────────────────┘   │               │   │
│   │       │                      │                     │               │   │
│   │       ▼                      ▼                     ▼               │   │
│   │  ┌─────────────────────────────────────────────────────────────┐   │   │
│   │  │                   SHARED DATABASE                            │   │   │
│   │  │   CPU: ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ 40%         │   │   │
│   │  │   All tenants perform consistently!                          │   │   │
│   │  └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: Resource Governor (Azure SQL)

```sql
-- Create workload groups with resource limits per tenant tier

-- Create resource pool
CREATE RESOURCE POOL TenantStandardPool
WITH (
    MIN_CPU_PERCENT = 0,
    MAX_CPU_PERCENT = 20,
    MIN_MEMORY_PERCENT = 0,
    MAX_MEMORY_PERCENT = 20
);

CREATE RESOURCE POOL TenantPremiumPool
WITH (
    MIN_CPU_PERCENT = 10,
    MAX_CPU_PERCENT = 50,
    MIN_MEMORY_PERCENT = 10,
    MAX_MEMORY_PERCENT = 50
);

-- Create workload groups
CREATE WORKLOAD GROUP StandardTenants
USING TenantStandardPool;

CREATE WORKLOAD GROUP PremiumTenants
USING TenantPremiumPool;

-- Classifier function to route based on tenant
CREATE FUNCTION dbo.TenantClassifier()
RETURNS SYSNAME WITH SCHEMABINDING
AS
BEGIN
    DECLARE @WorkloadGroup SYSNAME;
    DECLARE @TenantTier NVARCHAR(50);

    SELECT @TenantTier = TenantTier
    FROM dbo.TenantContext
    WHERE SessionId = @@SPID;

    SET @WorkloadGroup = CASE @TenantTier
        WHEN 'Premium' THEN 'PremiumTenants'
        ELSE 'StandardTenants'
    END;

    RETURN @WorkloadGroup;
END;

ALTER RESOURCE GOVERNOR WITH (CLASSIFIER_FUNCTION = dbo.TenantClassifier);
ALTER RESOURCE GOVERNOR RECONFIGURE;
```

### Solution 2: API Rate Limiting

```csharp
// SOLUTION: Rate limiting per tenant

// Program.cs
builder.Services.AddRateLimiter(options =>
{
    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;

    options.AddPolicy("TenantRateLimit", context =>
    {
        var tenantId = context.Request.Headers["X-Tenant-Id"].FirstOrDefault();
        return RateLimitPartition.GetTokenBucketLimiter(tenantId, _ =>
            new TokenBucketRateLimiterOptions
            {
                TokenLimit = 100,
                ReplenishmentPeriod = TimeSpan.FromMinutes(1),
                TokensPerPeriod = 100,
                QueueProcessingOrder = QueueProcessingOrder.OldestFirst,
                QueueLimit = 10
            });
    });
});

// Controller
[HttpGet("search")]
[EnableRateLimiting("TenantRateLimit")]
public async Task<ActionResult<SearchResults>> Search([FromQuery] string query)
{
    return await _searchService.SearchAsync(HttpContext.GetTenantId(), query);
}
```

### Solution 3: Query Timeouts and Pagination

```csharp
// SOLUTION: Enforce query limits per tenant

public class OrderRepository
{
    private readonly TenantSettings _tenantSettings;

    public async Task<PagedResult<Order>> GetOrdersAsync(
        int tenantId,
        int page = 1,
        int pageSize = 50)
    {
        var settings = _tenantSettings.GetSettings(tenantId);

        // Enforce maximum page size based on tenant tier
        pageSize = Math.Min(pageSize, settings.MaxPageSize);

        using var cts = new CancellationTokenSource(settings.QueryTimeout);

        try
        {
            var query = _db.Orders
                .Where(o => o.TenantId == tenantId)
                .OrderByDescending(o => o.OrderDate);

            var totalCount = await query.CountAsync(cts.Token);
            var items = await query
                .Skip((page - 1) * pageSize)
                .Take(pageSize)
                .ToListAsync(cts.Token);

            return new PagedResult<Order>
            {
                Items = items,
                TotalCount = totalCount,
                Page = page,
                PageSize = pageSize
            };
        }
        catch (OperationCanceledException)
        {
            throw new TenantQuotaExceededException(
                $"Query exceeded timeout of {settings.QueryTimeout}");
        }
    }
}
```

### Solution 4: Storage Quotas

```csharp
// SOLUTION: Enforce storage limits per tenant

public class DocumentService
{
    private readonly TenantQuotaService _quotaService;

    public async Task UploadDocumentAsync(int tenantId, Stream content, string fileName)
    {
        var fileSize = content.Length;

        // Check quota before upload
        var usage = await _quotaService.GetStorageUsageAsync(tenantId);
        var quota = await _quotaService.GetStorageQuotaAsync(tenantId);

        if (usage + fileSize > quota)
        {
            throw new QuotaExceededException(
                $"Storage quota exceeded. Used: {usage / 1024 / 1024}MB, " +
                $"Quota: {quota / 1024 / 1024}MB");
        }

        // Check file size limit
        var maxFileSize = await _quotaService.GetMaxFileSizeAsync(tenantId);
        if (fileSize > maxFileSize)
        {
            throw new QuotaExceededException(
                $"File size {fileSize / 1024 / 1024}MB exceeds limit of " +
                $"{maxFileSize / 1024 / 1024}MB");
        }

        // Upload with metadata
        var blobClient = _container.GetBlobClient($"{tenantId}/{fileName}");
        await blobClient.UploadAsync(content, new BlobUploadOptions
        {
            Metadata = new Dictionary<string, string>
            {
                ["TenantId"] = tenantId.ToString(),
                ["FileSize"] = fileSize.ToString()
            }
        });

        // Update usage
        await _quotaService.IncrementUsageAsync(tenantId, fileSize);
    }
}
```

### Solution 5: Separate Queues per Tenant Tier

```csharp
// SOLUTION: Priority queues based on tenant tier

public class NotificationService
{
    private readonly Dictionary<TenantTier, ServiceBusSender> _senders;

    public NotificationService(ServiceBusClient client)
    {
        _senders = new Dictionary<TenantTier, ServiceBusSender>
        {
            [TenantTier.Enterprise] = client.CreateSender("notifications-enterprise"),
            [TenantTier.Premium] = client.CreateSender("notifications-premium"),
            [TenantTier.Standard] = client.CreateSender("notifications-standard")
        };
    }

    public async Task SendNotificationAsync(int tenantId, Notification notification)
    {
        var tier = await _tenantService.GetTierAsync(tenantId);
        var sender = _senders[tier];

        await sender.SendMessageAsync(new ServiceBusMessage(
            JsonSerializer.Serialize(notification)));
    }
}

// Different processing capacity per queue
// Enterprise: 10 consumers
// Premium: 5 consumers
// Standard: 2 consumers
```

### Solution 6: Database Sharding for High-Value Tenants

```csharp
// SOLUTION: Dedicated databases for enterprise tenants

public class TenantDbContextFactory
{
    private readonly Dictionary<int, string> _enterpriseTenantConnections;
    private readonly string _sharedConnection;

    public AppDbContext CreateContext(int tenantId)
    {
        string connectionString;

        if (_enterpriseTenantConnections.TryGetValue(tenantId, out var dedicated))
        {
            // Enterprise tenant gets dedicated database
            connectionString = dedicated;
        }
        else
        {
            // Standard tenants share database
            connectionString = _sharedConnection;
        }

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer(connectionString)
            .Options;

        var context = new AppDbContext(options);
        context.TenantId = tenantId;
        return context;
    }
}
```

### Azure-Specific Solutions

#### Azure SQL Elastic Pool with Per-Database DTU

```bicep
// Bicep: Elastic pool with per-DB limits
resource elasticPool 'Microsoft.Sql/servers/elasticPools@2023-05-01-preview' = {
  parent: sqlServer
  name: 'tenant-pool'
  location: location
  sku: {
    name: 'StandardPool'
    tier: 'Standard'
    capacity: 200 // Total DTUs
  }
  properties: {
    perDatabaseSettings: {
      minCapacity: 0
      maxCapacity: 50 // Max 50 DTUs per database (tenant)
    }
  }
}
```

#### Cosmos DB with Throughput Limits

```csharp
// Per-tenant containers with dedicated throughput
public async Task CreateTenantContainerAsync(int tenantId, TenantTier tier)
{
    var throughput = tier switch
    {
        TenantTier.Enterprise => 10000,
        TenantTier.Premium => 2000,
        TenantTier.Standard => 400,
        _ => 400
    };

    await _database.CreateContainerAsync(new ContainerProperties
    {
        Id = $"tenant-{tenantId}",
        PartitionKeyPath = "/category"
    }, throughput);
}
```

## Test Cases

### Unit Test: Rate Limiting

```csharp
[Fact]
public async Task RateLimiter_ShouldBlockExcessiveRequests()
{
    // Arrange
    var client = _factory.CreateClient();
    client.DefaultRequestHeaders.Add("X-Tenant-Id", "test-tenant");

    // Act - Make requests up to limit
    var tasks = Enumerable.Range(0, 150)
        .Select(_ => client.GetAsync("/api/search?q=test"));
    var responses = await Task.WhenAll(tasks);

    // Assert - Some should be rate limited
    var successCount = responses.Count(r => r.IsSuccessStatusCode);
    var rateLimitedCount = responses.Count(r => r.StatusCode == HttpStatusCode.TooManyRequests);

    Assert.True(successCount <= 100, "Should not exceed rate limit");
    Assert.True(rateLimitedCount > 0, "Excess requests should be rate limited");
}
```

### Integration Test: Query Timeout

```csharp
[Fact]
public async Task LongRunningQuery_ShouldTimeout()
{
    // Arrange
    var tenantId = 1;
    await _tenantSettings.SetQueryTimeoutAsync(tenantId, TimeSpan.FromSeconds(1));

    // Seed lots of data to make query slow
    await SeedManyOrdersAsync(tenantId, 1000000);

    // Act & Assert
    await Assert.ThrowsAsync<TenantQuotaExceededException>(
        () => _repository.GetOrdersAsync(tenantId, page: 1, pageSize: 10000));
}
```

### Load Test: Tenant Isolation

```csharp
[Fact]
public async Task NoisyTenant_ShouldNotAffectOthers()
{
    // Arrange
    var normalTenantId = 1;
    var noisyTenantId = 2;

    // Start noisy tenant making heavy requests
    var noisyTask = GenerateHeavyLoadAsync(noisyTenantId);

    // Act - Measure normal tenant response times
    var stopwatch = Stopwatch.StartNew();
    await _service.GetOrdersAsync(normalTenantId);
    var normalLatency = stopwatch.ElapsedMilliseconds;

    // Assert - Normal tenant should not be affected
    Assert.True(normalLatency < 100,
        $"Normal tenant latency ({normalLatency}ms) should be < 100ms even with noisy neighbor");
}
```

## Best Practices Checklist

- [ ] Implement rate limiting per tenant
- [ ] Set query timeouts per tenant tier
- [ ] Enforce storage quotas
- [ ] Use Resource Governor in SQL
- [ ] Consider database-per-tenant for enterprise
- [ ] Implement separate queues by priority
- [ ] Add per-tenant monitoring and alerting
- [ ] Document SLAs per tier
- [ ] Implement tenant usage dashboards
- [ ] Plan for tenant isolation upgrades

## AWS to Azure Migration Note

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| DynamoDB per-table capacity | Cosmos DB per-container throughput |
| RDS instance per tenant | Azure SQL Elastic Pool / MI |
| API Gateway usage plans | APIM rate limiting policies |
| SQS per-tenant queues | Service Bus per-tier queues |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [No Caching](07-no-caching.md)*

*Next: [Retry Storm](09-retry-storm.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
