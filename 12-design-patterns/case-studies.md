# Cloud Design Patterns Case Studies

*When theory meets production: stories from the trenches*

---

## Case Study 1: The Circuit Breaker That Saved Black Friday

### The Setup

**Company:** MegaMart Online
**Industry:** E-commerce
**The Day:** Black Friday 2024, 2:47 AM

### The Problem

```
THE 2:47 AM INCIDENT:
─────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Everything was fine until the Payment Service started slowing...  │
│                                                                      │
│   Timeline:                                                          │
│   ─────────                                                          │
│   2:45 AM  Payment API latency: 200ms → 2s (payment provider issue) │
│   2:46 AM  Checkout service waiting on payment... threads exhausted │
│   2:47 AM  Order service waiting on checkout... cascading failure   │
│   2:48 AM  Product service overwhelmed with retries                 │
│   2:49 AM  Homepage returning 503 errors                            │
│   2:50 AM  "MEGAMART IS DOWN" trending on Twitter                   │
│   2:51 AM  CEO's phone rings                                        │
│                                                                      │
│   THE CASCADE:                                                       │
│   ┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐     │
│   │Homepage │────►│ Product │────►│ Order   │────►│ Payment │     │
│   │  ✗      │     │   ✗     │     │   ✗     │     │   🐌    │     │
│   └─────────┘     └─────────┘     └─────────┘     └─────────┘     │
│                                                                      │
│   One slow service = entire platform down                           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Solution: Circuit Breaker + Bulkhead

After the incident, they implemented proper resilience patterns:

```csharp
// Before: Naive implementation
public async Task<PaymentResult> ProcessPayment(Order order)
{
    // If this hangs, EVERYTHING hangs
    return await _httpClient.PostAsync("/api/payments", order);
}

// After: Circuit Breaker + Timeout + Bulkhead
public class ResilientPaymentService
{
    private readonly IAsyncPolicy<HttpResponseMessage> _resiliencePolicy;

    public ResilientPaymentService(IHttpClientFactory httpClientFactory)
    {
        // Circuit Breaker: After 5 failures, stop trying for 30 seconds
        var circuitBreaker = Policy<HttpResponseMessage>
            .Handle<HttpRequestException>()
            .OrResult(r => !r.IsSuccessStatusCode)
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (result, duration) =>
                {
                    _logger.LogWarning(
                        "Circuit breaker opened for {Duration}s",
                        duration.TotalSeconds);
                    _metrics.CircuitBreakerOpened();
                },
                onReset: () =>
                {
                    _logger.LogInformation("Circuit breaker reset");
                    _metrics.CircuitBreakerReset();
                }
            );

        // Retry: 3 attempts with exponential backoff
        var retry = Policy<HttpResponseMessage>
            .Handle<HttpRequestException>()
            .OrResult(r => r.StatusCode == HttpStatusCode.ServiceUnavailable)
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt =>
                    TimeSpan.FromSeconds(Math.Pow(2, attempt)),
                onRetry: (result, delay, attempt, context) =>
                {
                    _logger.LogWarning(
                        "Retry {Attempt} after {Delay}s",
                        attempt, delay.TotalSeconds);
                }
            );

        // Timeout: Don't wait forever
        var timeout = Policy.TimeoutAsync<HttpResponseMessage>(
            TimeSpan.FromSeconds(5));

        // Combine policies: Timeout → Retry → Circuit Breaker
        _resiliencePolicy = Policy.WrapAsync(timeout, retry, circuitBreaker);
    }

    public async Task<PaymentResult> ProcessPayment(Order order)
    {
        try
        {
            var response = await _resiliencePolicy.ExecuteAsync(async () =>
                await _httpClient.PostAsync("/api/payments", order));

            return await response.Content.ReadAsAsync<PaymentResult>();
        }
        catch (BrokenCircuitException)
        {
            // Circuit is open - don't even try
            return new PaymentResult
            {
                Success = false,
                Message = "Payment service temporarily unavailable. " +
                          "Your order will be processed shortly.",
                Queued = true  // We'll process it when service recovers
            };
        }
    }
}
```

### The Bulkhead Pattern

```
BULKHEAD ISOLATION:
───────────────────

