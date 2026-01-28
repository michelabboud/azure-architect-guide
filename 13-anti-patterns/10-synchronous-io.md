# Anti-Pattern: Synchronous I/O

## Overview

The Synchronous I/O anti-pattern occurs when threads are blocked while waiting for I/O operations (database queries, HTTP requests, file operations) to complete. This wastes thread pool resources and severely limits application scalability under load.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SYNCHRONOUS I/O ANTI-PATTERN                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Thread Pool (Limited: ~100 threads)                                       │
│                                                                              │
│   Request 1: ████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░       │
│              │ Working │   BLOCKED waiting for database                     │
│                                                                              │
│   Request 2: ████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░       │
│              │ Working │   BLOCKED waiting for HTTP API                     │
│                                                                              │
│   Request 3: ████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░       │
│              │ Working │   BLOCKED waiting for file I/O                     │
│                                                                              │
│   ...                                                                        │
│                                                                              │
│   Request 100: Thread pool exhausted! New requests QUEUED                   │
│                                                                              │
│   Request 101: ⏳ Waiting for thread...                                     │
│   Request 102: ⏳ Waiting for thread...                                     │
│   Request 103: ⏳ Waiting for thread...                                     │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │   SYMPTOMS:                                                          │   │
│   │   • Response times increase dramatically under load                  │   │
│   │   • CPU usage LOW despite high latency (threads just waiting)       │   │
│   │   • Thread pool starvation errors                                   │   │
│   │   • Timeouts and request queuing                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Using .Result or .Wait() on Tasks

```csharp
// ANTI-PATTERN: Blocking on async operations
public class OrderService
{
    public Order GetOrder(int orderId)
    {
        // BLOCKS the thread until complete!
        var order = _db.Orders.FindAsync(orderId).Result;

        // More blocking
        var customer = _customerApi.GetCustomerAsync(order.CustomerId).Result;

        return order;
    }
}
```

### 2. Synchronous HTTP Calls

```csharp
// ANTI-PATTERN: Synchronous HTTP requests
public class WeatherService
{
    public Weather GetWeather(string city)
    {
        // Creates new HttpClient (also an anti-pattern!)
        using var client = new HttpClient();

        // Synchronous call blocks thread
        var response = client.GetAsync($"https://api.weather.com/{city}").Result;
        var content = response.Content.ReadAsStringAsync().Result;

        return JsonSerializer.Deserialize<Weather>(content);
    }
}
```

### 3. Synchronous File Operations

```csharp
// ANTI-PATTERN: Synchronous file I/O
public class DocumentService
{
    public string ReadDocument(string path)
    {
        // Blocks thread during file I/O
        return File.ReadAllText(path);
    }

    public void WriteDocument(string path, string content)
    {
        // Blocks thread during write
        File.WriteAllText(path, content);
    }
}
```

### 4. Mixing Sync and Async (Sync-over-Async)

```csharp
// ANTI-PATTERN: Calling async from sync context
public class ReportService
{
    public Report GenerateReport()
    {
        // Blocks thread waiting for async method
        var data = GetDataAsync().GetAwaiter().GetResult();

        return ProcessData(data);
    }

    private async Task<Data> GetDataAsync()
    {
        // This is async, but caller blocks!
        return await _db.Data.ToListAsync();
    }
}
```

### 5. Database Calls Without Async

