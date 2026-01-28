# API Integration Case Studies

*Real-world scenarios for mastering Azure integration services*

---

## Case Study 1: The "We Built Our Own API Gateway" Disaster

### The Setup 🎭

**Company:** QuickShip Logistics
**Industry:** Shipping & Delivery
**Team Size:** 45 developers across 8 teams

### The Problem

QuickShip's CTO had a brilliant idea in 2019: "Why pay for API Gateway when we can build our own? It's just routing!"

Three years later, they had:
- **47,000 lines** of custom gateway code
- **3 full-time developers** just maintaining the gateway
- **Zero documentation** (the original author left)
- **Rate limiting** that worked "most of the time"
- **Authentication** that was "probably secure"

The breaking point came during Black Friday when their custom gateway crashed under load, taking down all 23 microservices simultaneously.

### The Discovery

```
WHAT THE CUSTOM GATEWAY "HANDLED":
───────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   "Simple routing" they said...                                     │
│                                                                      │
│   ┌─────────────────┐                                               │
│   │ Custom Gateway  │                                               │
│   │                 │                                               │
│   │ • Routing      ─┼── 2,000 lines (this part worked)             │
│   │ • Auth         ─┼── 8,000 lines (OAuth2 "close enough")        │
│   │ • Rate limit   ─┼── 5,000 lines (leaked memory)                │
│   │ • Logging      ─┼── 3,000 lines (logged to /dev/null in prod)  │
│   │ • SSL          ─┼── 4,000 lines (expired certs monthly)        │
│   │ • Retry logic  ─┼── 6,000 lines (infinite retry loops)         │
│   │ • Caching      ─┼── 7,000 lines (cache invalidation bugs)      │
│   │ • Transforms   ─┼── 12,000 lines (XML/JSON nightmares)         │
│   │                 │                                               │
│   └─────────────────┘                                               │
│                                                                      │
│   Total: 47,000 lines to replace what APIM does out of the box     │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Migration to Azure API Management

**Phase 1: Assessment (2 weeks)**

```
API INVENTORY DISCOVERY:
────────────────────────

┌──────────────────┬───────────┬─────────────────┬────────────────────┐
│ Service          │ Endpoints │ Daily Calls     │ Auth Method        │
├──────────────────┼───────────┼─────────────────┼────────────────────┤
│ Orders API       │ 34        │ 2.5M            │ API Key + JWT      │
│ Tracking API     │ 12        │ 15M             │ API Key only       │
│ Partners API     │ 28        │ 500K            │ Certificate + JWT  │
│ Inventory API    │ 45        │ 800K            │ IP Whitelist (!)   │
│ Customer API     │ 67        │ 1.2M            │ JWT                │
│ Webhooks         │ 8         │ 3M              │ Shared Secret      │
└──────────────────┴───────────┴─────────────────┴────────────────────┘

Total: 194 endpoints, 23M calls/day, 6 different auth methods
```

**Phase 2: APIM Design**

```bicep
// APIM deployment with proper enterprise setup
resource apiManagement 'Microsoft.ApiManagement/service@2023-05-01-preview' = {
  name: 'quickship-apim'
  location: location
  sku: {
    name: 'Premium'
    capacity: 2
  }
  properties: {
    publisherEmail: 'platform@quickship.com'
    publisherName: 'QuickShip Platform'
    virtualNetworkType: 'Internal'  // Not external!
  }
  identity: {
    type: 'SystemAssigned'
  }
}

// What took them 5,000 lines of buggy code
resource rateLimitPolicy 'Microsoft.ApiManagement/service/policies@2023-05-01-preview' = {
  parent: apiManagement
  name: 'policy'
  properties: {
    value: '''
      <policies>
        <inbound>
          <rate-limit-by-key
            calls="1000"
            renewal-period="60"
            counter-key="@(context.Subscription.Key)" />
          <quota-by-key
            calls="50000"
            renewal-period="86400"
            counter-key="@(context.Subscription.Key)" />
        </inbound>
      </policies>
    '''
  }
}
```

**Phase 3: Strangler Fig Migration**

```
INCREMENTAL MIGRATION:
──────────────────────

Week 1-2: Low-risk APIs
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Traffic ───► Custom GW ───┬──► Orders API (via APIM) ✓           │
│                             ├──► Other APIs (direct)                │
│                             └──► Custom GW handles rest            │
│                                                                      │
│   Result: Orders API saw 40% latency improvement                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

