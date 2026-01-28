# Reliability Patterns

## Circuit Breaker Pattern

### Problem

When a remote service is failing, continuing to call it wastes resources and can cause cascade failures across your system.

### Solution

```
CIRCUIT BREAKER STATES:
───────────────────────

┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│    ┌─────────┐     failures > threshold    ┌─────────┐                 │
│    │ CLOSED  │ ─────────────────────────▶ │  OPEN   │                  │
│    │(Normal) │                             │(Failing)│                  │
│    └─────────┘                             └─────────┘                  │
│         ▲                                       │                        │
│         │                                       │ timeout elapsed        │
│         │ success                               ▼                        │
│         │                                 ┌───────────┐                  │
│         └──────────────────────────────── │ HALF-OPEN │                 │
│                    success                │ (Testing) │                 │
│                                           └───────────┘                  │
│                                                 │                        │
│                                                 │ failure                │
│                                                 ▼                        │
│                                           Back to OPEN                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

STATES:
• CLOSED: Normal operation, requests pass through
• OPEN: Failures exceeded threshold, requests fail immediately
• HALF-OPEN: After timeout, allow one test request
```

### Azure Implementation

```csharp
// Circuit Breaker with Polly
public class ResilientHttpClient
{
    private readonly HttpClient _httpClient;
    private readonly AsyncCircuitBreakerPolicy<HttpResponseMessage> _circuitBreaker;
    private readonly ILogger _logger;

    public ResilientHttpClient(HttpClient httpClient, ILogger<ResilientHttpClient> logger)
    {
        _httpClient = httpClient;
        _logger = logger;

        _circuitBreaker = Policy<HttpResponseMessage>
            .Handle<HttpRequestException>()
            .OrResult(r => (int)r.StatusCode >= 500)
            .AdvancedCircuitBreakerAsync(
                failureThreshold: 0.5,           // 50% failure rate
                samplingDuration: TimeSpan.FromSeconds(30),
                minimumThroughput: 10,           // Min requests before tripping
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (result, duration) =>
                {
                    _logger.LogWarning("Circuit OPEN for {Duration}s. Reason: {Reason}",
                        duration.TotalSeconds,
                        result.Exception?.Message ?? result.Result.StatusCode.ToString());
                },
                onReset: () => _logger.LogInformation("Circuit CLOSED - normal operation"),
                onHalfOpen: () => _logger.LogInformation("Circuit HALF-OPEN - testing"));
    }

    public async Task<HttpResponseMessage> SendAsync(HttpRequestMessage request)
    {
        return await _circuitBreaker.ExecuteAsync(async () =>
        {
            return await _httpClient.SendAsync(request);
        });
    }
}
```

### When to Use

```
USE CIRCUIT BREAKER WHEN:
✅ Calling external services (APIs, databases)
✅ Service failures are temporary
✅ You want to fail fast instead of waiting
✅ You need to prevent cascade failures

DON'T USE WHEN:
❌ Failures are permanent (bad request)
❌ In-memory operations
❌ Very fast operations where overhead matters
```

---

## Retry Pattern

### Problem

Transient failures (network blips, temporary unavailability) can cause requests to fail, but succeed on retry.

### Solution

```
RETRY STRATEGIES:
─────────────────

IMMEDIATE RETRY:
Request → Fail → Retry → Retry → Retry → Give up

FIXED DELAY:
Request → Fail → Wait 1s → Retry → Wait 1s → Retry

EXPONENTIAL BACKOFF (Recommended):
Request → Fail → Wait 1s → Retry → Wait 2s → Retry → Wait 4s → Retry

EXPONENTIAL BACKOFF WITH JITTER (Best):
Request → Fail → Wait 1s ± random → Retry → Wait 2s ± random → Retry

Why jitter? Prevents "thundering herd" when many clients retry simultaneously
```

### Azure Implementation