Before (One Thread Pool):
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   All requests share same thread pool (100 threads)                 │
│                                                                      │
│   Thread Pool: [||||||||||||||||||||||||||||||] 100 threads         │
│                                                                      │
│   Payment API slow? All 100 threads waiting on payment.             │
│   Product listings? Sorry, no threads available.                    │
│   Homepage? 503 Service Unavailable.                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

After (Isolated Bulkheads):
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Separate thread pools per downstream service                      │
│                                                                      │
│   ┌──────────────────────────┐                                      │
│   │ Product Pool (30 threads)│  → Products keep working!           │
│   │ [||||||||||||||]         │                                      │
│   └──────────────────────────┘                                      │
│                                                                      │
│   ┌──────────────────────────┐                                      │
│   │ Order Pool (30 threads)  │  → Orders queue up gracefully       │
│   │ [|||||||||||||]          │                                      │
│   └──────────────────────────┘                                      │
│                                                                      │
│   ┌──────────────────────────┐                                      │
│   │ Payment Pool (20 threads)│  → Payment threads exhausted, but   │
│   │ [XXXXXXXXXX] ← FULL      │     contained! Others unaffected.   │
│   └──────────────────────────┘                                      │
│                                                                      │
│   ┌──────────────────────────┐                                      │
│   │ General Pool (20 threads)│  → Homepage still loads!            │
│   │ [|||||||]                │                                      │
│   └──────────────────────────┘                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Results: The Next Black Friday

```
BLACK FRIDAY 2025 - SAME PAYMENT ISSUE, DIFFERENT OUTCOME:
──────────────────────────────────────────────────────────

3:12 AM  Payment latency spikes to 5s
3:12 AM  Circuit breaker opens after 5 timeouts
3:12 AM  Orders queued to Service Bus (processed when recovered)
3:13 AM  Customers see: "Payment processing delayed, we'll confirm by email"
3:13 AM  Shopping continues! Product browsing, cart additions work fine
3:45 AM  Payment provider recovers
3:45 AM  Circuit breaker half-opens, tests with 1 request
3:46 AM  Circuit breaker closes, normal operation
3:47 AM  Queued orders processed, confirmation emails sent

IMPACT COMPARISON:
┌─────────────────────┬─────────────────┬─────────────────┐
│ Metric              │ 2024 (No CB)    │ 2025 (With CB)  │
├─────────────────────┼─────────────────┼─────────────────┤
│ Downtime            │ 47 minutes      │ 0 minutes       │
│ Lost Revenue        │ $2.3M           │ ~$0             │
│ Twitter mentions    │ 12,000 (angry)  │ 50 (praise!)    │
│ CEO sleep lost      │ All night       │ None            │
└─────────────────────┴─────────────────┴─────────────────┘
```

---

## Case Study 2: Cache-Aside Pattern at Scale

### The Setup

**Company:** NewsFlow
**Industry:** Digital Media
**Challenge:** Serve 50M article reads per day with sub-100ms latency

### The Problem

```
THE PERFORMANCE CRISIS:
───────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Direct database queries for every article read:                   │
│                                                                      │
│   User Request ──► API Server ──► Azure SQL ──► Response            │
│                                                                      │
│   Database stats:                                                    │
│   ├── 50M reads/day = 580 reads/second average                      │
│   ├── Peak: 3,000 reads/second (breaking news)                      │
│   ├── Average query time: 150ms                                     │
│   ├── P99 latency: 2.5 seconds (unacceptable)                       │
│   │                                                                  │
│   ├── Same viral article read 500,000 times?                        │
│   │   500,000 identical database queries. Genius. 🤦                │
│   │                                                                  │
│   └── Database CPU at 95%, auto-scaling maxed out                   │
│       Monthly cost: $12,000 (for a mostly-read workload!)           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Solution: Cache-Aside with Redis

```
CACHE-ASIDE PATTERN:
────────────────────

