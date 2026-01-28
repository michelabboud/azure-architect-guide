# Anti-Pattern: Monolithic Persistence

## Overview

The Monolithic Persistence anti-pattern occurs when an application uses a single data store for all data, regardless of the data's characteristics, access patterns, or requirements. This approach leads to suboptimal performance because different types of data have different storage needs.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   MONOLITHIC PERSISTENCE ANTI-PATTERN                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   All Data Types → Single Database                                          │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │  Transactional Data    ──┐                                          │   │
│   │  (Orders, Payments)       │                                          │   │
│   │                           │      ┌────────────────────────────────┐ │   │
│   │  Session Data          ──┼─────►│                                │ │   │
│   │  (User sessions)          │      │      SQL Server / RDS          │ │   │
│   │                           │      │                                │ │   │
│   │  Search Data           ──┼─────►│      Single Database           │ │   │
│   │  (Product catalog)        │      │                                │ │   │
│   │                           │      │      ❌ Jack of all trades     │ │   │
│   │  Time-Series Data      ──┼─────►│      ❌ Master of none         │ │   │
│   │  (IoT sensor data)        │      │                                │ │   │
│   │                           │      └────────────────────────────────┘ │   │
│   │  Graph Data            ──┼                                          │   │
│   │  (Social connections)     │                                          │   │
│   │                           │                                          │   │
│   │  Binary Data           ──┘                                          │   │
│   │  (Images, files)                                                     │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Problems:                                                                  │
│   • Full-text search is slow (SQL LIKE queries)                             │
│   • Graph queries require complex recursive CTEs                            │
│   • Time-series data causes table bloat                                     │
│   • Sessions don't need ACID guarantees                                     │
│   • Binary files waste expensive database storage                           │
│   • Can't scale read-heavy and write-heavy workloads independently         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Everything in SQL

```csharp
// ANTI-PATTERN: All data types in single SQL database

// User sessions (high read/write, ephemeral)
await _db.Sessions.AddAsync(new Session
{
    SessionId = Guid.NewGuid(),
    UserId = userId,
    Data = JsonSerializer.Serialize(sessionData),
    ExpiresAt = DateTime.UtcNow.AddHours(1)
});

// Product search (needs full-text, facets)
var products = await _db.Products
    .Where(p => p.Name.Contains(searchTerm) || p.Description.Contains(searchTerm))
    .ToListAsync(); // Slow LIKE queries!

// Time-series IoT data (append-only, high volume)
await _db.SensorReadings.AddAsync(new SensorReading
{
    SensorId = sensorId,
    Timestamp = DateTime.UtcNow,
    Value = reading
}); // Millions of rows!

// Social graph (needs traversal queries)
var friends = await _db.Relationships
    .Where(r => r.UserId == userId && r.Type == "friend")
    .ToListAsync();
var friendsOfFriends = await _db.Relationships
    .Where(r => friends.Select(f => f.TargetUserId).Contains(r.UserId))
    .ToListAsync(); // Complex, inefficient!

// Files stored as BLOBs
var document = await _db.Documents
    .Where(d => d.Id == documentId)
    .Select(d => d.FileContent)
    .FirstOrDefaultAsync(); // 50MB BLOB in database!
```

### 2. NoSQL for Everything

```javascript
// ANTI-PATTERN: All data in document database (opposite extreme)

// Transactional data that needs ACID
await ordersCollection.insertOne({
    orderId: generateId(),
    items: items,
    total: calculateTotal(items),
    payment: paymentDetails
}); // No multi-document transactions!

// Financial data requiring strong consistency
await accountsCollection.updateOne(
    { accountId: fromAccount },
    { $inc: { balance: -amount } }
);
await accountsCollection.updateOne(
    { accountId: toAccount },
    { $inc: { balance: amount } }
); // Race condition risk!
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| Database bloat | Single database growing to TB+ |
| Mixed workloads | OLTP and OLAP queries competing |
| Slow queries | Some query types always slow |
| Scaling issues | Can't scale specific workloads |
| High costs | Paying for premium tier for all data |
| Backup problems | Backups take hours |

### Detection Queries

```sql
-- Find tables with different access patterns
SELECT
    t.name AS TableName,
    SUM(ios.num_of_reads) AS Reads,
    SUM(ios.num_of_writes) AS Writes,
    CASE
        WHEN SUM(ios.num_of_reads) > SUM(ios.num_of_writes) * 10 THEN 'Read-Heavy'
        WHEN SUM(ios.num_of_writes) > SUM(ios.num_of_reads) * 10 THEN 'Write-Heavy'
        ELSE 'Mixed'
    END AS Pattern
