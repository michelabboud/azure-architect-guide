# Case Study 1: Global E-commerce Platform

## Executive Summary

**Company**: RetailGlobal Inc.
**Industry**: E-commerce/Retail
**Challenge**: Migrate from AWS to Azure while scaling for global expansion
**Outcome**: 40% reduction in latency, 99.99% availability, $2.3M annual savings

### The Business Context

RetailGlobal operates in 12 countries with 50M monthly active users. Their AWS infrastructure was hitting scaling limits during flash sales and Black Friday events. The decision to move to Azure was driven by:

1. Existing Microsoft 365 and Dynamics 365 investments
2. Better Azure pricing through EA agreement
3. Superior Azure presence in target expansion markets (Middle East, Asia)

## Architecture Overview

```
                                    ┌─────────────────────────────────────┐
                                    │         Azure Front Door            │
                                    │    (Global Load Balancing + WAF)    │
                                    └──────────────┬──────────────────────┘
                                                   │
                    ┌──────────────────────────────┼──────────────────────────────┐
                    │                              │                              │
           ┌────────▼────────┐           ┌────────▼────────┐           ┌────────▼────────┐
           │   West Europe   │           │    East US      │           │   Southeast     │
           │     Region      │           │     Region      │           │   Asia Region   │
           └────────┬────────┘           └────────┬────────┘           └────────┬────────┘
                    │                              │                              │
    ┌───────────────┼───────────────┐              │              ┌───────────────┼───────────────┐
    │               │               │              │              │               │               │
┌───▼───┐      ┌───▼───┐      ┌───▼───┐      ┌───▼───┐      ┌───▼───┐      ┌───▼───┐      ┌───▼───┐
│  AKS  │      │ Redis │      │ Blob  │      │  AKS  │      │  AKS  │      │ Redis │      │ Blob  │
│Cluster│      │ Cache │      │Storage│      │Cluster│      │Cluster│      │ Cache │      │Storage│
└───┬───┘      └───────┘      └───────┘      └───┬───┘      └───┬───┘      └───────┘      └───────┘
    │                                             │              │
    └─────────────────────────────────────────────┼──────────────┘
                                                  │
                                    ┌─────────────▼─────────────┐
                                    │        Cosmos DB          │
                                    │   (Multi-Region Write)    │
                                    │  ┌─────┐ ┌─────┐ ┌─────┐  │
                                    │  │ EU  │ │ US  │ │ Asia│  │
                                    │  └─────┘ └─────┘ └─────┘  │
                                    └───────────────────────────┘
```

## AWS to Azure Service Mapping

| Component | AWS (Before) | Azure (After) | Migration Rationale |
|-----------|-------------|---------------|---------------------|
| Global LB | CloudFront + Route 53 | Azure Front Door | Native WAF, better PoPs in target markets |
| Container Orchestration | EKS | AKS | Integrated Azure AD, lower node costs |
| Product Catalog | DynamoDB | Cosmos DB | Multi-region writes, more consistency options |
| Session/Cart | ElastiCache Redis | Azure Cache for Redis | Enterprise tier with geo-replication |
| Product Images | S3 + CloudFront | Blob Storage + CDN | Integrated with Front Door caching |
| Search | OpenSearch | Azure Cognitive Search | AI-powered semantic search |
| Payments | Lambda + SQS | Azure Functions + Service Bus | Durable Functions for sagas |
| Analytics | Kinesis + Redshift | Event Hubs + Synapse | Real-time inventory insights |

## Key Architectural Decisions

### Decision 1: Database Selection for Product Catalog

**The Debate:**

| Option | Pros | Cons |
|--------|------|------|
| **Azure SQL** | Familiar relational model, strong consistency | Scaling writes globally is complex |
| **Cosmos DB** | Multi-region writes, automatic scaling | Eventual consistency, higher cost |
| **PostgreSQL Flexible** | Open source, JSONB support | Manual geo-replication setup |

**Decision: Cosmos DB with Session Consistency**

**Rationale:**
1. Product catalog needs low-latency reads globally
2. Inventory updates can tolerate eventual consistency (bounded staleness)
3. Flash sales require automatic scaling without capacity planning
4. Multi-region writes eliminate cross-region latency for catalog updates

**Configuration:**
```json
{
  "consistencyPolicy": {
    "defaultConsistencyLevel": "Session",
    "maxIntervalInSeconds": 5,
    "maxStalenessPrefix": 100
  },
  "locations": [
    { "locationName": "West Europe", "failoverPriority": 0 },
    { "locationName": "East US", "failoverPriority": 1 },
    { "locationName": "Southeast Asia", "failoverPriority": 2 }
  ],
  "enableMultipleWriteLocations": true
}
```

