# Design Principles for Cloud Applications

## Overview

These design principles guide architects in building reliable, scalable, and maintainable cloud applications. Each principle addresses specific challenges in distributed systems.

---

## Principle 1: Design for Self-Healing

### Concept

Systems should detect failures and recover automatically without human intervention. In distributed systems, failures are inevitable—design for MTTR (Mean Time To Recovery) not just MTBF (Mean Time Between Failures).

### Implementation Strategies

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          SELF-HEALING PATTERNS                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   1. RETRY PATTERN                                                               │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                                                                          │  │
│   │   Request ──► Failure ──► Wait ──► Retry ──► Success                    │  │
│   │                  │         │                                             │  │
│   │                  │    Exponential                                        │  │
│   │                  │     Backoff                                           │  │
│   │                  │                                                       │  │
│   │                  └──► Max Retries ──► Circuit Breaker ──► Fallback      │  │
│   │                                                                          │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   2. CIRCUIT BREAKER PATTERN                                                     │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                                                                          │  │
│   │   CLOSED ◄─────────────────────────── HALF-OPEN                         │  │
│   │     │                                     │                              │  │
│   │     │ Failures                           │ Success                      │  │
│   │     │ exceed                             │                              │  │
│   │     │ threshold                          │                              │  │
│   │     ▼                                    │                              │  │
│   │   OPEN ─────── Timeout ─────────────────►│                              │  │
│   │     │                                                                    │  │
│   │     └──► Return fallback / fail fast                                    │  │
│   │                                                                          │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   3. HEALTH ENDPOINT MONITORING                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                                                                          │  │
│   │   Load Balancer ──► /health ──► Check dependencies                      │  │
│   │        │                              │                                  │  │
│   │        │                              ├── Database connection            │  │
│   │        │                              ├── Cache availability             │  │
│   │   Remove unhealthy                    ├── External API                   │  │
│   │   instances                           └── Disk space                     │  │
│   │                                                                          │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Polly resilience policies
public static class ResiliencePolicies
{
    public static IAsyncPolicy<HttpResponseMessage> GetRetryPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .WaitAndRetryAsync(
                retryCount: 3,
                sleepDurationProvider: attempt =>
                    TimeSpan.FromSeconds(Math.Pow(2, attempt)), // Exponential backoff
                onRetry: (outcome, timespan, retryCount, context) =>
                {
                    Log.Warning($"Retry {retryCount} after {timespan.TotalSeconds}s due to {outcome.Exception?.Message}");
                });
    }

    public static IAsyncPolicy<HttpResponseMessage> GetCircuitBreakerPolicy()
    {
        return HttpPolicyExtensions
            .HandleTransientHttpError()
            .CircuitBreakerAsync(
                handledEventsAllowedBeforeBreaking: 5,
                durationOfBreak: TimeSpan.FromSeconds(30),
                onBreak: (outcome, breakDelay) =>
                    Log.Warning($"Circuit broken for {breakDelay.TotalSeconds}s"),
                onReset: () => Log.Information("Circuit reset"),
                onHalfOpen: () => Log.Information("Circuit half-open"));
    }

    // Combined policy
    public static IAsyncPolicy<HttpResponseMessage> GetResiliencePolicy()
    {
        return Policy.WrapAsync(
            GetRetryPolicy(),
            GetCircuitBreakerPolicy());
    }
}

// Health check implementation
public class DatabaseHealthCheck : IHealthCheck
{
    private readonly IDbConnection _connection;