FROM sys.dm_db_index_operational_stats(DB_ID(), NULL, NULL, NULL) ios
JOIN sys.tables t ON ios.object_id = t.object_id
GROUP BY t.name
ORDER BY Reads + Writes DESC;
```

```sql
-- Find large tables that might be candidates for separate storage
SELECT
    t.name AS TableName,
    p.rows AS RowCount,
    (SUM(a.total_pages) * 8) / 1024 AS TotalSpaceMB
FROM sys.tables t
JOIN sys.indexes i ON t.object_id = i.object_id
JOIN sys.partitions p ON i.object_id = p.object_id AND i.index_id = p.index_id
JOIN sys.allocation_units a ON p.partition_id = a.container_id
GROUP BY t.name, p.rows
ORDER BY TotalSpaceMB DESC;
```

## Solution

### Correct Architecture: Polyglot Persistence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    POLYGLOT PERSISTENCE ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Different Data → Appropriate Storage                                      │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │  Transactional Data    ────────► Azure SQL / PostgreSQL             │   │
│   │  (Orders, Payments)              • ACID transactions                 │   │
│   │                                  • Referential integrity             │   │
│   │                                                                      │   │
│   │  Session/Cache Data    ────────► Azure Cache for Redis              │   │
│   │  (User sessions)                 • Sub-millisecond latency          │   │
│   │                                  • Built-in expiration              │   │
│   │                                                                      │   │
│   │  Search Data           ────────► Azure Cognitive Search             │   │
│   │  (Product catalog)               • Full-text search                  │   │
│   │                                  • Facets, filters, scoring          │   │
│   │                                                                      │   │
│   │  Document Data         ────────► Cosmos DB                          │   │
│   │  (User profiles)                 • Flexible schema                   │   │
│   │                                  • Global distribution               │   │
│   │                                                                      │   │
│   │  Time-Series Data      ────────► Azure Data Explorer                │   │
│   │  (IoT, metrics)                  • Optimized for append             │   │
│   │                                  • Powerful analytics                │   │
│   │                                                                      │   │
│   │  Graph Data            ────────► Cosmos DB (Gremlin)                │   │
│   │  (Social connections)            • Native graph traversal           │   │
│   │                                                                      │   │
│   │  Binary Files          ────────► Azure Blob Storage                 │   │
│   │  (Images, documents)             • Cheap, scalable storage          │   │
│   │                                  • CDN integration                   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ✅ Each data type optimized for its access pattern                        │
│   ✅ Scale each store independently                                         │
│   ✅ Right cost model for each workload                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution Implementation

```csharp
// SOLUTION: Polyglot persistence with appropriate stores

public class ApplicationServices
{
    // Transactional data - SQL
    private readonly AppDbContext _sqlDb;

    // Session/Cache - Redis
    private readonly IConnectionMultiplexer _redis;

    // Search - Cognitive Search
    private readonly SearchClient _searchClient;

    // Documents - Cosmos DB
    private readonly CosmosClient _cosmosClient;

    // Files - Blob Storage
    private readonly BlobServiceClient _blobClient;

    // Time-series - Data Explorer
    private readonly ICslQueryProvider _kustoClient;

    public ApplicationServices(/* inject all clients */) { }
}
```

### Session Data → Redis

```csharp
// Session data in Redis
public class SessionService
{
    private readonly IDatabase _redis;

    public async Task CreateSessionAsync(string sessionId, SessionData data)
    {
        var json = JsonSerializer.Serialize(data);
        await _redis.StringSetAsync(
            $"session:{sessionId}",
            json,
            expiry: TimeSpan.FromHours(1)
        );
    }

