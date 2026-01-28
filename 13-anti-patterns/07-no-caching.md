# Anti-Pattern: No Caching

## Overview

The No Caching anti-pattern occurs when an application repeatedly fetches the same data from slower data stores (databases, external APIs) when that data could be cached. This creates unnecessary load on backend systems, increases latency, and wastes resources.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         NO CACHING ANTI-PATTERN                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   User 1                  User 2                  User 3                    │
│     │                       │                       │                        │
│     │ GET /products/123     │ GET /products/123     │ GET /products/123     │
│     │                       │                       │                        │
│     ▼                       ▼                       ▼                        │
│   ┌───────────────────────────────────────────────────────────────────────┐ │
│   │                         Application                                    │ │
│   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐              │ │
│   │   │   Request   │    │   Request   │    │   Request   │              │ │
│   │   │      1      │    │      2      │    │      3      │              │ │
│   │   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘              │ │
│   │          │                  │                  │                      │ │
│   │          │ Query            │ Query            │ Query                │ │
│   │          ▼                  ▼                  ▼                      │ │
│   │   ┌─────────────────────────────────────────────────────────────────┐│ │
│   │   │                      DATABASE                                    ││ │
│   │   │                                                                  ││ │
│   │   │   Same query executed 3 times!                                   ││ │
│   │   │   SELECT * FROM Products WHERE Id = 123                         ││ │
│   │   │                                                                  ││ │
│   │   └─────────────────────────────────────────────────────────────────┘│ │
│   └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│   Problems:                                                                  │
│   • 3x database load for identical data                                     │
│   • 3x network latency (50ms × 3 = 150ms cumulative)                       │
│   • 3x connection pool usage                                                │
│   • Wasted compute resources                                                │
│   • Poor scalability under load                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Fetching Configuration Repeatedly

```csharp
// ANTI-PATTERN: Reading config from database on every request
public class PricingService
{
    public async Task<decimal> CalculatePriceAsync(int productId)
    {
        // Hits database every time!
        var taxRate = await _db.Settings
            .Where(s => s.Key == "TaxRate")
            .Select(s => s.Value)
            .FirstOrDefaultAsync();

        var discountRules = await _db.Settings
            .Where(s => s.Key.StartsWith("Discount"))
            .ToListAsync();

        var product = await _db.Products.FindAsync(productId);
        return CalculateWithRules(product, taxRate, discountRules);
    }
}
// Tax rate changes once a year, but queried millions of times daily!
```

### 2. Repeated API Calls

```csharp
// ANTI-PATTERN: Calling external API for same data
public class WeatherService
{
    public async Task<Weather> GetWeatherAsync(string city)
    {
        // External API called every time!
        var response = await _httpClient.GetAsync(
            $"https://api.weather.com/current?city={city}");

        return await response.Content.ReadFromJsonAsync<Weather>();
    }
}
// Weather changes hourly, but API called every second!
```

### 3. No Session Caching

```csharp
// ANTI-PATTERN: Loading user data on every request
public class UserContextMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        var userId = context.User.GetUserId();

        // Database call on EVERY request
        var user = await _db.Users
            .Include(u => u.Roles)
            .Include(u => u.Permissions)
            .FirstOrDefaultAsync(u => u.Id == userId);

        context.Items["User"] = user;
        await _next(context);
    }
}
```

### 4. Repeated Aggregations

```csharp
// ANTI-PATTERN: Computing expensive aggregations repeatedly
public class DashboardService
{
    public async Task<DashboardStats> GetStatsAsync()
    {
        // Expensive queries on every page load
        var totalOrders = await _db.Orders.CountAsync();
        var revenue = await _db.Orders.SumAsync(o => o.Total);
        var topProducts = await _db.OrderItems
            .GroupBy(oi => oi.ProductId)
            .OrderByDescending(g => g.Sum(oi => oi.Quantity))
            .Take(10)
            .ToListAsync();

        return new DashboardStats { /* ... */ };
    }
}
// Dashboard viewed 1000 times/minute, stats change once/minute
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| High database load | CPU/DTU constantly high |
| Repeated queries | Same queries in slow query log |
| High latency | Response times increase under load |
| API rate limiting | External APIs throttling requests |
| Linear scaling issues | Adding users linearly increases DB load |

### Detection Queries

```sql
-- Find repeated identical queries
SELECT TOP 20
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count AS avg_time,
    SUBSTRING(st.text, 1, 200) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.execution_count > 1000