Week 3-4: High-traffic APIs
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Traffic ───► APIM ───┬──► Orders API ✓                           │
│                        ├──► Tracking API ✓ (15M calls handled!)    │
│                        ├──► Customer API ✓                         │
│                        └──► Custom GW (only Partners left)         │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

Week 5-6: Certificate-based auth (the scary one)
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Partners ───► APIM ───► Partners API                             │
│        │                                                            │
│        └── Mutual TLS (what took 4,000 lines now = 10 line policy) │
│                                                                      │
│   <validate-client-certificate                                      │
│     validate-revocation="true"                                      │
│     validate-trust="true"                                           │
│     validate-not-before="true"                                      │
│     validate-not-after="true" />                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Results

| Metric | Before (Custom GW) | After (APIM) | Improvement |
|--------|-------------------|--------------|-------------|
| P99 Latency | 850ms | 95ms | 89% faster |
| Availability | 99.2% | 99.99% | Goodbye 3am pages |
| Developer Hours/month | 320 (gateway maint) | 8 (policy updates) | 97% reduction |
| Security Incidents | 4/year | 0/year | Priceless |
| Black Friday Survival | ❌ | ✅ | Company still exists |

### Cost Analysis

```
TOTAL COST OF OWNERSHIP (ANNUAL):
─────────────────────────────────

Custom Gateway:
├── 3 FTE maintaining gateway: $450,000
├── Infrastructure (over-provisioned): $120,000
├── Incident response (downtime): $80,000
├── Security audit remediation: $50,000
└── Total: $700,000/year

Azure APIM (Premium, 2 units):
├── APIM license: $86,400
├── 0.2 FTE for policy management: $30,000
├── Application Gateway: $8,760
└── Total: $125,160/year

Annual Savings: $574,840 🎉
```

### Key Lessons

1. **"Build vs Buy" should usually be "Buy"** - API Management is a solved problem
2. **Strangler Fig works** - Migrate incrementally, not big-bang
3. **Policies > Code** - Declarative is better than imperative for cross-cutting concerns
4. **Premium tier is worth it** - VNet integration prevented many security headaches

---

## Case Study 2: Event-Driven Order Processing at Scale

### The Setup

**Company:** FreshMart
**Industry:** Online Grocery
**Challenge:** Process 50,000 orders/hour during peak times

### The Original Architecture (The Problem)

```
THE "SYNCHRONOUS EVERYTHING" DESIGN:
────────────────────────────────────

Customer places order:

┌──────────┐    ┌──────────────┐    ┌─────────────┐    ┌──────────────┐
│  Order   │───►│  Inventory   │───►│  Payment    │───►│  Fulfillment │
│  Service │    │  Service     │    │  Service    │    │  Service     │
└──────────┘    └──────────────┘    └─────────────┘    └──────────────┘
                                           │
     Total time: 3-8 seconds               │
     If ANY service slow = customer waits  ▼
     If ANY service down = order fails     😭

PEAK HOUR DISASTER:
├── Inventory service: 2s response (DB locked)
├── Payment service: 3s response (provider slow)
├── Fulfillment service: timeout (queue full)
└── Result: 8-second order = customer leaves

Cart abandonment during peak: 34%
```

### The Solution: Event-Driven Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│              EVENT-DRIVEN ORDER PROCESSING                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────┐     ┌─────────────────────────────────────────────┐  │
│  │  Order   │────►│           Azure Service Bus                  │  │
│  │  API     │     │                                              │  │
│  └──────────┘     │  orders-placed ─────────────────────────────┼──┼─► Inventory
│       │           │       │                                      │  │
│       │           │       ├─► inventory-reserved ────────────────┼──┼─► Payment
│       │           │       │         │                            │  │
│       │           │       │         └─► payment-completed ───────┼──┼─► Fulfillment
│       │           │       │                   │                  │  │
│  200 OK           │       │                   └─► order-ready ───┼──┼─► Notification
│  (in 200ms!)      │       │                                      │  │
│       │           └───────┴──────────────────────────────────────┘  │
│       ▼                                                              │
│  "Order received,                                                   │
│   we'll email you!"        Customer is HAPPY!                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Service Bus Configuration