    public async Task<SessionData?> GetSessionAsync(string sessionId)
    {
        var json = await _redis.StringGetAsync($"session:{sessionId}");
        if (json.IsNullOrEmpty) return null;
        return JsonSerializer.Deserialize<SessionData>(json!);
    }
}
```

### Search Data → Cognitive Search

```csharp
// Product search in Azure Cognitive Search
public class ProductSearchService
{
    private readonly SearchClient _searchClient;

    public async Task<SearchResults<Product>> SearchProductsAsync(
        string query,
        List<string> facets,
        Dictionary<string, string> filters)
    {
        var options = new SearchOptions
        {
            IncludeTotalCount = true,
            Filter = BuildFilter(filters),
            Facets = { "category", "brand", "priceRange" },
            HighlightFields = { "name", "description" },
            SearchMode = SearchMode.All
        };

        return await _searchClient.SearchAsync<Product>(query, options);
    }

    public async Task IndexProductAsync(Product product)
    {
        await _searchClient.MergeOrUploadDocumentsAsync(new[] { product });
    }
}
```

### Document Data → Cosmos DB

```csharp
// User profiles in Cosmos DB (flexible schema)
public class UserProfileService
{
    private readonly Container _container;

    public async Task<UserProfile> GetProfileAsync(string userId)
    {
        return await _container.ReadItemAsync<UserProfile>(
            userId,
            new PartitionKey(userId)
        );
    }

    public async Task UpdateProfileAsync(UserProfile profile)
    {
        await _container.UpsertItemAsync(profile, new PartitionKey(profile.UserId));
    }
}
```

### Time-Series Data → Data Explorer

```csharp
// IoT data in Azure Data Explorer
public class TelemetryService
{
    private readonly ICslQueryProvider _kustoClient;
    private readonly IKustoIngestClient _ingestClient;

    public async Task IngestTelemetryAsync(IEnumerable<SensorReading> readings)
    {
        // Use streaming ingestion for real-time
        using var stream = new MemoryStream();
        await JsonSerializer.SerializeAsync(stream, readings);
        stream.Position = 0;

        await _ingestClient.IngestFromStreamAsync(
            stream,
            new KustoIngestionProperties("iotdb", "telemetry"));
    }

    public async Task<IDataReader> QueryTelemetryAsync(
        string sensorId,
        DateTime start,
        DateTime end)
    {
        var query = $@"
            telemetry
            | where SensorId == '{sensorId}'
            | where Timestamp between (datetime({start:O}) .. datetime({end:O}))
            | summarize AvgValue = avg(Value), MaxValue = max(Value) by bin(Timestamp, 1h)
            | order by Timestamp asc";

        return await _kustoClient.ExecuteQueryAsync("iotdb", query);
    }
}
```

### Binary Files → Blob Storage

```csharp
// Files in Blob Storage
public class DocumentStorageService
{
    private readonly BlobContainerClient _container;
    private readonly AppDbContext _db;

    public async Task<string> UploadDocumentAsync(Stream content, string fileName)
    {
        // Store file in Blob Storage
        var blobName = $"{Guid.NewGuid()}/{fileName}";
        var blobClient = _container.GetBlobClient(blobName);
        await blobClient.UploadAsync(content);

        // Store metadata in SQL
        var doc = new DocumentMetadata
        {
            Id = Guid.NewGuid(),
            FileName = fileName,
            BlobPath = blobName,
            UploadedAt = DateTime.UtcNow
        };
        _db.Documents.Add(doc);
        await _db.SaveChangesAsync();

        return blobClient.Uri.ToString();
    }

