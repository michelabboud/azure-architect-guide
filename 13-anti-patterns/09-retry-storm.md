# Anti-Pattern: Retry Storm

## Overview

The Retry Storm anti-pattern occurs when multiple clients simultaneously retry failed requests, creating a surge of traffic that overwhelms an already struggling service. Instead of allowing the service to recover, the retry storm makes the situation worse, potentially causing complete system failure.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RETRY STORM ANTI-PATTERN                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Timeline of Disaster                                                      │
│                                                                              │
│   T=0: Service becomes slow (maybe due to load spike)                       │
│                                                                              │
│   T=1: Requests start timing out                                            │
│        ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐              │
│        │Client 1│ │Client 2│ │Client 3│ │Client 4│ │Client 5│              │
│        │ × Fail │ │ × Fail │ │ × Fail │ │ × Fail │ │ × Fail │              │
│        └────────┘ └────────┘ └────────┘ └────────┘ └────────┘              │
│                                                                              │
│   T=2: All clients retry IMMEDIATELY                                        │
│        ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐              │
│        │ Retry! │ │ Retry! │ │ Retry! │ │ Retry! │ │ Retry! │              │
│        └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘              │
│            │          │          │          │          │                    │
│            └──────────┴─────┬────┴──────────┴──────────┘                    │
│                             │                                                │
│                             ▼                                                │
│        ┌────────────────────────────────────────────────────────────┐       │
│        │                      SERVICE                               │       │
│        │                                                            │       │
│        │   Was handling 100 req/s, now receiving 500 req/s         │       │
│        │   (Original load + All retries hitting at same time)       │       │
│        │                                                            │       │
│        │   ❌ SERVICE CRASHES                                       │       │
│        │                                                            │       │
│        └────────────────────────────────────────────────────────────┘       │
│                                                                              │
│   T=3: More failures → More retries → Service can't recover                │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │   CASCADING FAILURE                                                  │   │
│   │                                                                      │   │
│   │   Service A fails → Service B retries → Service B fails             │   │
│   │       ↓                                                              │   │
│   │   Service C retries → Service C fails → ...                         │   │
│   │                                                                      │   │
│   │   Entire system down!                                                │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Immediate Retries Without Backoff

```csharp
// ANTI-PATTERN: Immediate retry with no backoff
public async Task<Response> CallServiceAsync()
{
    int retries = 3;

    while (retries > 0)
    {
        try
        {
            return await _httpClient.GetAsync("/api/data");
        }
        catch (HttpRequestException)
        {
            retries--;
            // Retries immediately! No delay!
        }
    }

    throw new Exception("Service unavailable");
}
```

### 2. Fixed Interval Retries (Synchronized)

```csharp
// ANTI-PATTERN: Fixed interval causes synchronized retries
public async Task<Response> CallServiceAsync()
{
    int retries = 3;

    while (retries > 0)
    {
        try
        {
            return await _httpClient.GetAsync("/api/data");
        }
        catch (HttpRequestException)
        {
            retries--;
            await Task.Delay(1000); // All clients retry at exactly 1 second!
        }
    }

    throw new Exception("Service unavailable");
}
// If 100 clients fail at T=0, all 100 retry at T=1!
```

### 3. Unlimited Retries

```csharp
// ANTI-PATTERN: No limit on retries
public async Task<Response> CallServiceAsync()
{
    while (true) // Never gives up!
    {
        try
        {
            return await _httpClient.GetAsync("/api/data");
        }
        catch (HttpRequestException)
        {
            await Task.Delay(100); // Keeps hammering the service
        }
    }
}
```

### 4. No Circuit Breaker

```csharp
// ANTI-PATTERN: Keeps trying even when service is clearly down
public async Task ProcessBatchAsync(List<Item> items)
{
    foreach (var item in items)
    {
        // If service is down, every item generates retries!
        await CallServiceWithRetriesAsync(item);
    }
}
// 1000 items × 3 retries = 3000 requests to a dead service
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| Synchronized spikes | Traffic spikes at regular intervals |
| Cascading failures | Multiple services failing together |
| Recovery failure | Service can't stabilize after issues |
| Exponential traffic | Traffic increases during outages |
| Resource exhaustion | Connection pools depleted |

### Detection Queries

```kusto
// Application Insights: Detect retry storms
requests
| where timestamp > ago(1h)
| summarize RequestCount = count() by bin(timestamp, 1s), name
| where RequestCount > 100
| order by timestamp asc