```csharp
// ANTI-PATTERN: Synchronous database operations
public class ProductRepository
{
    public List<Product> GetProducts()
    {
        // ToList() is synchronous - blocks thread
        return _db.Products.ToList();
    }

    public void UpdateProduct(Product product)
    {
        _db.Products.Update(product);
        // SaveChanges() is synchronous
        _db.SaveChanges();
    }
}
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| Thread pool starvation | `ThreadPool.GetAvailableThreads()` returns 0 |
| Low CPU with high latency | CPU < 20% but requests take seconds |
| Timeouts under load | Works fine for few users, fails at scale |
| Thread count growth | Thread count keeps increasing |
| ASP.NET queue delays | Requests queued waiting for threads |

### Detection Code

```csharp
// Middleware to detect thread pool issues
public class ThreadPoolDiagnosticsMiddleware
{
    public async Task InvokeAsync(HttpContext context)
    {
        ThreadPool.GetAvailableThreads(out int workerThreads, out int ioThreads);
        ThreadPool.GetMaxThreads(out int maxWorkerThreads, out int maxIoThreads);

        var workerUsage = (maxWorkerThreads - workerThreads) * 100.0 / maxWorkerThreads;
        var ioUsage = (maxIoThreads - ioThreads) * 100.0 / maxIoThreads;

        if (workerUsage > 80)
        {
            _logger.LogWarning(
                "Thread pool worker usage high: {Usage}% ({Available}/{Max})",
                workerUsage, workerThreads, maxWorkerThreads);
        }

        context.Response.Headers.Add("X-ThreadPool-Workers",
            $"{maxWorkerThreads - workerThreads}/{maxWorkerThreads}");

        await _next(context);
    }
}
```

### Application Insights Detection

```kusto
// Detect thread pool exhaustion
performanceCounters
| where name == "Thread Count" or name == "Requests Queued"
| summarize AvgValue = avg(value) by name, bin(timestamp, 1m)
| render timechart

// Find slow requests with low CPU
requests
| where duration > 5000 // > 5 seconds
| join kind=inner (
    performanceCounters
    | where name == "% Processor Time"
    | summarize AvgCPU = avg(value) by bin(timestamp, 1m)
) on $left.timestamp == $right.timestamp
| where AvgCPU < 20 // Low CPU
| project timestamp, name, duration, AvgCPU
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Async I/O: Thread returns to pool during I/O wait                        │
│                                                                              │
│   Request 1: ████░░░░░░░░░░░░░░░░░░░░░░████████████████████████████        │
│              │Work│ Return thread      │ Continue when I/O complete         │
│                   │ (thread available) │                                    │
│                                                                              │
│   Request 2: ████░░░░░░░░████████████████████████████░░░░░░░░░░░░░░        │
│              │Work│ Return │ Continue    │ Work      │ Return              │
│                                                                              │
│   Request 3: ████████████████████░░░░░░░░░░░░████████████████████░░        │
│                                                                              │
│   Thread reused while waiting for I/O!                                      │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │   BENEFITS:                                                          │   │
│   │   • Handle 1000s of concurrent requests with ~100 threads           │   │
│   │   • Threads do real work, not waiting                               │   │
│   │   • Better resource utilization                                     │   │
│   │   • Consistent performance under load                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: Async/Await Throughout

```csharp
// SOLUTION: Async all the way down

public class OrderService
{
    private readonly AppDbContext _db;
    private readonly ICustomerApiClient _customerApi;

    public async Task<OrderDetails> GetOrderAsync(int orderId)
    {
        // Thread returns to pool during database call
        var order = await _db.Orders.FindAsync(orderId);

        // Thread returns to pool during HTTP call
        var customer = await _customerApi.GetCustomerAsync(order.CustomerId);

        return new OrderDetails { Order = order, Customer = customer };
    }
}

// Controller - also async
[HttpGet("{id}")]
public async Task<ActionResult<OrderDetails>> GetOrder(int id)
{
    var details = await _orderService.GetOrderAsync(id);
    return Ok(details);
}
```

### Solution 2: Async HTTP Client

```csharp
// SOLUTION: Async HTTP operations with IHttpClientFactory

public class WeatherService
{
    private readonly HttpClient _client;

    public WeatherService(IHttpClientFactory factory)
    {
        _client = factory.CreateClient("WeatherApi");
    }

    public async Task<Weather> GetWeatherAsync(string city)
    {
        // Async HTTP call
        var response = await _client.GetAsync($"/weather/{city}");
        response.EnsureSuccessStatusCode();

        // Async content read
        return await response.Content.ReadFromJsonAsync<Weather>();
    }
}
```