    public async Task<HealthCheckResult> CheckHealthAsync(
        HealthCheckContext context,
        CancellationToken cancellationToken = default)
    {
        try
        {
            await _connection.OpenAsync(cancellationToken);
            await _connection.ExecuteScalarAsync("SELECT 1");
            return HealthCheckResult.Healthy();
        }
        catch (Exception ex)
        {
            return HealthCheckResult.Unhealthy(
                description: "Database connection failed",
                exception: ex);
        }
    }
}
```

---

## Principle 2: Make All Things Redundant

### Concept

Eliminate single points of failure by building redundancy at every layer. The level of redundancy should match business requirements and risk tolerance.

### Redundancy Levels

| Level | Description | Azure Implementation | RTO/RPO |
|-------|-------------|---------------------|---------|
| **Local** | Same datacenter | Availability Sets | Minutes |
| **Zonal** | Multiple AZs | Availability Zones | Seconds-Minutes |
| **Regional** | Multiple regions | Geo-replication | Minutes-Hours |
| **Global** | Active-active | Traffic Manager + Multi-region | Near-zero |

### Architecture Example

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                       MULTI-REGION REDUNDANCY                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                         Azure Traffic Manager                                    │
│                         (Global Load Balancer)                                   │
│                                │                                                 │
│              ┌─────────────────┼─────────────────┐                              │
│              │                 │                 │                              │
│              ▼                 │                 ▼                              │
│   ┌──────────────────┐        │        ┌──────────────────┐                    │
│   │   EAST US        │        │        │   WEST US        │                    │
│   │   (Primary)      │        │        │   (Secondary)    │                    │
│   │                  │        │        │                  │                    │
│   │  ┌────────────┐  │        │        │  ┌────────────┐  │                    │
│   │  │ AZ 1 │ AZ 2│  │        │        │  │ AZ 1 │ AZ 2│  │                    │
│   │  │      │     │  │        │        │  │      │     │  │                    │
│   │  │ App  │ App │  │        │        │  │ App  │ App │  │                    │
│   │  └──────┴─────┘  │        │        │  └──────┴─────┘  │                    │
│   │        │         │        │        │        │         │                    │
│   │  ┌─────▼─────┐   │        │        │  ┌─────▼─────┐   │                    │
│   │  │  SQL DB   │   │◄── Geo-Replication ──►│  SQL DB   │   │                    │
│   │  │ (Primary) │   │        │        │  │(Secondary)│   │                    │
│   │  └───────────┘   │        │        │  └───────────┘   │                    │
│   │                  │        │        │                  │                    │
│   └──────────────────┘        │        └──────────────────┘                    │
│                               │                                                 │
│                    Automatic Failover Group                                     │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Principle 3: Minimize Coordination

### Concept

Coordination between services creates coupling and reduces scalability. Design systems that can operate independently using eventual consistency where possible.

### Strategies

| Strategy | Description | Trade-off |
|----------|-------------|-----------|
| **Async messaging** | Decouple via queues | Higher latency |
| **Event sourcing** | Events as source of truth | Complexity |
| **CQRS** | Separate read/write | Eventually consistent |
| **Saga pattern** | Distributed transactions | Compensation logic |

### Anti-Pattern vs Pattern

```
ANTI-PATTERN: Distributed Transaction
──────────────────────────────────────
   Order Service ───► Payment Service ───► Inventory Service
        │                   │                    │
        └───────── 2PC / XA Transaction ─────────┘
                    (Locks resources)

PATTERN: Choreography with Events
─────────────────────────────────
   Order Service ──publish──► "OrderCreated"
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
        ▼                          ▼                          ▼
   Payment Service          Inventory Service         Notification Service
   (subscribes)             (subscribes)              (subscribes)
        │                          │                          │
        └──publish──► "PaymentProcessed"   └──publish──► "StockReserved"