```csharp
// Comprehensive retry policy
public static class RetryPolicies
{
    public static AsyncRetryPolicy<HttpResponseMessage> GetHttpRetryPolicy(ILogger logger)
    {
        return Policy<HttpResponseMessage>
            .Handle<HttpRequestException>()
            .OrResult(r => IsTransientError(r))
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: (retryAttempt, response, context) =>
                {
                    // Respect Retry-After header if present
                    if (response.Result?.Headers.RetryAfter?.Delta != null)
                    {
                        return response.Result.Headers.RetryAfter.Delta.Value;
                    }

                    // Exponential backoff with jitter
                    var delay = TimeSpan.FromSeconds(Math.Pow(2, retryAttempt));
                    var jitter = TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000));
                    return delay + jitter;
                },
                onRetryAsync: async (outcome, timespan, retryAttempt, context) =>
                {
                    logger.LogWarning(
                        "Retry {Attempt} after {Delay}ms. Reason: {Reason}",
                        retryAttempt,
                        timespan.TotalMilliseconds,
                        outcome.Exception?.Message ?? outcome.Result.StatusCode.ToString());
                    await Task.CompletedTask;
                });
    }

    private static bool IsTransientError(HttpResponseMessage response)
    {
        return response.StatusCode == HttpStatusCode.RequestTimeout ||   // 408
               response.StatusCode == HttpStatusCode.TooManyRequests ||  // 429
               (int)response.StatusCode >= 500;                          // 5xx
    }
}

// Azure SDK built-in retries
var options = new ServiceBusClientOptions
{
    RetryOptions = new ServiceBusRetryOptions
    {
        Mode = ServiceBusRetryMode.Exponential,
        MaxRetries = 3,
        Delay = TimeSpan.FromSeconds(1),
        MaxDelay = TimeSpan.FromSeconds(30)
    }
};
```

### When to Use

```
RETRY THESE ERRORS:
✅ 408 Request Timeout
✅ 429 Too Many Requests
✅ 500 Internal Server Error
✅ 502 Bad Gateway
✅ 503 Service Unavailable
✅ 504 Gateway Timeout
✅ Network timeouts
✅ Connection refused (temporary)

DON'T RETRY THESE:
❌ 400 Bad Request
❌ 401 Unauthorized
❌ 403 Forbidden
❌ 404 Not Found
❌ 409 Conflict
❌ 422 Unprocessable Entity
```

---

## Bulkhead Pattern

### Problem

A failure in one component consumes all shared resources (threads, connections), causing the entire system to fail.

### Solution

```
BULKHEAD ISOLATION:
───────────────────

WITHOUT BULKHEAD:
┌────────────────────────────────────────────────────────────────────────┐
│                        SHARED THREAD POOL (100)                        │
│                                                                        │
│  Service A calls: ████████████████████████████████████ (90 threads)   │
│  Service B calls: █ (10 threads, waiting...)                          │
│  Service C calls: (blocked, no threads available)                     │
│                                                                        │
│  If Service A is slow, it consumes all threads → total outage         │
└────────────────────────────────────────────────────────────────────────┘

WITH BULKHEAD:
┌────────────────────────────────────────────────────────────────────────┐
│  Service A Pool (40)   Service B Pool (30)   Service C Pool (30)      │
│  ██████████████████    ████████████          ████████████             │
│                                                                        │
│  If Service A is slow, only its pool is affected                      │
│  Services B and C continue operating normally                          │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Bulkhead with Polly
public class BulkheadedServices
{
    private readonly AsyncBulkheadPolicy _criticalServiceBulkhead;
    private readonly AsyncBulkheadPolicy _nonCriticalServiceBulkhead;

    public BulkheadedServices()
    {
        // Critical service: Allow 20 concurrent, 10 queued
        _criticalServiceBulkhead = Policy.BulkheadAsync(
            maxParallelization: 20,
            maxQueuingActions: 10,
            onBulkheadRejectedAsync: async context =>
            {
                await Task.CompletedTask;
                throw new BulkheadRejectedException("Critical service at capacity");
            });

        // Non-critical service: Allow 10 concurrent, 5 queued
        _nonCriticalServiceBulkhead = Policy.BulkheadAsync(
            maxParallelization: 10,
            maxQueuingActions: 5);
    }

    public async Task<PaymentResult> ProcessPaymentAsync(Payment payment)
    {
        return await _criticalServiceBulkhead.ExecuteAsync(async () =>
        {
            return await _paymentService.ProcessAsync(payment);
        });
    }

    public async Task<ReportResult> GenerateReportAsync(ReportRequest request)
    {
        return await _nonCriticalServiceBulkhead.ExecuteAsync(async () =>
        {
            return await _reportService.GenerateAsync(request);
        });
    }
}
```

### When to Use

```
USE BULKHEAD WHEN:
✅ Multiple downstream services with different SLAs
✅ Some services are more critical than others
✅ Want to prevent cascade failures
✅ Resource-intensive operations could starve others

IMPLEMENTATION OPTIONS:
• Thread pool isolation (Polly Bulkhead)
• Process isolation (separate containers/services)
• Tenant isolation (per-customer resources)
• Connection pool isolation (separate DB connections)
```