    public async Task<Stream> DownloadDocumentAsync(Guid documentId)
    {
        var metadata = await _db.Documents.FindAsync(documentId);
        var blobClient = _container.GetBlobClient(metadata.BlobPath);
        return await blobClient.OpenReadAsync();
    }
}
```

## Data Store Selection Guide

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      DATA STORE SELECTION GUIDE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Question                              → Recommended Store                  │
│   ────────                                 ─────────────────                  │
│                                                                              │
│   Need ACID transactions?               → Azure SQL / PostgreSQL            │
│   Need referential integrity?           → Azure SQL / PostgreSQL            │
│                                                                              │
│   High-speed cache/sessions?            → Azure Cache for Redis             │
│   Ephemeral data with TTL?              → Azure Cache for Redis             │
│                                                                              │
│   Full-text search with facets?         → Azure Cognitive Search            │
│   Relevance scoring needed?             → Azure Cognitive Search            │
│                                                                              │
│   Flexible/evolving schema?             → Cosmos DB                         │
│   Global distribution needed?           → Cosmos DB                         │
│   Multi-model (doc, graph, table)?      → Cosmos DB                         │
│                                                                              │
│   Time-series / IoT data?               → Azure Data Explorer               │
│   Log analytics?                        → Azure Data Explorer               │
│                                                                              │
│   Graph traversal queries?              → Cosmos DB (Gremlin)               │
│   Social network relationships?         → Cosmos DB (Gremlin)               │
│                                                                              │
│   Large binary files?                   → Azure Blob Storage                │
│   Static content / CDN?                 → Azure Blob Storage + CDN          │
│                                                                              │
│   OLAP / Analytics?                     → Azure Synapse Analytics           │
│   Data warehousing?                     → Azure Synapse Analytics           │
│                                                                              │
│   Message queuing?                      → Azure Service Bus                 │
│   Event streaming?                      → Azure Event Hubs                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Test Cases

### Unit Test: Verify Correct Store Usage

```csharp
[Fact]
public async Task SessionData_ShouldUseRedis()
{
    // Arrange
    var redisMock = new Mock<IDatabase>();
    var sessionService = new SessionService(redisMock.Object);

    // Act
    await sessionService.CreateSessionAsync("test-session", new SessionData());

    // Assert - verify Redis was called, not SQL
    redisMock.Verify(r => r.StringSetAsync(
        It.Is<RedisKey>(k => k.ToString().StartsWith("session:")),
        It.IsAny<RedisValue>(),
        It.IsAny<TimeSpan?>(),
        It.IsAny<bool>(),
        It.IsAny<When>(),
        It.IsAny<CommandFlags>()
    ), Times.Once);
}
```

### Performance Test: Query Latency

```csharp
[Theory]
[InlineData("Redis", 10)]      // Sessions should be < 10ms
[InlineData("Search", 100)]    // Search should be < 100ms
[InlineData("SQL", 50)]        // Transactions should be < 50ms
public async Task QueryLatency_ShouldMeetSLA(string storeType, int maxMs)
{
    var stopwatch = Stopwatch.StartNew();

    switch (storeType)
    {
        case "Redis":
            await _sessionService.GetSessionAsync("test");
            break;
        case "Search":
            await _searchService.SearchProductsAsync("widget", null, null);
            break;
        case "SQL":
            await _orderService.GetOrderAsync(1);
            break;
    }

    Assert.True(stopwatch.ElapsedMilliseconds < maxMs,
        $"{storeType} query took {stopwatch.ElapsedMilliseconds}ms, expected < {maxMs}ms");
}
```

## Best Practices Checklist

- [ ] Analyze data access patterns before choosing storage
- [ ] Use SQL for transactional data requiring ACID
- [ ] Use Redis for sessions and caching
- [ ] Use specialized search services for full-text search
- [ ] Use Cosmos DB for flexible schema documents
- [ ] Use Blob Storage for binary files
- [ ] Use Data Explorer for time-series/logs
- [ ] Implement data synchronization where needed
- [ ] Consider eventual consistency trade-offs
- [ ] Monitor each data store independently

## AWS to Azure Migration Note

| AWS Service | Azure Equivalent |
|-------------|-----------------|
| RDS | Azure SQL / PostgreSQL |
| ElastiCache | Azure Cache for Redis |
| DynamoDB | Cosmos DB |
| Elasticsearch | Azure Cognitive Search |
| S3 | Azure Blob Storage |
| Timestream | Azure Data Explorer |
| Neptune | Cosmos DB (Gremlin) |
| Redshift | Azure Synapse Analytics |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [Improper Instantiation](05-improper-instantiation.md)*

*Next: [No Caching](07-no-caching.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