ORDER BY qs.execution_count DESC;
```

```kusto
// Application Insights: Find repetitive dependencies
dependencies
| where timestamp > ago(1h)
| summarize Count = count(), AvgDuration = avg(duration) by target, name, data
| where Count > 100
| order by Count desc
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   User 1                  User 2                  User 3                    │
│     │                       │                       │                        │
│     │ GET /products/123     │ GET /products/123     │ GET /products/123     │
│     │                       │                       │                        │
│     ▼                       ▼                       ▼                        │
│   ┌───────────────────────────────────────────────────────────────────────┐ │
│   │                         Application                                    │ │
│   │                                                                        │ │
│   │   ┌─────────────────────────────────────────────────────────────────┐ │ │
│   │   │                     CACHE (Redis)                                │ │ │
│   │   │                                                                  │ │ │
│   │   │   Key: product:123                                               │ │ │
│   │   │   Value: { "id": 123, "name": "Widget", ... }                   │ │ │
│   │   │   TTL: 5 minutes                                                 │ │ │
│   │   │                                                                  │ │ │
│   │   │   ✅ User 1: Cache MISS → Query DB → Store in cache             │ │ │
│   │   │   ✅ User 2: Cache HIT → Return instantly                       │ │ │
│   │   │   ✅ User 3: Cache HIT → Return instantly                       │ │ │
│   │   │                                                                  │ │ │
│   │   └─────────────────────────────────────────────────────────────────┘ │ │
│   │                              │                                         │ │
│   │                    Only on cache miss                                  │ │
│   │                              ▼                                         │ │
│   │   ┌─────────────────────────────────────────────────────────────────┐ │ │
│   │   │                      DATABASE                                    │ │ │
│   │   │   Query executed only ONCE every 5 minutes                      │ │ │
│   │   └─────────────────────────────────────────────────────────────────┘ │ │
│   └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│   Result: 99% reduction in database load!                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: Distributed Caching with Redis

```csharp
// SOLUTION: Cache-aside pattern with Redis

public class ProductService
{
    private readonly IDatabase _cache;
    private readonly AppDbContext _db;
    private readonly TimeSpan _cacheDuration = TimeSpan.FromMinutes(5);

    public async Task<Product?> GetProductAsync(int productId)
    {
        var cacheKey = $"product:{productId}";

        // Try cache first
        var cached = await _cache.StringGetAsync(cacheKey);
        if (!cached.IsNullOrEmpty)
        {
            return JsonSerializer.Deserialize<Product>(cached!);
        }

        // Cache miss - get from database
        var product = await _db.Products.FindAsync(productId);
        if (product != null)
        {
            // Store in cache
            await _cache.StringSetAsync(
                cacheKey,
                JsonSerializer.Serialize(product),
                _cacheDuration);
        }

        return product;
    }

    public async Task UpdateProductAsync(Product product)
    {
        _db.Products.Update(product);
        await _db.SaveChangesAsync();

        // Invalidate cache
        await _cache.KeyDeleteAsync($"product:{product.Id}");
    }
}
```

### Solution 2: In-Memory Caching for Hot Data

```csharp
// SOLUTION: IMemoryCache for frequently accessed, small data

public class ConfigurationService
{
    private readonly IMemoryCache _cache;
    private readonly AppDbContext _db;

    public async Task<AppSettings> GetSettingsAsync()
    {
        return await _cache.GetOrCreateAsync("app-settings", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(15);
            entry.Priority = CacheItemPriority.High;

            return await _db.Settings
                .AsNoTracking()
                .ToDictionaryAsync(s => s.Key, s => s.Value);
        });
    }
}

// Register in DI
builder.Services.AddMemoryCache();
```

### Solution 3: Output Caching for API Responses