---

## Health Endpoint Monitoring Pattern

### Problem

You need to verify that applications and services are functioning correctly and available.

### Solution

```
HEALTH CHECK TYPES:
───────────────────

LIVENESS: Is the process alive?
┌────────────────────────────────────────────────────────────────────────┐
│ Check: Process is running, not deadlocked                              │
│ Response: 200 OK (healthy) or 503 (unhealthy)                         │
│ Action on failure: Restart the container/process                      │
│ Example: /health/live                                                  │
└────────────────────────────────────────────────────────────────────────┘

READINESS: Can it handle traffic?
┌────────────────────────────────────────────────────────────────────────┐
│ Check: Dependencies connected, warmed up, ready                        │
│ Response: 200 OK (ready) or 503 (not ready)                           │
│ Action on failure: Remove from load balancer                          │
│ Example: /health/ready                                                 │
└────────────────────────────────────────────────────────────────────────┘

DEEP HEALTH: Are dependencies healthy?
┌────────────────────────────────────────────────────────────────────────┐
│ Check: Database connected, cache reachable, APIs responding           │
│ Response: Detailed status of each dependency                          │
│ Use for: Diagnostics, monitoring dashboards                           │
│ Example: /health/deep                                                  │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// ASP.NET Core Health Checks
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddHealthChecks()
            // Database health
            .AddSqlServer(
                connectionString: Configuration.GetConnectionString("Database"),
                name: "sqlserver",
                tags: new[] { "ready", "db" })
            // Redis health
            .AddRedis(
                redisConnectionString: Configuration.GetConnectionString("Redis"),
                name: "redis",
                tags: new[] { "ready", "cache" })
            // External API health
            .AddUrlGroup(
                new Uri("https://api.external.com/health"),
                name: "external-api",
                tags: new[] { "ready", "external" })
            // Custom health check
            .AddCheck<CustomHealthCheck>("custom");
    }

    public void Configure(IApplicationBuilder app)
    {
        // Liveness - just checks if app is running
        app.MapHealthChecks("/health/live", new HealthCheckOptions
        {
            Predicate = _ => false  // No dependency checks
        });

        // Readiness - checks if app can handle traffic
        app.MapHealthChecks("/health/ready", new HealthCheckOptions
        {
            Predicate = check => check.Tags.Contains("ready")
        });

        // Deep health - detailed status
        app.MapHealthChecks("/health/deep", new HealthCheckOptions
        {
            ResponseWriter = WriteDetailedResponse
        });
    }

    private static async Task WriteDetailedResponse(
        HttpContext context, HealthReport report)
    {
        context.Response.ContentType = "application/json";

        var response = new
        {
            status = report.Status.ToString(),
            checks = report.Entries.Select(e => new
            {
                name = e.Key,
                status = e.Value.Status.ToString(),
                duration = e.Value.Duration.TotalMilliseconds,
                error = e.Value.Exception?.Message
            }),
            totalDuration = report.TotalDuration.TotalMilliseconds
        };

        await context.Response.WriteAsJsonAsync(response);
    }
}

// Custom health check
public class CustomHealthCheck : IHealthCheck
{
    private readonly IExternalService _service;

    public CustomHealthCheck(IExternalService service)
    {
        _service = service;
    }

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            var isHealthy = await _service.PingAsync(cancellationToken);

            return isHealthy
                ? HealthCheckResult.Healthy("Service is responding")
                : HealthCheckResult.Degraded("Service is slow");
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy("Service unreachable", ex);
        }
    }
}
```

### Azure Integration

```bicep
// App Service health check
resource appService 'Microsoft.Web/sites@2022-09-01' = {
  name: 'myapp'
  properties: {
    siteConfig: {
      healthCheckPath: '/health/ready'
    }
  }
}

// AKS health probes (in deployment.yaml)
// livenessProbe:
//   httpGet:
//     path: /health/live
//     port: 8080
//   initialDelaySeconds: 10
//   periodSeconds: 10
//
// readinessProbe:
//   httpGet:
//     path: /health/ready
//     port: 8080
//   initialDelaySeconds: 5
//   periodSeconds: 5
```

---

## Compensating Transaction Pattern

### Problem

In distributed systems, you can't use traditional database transactions. If a multi-step operation fails partway through, you need to undo the completed steps.

### Solution