### Decision 2: AKS vs App Service for Microservices

**The Debate:**

| Criteria | AKS | App Service |
|----------|-----|-------------|
| Team Skills | Strong Kubernetes experience | Limited container expertise |
| Control | Full control over runtime | Managed, less customization |
| Scaling | Complex but powerful | Simple but limited |
| Cost | Lower at scale | Higher per-unit but simpler |
| Migration | Direct port from EKS | Refactoring required |

**Decision: AKS with KEDA for auto-scaling**

**Rationale:**
1. Team already proficient with Kubernetes from EKS
2. Direct migration path with minimal refactoring
3. KEDA enables event-driven scaling for flash sales
4. Cost optimization through spot node pools for background jobs

**KEDA Configuration for Flash Sales:**
```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
spec:
  scaleTargetRef:
    name: order-processor
  minReplicaCount: 5
  maxReplicaCount: 500
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      messageCount: "50"
      connectionFromEnv: SERVICEBUS_CONNECTION
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: http_requests_total
      threshold: "100"
      query: sum(rate(http_requests_total{service="checkout"}[1m]))
```

### Decision 3: Cart Persistence Strategy

**The Debate:**

| Option | Latency | Durability | Cost | Complexity |
|--------|---------|------------|------|------------|
| Redis only | <1ms | Low (volatile) | $ | Low |
| Cosmos DB only | ~10ms | High | $$$ | Low |
| Redis + async Cosmos | <1ms | High | $$ | Medium |
| Redis + periodic backup | <1ms | Medium | $ | Medium |

**Decision: Redis Cache with Cosmos DB async persistence**

**Rationale:**
1. Cart operations need sub-millisecond latency
2. Losing carts has significant revenue impact
3. Async write to Cosmos provides durability without latency hit
4. 30-minute TTL in Redis, permanent in Cosmos

**Implementation Pattern:**
```csharp
public class CartService
{
    private readonly IDatabase _redis;
    private readonly Container _cosmosContainer;
    private readonly IBackgroundJobClient _hangfire;

    public async Task<Cart> AddItemAsync(string userId, CartItem item)
    {
        // Read from Redis (or Cosmos if cache miss)
        var cart = await GetCartAsync(userId);
        cart.AddItem(item);

        // Write to Redis immediately
        await _redis.StringSetAsync(
            $"cart:{userId}",
            JsonSerializer.Serialize(cart),
            TimeSpan.FromMinutes(30));

        // Queue async write to Cosmos
        _hangfire.Enqueue(() => PersistCartAsync(userId, cart));

        return cart;
    }

    public async Task PersistCartAsync(string userId, Cart cart)
    {
        await _cosmosContainer.UpsertItemAsync(cart, new PartitionKey(userId));
    }
}
```

## Technical Deep Dive

### Flash Sale Architecture

The platform handles 10x normal traffic during flash sales. Here's the architecture:

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                    Flash Sale Flow                       │
                    └─────────────────────────────────────────────────────────┘
                                              │
                    ┌─────────────────────────▼─────────────────────────┐
                    │              Azure Front Door                      │
                    │  • Rate limiting: 1000 req/s per IP               │
                    │  • WAF rules: Bot detection enabled               │
                    │  • Geo-filter: Block non-target regions           │
                    └─────────────────────────┬─────────────────────────┘
                                              │
                    ┌─────────────────────────▼─────────────────────────┐
                    │              Inventory Service (AKS)               │
                    │  • Pre-warmed: 50 pods before sale                │
                    │  • KEDA scaling: 0-500 pods                       │
                    │  • Circuit breaker: Polly (5 failures = open)     │
                    └─────────────────────────┬─────────────────────────┘
                                              │
            ┌─────────────────────────────────┼─────────────────────────────────┐
            │                                 │                                 │
┌───────────▼───────────┐       ┌────────────▼────────────┐       ┌────────────▼────────────┐
│     Redis Cache       │       │     Cosmos DB           │       │    Service Bus          │
│  • Stock counters     │       │  • Order documents      │       │  • Order queue          │
│  • Distributed locks  │       │  • Strong consistency   │       │  • Dead letter queue    │
│  • Lua scripts        │       │  • Request Units: 100K  │       │  • 10K msg/s capacity   │
└───────────────────────┘       └─────────────────────────┘       └─────────────────────────┘
```

**Stock Reservation with Redis Lua Script:**
```lua
-- reserve_stock.lua
local stock_key = KEYS[1]
local reservation_key = KEYS[2]
local user_id = ARGV[1]
local quantity = tonumber(ARGV[2])
local ttl = tonumber(ARGV[3])

-- Check available stock
local current_stock = tonumber(redis.call('GET', stock_key) or 0)