### Solution 3: Async File Operations

```csharp
// SOLUTION: Async file I/O

public class DocumentService
{
    public async Task<string> ReadDocumentAsync(string path)
    {
        // Async file read
        return await File.ReadAllTextAsync(path);
    }

    public async Task WriteDocumentAsync(string path, string content)
    {
        // Async file write
        await File.WriteAllTextAsync(path, content);
    }

    public async Task<byte[]> ReadBinaryAsync(string path)
    {
        await using var stream = new FileStream(
            path,
            FileMode.Open,
            FileAccess.Read,
            FileShare.Read,
            bufferSize: 4096,
            useAsync: true); // Important: enable async

        var buffer = new byte[stream.Length];
        await stream.ReadAsync(buffer);
        return buffer;
    }
}
```

### Solution 4: Async Database Operations

```csharp
// SOLUTION: Async Entity Framework operations

public class ProductRepository
{
    private readonly AppDbContext _db;

    public async Task<List<Product>> GetProductsAsync()
    {
        // ToListAsync() releases thread during query
        return await _db.Products.ToListAsync();
    }

    public async Task<Product?> GetProductAsync(int id)
    {
        // FindAsync() is async
        return await _db.Products.FindAsync(id);
    }

    public async Task UpdateProductAsync(Product product)
    {
        _db.Products.Update(product);
        // SaveChangesAsync() releases thread during commit
        await _db.SaveChangesAsync();
    }

    public async Task<int> CountProductsAsync(string category)
    {
        // CountAsync() is async
        return await _db.Products
            .Where(p => p.Category == category)
            .CountAsync();
    }
}
```

### Solution 5: Parallel Async Operations

```csharp
// SOLUTION: Run independent async operations in parallel

public class DashboardService
{
    public async Task<Dashboard> GetDashboardAsync(int userId)
    {
        // Start all independent operations
        var userTask = _userService.GetUserAsync(userId);
        var ordersTask = _orderService.GetRecentOrdersAsync(userId);
        var recommendationsTask = _recService.GetRecommendationsAsync(userId);
        var notificationsTask = _notificationService.GetUnreadAsync(userId);

        // Wait for all to complete
        await Task.WhenAll(userTask, ordersTask, recommendationsTask, notificationsTask);

        return new Dashboard
        {
            User = userTask.Result,
            RecentOrders = ordersTask.Result,
            Recommendations = recommendationsTask.Result,
            Notifications = notificationsTask.Result
        };
    }
}
```

### Azure-Specific Solutions

#### Azure SDK Async Operations

```csharp
// All Azure SDK operations are async by default

public class BlobService
{
    private readonly BlobContainerClient _container;

    public async Task<string> UploadAsync(Stream content, string fileName)
    {
        var blobClient = _container.GetBlobClient(fileName);

        // Async upload
        await blobClient.UploadAsync(content, overwrite: true);

        return blobClient.Uri.ToString();
    }

    public async Task<Stream> DownloadAsync(string fileName)
    {
        var blobClient = _container.GetBlobClient(fileName);

        // Async download
        var response = await blobClient.DownloadAsync();
        return response.Value.Content;
    }

    public async IAsyncEnumerable<BlobItem> ListBlobsAsync(string prefix)
    {
        // Async enumerable for large lists
        await foreach (var blob in _container.GetBlobsAsync(prefix: prefix))
        {
            yield return blob;
        }
    }
}
```

#### Service Bus Async Processing

```csharp
// Async message processing

public class MessageProcessor : BackgroundService
{
    private readonly ServiceBusProcessor _processor;

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _processor.ProcessMessageAsync += async args =>
        {
            // Async processing
            await ProcessMessageAsync(args.Message);
            await args.CompleteMessageAsync(args.Message);
        };

        _processor.ProcessErrorAsync += args =>
        {
            _logger.LogError(args.Exception, "Error processing message");
            return Task.CompletedTask;
        };

        await _processor.StartProcessingAsync(stoppingToken);

        // Keep running
        await Task.Delay(Timeout.Infinite, stoppingToken);
    }
}
```