```
COMPENSATING TRANSACTION FLOW:
──────────────────────────────

HAPPY PATH:
┌────────────────────────────────────────────────────────────────────────┐
│ Step 1: Reserve Inventory  ✓                                          │
│ Step 2: Charge Payment     ✓                                          │
│ Step 3: Create Shipment    ✓                                          │
│ → Success! Order complete                                             │
└────────────────────────────────────────────────────────────────────────┘

FAILURE WITH COMPENSATION:
┌────────────────────────────────────────────────────────────────────────┐
│ Step 1: Reserve Inventory  ✓                                          │
│ Step 2: Charge Payment     ✓                                          │
│ Step 3: Create Shipment    ✗ (Failed - no carrier available)         │
│                                                                        │
│ COMPENSATE:                                                            │
│ Step 2 Undo: Refund Payment ✓                                        │
│ Step 1 Undo: Release Inventory ✓                                     │
│ → Rolled back to original state                                       │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation (Durable Functions)

```csharp
[FunctionName("OrderSagaOrchestrator")]
public static async Task<OrderResult> RunOrderSaga(
    [OrchestrationTrigger] IDurableOrchestrationContext context,
    ILogger log)
{
    var order = context.GetInput<OrderRequest>();
    var compensations = new Stack<Func<Task>>();

    try
    {
        // Step 1: Reserve Inventory
        var inventoryResult = await context.CallActivityAsync<InventoryResult>(
            "ReserveInventory", order);

        if (!inventoryResult.Success)
            return OrderResult.Failed("No inventory available");

        // Add compensation for this step
        compensations.Push(() => context.CallActivityAsync(
            "ReleaseInventory", inventoryResult.ReservationId));

        // Step 2: Process Payment
        var paymentResult = await context.CallActivityAsync<PaymentResult>(
            "ProcessPayment", new PaymentRequest
            {
                OrderId = order.Id,
                Amount = order.Total,
                CustomerId = order.CustomerId
            });

        if (!paymentResult.Success)
        {
            await ExecuteCompensations(compensations);
            return OrderResult.Failed("Payment declined");
        }

        compensations.Push(() => context.CallActivityAsync(
            "RefundPayment", paymentResult.TransactionId));

        // Step 3: Create Shipment
        var shipmentResult = await context.CallActivityAsync<ShipmentResult>(
            "CreateShipment", new ShipmentRequest
            {
                OrderId = order.Id,
                Address = order.ShippingAddress,
                Items = order.Items
            });

        if (!shipmentResult.Success)
        {
            await ExecuteCompensations(compensations);
            return OrderResult.Failed("Shipment creation failed");
        }

        // All steps succeeded
        return OrderResult.Succeeded(shipmentResult.TrackingNumber);
    }
    catch (Exception ex)
    {
        log.LogError(ex, "Order saga failed unexpectedly");
        await ExecuteCompensations(compensations);
        throw;
    }
}

private static async Task ExecuteCompensations(Stack<Func<Task>> compensations)
{
    // Execute compensations in reverse order
    while (compensations.Count > 0)
    {
        var compensation = compensations.Pop();
        try
        {
            await compensation();
        }
        catch (Exception ex)
        {
            // Log but continue with other compensations
            // May need manual intervention for failed compensations
        }
    }
}
```

---

## Combining Patterns

### Resilient HTTP Client

```csharp
// Combining Retry + Circuit Breaker + Bulkhead
public static class HttpClientResilience
{
    public static IHttpClientBuilder AddResiliencePolicies(
        this IHttpClientBuilder builder)
    {
        return builder
            .AddPolicyHandler(GetRetryPolicy())
            .AddPolicyHandler(GetCircuitBreakerPolicy())
            .AddPolicyHandler(GetBulkheadPolicy());
    }

    private static AsyncRetryPolicy<HttpResponseMessage> GetRetryPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .WaitAndRetryAsync(3, retry =>
                TimeSpan.FromSeconds(Math.Pow(2, retry)));
    }

    private static AsyncCircuitBreakerPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(5, TimeSpan.FromSeconds(30));
    }

    private static AsyncBulkheadPolicy<HttpResponseMessage> GetBulkheadPolicy()
    {
        return Policy.BulkheadAsync<HttpResponseMessage>(10, 5);
    }
}

// Registration
services.AddHttpClient<IPaymentService, PaymentService>()
    .AddResiliencePolicies();
```

---

*Next: [Messaging Patterns](02-messaging-patterns.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