```bicep
// Service Bus with proper enterprise patterns
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'freshmart-orders'
  location: location
  sku: {
    name: 'Premium'  // Required for VNet, partitioning, geo-DR
    tier: 'Premium'
    capacity: 4      // 4 messaging units for 50K msg/hour peak
  }
  properties: {
    minimumTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'
  }
}

// Order processing topic with smart subscriptions
resource orderTopic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: serviceBusNamespace
  name: 'orders'
  properties: {
    maxSizeInMegabytes: 5120
    defaultMessageTimeToLive: 'P1D'  // 1 day TTL
    enablePartitioning: true         // Scale across partitions
    duplicateDetectionHistoryTimeWindow: 'PT10M'
  }
}

// Each service gets its own subscription with filters
resource inventorySubscription 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: orderTopic
  name: 'inventory-processor'
  properties: {
    maxDeliveryCount: 10
    deadLetteringOnMessageExpiration: true
    lockDuration: 'PT5M'  // 5 min lock for processing
  }
}

// SQL Filter: Only handle orders with items
resource inventoryRule 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: inventorySubscription
  name: 'has-items-filter'
  properties: {
    filterType: 'SqlFilter'
    sqlFilter: {
      sqlExpression: 'itemCount > 0 AND orderType <> \'subscription\''
    }
  }
}
```

### The Saga Pattern for Distributed Transactions

```
ORDER SAGA (BECAUSE DISTRIBUTED TRANSACTIONS ARE A LIE):
────────────────────────────────────────────────────────

Happy Path:
┌──────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  OrderPlaced ──► ReserveInventory ──► ProcessPayment ──► Schedule    │
│                        ✓                    ✓               ✓        │
│                                                                       │
│  Result: Order Completed! 🎉                                         │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘

Sad Path (Payment Fails):
┌──────────────────────────────────────────────────────────────────────┐
│                                                                       │
│  OrderPlaced ──► ReserveInventory ──► ProcessPayment                 │
│                        ✓                    ✗                         │
│                        │                    │                         │
│                        ◄────────────────────┘                         │
│                        │                                              │
│                  ReleaseInventory (compensating action)               │
│                        ✓                                              │
│                        │                                              │
│                  Notify Customer: "Payment failed"                    │
│                                                                       │
└──────────────────────────────────────────────────────────────────────┘
```

### Durable Functions Implementation

```csharp
// Order Saga Orchestrator
[FunctionName("OrderSaga")]
public static async Task<OrderResult> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context,
    ILogger log)
{
    var order = context.GetInput<Order>();
    var result = new OrderResult { OrderId = order.Id };

    try
    {
        // Step 1: Reserve inventory
        var inventoryResult = await context.CallActivityAsync<InventoryResult>(
            "ReserveInventory", order);

        if (!inventoryResult.Success)
        {
            result.Status = "Failed";
            result.Reason = "Items out of stock";
            return result;
        }

        // Step 2: Process payment
        var paymentResult = await context.CallActivityAsync<PaymentResult>(
            "ProcessPayment", order);

        if (!paymentResult.Success)
        {
            // COMPENSATE: Release the inventory we reserved
            await context.CallActivityAsync("ReleaseInventory", inventoryResult);
            result.Status = "Failed";
            result.Reason = "Payment declined";
            return result;
        }

        // Step 3: Schedule fulfillment
        await context.CallActivityAsync("ScheduleFulfillment", new {
            Order = order,
            Payment = paymentResult
        });

        // Step 4: Notify customer (fire and forget via queue)
        await context.CallActivityAsync("QueueNotification", new {
            OrderId = order.Id,
            Type = "OrderConfirmed"
        });

        result.Status = "Completed";
        return result;
    }
    catch (Exception ex)
    {
        // Saga failed - compensate everything
        await context.CallActivityAsync("CompensateOrder", new {
            Order = order,
            FailureReason = ex.Message
        });
        throw;
    }
}
```

### Results

| Metric | Synchronous | Event-Driven | Improvement |
|--------|-------------|--------------|-------------|
| Order Response Time | 3-8 seconds | 200ms | 95% faster |
| Peak Throughput | 12,000/hour | 80,000/hour | 6.7x capacity |
| Cart Abandonment | 34% | 8% | $2.4M/month saved |
| Service Coupling | Tightly coupled | Loosely coupled | Independent deployments |
| Failure Blast Radius | All services | Single service | Happy on-call engineers |