```

---

## Principle 4: Design to Scale Out

### Concept

Design for horizontal scaling (adding more instances) rather than vertical scaling (bigger instances). Horizontal scaling provides better resilience and cost optimization.

### Implementation Checklist

- [ ] **Stateless services** - No session affinity required
- [ ] **External state** - Use Redis, databases for session state
- [ ] **Idempotent operations** - Safe to retry
- [ ] **No shared mutable state** - Avoid race conditions
- [ ] **Partition data** - Distribute load across shards

### Azure Auto-Scaling

```bicep
// Auto-scale rules for App Service
resource autoscale 'Microsoft.Insights/autoscalesettings@2022-10-01' = {
  name: 'autoscale-settings'
  location: location
  properties: {
    enabled: true
    targetResourceUri: appServicePlan.id
    profiles: [
      {
        name: 'Auto Scale Profile'
        capacity: {
          minimum: '2'
          maximum: '10'
          default: '2'
        }
        rules: [
          {
            // Scale OUT when CPU > 70%
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: appServicePlan.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT5M'
              timeAggregation: 'Average'
              operator: 'GreaterThan'
              threshold: 70
            }
            scaleAction: {
              direction: 'Increase'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT5M'
            }
          }
          {
            // Scale IN when CPU < 30%
            metricTrigger: {
              metricName: 'CpuPercentage'
              metricResourceUri: appServicePlan.id
              timeGrain: 'PT1M'
              statistic: 'Average'
              timeWindow: 'PT10M'
              timeAggregation: 'Average'
              operator: 'LessThan'
              threshold: 30
            }
            scaleAction: {
              direction: 'Decrease'
              type: 'ChangeCount'
              value: '1'
              cooldown: 'PT10M'
            }
          }
        ]
      }
    ]
  }
}
```

---

## Principle 5: Partition Around Limits

### Concept

Use partitioning to work around service limits for databases, networks, and compute. Distribute load across multiple resources.

### Partitioning Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Horizontal (Sharding)** | Distribute rows across databases | High-volume transactional |
| **Vertical** | Split columns into tables | Different access patterns |
| **Functional** | Separate by business function | Microservices |

### Cosmos DB Partitioning Example

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        COSMOS DB PARTITIONING                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Good Partition Key: /customerId                                                │
│   ────────────────────────────────                                               │
│                                                                                  │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│   │ Partition 1  │  │ Partition 2  │  │ Partition 3  │  │ Partition N  │       │
│   │              │  │              │  │              │  │              │       │
│   │ Customer A   │  │ Customer B   │  │ Customer C   │  │ Customer Z   │       │
│   │ Orders: 50   │  │ Orders: 75   │  │ Orders: 60   │  │ Orders: 45   │       │
│   │              │  │              │  │              │  │              │       │
│   │ Even         │  │ Even         │  │ Even         │  │ Even         │       │
│   │ Distribution │  │ Distribution │  │ Distribution │  │ Distribution │       │
│   └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                                                  │
│   Bad Partition Key: /status (Hot Partition!)                                   │
│   ──────────────────────────────────────────                                    │
│                                                                                  │
│   ┌──────────────────────────────────────┐  ┌───────┐  ┌───────┐               │
│   │           Partition 1                 │  │ P2    │  │ P3    │               │
│   │                                       │  │       │  │       │               │
│   │   status = "active"                   │  │"done" │  │"new"  │               │
│   │   99% of all requests                │  │ 0.5%  │  │ 0.5%  │               │
│   │                                       │  │       │  │       │               │
│   │   HOT PARTITION - Throttling!        │  │       │  │       │               │
│   └──────────────────────────────────────┘  └───────┘  └───────┘               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Principle 6: Design for Operations

### Concept

Build systems with operations in mind from the start. Include logging, monitoring, debugging, and management capabilities.

### Observability Pillars

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          OBSERVABILITY PILLARS                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                │
│   │     METRICS     │  │      LOGS       │  │     TRACES      │                │
│   │                 │  │                 │  │                 │                │
│   │  • CPU, Memory  │  │  • Application  │  │  • Request flow │                │
│   │  • Request rate │  │  • Errors       │  │  • Latency      │                │
│   │  • Error rate   │  │  • Audit        │  │  • Dependencies │                │
│   │  • Latency      │  │  • Security     │  │  • Bottlenecks  │                │
│   │                 │  │                 │  │                 │                │
│   │  Azure Monitor  │  │  Log Analytics  │  │  App Insights   │                │
│   │  Metrics        │  │                 │  │  Distributed    │                │
│   │                 │  │                 │  │  Tracing        │                │
│   └────────┬────────┘  └────────┬────────┘  └────────┬────────┘                │
│            │                    │                    │                          │
│            └────────────────────┼────────────────────┘                          │
│                                 │                                               │
│                    ┌────────────▼────────────┐                                  │
│                    │      ALERTING           │                                  │
│                    │                         │                                  │
│                    │  • Action Groups        │                                  │
│                    │  • Auto-remediation     │                                  │
│                    │  • Escalation           │                                  │
│                    └─────────────────────────┘                                  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Principle 7: Use Managed Services

### Concept

Prefer PaaS and managed services over IaaS. Managed services reduce operational burden, provide built-in scaling, and include security features.

### Service Selection Matrix

| Requirement | IaaS Option | PaaS Option (Preferred) |
|-------------|-------------|-------------------------|
| Web hosting | VMs + IIS/nginx | App Service |
| Databases | SQL on VMs | Azure SQL Database |
| Message queues | RabbitMQ on VMs | Service Bus |
| Caching | Redis on VMs | Azure Cache for Redis |
| Containers | Self-managed K8s | AKS (managed) |

---

## Principle 8: Design for Evolution

### Concept

Design systems that can evolve as requirements change. Embrace loose coupling, well-defined APIs, and versioning from the start.

### Evolution Strategies

```
API VERSIONING
──────────────
/api/v1/customers  ──►  /api/v2/customers
     │                        │
     │ (Keep for             │ (New features)
     │  backward             │
     │  compatibility)       │

DATABASE EVOLUTION
──────────────────
Schema changes via migrations:
├── Add columns (backward compatible)
├── Deprecate columns (keep temporarily)
├── Remove columns (after migration period)
└── Split tables (when needed)

SERVICE DECOMPOSITION
─────────────────────
Monolith ──► Strangler Fig ──► Microservices
    │              │
    │     Extract service
    │     by service
    │
    └── Keep working while evolving
```

---

## Principle Summary Checklist

| Principle | Checklist Item |
|-----------|----------------|
| Self-Healing | ☐ Retry policies configured |
| | ☐ Circuit breakers implemented |
| | ☐ Health endpoints exposed |
| Redundancy | ☐ Multiple instances deployed |
| | ☐ Zone redundancy enabled |
| | ☐ DR plan documented |
| Minimize Coordination | ☐ Async messaging used |
| | ☐ Eventual consistency acceptable |
| | ☐ Services independently deployable |
| Scale Out | ☐ Stateless services |
| | ☐ Auto-scaling configured |
| | ☐ No session affinity |
| Partitioning | ☐ Partition key selected |
| | ☐ Hot partitions avoided |
| | ☐ Limits documented |
| Operations | ☐ Logging implemented |
| | ☐ Metrics collected |
| | ☐ Distributed tracing enabled |
| Managed Services | ☐ PaaS preferred over IaaS |
| | ☐ Built-in security leveraged |
| Evolution | ☐ APIs versioned |
| | ☐ Loose coupling maintained |
| | ☐ Contracts defined |

---

*Continue to [Technology Choices](03-technology-choices.md)*

*Back to [Architecture Styles Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
