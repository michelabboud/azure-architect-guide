# Anti-Pattern: Improper Instantiation

## Overview

The Improper Instantiation anti-pattern occurs when applications repeatedly create and destroy objects that are designed to be shared or reused. This is particularly problematic for objects that manage expensive resources like network connections, database connections, or HTTP clients.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   IMPROPER INSTANTIATION ANTI-PATTERN                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Request 1                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  new HttpClient()  →  Open Socket  →  HTTP Request  →  Dispose()   │   │
│   │  new HttpClient()  →  Open Socket  →  HTTP Request  →  Dispose()   │   │
│   │  new HttpClient()  →  Open Socket  →  HTTP Request  →  Dispose()   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Request 2                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  new HttpClient()  →  Open Socket  →  HTTP Request  →  Dispose()   │   │
│   │  new HttpClient()  →  Open Socket  →  HTTP Request  →  Dispose()   │   │
│   │  new HttpClient()  →  Open Socket  →  HTTP Request  →  Dispose()   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Operating System                                                          │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  Sockets in TIME_WAIT state:                                        │   │
│   │  ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐ ┌────┐   │   │
│   │  │WAIT│ │WAIT│ │WAIT│ │WAIT│ │WAIT│ │WAIT│ │WAIT│ │WAIT│ │WAIT│   │   │
│   │  └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘ └────┘   │   │
│   │                                                                     │   │
│   │  ❌ Socket exhaustion: No more ports available!                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Problems:                                                                  │
│   • Socket exhaustion (ports stuck in TIME_WAIT)                            │
│   • Connection pool exhaustion                                              │
│   • DNS caching issues (stale DNS entries)                                  │
│   • Memory pressure from object creation                                    │
│   • GC overhead from frequent allocations                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Creating HttpClient Per Request

```csharp
// ANTI-PATTERN: New HttpClient for each request
public class OrderService
{
    public async Task<Order> GetOrderAsync(int orderId)
    {
        // Creates new socket connection each time
        using var client = new HttpClient();
        client.BaseAddress = new Uri("https://api.example.com");

        var response = await client.GetAsync($"/orders/{orderId}");
        return await response.Content.ReadFromJsonAsync<Order>();
    }
    // Client disposed, but socket stays in TIME_WAIT for ~240 seconds
}
```

### 2. Creating DbContext Per Operation

```csharp
// ANTI-PATTERN: New DbContext for each database operation
public class ProductRepository
{
    public async Task<Product> GetProductAsync(int id)
    {
        using var context = new AppDbContext();
        return await context.Products.FindAsync(id);
    }

    public async Task<List<Product>> GetAllProductsAsync()
    {
        using var context = new AppDbContext(); // Another instance
        return await context.Products.ToListAsync();
    }

    public async Task UpdateProductAsync(Product product)
    {
        using var context = new AppDbContext(); // Yet another
        context.Products.Update(product);
        await context.SaveChangesAsync();
    }
}
```

### 3. Creating Service Clients Repeatedly

```csharp
// ANTI-PATTERN: New Azure clients for each operation
public class BlobService
{
    private readonly string _connectionString;

    public async Task<Stream> GetBlobAsync(string container, string blob)
    {
        // Creates new connection for each call
        var client = new BlobServiceClient(_connectionString);
        var containerClient = client.GetBlobContainerClient(container);
        var blobClient = containerClient.GetBlobClient(blob);

        return await blobClient.OpenReadAsync();
    }
}
```

### 4. Expensive Object Creation in Loops

```csharp
// ANTI-PATTERN: Creating expensive objects in loops
public async Task ProcessOrdersAsync(List<Order> orders)
{
    foreach (var order in orders)
    {
        // JsonSerializer options created for each order
        var options = new JsonSerializerOptions
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
            WriteIndented = true
        };

        var json = JsonSerializer.Serialize(order, options);
        await SendToQueueAsync(json);
    }
}
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| Socket exhaustion | `SocketException: Address already in use` |
| Connection failures | `HttpRequestException` under load |
| Memory spikes | GC frequently collecting Gen2 |
| Port exhaustion | `netstat` shows many TIME_WAIT connections |
| Slow performance | Latency increases under sustained load |

### Detection Commands

```bash
# Check for socket exhaustion on Linux
netstat -an | grep TIME_WAIT | wc -l

# Check for high port usage
ss -s

# Check established connections
netstat -ant | grep ESTABLISHED | wc -l
```

### Application Insights Detection

```kusto
// High connection exceptions
exceptions
| where timestamp > ago(1h)
| where type contains "Socket" or type contains "Http"
| summarize Count = count() by type, outerMessage
| order by Count desc