Read Flow:
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   1. Check cache first                                              │
│   ┌────────────┐         ┌────────────────┐                        │
│   │   Client   │────────►│  API Server    │                        │
│   └────────────┘         └───────┬────────┘                        │
│                                  │                                   │
│                                  ▼                                   │
│                          ┌──────────────┐                           │
│                          │ Azure Redis  │  "Is article in cache?"   │
│                          │   Cache      │                           │
│                          └──────┬───────┘                           │
│                                 │                                    │
│                    ┌────────────┴────────────┐                      │
│                    │                         │                       │
│               Cache HIT ✓                Cache MISS                  │
│               (95% of time)              (5% of time)                │
│                    │                         │                       │
│                    │                         ▼                       │
│                    │                 ┌──────────────┐               │
│                    │                 │  Azure SQL   │               │
│                    │                 │  Database    │               │
│                    │                 └──────┬───────┘               │
│                    │                        │                        │
│                    │                        ▼                        │
│                    │                 Store in Redis                  │
│                    │                 (TTL: 5 minutes)                │
│                    │                        │                        │
│                    └───────────┬────────────┘                       │
│                                │                                     │
│                                ▼                                     │
│                         Return Article                               │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Implementation

```csharp
public class ArticleService
{
    private readonly IDatabase _cache;
    private readonly ArticleRepository _db;
    private readonly ILogger<ArticleService> _logger;

    private static readonly TimeSpan CacheDuration = TimeSpan.FromMinutes(5);
    private static readonly TimeSpan LockTimeout = TimeSpan.FromSeconds(10);

    public async Task<Article> GetArticleAsync(string articleId)
    {
        var cacheKey = $"article:{articleId}";

        // 1. Try cache first
        var cached = await _cache.StringGetAsync(cacheKey);
        if (cached.HasValue)
        {
            _logger.LogDebug("Cache HIT for article {ArticleId}", articleId);
            return JsonSerializer.Deserialize<Article>(cached!);
        }

        // 2. Cache miss - need to load from database
        _logger.LogDebug("Cache MISS for article {ArticleId}", articleId);

        // 3. Use distributed lock to prevent cache stampede
        //    (1000 requests for same article = 1 DB query, not 1000)
        var lockKey = $"lock:article:{articleId}";
        await using var redLock = await _lockFactory.CreateLockAsync(
            lockKey,
            LockTimeout);

        if (redLock.IsAcquired)
        {
            // Double-check cache (another thread might have populated it)
            cached = await _cache.StringGetAsync(cacheKey);
            if (cached.HasValue)
            {
                return JsonSerializer.Deserialize<Article>(cached!);
            }

            // Load from database
            var article = await _db.GetArticleAsync(articleId);

            if (article != null)
            {
                // Store in cache
                await _cache.StringSetAsync(
                    cacheKey,
                    JsonSerializer.Serialize(article),
                    CacheDuration);
            }

            return article;
        }

        // Couldn't acquire lock - wait and retry from cache
        await Task.Delay(100);
        cached = await _cache.StringGetAsync(cacheKey);
        return cached.HasValue
            ? JsonSerializer.Deserialize<Article>(cached!)
            : await _db.GetArticleAsync(articleId);  // Fallback
    }

    // Write-through for article updates
    public async Task UpdateArticleAsync(Article article)
    {
        // 1. Update database first (source of truth)
        await _db.UpdateArticleAsync(article);

        // 2. Invalidate cache (don't update - let next read repopulate)
        var cacheKey = $"article:{article.Id}";
        await _cache.KeyDeleteAsync(cacheKey);

        _logger.LogInformation("Article {ArticleId} updated, cache invalidated",
            article.Id);
    }
}
```

### Bicep for Redis Cache

```bicep
// Premium Redis for production workload
resource redisCache 'Microsoft.Cache/redis@2023-08-01' = {
  name: 'newsflow-cache'
  location: location
  properties: {
    sku: {
      name: 'Premium'
      family: 'P'
      capacity: 2  // P2: 6GB, 25K connections
    }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
    redisConfiguration: {
      'maxmemory-policy': 'allkeys-lru'  // Evict least recently used
      'maxfragmentationmemory-reserved': '125'
      'maxmemory-reserved': '125'
    }
    replicasPerPrimary: 2  // High availability
  }
}

// Private endpoint for security
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'redis-pe'
  location: location
  properties: {
    subnet: { id: subnetId }
    privateLinkServiceConnections: [
      {
        name: 'redis-connection'
        properties: {
          privateLinkServiceId: redisCache.id
          groupIds: ['redisCache']
        }
      }
    ]
  }
}
```