// Visualize synchronized retries
requests
| where timestamp > ago(10m)
| where success == false
| summarize FailedCount = count() by bin(timestamp, 100ms)
| render timechart
```

```kusto
// Detect circuit breaker trips
customEvents
| where name == "CircuitBreakerStateChange"
| project timestamp, State = tostring(customDimensions.State), Service = tostring(customDimensions.Service)
| order by timestamp desc
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Exponential Backoff with Jitter + Circuit Breaker                         │
│                                                                              │
│   T=0: Service becomes slow                                                 │
│                                                                              │
│   T=1: Requests timeout, clients apply backoff + jitter                     │
│        ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐              │
│        │Client 1│ │Client 2│ │Client 3│ │Client 4│ │Client 5│              │
│        │ × Fail │ │ × Fail │ │ × Fail │ │ × Fail │ │ × Fail │              │
│        │Wait 2.1s│ │Wait 1.8s│ │Wait 2.3s│ │Wait 1.9s│ │Wait 2.2s│         │
│        └────────┘ └────────┘ └────────┘ └────────┘ └────────┘              │
│                                                                              │
│   T=2-4: Retries spread out over time (jitter)                              │
│                                                                              │
│        │    │      │  │     │    │       │   │      │                       │
│        ▼    ▼      ▼  ▼     ▼    ▼       ▼   ▼      ▼                       │
│        ┌────────────────────────────────────────────────────────────┐       │
│        │                      SERVICE                               │       │
│        │   Receives spread-out retry traffic                        │       │
│        │   Has time to recover between requests                     │       │
│        │   ✅ SERVICE RECOVERS                                      │       │
│        └────────────────────────────────────────────────────────────┘       │
│                                                                              │
│   If failures continue: Circuit breaker OPENS                               │
│                                                                              │
│        ┌────────┐ ┌────────┐ ┌────────┐                                    │
│        │Client 1│ │Client 2│ │Client 3│                                    │
│        │Circuit │ │Circuit │ │Circuit │                                    │
│        │ OPEN   │ │ OPEN   │ │ OPEN   │                                    │
│        │        │ │        │ │        │                                    │
│        │Fail    │ │Fail    │ │Fail    │                                    │
│        │ Fast!  │ │ Fast!  │ │ Fast!  │                                    │
│        └────────┘ └────────┘ └────────┘                                    │
│              │          │          │                                        │
│              X          X          X   ← No requests sent!                 │
│                                                                              │
│        Service has time to recover without any traffic                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: Exponential Backoff with Jitter (Polly)

```csharp
// SOLUTION: Exponential backoff with jitter using Polly

builder.Services.AddHttpClient("ExternalApi")
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());

static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
{
    // Decorrelated jitter strategy (recommended)
    var delay = Backoff.DecorrelatedJitterBackoffV2(
        medianFirstRetryDelay: TimeSpan.FromSeconds(1),
        retryCount: 5);

    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .OrResult(msg => msg.StatusCode == System.Net.HttpStatusCode.TooManyRequests)
        .WaitAndRetryAsync(delay, onRetry: (outcome, timespan, retryAttempt, context) =>
        {
            Log.Warning("Retry {RetryAttempt} after {Delay}ms due to {StatusCode}",
                retryAttempt, timespan.TotalMilliseconds, outcome.Result?.StatusCode);
        });
}

static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
{
    return HttpPolicyExtensions
        .HandleTransientHttpError()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak: (outcome, breakDelay) =>
            {
                Log.Warning("Circuit breaker opened for {BreakDelay}s", breakDelay.TotalSeconds);
            },
            onReset: () =>
            {
                Log.Information("Circuit breaker reset");
            },
            onHalfOpen: () =>
            {
                Log.Information("Circuit breaker half-open, testing...");
            });
}
```

### Solution 2: Manual Exponential Backoff with Jitter

```csharp
// SOLUTION: Manual implementation with decorrelated jitter
public class RetryWithJitter
{
    private static readonly Random _random = new();

    public async Task<T> ExecuteAsync<T>(
        Func<Task<T>> operation,
        int maxRetries = 5,
        double baseDelayMs = 1000)
    {
        int retryCount = 0;
        double delay = baseDelayMs;

        while (true)
        {
            try
            {
                return await operation();
            }
            catch (HttpRequestException ex) when (retryCount < maxRetries)
            {
                retryCount++;

                // Decorrelated jitter: delay = random(baseDelay, previousDelay * 3)
                delay = Math.Min(
                    baseDelayMs + _random.NextDouble() * (delay * 3 - baseDelayMs),
                    60000); // Cap at 60 seconds

                Log.Warning("Attempt {Attempt} failed. Retrying in {Delay}ms",
                    retryCount, delay);

                await Task.Delay(TimeSpan.FromMilliseconds(delay));
            }
        }
    }
}
```

### Solution 3: Circuit Breaker Pattern