if current_stock >= quantity then
    -- Decrement stock
    redis.call('DECRBY', stock_key, quantity)

    -- Create reservation with TTL
    local reservation_id = redis.call('INCR', 'reservation:counter')
    local reservation_data = cjson.encode({
        user_id = user_id,
        quantity = quantity,
        timestamp = redis.call('TIME')[1]
    })
    redis.call('HSET', reservation_key, reservation_id, reservation_data)
    redis.call('EXPIRE', reservation_key, ttl)

    return reservation_id
else
    return -1  -- Insufficient stock
end
```

### Real-time Inventory Sync

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│   Order     │────▶│  Event Hub  │────▶│   Stream    │────▶│   Cosmos    │
│  Service    │     │  (Capture)  │     │  Analytics  │     │ (Inventory) │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                           │                   │
                           │                   │
                           ▼                   ▼
                    ┌─────────────┐     ┌─────────────┐
                    │   Synapse   │     │   Signal R  │
                    │ (Analytics) │     │ (Real-time) │
                    └─────────────┘     └─────────────┘
```

**Stream Analytics Query:**
```sql
-- Real-time inventory aggregation
WITH OrderEvents AS (
    SELECT
        productId,
        SUM(quantity) as totalOrdered,
        COUNT(*) as orderCount,
        System.Timestamp() as windowEnd
    FROM orders TIMESTAMP BY eventTime
    GROUP BY
        productId,
        TumblingWindow(second, 5)
)

SELECT
    o.productId,
    o.totalOrdered,
    o.orderCount,
    o.windowEnd,
    i.currentStock - o.totalOrdered as projectedStock
INTO inventoryOutput
FROM OrderEvents o
JOIN inventoryReference i ON o.productId = i.productId
WHERE i.currentStock - o.totalOrdered < i.reorderThreshold
```

## Cost Analysis

### Monthly Cost Breakdown (Production)

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| Azure Front Door | Premium, 100TB egress | $12,500 |
| AKS (3 regions) | 30 D8s_v4 nodes + 20 spot | $28,000 |
| Cosmos DB | 100K RU/s multi-region | $45,000 |
| Azure Cache for Redis | P4 Enterprise (3 regions) | $18,000 |
| Blob Storage | 50TB hot, 200TB cool | $6,000 |
| Event Hubs | Dedicated, 4 CU | $8,500 |
| Service Bus | Premium, 8 MU | $4,800 |
| Azure Functions | Consumption (burst) | $2,200 |
| **Total** | | **$125,000** |

### Cost Optimization Strategies

1. **Reserved Capacity**: 3-year Cosmos DB reserved = 65% savings
2. **Spot Nodes**: Background processing on 80% cheaper spot VMs
3. **Auto-pause**: Development environments pause after hours
4. **Storage Tiering**: Automatic lifecycle policies for older images

**AWS vs Azure Cost Comparison:**
| Category | AWS (Previous) | Azure (Current) | Savings |
|----------|---------------|-----------------|---------|
| Compute | $52,000 | $28,000 | 46% |
| Database | $68,000 | $45,000 | 34% |
| Network | $35,000 | $25,000 | 29% |
| Total | $155,000 | $125,000 | **$30,000/mo** |

## Lessons Learned

### What Worked Well

1. **Cosmos DB Multi-Region Writes**: Eliminated the need for conflict resolution logic that existed with DynamoDB Global Tables
2. **Azure Front Door**: Simpler configuration than CloudFront + Route 53 combination
3. **AKS + KEDA**: Seamless migration from EKS, better scaling for event-driven workloads

### Challenges Encountered

1. **Cosmos DB Partition Design**: Initial partition key choice caused hot partitions during flash sales; redesigned using composite keys
2. **Redis Enterprise Geo-Replication**: Active-active replication had learning curve; required careful conflict resolution configuration
3. **Service Principal Management**: Initially used too many service principals; consolidated with managed identities

### Recommendations

1. **Start with Cosmos DB capacity mode**: Use serverless for dev, autoscale for prod
2. **Implement feature flags**: Used Azure App Configuration for gradual rollout
3. **Plan for observability from day 1**: Application Insights + Azure Monitor + Log Analytics

## Discussion Questions

1. **Why was session consistency chosen over strong consistency for Cosmos DB?**

2. **What would change if the platform needed to support 100M users instead of 50M?**

3. **How would you handle a scenario where Redis goes down during a flash sale?**

4. **What are the trade-offs of using Service Bus vs Event Grid for order processing?**

5. **How would you implement a circuit breaker for the payment service?**

---

*Continue to [Case Study 2: Enterprise Data Platform](02-enterprise-data-platform.md)*

*Back to [Chapter 08: Case Studies](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