### Results

```
BEFORE vs AFTER:
────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Metric              Before (No Cache)    After (Redis)            │
│   ────────────────────────────────────────────────────────────────  │
│   Cache hit rate      0%                   95%                       │
│   P50 latency         150ms                8ms                       │
│   P99 latency         2,500ms              45ms                      │
│   Database QPS        580                  29 (95% reduction!)       │
│   Database cost       $12,000/month        $800/month                │
│   Redis cost          $0                   $700/month                │
│   Total cost          $12,000/month        $1,500/month              │
│   ────────────────────────────────────────────────────────────────  │
│   Savings                                  $10,500/month (87.5%)     │
│                                                                      │
│   Breaking news performance:                                         │
│   ├── Viral article (500K reads): 500K DB queries → 1 DB query      │
│   ├── Database CPU at peak: 95% → 15%                               │
│   └── User experience: Fast, even during traffic spikes             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Case Study 3: Saga Pattern for Distributed Transactions

### The Setup

**Company:** TripPlanner Pro
**Industry:** Travel Booking
**Challenge:** Book flight + hotel + car atomically across 3 external providers

### The Problem: No Distributed Transactions

```
THE IMPOSSIBILITY:
──────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Customer wants:                                                    │
│   ├── Flight: United Airlines API                                   │
│   ├── Hotel: Marriott API                                           │
│   └── Car: Hertz API                                                │
│                                                                      │
│   The dream (impossible):                                            │
│   BEGIN DISTRIBUTED TRANSACTION                                      │
│       Book flight with United                                        │
│       Book hotel with Marriott                                       │
│       Book car with Hertz                                            │
│   COMMIT (all or nothing)                                            │
│                                                                      │
│   The reality:                                                       │
│   ├── These are 3 separate external companies                       │
│   ├── No shared database                                             │
│   ├── No 2-phase commit protocol                                    │
│   └── If hotel fails after flight booked, customer has orphan flight│
│                                                                      │
│   What actually happened to customers:                              │
│   "I paid for a flight but my hotel wasn't booked!"                 │
│   "I got charged twice when I retried!"                             │
│   "My car reservation vanished but I was still charged!"            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Solution: Saga Pattern with Compensations