```csharp
// SOLUTION: Circuit breaker with state management

public class CircuitBreaker
{
    private CircuitState _state = CircuitState.Closed;
    private int _failureCount;
    private DateTime _lastFailureTime;
    private readonly int _threshold = 5;
    private readonly TimeSpan _timeout = TimeSpan.FromSeconds(30);

    public async Task<T> ExecuteAsync<T>(Func<Task<T>> operation)
    {
        if (_state == CircuitState.Open)
        {
            if (DateTime.UtcNow - _lastFailureTime > _timeout)
            {
                _state = CircuitState.HalfOpen;
            }
            else
            {
                throw new CircuitBreakerOpenException("Circuit is open");
            }
        }

        try
        {
            var result = await operation();

            if (_state == CircuitState.HalfOpen)
            {
                Reset();
            }

            return result;
        }
        catch (Exception)
        {
            RecordFailure();
            throw;
        }
    }

    private void RecordFailure()
    {
        _failureCount++;
        _lastFailureTime = DateTime.UtcNow;

        if (_failureCount >= _threshold)
        {
            _state = CircuitState.Open;
            Log.Warning("Circuit breaker opened after {Failures} failures", _failureCount);
        }
    }

    private void Reset()
    {
        _state = CircuitState.Closed;
        _failureCount = 0;
        Log.Information("Circuit breaker reset");
    }
}

public enum CircuitState { Closed, Open, HalfOpen }
```

### Solution 4: Bulkhead Isolation

```csharp
// SOLUTION: Bulkhead to limit concurrent calls

builder.Services.AddHttpClient("ExternalApi")
    .AddPolicyHandler(Policy.BulkheadAsync<HttpResponseMessage>(
        maxParallelization: 10,      // Max concurrent executions
        maxQueuingActions: 100,      // Max queued requests
        onBulkheadRejectedAsync: context =>
        {
            Log.Warning("Request rejected by bulkhead");
            return Task.CompletedTask;
        }))
    .AddPolicyHandler(GetRetryPolicy())
    .AddPolicyHandler(GetCircuitBreakerPolicy());
```

### Solution 5: Queue-Based Load Leveling

```csharp
// SOLUTION: Use queue to smooth out traffic spikes

public class QueuedRequestHandler
{
    private readonly ServiceBusSender _sender;
    private readonly IMemoryCache _cache;

    public async Task<Response> RequestAsync(Request request)
    {
        var requestId = Guid.NewGuid().ToString();

        // Send to queue instead of direct call
        await _sender.SendMessageAsync(new ServiceBusMessage(
            JsonSerializer.Serialize(request))
        {
            MessageId = requestId,
            TimeToLive = TimeSpan.FromMinutes(5)
        });

        // Return immediately with request ID
        // Client polls for result or uses SignalR for notification
        return new Response { RequestId = requestId, Status = "Processing" };
    }
}

// Background processor handles at controlled rate
public class QueueProcessor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var message in _receiver.ReceiveMessagesAsync(stoppingToken))
        {
            try
            {
                await ProcessWithRetryAsync(message);
                await _receiver.CompleteMessageAsync(message);
            }
            catch (Exception ex)
            {
                // Dead letter after max retries
                if (message.DeliveryCount >= 3)
                {
                    await _receiver.DeadLetterMessageAsync(message);
                }
            }
        }
    }
}
```

### Azure-Specific Solutions

#### Azure Service Bus Retry Policy

```csharp
// Service Bus client with built-in retry
var clientOptions = new ServiceBusClientOptions
{
    RetryOptions = new ServiceBusRetryOptions
    {
        Mode = ServiceBusRetryMode.Exponential,
        MaxRetries = 5,
        Delay = TimeSpan.FromSeconds(1),
        MaxDelay = TimeSpan.FromSeconds(60),
        TryTimeout = TimeSpan.FromSeconds(30)
    }
};

var client = new ServiceBusClient(connectionString, clientOptions);
```

#### Azure SDK Default Retry Policy

```csharp
// All Azure SDK clients have built-in retry with jitter
var blobClientOptions = new BlobClientOptions
{
    Retry =
    {
        Mode = RetryMode.Exponential,
        MaxRetries = 5,
        Delay = TimeSpan.FromSeconds(1),
        MaxDelay = TimeSpan.FromSeconds(60),
        NetworkTimeout = TimeSpan.FromSeconds(100)
    }
};

var blobClient = new BlobServiceClient(connectionString, blobClientOptions);
```