// Memory pressure indicators
performanceCounters
| where timestamp > ago(1h)
| where category == "Memory"
| summarize AvgValue = avg(value) by name
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Application (using IHttpClientFactory)                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                                                                      │   │
│   │  Request 1 ──► Factory ──► Handler Pool ──► Shared Connections      │   │
│   │  Request 2 ──► Factory ──► Handler Pool ──► Shared Connections      │   │
│   │  Request 3 ──► Factory ──► Handler Pool ──► Shared Connections      │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Connection Pool                                                           │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │   │
│   │  │   Connection 1   │  │   Connection 2   │  │   Connection 3   │   │   │
│   │  │   (Reused)       │  │   (Reused)       │  │   (Reused)       │   │   │
│   │  └──────────────────┘  └──────────────────┘  └──────────────────┘   │   │
│   │                                                                      │   │
│   │  ✅ DNS refreshed periodically (2 minutes default)                  │   │
│   │  ✅ Connections reused across requests                              │   │
│   │  ✅ Handler lifecycle managed                                       │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: IHttpClientFactory

```csharp
// SOLUTION: Use IHttpClientFactory

// Program.cs - Register named client
builder.Services.AddHttpClient("OrdersApi", client =>
{
    client.BaseAddress = new Uri("https://api.example.com");
    client.DefaultRequestHeaders.Add("Accept", "application/json");
})
.AddPolicyHandler(GetRetryPolicy())
.AddPolicyHandler(GetCircuitBreakerPolicy());

// Service - Inject factory
public class OrderService
{
    private readonly IHttpClientFactory _clientFactory;

    public OrderService(IHttpClientFactory clientFactory)
    {
        _clientFactory = clientFactory;
    }

    public async Task<Order> GetOrderAsync(int orderId)
    {
        // Factory manages client lifecycle
        var client = _clientFactory.CreateClient("OrdersApi");
        var response = await client.GetAsync($"/orders/{orderId}");
        return await response.Content.ReadFromJsonAsync<Order>();
    }
}
```

### Solution 2: Typed HttpClient

```csharp
// SOLUTION: Typed client with DI

// Define typed client
public class OrdersApiClient
{
    private readonly HttpClient _client;

    public OrdersApiClient(HttpClient client)
    {
        _client = client;
        _client.BaseAddress = new Uri("https://api.example.com");
    }

    public async Task<Order> GetOrderAsync(int orderId)
    {
        return await _client.GetFromJsonAsync<Order>($"/orders/{orderId}");
    }
}

// Register
builder.Services.AddHttpClient<OrdersApiClient>()
    .AddPolicyHandler(GetRetryPolicy());

// Use - simply inject the typed client
public class OrderController
{
    private readonly OrdersApiClient _client;

    public OrderController(OrdersApiClient client)
    {
        _client = client;
    }
}
```

### Solution 3: DbContext with DI

```csharp
// SOLUTION: DbContext managed by DI container

// Program.cs
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(connectionString));

// Repository with scoped DbContext
public class ProductRepository
{
    private readonly AppDbContext _context;

    public ProductRepository(AppDbContext context)
    {
        _context = context; // Scoped lifetime, shared within request
    }

    public async Task<Product> GetProductAsync(int id)
        => await _context.Products.FindAsync(id);

    public async Task<List<Product>> GetAllProductsAsync()
        => await _context.Products.ToListAsync();

    public async Task UpdateProductAsync(Product product)
    {
        _context.Products.Update(product);
        await _context.SaveChangesAsync();
    }
}
```

### Solution 4: Singleton Azure Clients

```csharp
// SOLUTION: Register Azure clients as singletons

// Program.cs
builder.Services.AddSingleton(new BlobServiceClient(connectionString));
builder.Services.AddSingleton<IBlobService, BlobService>();

// Service with injected client
public class BlobService : IBlobService
{
    private readonly BlobServiceClient _client;

    public BlobService(BlobServiceClient client)
    {
        _client = client;
    }

    public async Task<Stream> GetBlobAsync(string container, string blob)
    {
        var containerClient = _client.GetBlobContainerClient(container);
        var blobClient = containerClient.GetBlobClient(blob);
        return await blobClient.OpenReadAsync();
    }
}

// Or use Azure.Extensions.AspNetCore.Configuration.Secrets
builder.Services.AddAzureClients(builder =>
{
    builder.AddBlobServiceClient(connectionString);
    builder.AddServiceBusClient(serviceBusConnection);
});
```

### Solution 5: Reuse Expensive Objects