```csharp
// SOLUTION: Cache entire API responses

// Program.cs
builder.Services.AddOutputCache(options =>
{
    options.AddBasePolicy(builder => builder.Expire(TimeSpan.FromSeconds(30)));

    options.AddPolicy("Products", builder => builder
        .Expire(TimeSpan.FromMinutes(5))
        .Tag("products"));
});

// Controller
[HttpGet("{id}")]
[OutputCache(PolicyName = "Products")]
public async Task<ActionResult<Product>> GetProduct(int id)
{
    return await _productService.GetProductAsync(id);
}

// Invalidate when product changes
[HttpPut("{id}")]
public async Task<IActionResult> UpdateProduct(int id, Product product)
{
    await _productService.UpdateProductAsync(product);
    await _outputCacheStore.EvictByTagAsync("products", default);
    return NoContent();
}
```

### Solution 4: CDN for Static Content

```csharp
// SOLUTION: Use CDN for static/semi-static content

public class ImageService
{
    private readonly BlobServiceClient _blobClient;
    private readonly string _cdnEndpoint;

    public string GetImageUrl(string imagePath)
    {
        // Return CDN URL instead of blob URL
        return $"{_cdnEndpoint}/images/{imagePath}";
    }
}

// Azure CDN handles caching at edge locations
// Configure caching rules in Azure CDN profile
```

### Solution 5: Multi-Layer Caching

```csharp
// SOLUTION: L1 (memory) + L2 (distributed) caching

public class MultiLevelCache
{
    private readonly IMemoryCache _l1Cache;
    private readonly IDatabase _l2Cache;

    public async Task<T?> GetAsync<T>(string key) where T : class
    {
        // Try L1 first (fastest)
        if (_l1Cache.TryGetValue(key, out T? l1Value))
        {
            return l1Value;
        }

        // Try L2 (distributed cache)
        var l2Value = await _l2Cache.StringGetAsync(key);
        if (!l2Value.IsNullOrEmpty)
        {
            var value = JsonSerializer.Deserialize<T>(l2Value!);
            // Populate L1 for future requests
            _l1Cache.Set(key, value, TimeSpan.FromMinutes(1));
            return value;
        }

        return null;
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan l2Expiry)
    {
        // Set in both caches
        _l1Cache.Set(key, value, TimeSpan.FromMinutes(1));
        await _l2Cache.StringSetAsync(
            key,
            JsonSerializer.Serialize(value),
            l2Expiry);
    }
}
```

### Azure-Specific Solutions

#### Azure Cache for Redis Configuration

```csharp
// Program.cs
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = builder.Configuration.GetConnectionString("Redis");
    options.InstanceName = "myapp:";
});

// Usage with IDistributedCache
public class CachedProductService
{
    private readonly IDistributedCache _cache;
    private readonly AppDbContext _db;

    public async Task<Product?> GetProductAsync(int id)
    {
        var cacheKey = $"product:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null)
        {
            return JsonSerializer.Deserialize<Product>(cached);
        }

        var product = await _db.Products.FindAsync(id);
        if (product != null)
        {
            await _cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(product),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5),
                    SlidingExpiration = TimeSpan.FromMinutes(2)
                });
        }

        return product;
    }
}
```

#### Azure CDN with Blob Storage

```bicep
// Bicep for CDN-enabled storage
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'mystorageaccount'
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

resource cdnProfile 'Microsoft.Cdn/profiles@2023-05-01' = {
  name: 'mycdnprofile'
  location: 'global'
  sku: { name: 'Standard_Microsoft' }
}

resource cdnEndpoint 'Microsoft.Cdn/profiles/endpoints@2023-05-01' = {
  parent: cdnProfile
  name: 'myendpoint'
  location: 'global'
  properties: {
    originHostHeader: '${storageAccount.name}.blob.core.windows.net'
    origins: [{
      name: 'storage'
      properties: {
        hostName: '${storageAccount.name}.blob.core.windows.net'
      }
    }]
    deliveryPolicy: {
      rules: [{
        name: 'CacheStaticFiles'
        order: 1
        conditions: [{
          name: 'UrlFileExtension'
          parameters: {
            operator: 'Equal'
            matchValues: ['jpg', 'png', 'gif', 'css', 'js']
          }
        }]
        actions: [{
          name: 'CacheExpiration'
          parameters: {
            cacheBehavior: 'Override'
            cacheDuration: '7.00:00:00'
          }
        }]
      }]
    }
  }
}
```