---

## Case Study 3: The B2B Integration Challenge

### The Setup

**Company:** MediSupply Corp
**Industry:** Medical Supply Distribution
**Challenge:** Integrate with 200+ hospital systems using various formats

### The Problem

```
B2B INTEGRATION NIGHTMARE:
──────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Hospital A: Sends orders via HL7 v2.5 over SFTP                   │
│   Hospital B: SOAP web services with WS-Security                    │
│   Hospital C: REST API with OAuth2                                  │
│   Hospital D: Flat files via VAN (1990s EDI)                        │
│   Hospital E: Faxes (yes, actual faxes in 2024)                     │
│   ... 195 more with unique requirements ...                         │
│                                                                      │
│   Current state:                                                     │
│   ├── 47 custom integration scripts                                 │
│   ├── 12 FTP servers (some credentials from 2015)                  │
│   ├── 3 developers who "know where the bodies are buried"          │
│   └── CEO: "Why can't we add new hospitals faster?"                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Solution: Logic Apps + Integration Account

```
ENTERPRISE INTEGRATION ARCHITECTURE:
────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Hospital Systems                Azure Integration                  │
│   ────────────────               ───────────────────                │
│                                                                      │
│   ┌─────────────┐                ┌───────────────────────────────┐ │
│   │ HL7 Systems │──SFTP/HTTPS───►│                               │ │
│   └─────────────┘                │    Azure Logic Apps            │ │
│                                  │                               │ │
│   ┌─────────────┐                │  ┌─────────────────────────┐ │ │
│   │ EDI (X12)   │──AS2 Protocol──┤  │ Integration Account      │ │ │
│   └─────────────┘                │  │                         │ │ │
│                                  │  │ • Trading Partners      │ │ │
│   ┌─────────────┐                │  │ • Schemas (HL7, X12)    │ │ │
│   │ REST APIs   │──HTTPS/OAuth───┤  │ • Maps (transforms)     │ │ │
│   └─────────────┘                │  │ • Certificates          │ │ │
│                                  │  └─────────────────────────┘ │ │
│   ┌─────────────┐                │                               │ │
│   │ SOAP/WCF    │──WS-Security───┤                               │ │
│   └─────────────┘                │                               │ │
│                                  └───────────────┬───────────────┘ │
│                                                  │                  │
│                                                  ▼                  │
│                                  ┌───────────────────────────────┐ │
│                                  │   Canonical Order Format       │ │
│                                  │                               │ │
│                                  │   ─────► Service Bus ─────►   │ │
│                                  │                               │ │
│                                  │   Internal Order Processing   │ │
│                                  └───────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Logic App Implementation

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "triggers": {
      "When_HL7_message_received": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "hl7Message": { "type": "string" }
            }
          }
        }
      }
    },
    "actions": {
      "Parse_HL7_to_XML": {
        "type": "ApiConnection",
        "inputs": {
          "host": { "connection": { "name": "@parameters('$connections')['hl7v2']" } },
          "method": "post",
          "path": "/hl7v2/decode"
        }
      },
      "Transform_to_Canonical": {
        "type": "Xslt",
        "inputs": {
          "content": "@body('Parse_HL7_to_XML')",
          "integrationAccount": {
            "map": { "name": "HL7toCanonicalOrder" }
          }
        }
      },
      "Validate_Order": {
        "type": "XmlValidation",
        "inputs": {
          "content": "@body('Transform_to_Canonical')",
          "integrationAccount": {
            "schema": { "name": "CanonicalOrder" }
          }
        }
      },
      "Send_to_Processing": {
        "type": "ApiConnection",
        "inputs": {
          "host": { "connection": { "name": "@parameters('$connections')['servicebus']" } },
          "method": "post",
          "path": "/orders/messages",
          "body": "@body('Transform_to_Canonical')"
        }
      }
    }
  }
}
```

### Trading Partner Onboarding (Before vs After)

```
BEFORE (Manual Integration):
────────────────────────────
1. Receive partner specs         → 2 weeks
2. Develop custom integration    → 4-6 weeks
3. Test with partner             → 2-3 weeks
4. Deploy to production          → 1 week
5. Debug production issues       → 2 weeks
────────────────────────────────────────────
Total: 11-14 weeks per partner 😱

