# Anti-Pattern: Chatty I/O

## Overview

The Chatty I/O anti-pattern occurs when an application makes many small network requests instead of fewer, larger requests. Each request carries overhead (network latency, connection setup, serialization), and when multiplied by thousands of requests, this overhead dominates application performance.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CHATTY I/O ANTI-PATTERN                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Client                              Server / Database                      │
│                                                                              │
│   ┌─────────┐                        ┌─────────────────────┐                │
│   │         │ ───── Request 1 ─────► │                     │                │
│   │         │ ◄──── Response 1 ───── │                     │                │
│   │         │                        │                     │                │
│   │         │ ───── Request 2 ─────► │                     │                │
│   │         │ ◄──── Response 2 ───── │                     │                │
│   │         │                        │                     │                │
│   │         │ ───── Request 3 ─────► │                     │                │
│   │         │ ◄──── Response 3 ───── │  Database /         │                │
│   │ Client  │                        │  API Server         │                │
│   │         │ ───── Request 4 ─────► │                     │                │
│   │         │ ◄──── Response 4 ───── │                     │                │
│   │         │                        │                     │                │
│   │         │         ...            │                     │                │
│   │         │    (100s of trips)     │                     │                │
│   │         │                        │                     │                │
│   └─────────┘                        └─────────────────────┘                │
│                                                                              │
│   Each request: 50ms latency                                                │
│   100 requests: 5000ms total latency!                                       │
│                                                                              │
│   ❌ Network latency multiplied by request count                            │
│   ❌ Connection overhead per request                                        │
│   ❌ Serialization/deserialization overhead                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Fetching Related Data One-by-One

```csharp
// ANTI-PATTERN: Multiple database calls in a loop
public async Task<OrderDetails> GetOrderDetailsAsync(int orderId)
{
    var order = await _db.Orders.FindAsync(orderId);

    // N+1 query problem
    foreach (var lineItem in order.LineItems)
    {
        // Separate call for each product
        lineItem.Product = await _db.Products.FindAsync(lineItem.ProductId);

        // Separate call for each supplier
        lineItem.Product.Supplier = await _db.Suppliers.FindAsync(lineItem.Product.SupplierId);
    }

    // Separate call for customer
    order.Customer = await _db.Customers.FindAsync(order.CustomerId);

    return order;
}

// 10 line items = 1 + 10 + 10 + 1 = 22 database calls!
```

### 2. API Calls in Loops

```javascript
// ANTI-PATTERN: Making individual API calls in a loop
async function loadUserDashboard(userId) {
    const user = await fetch(`/api/users/${userId}`).then(r => r.json());

    // Fetching each item individually
    const orders = [];
    for (const orderId of user.orderIds) {
        const order = await fetch(`/api/orders/${orderId}`).then(r => r.json());
        orders.push(order);
    }

    const recommendations = [];
    for (const productId of user.viewedProducts) {
        const rec = await fetch(`/api/products/${productId}/recommendations`).then(r => r.json());
        recommendations.push(...rec);
    }

    return { user, orders, recommendations };
}
```

### 3. Polling for Updates

```javascript
// ANTI-PATTERN: Frequent polling
setInterval(async () => {
    const status = await fetch('/api/job/status').then(r => r.json());
    if (status.complete) {
        handleComplete(status);
    }
}, 100); // Polling every 100ms!
```

### 4. Single-Property Updates

```csharp
// ANTI-PATTERN: Updating properties one at a time
public async Task UpdateUserProfileAsync(UserProfile profile)
{
    await UpdateFieldAsync("users", profile.Id, "firstName", profile.FirstName);
    await UpdateFieldAsync("users", profile.Id, "lastName", profile.LastName);
    await UpdateFieldAsync("users", profile.Id, "email", profile.Email);
    await UpdateFieldAsync("users", profile.Id, "phone", profile.Phone);
    await UpdateFieldAsync("users", profile.Id, "address", profile.Address);
    // 5 separate network calls!
}
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| High latency | Response times don't match payload size |
| Low throughput | Network not fully utilized |
| Many connections | High connection count in metrics |
| Sequential waits | Waterfall pattern in network traces |
| DTU/RU consumption | Database shows many small operations |

### Detection Queries

```sql
-- Azure SQL: Find chatty query patterns
SELECT TOP 20
    qs.execution_count,
    qs.total_elapsed_time / qs.execution_count AS avg_elapsed_time,
    SUBSTRING(st.text, 1, 100) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE qs.execution_count > 1000