```
CHOREOGRAPHY-BASED SAGA:
────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   HAPPY PATH (All bookings succeed):                                │
│   ────────────────────────────────                                  │
│                                                                      │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    │
│   │  Start   │───►│  Book    │───►│  Book    │───►│  Book    │    │
│   │  Saga    │    │  Flight  │    │  Hotel   │    │  Car     │    │
│   └──────────┘    └────┬─────┘    └────┬─────┘    └────┬─────┘    │
│                        │ ✓             │ ✓             │ ✓        │
│                        ▼               ▼               ▼          │
│                   FlightBooked    HotelBooked     CarBooked       │
│                        │               │               │          │
│                        └───────────────┴───────────────┘          │
│                                        │                           │
│                                        ▼                           │
│                              ┌──────────────────┐                  │
│                              │ Trip Confirmed!  │                  │
│                              │ Email customer   │                  │
│                              └──────────────────┘                  │
│                                                                      │
│   SAD PATH (Hotel booking fails):                                   │
│   ───────────────────────────────                                   │
│                                                                      │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐                     │
│   │  Start   │───►│  Book    │───►│  Book    │                     │
│   │  Saga    │    │  Flight  │    │  Hotel   │                     │
│   └──────────┘    └────┬─────┘    └────┬─────┘                     │
│                        │ ✓             │ ✗                          │
│                        │               │                            │
│                        │      ┌────────┘                            │
│                        │      │ HotelBookingFailed                  │
│                        │      ▼                                     │
│                        │  ┌──────────────┐                         │
│                        └─►│ COMPENSATE:  │ Cancel the flight!      │
│                           │ Cancel Flight│                         │
│                           └──────┬───────┘                         │
│                                  │                                  │
│                                  ▼                                  │
│                         ┌──────────────────┐                       │
│                         │ Saga Failed      │                       │
│                         │ Notify customer  │                       │
│                         │ Full refund      │                       │
│                         └──────────────────┘                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Durable Functions Implementation

```csharp
[FunctionName("TripBookingSaga")]
public static async Task<TripResult> RunSaga(
    [OrchestrationTrigger] IDurableOrchestrationContext context,
    ILogger log)
{
    var request = context.GetInput<TripBookingRequest>();
    var compensations = new Stack<Func<Task>>();

    try
    {
        // Step 1: Book Flight
        log.LogInformation("Booking flight...");
        var flightResult = await context.CallActivityAsync<BookingResult>(
            "BookFlight",
            request.Flight);

        if (!flightResult.Success)
        {
            throw new BookingException("Flight booking failed", flightResult.Error);
        }

        // Register compensation
        compensations.Push(async () =>
        {
            await context.CallActivityAsync("CancelFlight", flightResult.ConfirmationId);
        });

        // Step 2: Book Hotel
        log.LogInformation("Booking hotel...");
        var hotelResult = await context.CallActivityAsync<BookingResult>(
            "BookHotel",
            request.Hotel);

        if (!hotelResult.Success)
        {
            throw new BookingException("Hotel booking failed", hotelResult.Error);
        }

        compensations.Push(async () =>
        {
            await context.CallActivityAsync("CancelHotel", hotelResult.ConfirmationId);
        });

        // Step 3: Book Car
        log.LogInformation("Booking car...");
        var carResult = await context.CallActivityAsync<BookingResult>(
            "BookCar",
            request.Car);

        if (!carResult.Success)
        {
            throw new BookingException("Car booking failed", carResult.Error);
        }

        // All succeeded! Return confirmation
        return new TripResult
        {
            Success = true,
            FlightConfirmation = flightResult.ConfirmationId,
            HotelConfirmation = hotelResult.ConfirmationId,
            CarConfirmation = carResult.ConfirmationId,
            TotalCost = flightResult.Cost + hotelResult.Cost + carResult.Cost
        };
    }
    catch (BookingException ex)
    {
        // Something failed - run compensations in reverse order
        log.LogWarning("Saga failed: {Error}. Running compensations...", ex.Message);

        while (compensations.Count > 0)
        {
            var compensation = compensations.Pop();
            try
            {
                await compensation();
                log.LogInformation("Compensation successful");
            }
            catch (Exception compEx)
            {
                // Log but continue - we need to try all compensations
                log.LogError(compEx, "Compensation failed, will retry later");

                // Queue for manual review
                await context.CallActivityAsync("QueueFailedCompensation", new
                {
                    SagaId = context.InstanceId,
                    Compensation = compensation.Method.Name,
                    Error = compEx.Message
                });
            }
        }

        return new TripResult
        {
            Success = false,
            Error = ex.Message
        };
    }
}

// Activity functions
[FunctionName("BookFlight")]
public static async Task<BookingResult> BookFlight(
    [ActivityTrigger] FlightRequest request,
    ILogger log)
{
    // Idempotency key prevents double-booking on retry
    var idempotencyKey = $"flight-{request.CustomerId}-{request.FlightId}-{request.Date:yyyyMMdd}";

    // Check if already booked (idempotent)
    var existing = await _flightService.GetBookingByIdempotencyKey(idempotencyKey);
    if (existing != null)
    {
        log.LogInformation("Flight already booked, returning existing confirmation");
        return new BookingResult { Success = true, ConfirmationId = existing.Id };
    }

    // Book with external API
    var result = await _unitedApi.BookFlight(request, idempotencyKey);

    return new BookingResult
    {
        Success = result.Status == "confirmed",
        ConfirmationId = result.ConfirmationNumber,
        Cost = result.TotalPrice,
        Error = result.ErrorMessage
    };
}