AFTER (Logic Apps + Integration Account):
────────────────────────────────────────────
1. Create trading partner profile   → 1 day
2. Upload their schemas             → 1 day
3. Create/reuse transform maps      → 3-5 days
4. Configure Logic App instance     → 1 day
5. Test with partner                → 3-5 days
6. Go live                          → 1 day
────────────────────────────────────────────
Total: 10-14 days per partner 🎉

Improvement: 85% faster onboarding
```

### Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Partner Onboarding | 12 weeks | 2 weeks | 83% faster |
| Integration Maintenance | 3 FTE | 0.5 FTE | 83% reduction |
| Failed Messages | 3.2%/day | 0.1%/day | 97% fewer failures |
| New Format Support | 2-4 weeks dev | 2-3 days config | 90% faster |

---

## Case Study 4: Real-Time Analytics Pipeline

### The Setup

**Company:** AdTechFlow
**Industry:** Digital Advertising
**Challenge:** Process 500,000 ad events/second for real-time bidding

### The Architecture

```
REAL-TIME AD ANALYTICS PIPELINE:
────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Ad Networks    Azure Event Grid     Event Hubs      Processing    │
│   ───────────    ─────────────────    ──────────      ──────────    │
│                                                                      │
│   ┌─────────┐                                                        │
│   │ Ad      │    ┌───────────────┐                                  │
│   │ Server 1│───►│               │    ┌──────────────┐              │
│   └─────────┘    │               │    │              │              │
│                  │  Event Grid   │───►│  Event Hubs  │              │
│   ┌─────────┐    │               │    │  (32 units)  │              │
│   │ Ad      │───►│  Fan-out to   │    │              │              │
│   │ Server 2│    │  multiple     │    │  500K/sec    │              │
│   └─────────┘    │  subscribers  │    │  throughput  │              │
│                  │               │    │              │              │
│   ┌─────────┐    └───────────────┘    └──────┬───────┘              │
│   │   ...   │                                │                       │
│   └─────────┘                                ▼                       │
│                                        ┌──────────────┐              │
│                                        │ Stream       │              │
│   ┌─────────────────────────────────►  │ Analytics    │              │
│   │                                    │              │              │
│   │   Real-time aggregations:          │ • CTR calc  │              │
│   │   • 5-sec sliding windows          │ • Fraud det │              │
│   │   • Per-campaign metrics           │ • Bidding   │              │
│   │   • Anomaly detection              └──────┬───────┘              │
│   │                                           │                      │
│   │                              ┌────────────┴────────────┐        │
│   │                              │                         │        │
│   │                              ▼                         ▼        │
│   │                       ┌─────────────┐          ┌─────────────┐  │
│   │                       │ Cosmos DB   │          │ Synapse     │  │
│   │                       │ (hot data)  │          │ (analytics) │  │
│   │                       └─────────────┘          └─────────────┘  │
│   │                              │                                   │
│   └──────────────────────────────┘                                   │
│          Real-time dashboard updates                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Event Hubs Configuration for Scale

```bicep
// Event Hubs for 500K events/second
resource eventHubNamespace 'Microsoft.EventHub/namespaces@2023-01-01-preview' = {
  name: 'adtech-events'
  location: location
  sku: {
    name: 'Premium'
    tier: 'Premium'
    capacity: 8  // 8 PUs = 800MB/s throughput
  }
  properties: {
    isAutoInflateEnabled: true
    maximumThroughputUnits: 20
    kafkaEnabled: true  // Kafka protocol support
    zoneRedundant: true
  }
}

resource adEventsHub 'Microsoft.EventHub/namespaces/eventhubs@2023-01-01-preview' = {
  parent: eventHubNamespace
  name: 'ad-impressions'
  properties: {
    partitionCount: 32  // Max parallelism
    messageRetentionInDays: 7
    captureDescription: {
      enabled: true
      encoding: 'Avro'
      intervalInSeconds: 300
      sizeLimitInBytes: 314572800  // 300MB files
      destination: {
        name: 'EventHubArchive.AzureBlockBlob'
        properties: {
          storageAccountResourceId: storageAccount.id
          blobContainer: 'ad-events-archive'
          archiveNameFormat: '{Namespace}/{EventHub}/{PartitionId}/{Year}/{Month}/{Day}/{Hour}/{Minute}/{Second}'
        }
      }
    }
  }
}
```