ORDER BY qs.execution_count DESC;
```

```kusto
// Application Insights: Find chatty dependencies
dependencies
| where timestamp > ago(1h)
| summarize Count = count(), AvgDuration = avg(duration) by target, name
| where Count > 100
| order by Count desc
```

```kusto
// Cosmos DB: High RU consumption from many small requests
AzureDiagnostics
| where ResourceType == "DOCUMENTDB"
| summarize RequestCount = count(), TotalRU = sum(requestCharge_s) by bin(TimeGenerated, 1m)
| where RequestCount > 1000
| order by TimeGenerated desc
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Client                              Server / Database                      │
│                                                                              │
│   ┌─────────┐                        ┌─────────────────────┐                │
│   │         │                        │                     │                │
│   │         │ ══════ Batch ════════► │                     │                │
│   │         │        Request         │  Process all        │                │
│   │ Client  │                        │  items together     │                │
│   │         │ ◄═════ Batch ════════  │                     │                │
│   │         │        Response        │                     │                │
│   │         │                        │                     │                │
│   └─────────┘                        └─────────────────────┘                │
│                                                                              │
│   Single request: 50ms latency                                              │
│   Batch of 100 items: Still ~50ms latency!                                  │
│                                                                              │
│   ✅ One round trip instead of many                                         │
│   ✅ Single connection, single serialization                                │
│   ✅ Network fully utilized                                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: Batch Database Queries

```csharp
// SOLUTION: Use Include/ThenInclude for eager loading
public async Task<OrderDetails> GetOrderDetailsAsync(int orderId)
{
    return await _db.Orders
        .Include(o => o.Customer)
        .Include(o => o.LineItems)
            .ThenInclude(li => li.Product)
                .ThenInclude(p => p.Supplier)
        .FirstOrDefaultAsync(o => o.OrderId == orderId);
}

// Single query with JOINs instead of 22 separate calls!
```

### Solution 2: Bulk API Operations

```javascript
// SOLUTION: Batch API endpoint
async function loadUserDashboard(userId) {
    // Single request for all data
    const response = await fetch('/api/dashboard', {
        method: 'POST',
        body: JSON.stringify({ userId })
    });

    return response.json();
}

// Server-side aggregation
[HttpPost("dashboard")]
public async Task<DashboardData> GetDashboard([FromBody] DashboardRequest request)
{
    // Parallel data fetching on server
    var userTask = _userService.GetUserAsync(request.UserId);
    var ordersTask = _orderService.GetUserOrdersAsync(request.UserId);
    var recommendationsTask = _recService.GetRecommendationsAsync(request.UserId);

    await Task.WhenAll(userTask, ordersTask, recommendationsTask);

    return new DashboardData
    {
        User = userTask.Result,
        Orders = ordersTask.Result,
        Recommendations = recommendationsTask.Result
    };
}
```

### Solution 3: GraphQL for Flexible Queries

```graphql
# SOLUTION: Single GraphQL query fetches all needed data
query GetOrderDetails($orderId: ID!) {
  order(id: $orderId) {
    id
    orderDate
    total
    customer {
      id
      name
      email
    }
    lineItems {
      quantity
      unitPrice
      product {
        name
        sku
        supplier {
          name
          contactEmail
        }
      }
    }
  }
}
```

### Solution 4: Replace Polling with Push

```javascript
// SOLUTION: Use SignalR instead of polling
const connection = new signalR.HubConnectionBuilder()
    .withUrl('/jobHub')
    .build();

connection.on('JobStatusUpdate', (status) => {
    updateUI(status);
    if (status.complete) {
        handleComplete(status);
    }
});

await connection.start();
await connection.invoke('SubscribeToJob', jobId);
```

### Solution 5: Batch Updates

```csharp
// SOLUTION: Single update for all fields
public async Task UpdateUserProfileAsync(UserProfile profile)
{
    await _db.Users
        .Where(u => u.Id == profile.Id)
        .ExecuteUpdateAsync(setters => setters
            .SetProperty(u => u.FirstName, profile.FirstName)
            .SetProperty(u => u.LastName, profile.LastName)
            .SetProperty(u => u.Email, profile.Email)
            .SetProperty(u => u.Phone, profile.Phone)
            .SetProperty(u => u.Address, profile.Address));
}
```

### Azure-Specific Solutions

#### Cosmos DB Bulk Operations

```csharp
// Cosmos DB bulk operations
public async Task ImportItemsAsync(IEnumerable<MyItem> items)
{
    var container = _cosmosClient.GetContainer("mydb", "mycontainer");

    var bulkOperations = items.Select(item =>
        container.CreateItemAsync(item, new PartitionKey(item.PartitionKey))
    );

    // Execute all operations in parallel
    var results = await Task.WhenAll(bulkOperations);
}

// Or use bulk executor for very large batches
container = await database.CreateContainerAsync(
    new ContainerProperties(containerId, "/pk")
    {
        AllowBulkExecution = true
    });
```