[FunctionName("CancelFlight")]
public static async Task CancelFlight(
    [ActivityTrigger] string confirmationId,
    ILogger log)
{
    log.LogInformation("Canceling flight {ConfirmationId}", confirmationId);

    // Idempotent cancellation
    var result = await _unitedApi.CancelFlight(confirmationId);

    if (!result.Success && result.ErrorCode != "ALREADY_CANCELLED")
    {
        throw new Exception($"Failed to cancel flight: {result.Error}");
    }
}
```

### Monitoring the Saga

```
SAGA STATE VISUALIZATION (Azure Portal):
────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Saga Instance: trip-booking-abc123                                │
│   Status: COMPENSATING                                               │
│   Customer: john.doe@email.com                                      │
│                                                                      │
│   Timeline:                                                          │
│   ──────────────────────────────────────────────────────────────────│
│   14:32:01  Saga started                                            │
│   14:32:02  BookFlight: SUCCESS (UA-789123)                         │
│   14:32:05  BookHotel: SUCCESS (MAR-456789)                         │
│   14:32:08  BookCar: FAILED (Hertz API timeout)                     │
│   14:32:08  Starting compensations...                               │
│   14:32:09  CancelHotel: SUCCESS                                    │
│   14:32:11  CancelFlight: SUCCESS                                   │
│   14:32:11  Saga completed: COMPENSATED                             │
│                                                                      │
│   Customer notification: "Sorry, we couldn't complete your          │
│   booking due to car rental availability. Full refund issued."      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Results

| Metric | Before (No Saga) | After (Saga Pattern) |
|--------|------------------|---------------------|
| Orphaned bookings | 3.2% of transactions | 0.01% (retry edge cases) |
| Customer complaints | 150/week | 5/week (93% reduction) |
| Manual reconciliation | 20 hours/week | 1 hour/week |
| Double-charges | 1.5% of retries | 0% (idempotency) |
| Support cost | $15,000/month | $2,000/month |

---

## Case Study 4: Strangler Fig Migration

### The Setup

**Company:** FinanceTrack
**Industry:** Fintech
**Challenge:** Modernize 15-year-old .NET Framework monolith without downtime

### The Strangler Fig Approach

```
THE GRADUAL STRANGULATION:
──────────────────────────

Year 1 (Before):
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   100% traffic to Legacy Monolith                                   │
│                                                                      │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │                      Legacy Monolith                          │ │
│   │                     (.NET Framework 4.5)                      │ │
│   │                                                                │ │
│   │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │ │
│   │   │ Accounts │ │ Payments │ │ Reports  │ │ Users    │       │ │
│   │   └──────────┘ └──────────┘ └──────────┘ └──────────┘       │ │
│   │                        │                                      │ │
│   │                   ┌────┴────┐                                 │ │
│   │                   │ Big DB  │                                 │ │
│   │                   └─────────┘                                 │ │
│   └──────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

Year 2 (Mid-Migration):
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│                    ┌──────────────────────┐                         │
│   Traffic ────────►│   Azure Front Door   │                         │
│                    │   (Facade/Router)    │                         │
│                    └──────────┬───────────┘                         │
│                               │                                      │
│          ┌────────────────────┼────────────────────┐                │
│          │                    │                    │                │
│          ▼                    ▼                    ▼                │
│   ┌─────────────┐     ┌─────────────┐     ┌─────────────┐          │
│   │  Reports    │     │  Legacy     │     │  Users      │          │
│   │  Service    │     │  Monolith   │     │  Service    │          │
│   │  (.NET 8)   │     │  (smaller)  │     │  (.NET 8)   │          │
│   │     ✓       │     │             │     │     ✓       │          │
│   └──────┬──────┘     └──────┬──────┘     └──────┬──────┘          │
│          │                   │                   │                  │
│          ▼                   ▼                   ▼                  │
│   ┌───────────┐       ┌───────────┐       ┌───────────┐            │
│   │ Cosmos DB │       │  Big DB   │       │ Azure SQL │            │
│   │ (Reports) │       │ (Legacy)  │       │ (Users)   │            │
│   └───────────┘       └───────────┘       └───────────┘            │
│                                                                      │
│   Extracted: Reports (30%), Users (20%)                             │
│   Remaining: Accounts, Payments, Core Logic (50%)                   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

Year 3 (Complete):
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│                    ┌──────────────────────┐                         │
│   Traffic ────────►│   API Management     │                         │
│                    │   (Unified API)      │                         │
│                    └──────────┬───────────┘                         │
│                               │                                      │
│     ┌─────────────┬──────────┼──────────┬─────────────┐            │
│     │             │          │          │             │            │
│     ▼             ▼          ▼          ▼             ▼            │
│ ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐        │
│ │Accounts│  │Payments│  │Reports │  │ Users  │  │Notifs  │        │
│ │Service │  │Service │  │Service │  │Service │  │Service │        │
│ │(.NET 8)│  │(.NET 8)│  │(.NET 8)│  │(.NET 8)│  │(.NET 8)│        │
│ └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘  └───┬────┘        │
│     │           │           │           │           │              │
│     ▼           ▼           ▼           ▼           ▼              │
│   Azure      Azure      Cosmos DB   Azure SQL   Service Bus        │
│   SQL        SQL                                                    │
│                                                                      │
│   Legacy Monolith: DECOMMISSIONED 🎉                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Anti-Corruption Layer

```csharp
// Anti-Corruption Layer: Translates between new and legacy worlds
public class AccountsAntiCorruptionLayer
{
    private readonly LegacyAccountsClient _legacy;
    private readonly IAccountRepository _modern;
    private readonly IFeatureFlags _features;