## Retry Strategy Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      RETRY STRATEGY COMPARISON                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Strategy              Delays                Risk              Use When    │
│   ────────              ──────                ────              ────────    │
│                                                                              │
│   Immediate             0, 0, 0              HIGH              Never!      │
│   Fixed Interval        1s, 1s, 1s           HIGH              Never!      │
│   Linear Backoff        1s, 2s, 3s           MEDIUM            Rarely      │
│   Exponential           1s, 2s, 4s, 8s       LOW-MEDIUM        Sometimes   │
│   Exp + Full Jitter     random(0, delay)     LOW               Better      │
│   Decorrelated Jitter   random(base, 3×prev) LOWEST            Best!       │
│                                                                              │
│   Full Jitter Example:                                                       │
│   Base: 1s, Cap: 60s                                                        │
│   Retry 1: random(0, 1) = 0.7s                                              │
│   Retry 2: random(0, 2) = 1.3s                                              │
│   Retry 3: random(0, 4) = 2.8s                                              │
│   Retry 4: random(0, 8) = 5.1s                                              │
│                                                                              │
│   Decorrelated Jitter Example:                                              │
│   Base: 1s, Previous: starts at base                                        │
│   Retry 1: random(1, 3×1) = 2.1s                                            │
│   Retry 2: random(1, 3×2.1) = 4.7s                                          │
│   Retry 3: random(1, 3×4.7) = 8.3s                                          │
│   Retry 4: random(1, 3×8.3) = 15.2s                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Test Cases

### Unit Test: Backoff Timing

```csharp
[Fact]
public async Task RetryPolicy_ShouldUseExponentialBackoff()
{
    // Arrange
    var delays = new List<TimeSpan>();
    var policy = Policy
        .Handle<HttpRequestException>()
        .WaitAndRetryAsync(
            retryCount: 3,
            sleepDurationProvider: (attempt, _) =>
            {
                var delay = TimeSpan.FromSeconds(Math.Pow(2, attempt));
                delays.Add(delay);
                return delay;
            });

    var callCount = 0;
    Task<string> FailingOperation() =>
        ++callCount <= 3
            ? throw new HttpRequestException()
            : Task.FromResult("Success");

    // Act
    await policy.ExecuteAsync(FailingOperation);

    // Assert - verify exponential delays
    Assert.Equal(3, delays.Count);
    Assert.True(delays[0] < delays[1]);
    Assert.True(delays[1] < delays[2]);
}
```

### Unit Test: Circuit Breaker

```csharp
[Fact]
public async Task CircuitBreaker_ShouldOpenAfterThreshold()
{
    // Arrange
    var breaker = Policy
        .Handle<HttpRequestException>()
        .CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 3,
            durationOfBreak: TimeSpan.FromSeconds(30));

    // Act - cause failures
    for (int i = 0; i < 3; i++)
    {
        await Assert.ThrowsAsync<HttpRequestException>(
            () => breaker.ExecuteAsync(() => throw new HttpRequestException()));
    }

    // Assert - circuit should be open
    await Assert.ThrowsAsync<BrokenCircuitException>(
        () => breaker.ExecuteAsync(() => Task.FromResult("test")));
}
```

### Load Test: No Synchronized Retries

```csharp
[Fact]
public async Task MultipleClients_ShouldNotRetrySynchronously()
{
    // Arrange
    var retryTimes = new ConcurrentBag<DateTime>();
    var policy = GetRetryPolicyWithJitter();

    Task SimulateClient() => policy.ExecuteAsync(async () =>
    {
        retryTimes.Add(DateTime.UtcNow);
        throw new HttpRequestException();
    });

    // Act - simulate 100 clients
    var tasks = Enumerable.Range(0, 100).Select(_ => SimulateClient());
    await Task.WhenAll(tasks.Select(t => t.ContinueWith(_ => { })));

    // Assert - retries should be spread out (not synchronized)
    var times = retryTimes.OrderBy(t => t).ToList();
    var burstCount = times.GroupBy(t => t.Ticks / TimeSpan.TicksPerSecond)
        .Max(g => g.Count());

    Assert.True(burstCount < 50,
        $"Too many synchronized retries: {burstCount} in same second");
}
```

## Best Practices Checklist

- [ ] Use exponential backoff with jitter
- [ ] Implement circuit breaker pattern
- [ ] Set maximum retry limits
- [ ] Use Polly for HTTP clients
- [ ] Implement bulkhead isolation
- [ ] Consider queue-based load leveling
- [ ] Monitor retry rates and patterns
- [ ] Log retry attempts for debugging
- [ ] Test failure scenarios
- [ ] Configure Azure SDK retry policies

## AWS to Azure Migration Note

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| AWS SDK retry config | Azure SDK RetryOptions |
| Step Functions retry | Durable Functions retry |
| SQS dead letter queue | Service Bus dead letter |
| Lambda retry behavior | Functions retry policies |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [Noisy Neighbor](08-noisy-neighbor.md)*

*Next: [Synchronous I/O](10-synchronous-io.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