```csharp
// SOLUTION: Static or cached expensive objects

public class OrderProcessor
{
    // Static - created once, shared across all instances
    private static readonly JsonSerializerOptions _jsonOptions = new()
    {
        PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        WriteIndented = true
    };

    public async Task ProcessOrdersAsync(List<Order> orders)
    {
        foreach (var order in orders)
        {
            // Reuse options instance
            var json = JsonSerializer.Serialize(order, _jsonOptions);
            await SendToQueueAsync(json);
        }
    }
}
```

### Azure-Specific Solutions

#### Azure SDK Best Practices

```csharp
// Register all Azure clients properly
builder.Services.AddAzureClients(clientBuilder =>
{
    // Blob Storage
    clientBuilder.AddBlobServiceClient(
        builder.Configuration.GetSection("Storage"));

    // Service Bus
    clientBuilder.AddServiceBusClient(
        builder.Configuration["ServiceBus:ConnectionString"]);

    // Key Vault
    clientBuilder.AddSecretClient(
        new Uri(builder.Configuration["KeyVault:Uri"]));

    // Configure defaults
    clientBuilder.ConfigureDefaults(
        builder.Configuration.GetSection("AzureDefaults"));

    // Use managed identity
    clientBuilder.UseCredential(new DefaultAzureCredential());
});
```

#### Connection Pooling for SQL

```csharp
// Configure SQL connection pooling
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connectionString, sqlOptions =>
    {
        sqlOptions.EnableRetryOnFailure(
            maxRetryCount: 5,
            maxRetryDelay: TimeSpan.FromSeconds(30),
            errorNumbersToAdd: null);

        // Connection pool is managed by SqlClient
        // Default: Min=0, Max=100 connections
    });
});

// Connection string pooling settings
// "Server=...;Min Pool Size=5;Max Pool Size=100;Connection Timeout=30;"
```

## Test Cases

### Unit Test: Verify Singleton Lifecycle

```csharp
[Fact]
public void HttpClientFactory_ShouldReuseSameHandler()
{
    // Arrange
    var services = new ServiceCollection();
    services.AddHttpClient("test");
    var provider = services.BuildServiceProvider();
    var factory = provider.GetRequiredService<IHttpClientFactory>();

    // Act
    var client1 = factory.CreateClient("test");
    var client2 = factory.CreateClient("test");

    // Assert - clients are different but share handlers
    Assert.NotSame(client1, client2);
    // Handler pooling is internal, but no socket exhaustion under load
}
```

### Integration Test: No Socket Exhaustion

```csharp
[Fact]
public async Task HighLoad_ShouldNotExhaustSockets()
{
    // Arrange
    var factory = _serviceProvider.GetRequiredService<IHttpClientFactory>();
    var tasks = new List<Task>();

    // Act - simulate high load
    for (int i = 0; i < 1000; i++)
    {
        tasks.Add(Task.Run(async () =>
        {
            var client = factory.CreateClient("test");
            await client.GetAsync("https://httpbin.org/get");
        }));
    }

    // Assert - should complete without socket exceptions
    await Task.WhenAll(tasks);
}
```

### Memory Test: No Memory Leaks

```csharp
[Fact]
public async Task Service_ShouldNotLeakMemory()
{
    var initialMemory = GC.GetTotalMemory(true);

    // Act - many operations
    for (int i = 0; i < 10000; i++)
    {
        await _service.ProcessAsync();
    }

    GC.Collect();
    GC.WaitForPendingFinalizers();
    var finalMemory = GC.GetTotalMemory(true);

    // Assert - memory should not grow significantly
    var growth = finalMemory - initialMemory;
    Assert.True(growth < 50_000_000, $"Memory grew by {growth / 1_000_000}MB");
}
```

## Best Practices Checklist

- [ ] Use IHttpClientFactory for HTTP clients
- [ ] Register Azure SDK clients as singletons
- [ ] Use DI for DbContext (scoped lifetime)
- [ ] Share JsonSerializerOptions instances
- [ ] Implement connection pooling
- [ ] Configure appropriate pool sizes
- [ ] Monitor for socket/connection exhaustion
- [ ] Use typed clients for cleaner code
- [ ] Add resilience policies to HTTP clients
- [ ] Review object lifetimes in DI container

## AWS to Azure Migration Note

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| AWS SDK client reuse | Azure SDK client registration |
| Connection pooling | Same concepts apply |
| Lambda cold starts | Azure Functions instance reuse |
| RDS connection management | SQL connection pooling |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [Extraneous Fetching](04-extraneous-fetching.md)*

*Next: [Monolithic Persistence](06-monolithic-persistence.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