## Common Async Pitfalls to Avoid

```csharp
// ❌ DON'T: Block on async
var result = GetDataAsync().Result;
var result = GetDataAsync().GetAwaiter().GetResult();
GetDataAsync().Wait();

// ✅ DO: Use async/await
var result = await GetDataAsync();

// ❌ DON'T: Async void (except event handlers)
public async void ProcessData() { }

// ✅ DO: Return Task
public async Task ProcessDataAsync() { }

// ❌ DON'T: Fire and forget without handling exceptions
_ = ProcessAsync(); // Exception lost!

// ✅ DO: Handle fire-and-forget properly
_ = ProcessAsync().ContinueWith(t =>
{
    if (t.IsFaulted) _logger.LogError(t.Exception, "Background task failed");
});

// ❌ DON'T: Use Task.Run for I/O
await Task.Run(() => _db.SaveChanges());

// ✅ DO: Use native async methods
await _db.SaveChangesAsync();

// ❌ DON'T: ConfigureAwait(false) in ASP.NET Core (usually unnecessary)
await _db.SaveChangesAsync().ConfigureAwait(false);

// ✅ DO: Just await (ASP.NET Core doesn't have SynchronizationContext)
await _db.SaveChangesAsync();
```

## Test Cases

### Unit Test: Verify Async Operations

```csharp
[Fact]
public async Task GetOrder_ShouldNotBlockThread()
{
    // Arrange
    var service = new OrderService(_mockDb.Object, _mockApi.Object);

    // Record thread before operation
    var threadBefore = Thread.CurrentThread.ManagedThreadId;

    // Act
    var order = await service.GetOrderAsync(1);

    // Assert - in async code, thread can change after await
    // This test ensures we're actually using async
    Assert.NotNull(order);
}
```

### Performance Test: Thread Pool Under Load

```csharp
[Fact]
public async Task HighLoad_ShouldNotExhaustThreadPool()
{
    // Arrange
    var client = _factory.CreateClient();
    var requests = Enumerable.Range(0, 1000)
        .Select(_ => client.GetAsync("/api/orders/1"));

    ThreadPool.GetMaxThreads(out int maxWorkers, out _);

    // Act
    var responses = await Task.WhenAll(requests);

    // Assert
    ThreadPool.GetAvailableThreads(out int availableWorkers, out _);
    var usedWorkers = maxWorkers - availableWorkers;

    Assert.True(usedWorkers < 100,
        $"Used {usedWorkers} threads for 1000 requests - should use fewer with async");
    Assert.All(responses, r => Assert.True(r.IsSuccessStatusCode));
}
```

### Integration Test: No Deadlocks

```csharp
[Fact]
public async Task AsyncOperations_ShouldNotDeadlock()
{
    // Arrange
    using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10));
    var service = _serviceProvider.GetRequiredService<OrderService>();

    // Act & Assert - should complete without deadlock
    var task = service.GetOrderAsync(1);

    await task.WaitAsync(cts.Token);
    // If we get here without timeout, no deadlock occurred
}
```

## Best Practices Checklist

- [ ] Use async/await throughout the call stack
- [ ] Never use .Result, .Wait(), or GetAwaiter().GetResult()
- [ ] Use async versions of all I/O methods
- [ ] Use IAsyncEnumerable for streaming data
- [ ] Avoid async void (except event handlers)
- [ ] Handle fire-and-forget tasks properly
- [ ] Use Task.WhenAll for parallel operations
- [ ] Monitor thread pool metrics
- [ ] Test under realistic concurrent load
- [ ] Use Azure SDK async methods

## AWS to Azure Migration Note

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| AWS SDK async | Azure SDK async (same pattern) |
| Lambda async handlers | Functions async (same pattern) |
| SQS async polling | Service Bus async processing |
| DynamoDB async | Cosmos DB async |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [Retry Storm](09-retry-storm.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