### Stream Analytics for Real-Time Processing

```sql
-- Click-through rate calculation (5-second tumbling window)
SELECT
    campaignId,
    COUNT(CASE WHEN eventType = 'impression' THEN 1 END) as impressions,
    COUNT(CASE WHEN eventType = 'click' THEN 1 END) as clicks,
    CAST(COUNT(CASE WHEN eventType = 'click' THEN 1 END) AS float) /
        NULLIF(COUNT(CASE WHEN eventType = 'impression' THEN 1 END), 0) * 100 as ctr,
    System.Timestamp() as windowEnd
INTO [cosmos-realtime]
FROM [ad-impressions] TIMESTAMP BY eventTime
GROUP BY campaignId, TumblingWindow(second, 5)

-- Anomaly detection for fraud
SELECT
    publisherId,
    AVG(CAST(responseTimeMs AS float)) as avgResponseTime,
    COUNT(*) as requestCount,
    AnomalyDetection_SpikeAndDip(CAST(requestCount AS float), 95, 60, 'spikesanddips')
        OVER(PARTITION BY publisherId LIMIT DURATION(minute, 5)) as anomalyScore
INTO [fraud-alerts]
FROM [ad-impressions]
GROUP BY publisherId, HoppingWindow(second, 60, 5)
HAVING anomalyScore.IsAnomaly = 1
```

### Cost Optimization Strategy

```
TIERED PROCESSING FOR COST EFFICIENCY:
──────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Hot Path (< 5 seconds)                                            │
│   ─────────────────────                                             │
│   Event Hubs → Stream Analytics → Cosmos DB                         │
│                                                                      │
│   Data: Last 24 hours, aggregated                                   │
│   Cost: High (but necessary for real-time bidding)                  │
│   Use: Campaign dashboards, bid optimization                        │
│                                                                      │
│   ───────────────────────────────────────────────────────────────   │
│                                                                      │
│   Warm Path (minutes to hours)                                       │
│   ────────────────────────────                                       │
│   Event Hubs Capture → ADLS Gen2 → Databricks                       │
│                                                                      │
│   Data: Last 30 days, detailed                                      │
│   Cost: Medium (batch processing more efficient)                    │
│   Use: Daily reports, trend analysis                                │
│                                                                      │
│   ───────────────────────────────────────────────────────────────   │
│                                                                      │
│   Cold Path (daily batch)                                            │
│   ────────────────────────                                           │
│   ADLS Gen2 → Synapse SQL Pool                                      │
│                                                                      │
│   Data: Historical (years)                                          │
│   Cost: Low (pay per query, compressed storage)                     │
│   Use: Historical analysis, ML training                             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

Cost comparison (monthly):
├── Hot path only (everything real-time):     $45,000
├── Tiered approach:                          $12,000
└── Savings:                                   $33,000/month
```

### Results

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| Processing Latency | 30-60 seconds | <3 seconds | 95% faster |
| Max Throughput | 50K events/sec | 500K events/sec | 10x capacity |
| Bid Optimization Speed | Near real-time | Real-time | $1.2M/year revenue |
| Infrastructure Cost | $75K/month | $12K/month | 84% reduction |
| Data Loss Rate | 0.5% | 0% | Reliable capture |

---

## Summary: Integration Pattern Selection

```
WHEN TO USE WHICH SERVICE:
──────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   "I need to manage and secure APIs"                                │
│   └──► API Management                                               │
│                                                                      │
│   "I need reliable message queuing"                                 │
│   └──► Service Bus Queues                                           │
│                                                                      │
│   "I need pub/sub with filtering"                                   │
│   └──► Service Bus Topics + Event Grid                              │
│                                                                      │
│   "I need high-throughput streaming"                                │
│   └──► Event Hubs (or Kafka on Event Hubs)                          │
│                                                                      │
│   "I need to orchestrate workflows"                                 │
│   └──► Logic Apps (low-code) or Durable Functions (code-first)      │
│                                                                      │
│   "I need B2B integration with EDI/XML"                             │
│   └──► Logic Apps + Integration Account                             │
│                                                                      │
│   "I need event-driven serverless"                                  │
│   └──► Event Grid + Azure Functions                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

*Navigation: [Quick Reference](quick-reference.md) | [README](README.md) | [Main Guide](../README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