## Caching Strategy Guide

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CACHING STRATEGY GUIDE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Data Characteristic              Caching Strategy         TTL             │
│   ────────────────────             ────────────────         ───             │
│                                                                              │
│   Static (images, JS, CSS)         CDN                      Days/Weeks     │
│   Reference data (countries)        Distributed + Memory    Hours          │
│   Configuration/Settings           Memory                   Minutes        │
│   User session data                Distributed (Redis)      Session TTL    │
│   API responses                    Output Cache             Seconds-Mins   │
│   Search results                   Distributed              Minutes        │
│   Real-time data                   Short TTL / None         Seconds        │
│   User-specific data               User-keyed cache         Minutes        │
│   Aggregations/Reports             Precomputed + Cache      Varies         │
│                                                                              │
│   Invalidation Strategies:                                                  │
│   • Time-based: Simplest, good for eventually consistent data              │
│   • Event-based: Invalidate on update, ensures freshness                   │
│   • Versioned: Include version in key, no explicit invalidation            │
│   • Cache-through: Write-through or write-behind patterns                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Test Cases

### Unit Test: Cache Hit Behavior

```csharp
[Fact]
public async Task GetProduct_CacheHit_ShouldNotQueryDatabase()
{
    // Arrange
    var product = new Product { Id = 1, Name = "Widget" };
    var cache = new Mock<IDistributedCache>();
    cache.Setup(c => c.GetAsync("product:1", default))
        .ReturnsAsync(JsonSerializer.SerializeToUtf8Bytes(product));

    var db = new Mock<AppDbContext>();
    var service = new CachedProductService(cache.Object, db.Object);

    // Act
    await service.GetProductAsync(1);

    // Assert - database should NOT be called
    db.Verify(d => d.Products, Times.Never);
}
```

### Integration Test: Cache Invalidation

```csharp
[Fact]
public async Task UpdateProduct_ShouldInvalidateCache()
{
    // Arrange
    var product = new Product { Id = 1, Name = "Widget" };
    await _service.GetProductAsync(1); // Prime cache

    // Act
    product.Name = "Updated Widget";
    await _service.UpdateProductAsync(product);

    // Assert - next get should hit database
    var cached = await _cache.GetStringAsync("product:1");
    Assert.Null(cached); // Cache should be invalidated
}
```

### Performance Test: Cache Effectiveness

```csharp
[Fact]
public async Task CacheShouldReduceDatabaseLoad()
{
    // Arrange
    var stopwatch = new Stopwatch();
    var iterations = 1000;

    // Act - First call (cache miss)
    stopwatch.Start();
    await _service.GetProductAsync(1);
    var firstCallMs = stopwatch.ElapsedMilliseconds;

    // Subsequent calls (cache hits)
    stopwatch.Restart();
    for (int i = 0; i < iterations - 1; i++)
    {
        await _service.GetProductAsync(1);
    }
    var cachedCallsMs = stopwatch.ElapsedMilliseconds;
    var avgCachedMs = cachedCallsMs / (iterations - 1.0);

    // Assert
    Assert.True(avgCachedMs < firstCallMs / 10,
        $"Cached calls ({avgCachedMs}ms avg) should be 10x faster than DB ({firstCallMs}ms)");
}
```

## Best Practices Checklist

- [ ] Identify frequently accessed, rarely changed data
- [ ] Use appropriate cache level (memory, distributed, CDN)
- [ ] Set appropriate TTLs based on data volatility
- [ ] Implement cache invalidation on writes
- [ ] Use cache-aside pattern for database data
- [ ] Configure CDN for static assets
- [ ] Monitor cache hit rates
- [ ] Handle cache failures gracefully
- [ ] Consider cache warming for cold starts
- [ ] Use consistent key naming conventions

## AWS to Azure Migration Note

| AWS Service | Azure Equivalent |
|-------------|-----------------|
| ElastiCache (Redis) | Azure Cache for Redis |
| CloudFront | Azure CDN / Front Door |
| DAX (DynamoDB) | Cosmos DB integrated cache |
| API Gateway caching | Azure API Management caching |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [Monolithic Persistence](06-monolithic-persistence.md)*

*Next: [Noisy Neighbor](08-noisy-neighbor.md)*