    public async Task<Account> GetAccountAsync(string accountId)
    {
        // During migration, check which system owns this account
        if (_features.IsEnabled("accounts.use-modern-service"))
        {
            try
            {
                // Try modern service first
                var modernAccount = await _modern.GetByIdAsync(accountId);
                if (modernAccount != null)
                {
                    return modernAccount;
                }
            }
            catch (Exception ex)
            {
                _logger.LogWarning(ex, "Modern service failed, falling back to legacy");
            }
        }

        // Fall back to legacy
        var legacyAccount = await _legacy.GetAccountByIdAsync(accountId);

        // Translate legacy response to modern domain model
        return TranslateFromLegacy(legacyAccount);
    }

    private Account TranslateFromLegacy(LegacyAccountResponse legacy)
    {
        // Legacy has different field names, types, and conventions
        return new Account
        {
            Id = legacy.AcctNo,  // Different name
            CustomerId = legacy.CustID.ToString(),  // int → string
            Balance = decimal.Parse(legacy.BalanceStr),  // string → decimal (why?!)
            Status = MapStatus(legacy.StatusCode),  // "A" → AccountStatus.Active
            CreatedAt = DateTime.ParseExact(
                legacy.CreateDt,
                "yyyyMMdd",  // Legacy date format
                CultureInfo.InvariantCulture),
            Type = MapAccountType(legacy.TypeCd)
        };
    }

    private AccountStatus MapStatus(string legacyCode) => legacyCode switch
    {
        "A" => AccountStatus.Active,
        "I" => AccountStatus.Inactive,
        "S" => AccountStatus.Suspended,
        "C" => AccountStatus.Closed,
        _ => throw new InvalidOperationException($"Unknown status: {legacyCode}")
    };
}
```

### Results

| Metric | Legacy Monolith | After Strangler | Improvement |
|--------|-----------------|-----------------|-------------|
| Deploy Frequency | Monthly | Daily | 30x faster |
| Downtime per deploy | 2 hours | 0 (blue-green) | 100% improvement |
| Dev Team Velocity | Declining | 3x increase | Teams unblocked |
| Security Updates | Months behind | Same day | Critical for fintech |
| Infrastructure Cost | $45K/month | $28K/month | 38% savings |
| Migration Duration | - | 18 months | Gradual, safe |

---

## Summary: Pattern Selection Quick Reference

```
CHOOSING THE RIGHT PATTERN:
───────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   PROBLEM                          PATTERN                          │
│   ───────────────────────────────────────────────────────────────   │
│   Service calls failing            Circuit Breaker + Retry          │
│   One service takes down all       Bulkhead                         │
│   Database is bottleneck           Cache-Aside                      │
│   Need atomic across services      Saga (not 2PC!)                  │
│   Migrating monolith               Strangler Fig + ACL              │
│   Traffic spikes                   Queue-Based Load Leveling        │
│   Need pub/sub                     Publisher/Subscriber             │
│   Large messages                   Claim Check                      │
│   Multi-region deployment          Deployment Stamps                │
│                                                                      │
│   REMEMBER:                                                         │
│   ├── Patterns are tools, not rules                                 │
│   ├── Combine patterns as needed                                    │
│   ├── Simplest solution that works is often best                   │
│   └── Monitor everything to know if patterns are helping           │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

*Navigation: [Quick Reference](quick-reference.md) | [README](README.md) | [Main Guide](../README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