#### Azure SQL Batch Operations

```csharp
// Use Table-Valued Parameters for batch inserts
public async Task InsertOrdersAsync(IEnumerable<Order> orders)
{
    var table = new DataTable();
    table.Columns.Add("OrderId", typeof(int));
    table.Columns.Add("CustomerId", typeof(int));
    table.Columns.Add("Total", typeof(decimal));

    foreach (var order in orders)
    {
        table.Rows.Add(order.OrderId, order.CustomerId, order.Total);
    }

    using var connection = new SqlConnection(_connectionString);
    await connection.OpenAsync();

    using var command = connection.CreateCommand();
    command.CommandText = "INSERT INTO Orders SELECT * FROM @Orders";
    command.Parameters.AddWithValue("@Orders", table);

    await command.ExecuteNonQueryAsync();
}
```

#### Azure Service Bus Batch Send

```csharp
// Batch message sending
public async Task SendMessagesAsync(IEnumerable<OrderMessage> messages)
{
    await using var sender = _serviceBusClient.CreateSender("orders");

    // Create batch
    using var batch = await sender.CreateMessageBatchAsync();

    foreach (var message in messages)
    {
        var sbMessage = new ServiceBusMessage(JsonSerializer.Serialize(message));
        if (!batch.TryAddMessage(sbMessage))
        {
            // Batch is full, send it
            await sender.SendMessagesAsync(batch);
            batch = await sender.CreateMessageBatchAsync();
            batch.TryAddMessage(sbMessage);
        }
    }

    // Send remaining messages
    if (batch.Count > 0)
    {
        await sender.SendMessagesAsync(batch);
    }
}
```

## Test Cases

### Unit Test: Verify Batch Operations

```csharp
[Fact]
public async Task GetOrderDetails_ShouldUseSingleQuery()
{
    // Arrange
    var options = new DbContextOptionsBuilder<AppDbContext>()
        .UseSqlServer(_connectionString)
        .LogTo(sql => _queryLog.Add(sql), LogLevel.Information)
        .Options;

    using var context = new AppDbContext(options);
    var service = new OrderService(context);

    // Act
    _queryLog.Clear();
    await service.GetOrderDetailsAsync(1);

    // Assert - should be single query, not N+1
    var selectQueries = _queryLog.Count(q => q.Contains("SELECT"));
    Assert.Equal(1, selectQueries);
}
```

### Performance Test: Network Calls

```csharp
[Fact]
public async Task Dashboard_ShouldMinimizeNetworkCalls()
{
    // Arrange
    var httpHandler = new MockHttpMessageHandler();
    var client = new HttpClient(httpHandler);
    var service = new DashboardService(client);

    // Act
    await service.LoadDashboardAsync(userId: 1);

    // Assert - should be 1 call, not many
    Assert.Equal(1, httpHandler.RequestCount);
}
```

### Load Test: Latency Under Load

```csharp
[Fact]
public async Task BatchOperation_ShouldScaleLinearly()
{
    var stopwatch = new Stopwatch();

    // Test with 10 items
    stopwatch.Start();
    await _service.ProcessBatchAsync(GenerateItems(10));
    var time10 = stopwatch.ElapsedMilliseconds;

    // Test with 100 items
    stopwatch.Restart();
    await _service.ProcessBatchAsync(GenerateItems(100));
    var time100 = stopwatch.ElapsedMilliseconds;

    // 10x more items should not be 10x slower
    Assert.True(time100 < time10 * 3,
        $"100 items took {time100}ms, expected < {time10 * 3}ms");
}
```

## Best Practices Checklist

- [ ] Use eager loading (Include) for related data
- [ ] Implement batch/bulk API endpoints
- [ ] Use GraphQL or OData for flexible queries
- [ ] Replace polling with WebSockets/SignalR
- [ ] Batch database operations (bulk insert/update)
- [ ] Use Table-Valued Parameters for SQL Server
- [ ] Enable Cosmos DB bulk execution
- [ ] Batch message queue operations
- [ ] Cache frequently accessed data
- [ ] Use connection pooling

## AWS to Azure Migration Note

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| DynamoDB BatchWriteItem | Cosmos DB Bulk Operations |
| SQS SendMessageBatch | Service Bus Batch Send |
| Lambda Batching | Azure Functions Batching |
| AppSync Batching | Hot Chocolate DataLoader |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [Busy Front End](02-busy-frontend.md)*

*Next: [Extraneous Fetching](04-extraneous-fetching.md)*
