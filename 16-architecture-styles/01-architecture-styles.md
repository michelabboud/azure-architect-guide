# Architecture Styles Deep Dive

## 1. N-Tier Architecture

### Overview

N-Tier (also called multi-tier) architecture divides an application into logical layers and physical tiers. Each layer has specific responsibilities and only communicates with layers directly below it.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            N-TIER ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Internet                                                                       │
│       │                                                                          │
│       ▼                                                                          │
│   ┌───────────────────────────────────────────────────────────────────────┐     │
│   │                    PRESENTATION TIER (Web)                             │     │
│   │  ┌─────────────────────────────────────────────────────────────────┐  │     │
│   │  │  Load Balancer (Azure App Gateway / AWS ALB)                    │  │     │
│   │  └─────────────────────────┬───────────────────────────────────────┘  │     │
│   │                            │                                          │     │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐         │     │
│   │  │  Web VM   │  │  Web VM   │  │  Web VM   │  │  Web VM   │         │     │
│   │  │   (AZ1)   │  │   (AZ2)   │  │   (AZ3)   │  │   (AZ1)   │         │     │
│   │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘         │     │
│   └────────┼──────────────┼──────────────┼──────────────┼────────────────┘     │
│            │              │              │              │                       │
│            └──────────────┼──────────────┼──────────────┘                       │
│                           │              │                                      │
│   ┌───────────────────────┼──────────────┼────────────────────────────────┐     │
│   │                       ▼              ▼       BUSINESS TIER (App)      │     │
│   │  ┌─────────────────────────────────────────────────────────────────┐  │     │
│   │  │  Internal Load Balancer                                         │  │     │
│   │  └─────────────────────────┬───────────────────────────────────────┘  │     │
│   │                            │                                          │     │
│   │  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐         │     │
│   │  │  App VM   │  │  App VM   │  │  App VM   │  │  App VM   │         │     │
│   │  │   (AZ1)   │  │   (AZ2)   │  │   (AZ3)   │  │   (AZ1)   │         │     │
│   │  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘         │     │
│   └────────┼──────────────┼──────────────┼──────────────┼────────────────┘     │
│            │              │              │              │                       │
│            └──────────────┼──────────────┼──────────────┘                       │
│                           │              │                                      │
│   ┌───────────────────────┼──────────────┼────────────────────────────────┐     │
│   │                       ▼              ▼       DATA TIER                │     │
│   │                                                                       │     │
│   │  ┌───────────────────────────────────────────────────────────────┐   │     │
│   │  │  Azure SQL (Primary)    ◄─── Geo-Replication ───►   (Secondary) │   │     │
│   │  │  or SQL Server Always On Availability Group                    │   │     │
│   │  └───────────────────────────────────────────────────────────────┘   │     │
│   │                                                                       │     │
│   └───────────────────────────────────────────────────────────────────────┘     │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### When to Use

| Scenario | Fit | Rationale |
|----------|-----|-----------|
| Migrating existing on-prem apps | Excellent | Minimal architectural changes |
| Traditional business applications | Good | Well-understood pattern |
| Low update frequency | Good | Deployment simplicity |
| Simple scaling requirements | Good | Scale tiers independently |
| Hybrid cloud scenarios | Excellent | Span on-prem and cloud |

### Azure Implementation

```bicep
// N-Tier Infrastructure Example
resource webSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = {
  name: 'web-subnet'
  properties: {
    addressPrefix: '10.0.1.0/24'
    networkSecurityGroup: { id: webNsg.id }
  }
}

resource appSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = {
  name: 'app-subnet'
  properties: {
    addressPrefix: '10.0.2.0/24'
    networkSecurityGroup: { id: appNsg.id }
  }
}

resource dataSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = {
  name: 'data-subnet'
  properties: {
    addressPrefix: '10.0.3.0/24'
    networkSecurityGroup: { id: dataNsg.id }
  }
}
```

### AWS vs Azure Comparison

| Component | AWS | Azure |
|-----------|-----|-------|
| Web Tier LB | Application Load Balancer | Application Gateway |
| App Tier LB | Network Load Balancer | Internal Load Balancer |
| Compute | EC2 Auto Scaling Groups | VM Scale Sets |
| Database | RDS Multi-AZ | Azure SQL with Zone Redundancy |
| Network | VPC Subnets | VNet Subnets |

---

## 2. Web-Queue-Worker Architecture

### Overview

Separates the web front end from background processing using a message queue. The web tier handles HTTP requests, the queue decouples processing, and workers handle resource-intensive tasks.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        WEB-QUEUE-WORKER ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   User Request                                                                   │
│       │                                                                          │
│       ▼                                                                          │
│   ┌───────────────────────────────────────────────────────────────────────┐     │
│   │                         WEB FRONT END                                  │     │
│   │                                                                        │     │
│   │   ┌──────────────────┐                                                │     │
│   │   │   App Service    │  ◄── Handles HTTP requests                    │     │
│   │   │   (Auto-scale)   │  ◄── Validates input                          │     │
│   │   │                  │  ◄── Enqueues work items                       │     │
│   │   └────────┬─────────┘                                                │     │
│   │            │                                                          │     │
│   └────────────┼──────────────────────────────────────────────────────────┘     │
│                │                                                                 │
│                │ Enqueue message                                                 │
│                ▼                                                                 │
│   ┌───────────────────────────────────────────────────────────────────────┐     │
│   │                         MESSAGE QUEUE                                  │     │
│   │                                                                        │     │
│   │   ┌──────────────────────────────────────────────────────────────┐   │     │
│   │   │                    Azure Service Bus                          │   │     │
│   │   │                    ═══════════════════                       │   │     │
│   │   │                    ═══════════════════                       │   │     │
│   │   │                    ═══════════════════                       │   │     │
│   │   │                                                              │   │     │
│   │   │   ┌─────────────────────────────────────────────────────┐   │   │     │
│   │   │   │              Dead Letter Queue                       │   │   │     │
│   │   │   │              (Failed messages)                       │   │   │     │
│   │   │   └─────────────────────────────────────────────────────┘   │   │     │
│   │   └──────────────────────────────────────────────────────────────┘   │     │
│   │                                                                        │     │
│   └────────────────────────────────┬──────────────────────────────────────┘     │
│                                    │                                             │
│                                    │ Dequeue and process                         │
│                                    ▼                                             │
│   ┌───────────────────────────────────────────────────────────────────────┐     │
│   │                         WORKER TIER                                    │     │
│   │                                                                        │     │
│   │   ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐         │     │
│   │   │  Azure    │  │  Azure    │  │  Azure    │  │  Azure    │         │     │
│   │   │ Function  │  │ Function  │  │ Function  │  │ Function  │         │     │
│   │   │ (KEDA)    │  │ (KEDA)    │  │ (KEDA)    │  │ (KEDA)    │         │     │
│   │   └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘         │     │
│   │         │              │              │              │                │     │
│   └─────────┼──────────────┼──────────────┼──────────────┼────────────────┘     │
│             │              │              │              │                       │
│             └──────────────┴──────────────┴──────────────┘                       │
│                                    │                                             │
│                                    ▼                                             │
│   ┌───────────────────────────────────────────────────────────────────────┐     │
│   │                         DATA STORE                                     │     │
│   │   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐            │     │
│   │   │  Azure SQL    │  │  Blob Storage │  │  Cosmos DB    │            │     │
│   │   │  (Results)    │  │  (Files)      │  │  (Documents)  │            │     │
│   │   └───────────────┘  └───────────────┘  └───────────────┘            │     │
│   └───────────────────────────────────────────────────────────────────────┘     │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Use Cases

| Scenario | Example |
|----------|---------|
| Image/Video Processing | Upload → Queue → Transcode → Store |
| Report Generation | Request → Queue → Generate PDF → Email |
| Order Processing | Submit → Queue → Validate → Fulfill |
| Data Import | Upload CSV → Queue → Parse → Insert |
| Email Campaigns | Trigger → Queue → Send batches |

### Azure Implementation

```csharp
// Web API - Enqueue work
[HttpPost("orders")]
public async Task<IActionResult> CreateOrder([FromBody] Order order)
{
    // Validate
    if (!ModelState.IsValid) return BadRequest();

    // Save to database
    await _dbContext.Orders.AddAsync(order);
    await _dbContext.SaveChangesAsync();

    // Enqueue for processing
    var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
    {
        MessageId = order.Id.ToString(),
        Subject = "ProcessOrder"
    };
    await _sender.SendMessageAsync(message);

    return Accepted(new { orderId = order.Id });
}

// Worker - Process queue
[FunctionName("ProcessOrder")]
public async Task Run(
    [ServiceBusTrigger("orders", Connection = "ServiceBus")] Order order,
    ILogger log)
{
    log.LogInformation($"Processing order {order.Id}");

    // Long-running processing
    await _paymentService.ProcessPaymentAsync(order);
    await _inventoryService.ReserveStockAsync(order);
    await _notificationService.SendConfirmationAsync(order);

    await _dbContext.Orders
        .Where(o => o.Id == order.Id)
        .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, "Completed"));
}
```

---

## 3. Microservices Architecture

### Overview

Decomposes applications into small, autonomous services that implement single business capabilities. Each service owns its data and communicates through well-defined APIs.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        MICROSERVICES ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Clients                                                                        │
│   ┌──────┐  ┌──────┐  ┌──────┐                                                  │
│   │Mobile│  │ Web  │  │ IoT  │                                                  │
│   └──┬───┘  └──┬───┘  └──┬───┘                                                  │
│      │         │         │                                                       │
│      └─────────┼─────────┘                                                       │
│                │                                                                 │
│   ┌────────────▼────────────────────────────────────────────────────────────┐   │
│   │                      API GATEWAY (Azure APIM)                            │   │
│   │   • Authentication & Authorization                                       │   │
│   │   • Rate Limiting & Throttling                                          │   │
│   │   • Request Routing                                                      │   │
│   │   • Protocol Translation                                                 │   │
│   └────────────┬────────────────────────────────────────────────────────────┘   │
│                │                                                                 │
│   ┌────────────▼────────────────────────────────────────────────────────────┐   │
│   │                    SERVICE MESH (Istio / Dapr)                           │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                │                                                                 │
│   ┌────────────┼────────────────────────────────────────────────────────────┐   │
│   │            │              KUBERNETES CLUSTER (AKS)                       │   │
│   │            │                                                             │   │
│   │  ┌─────────▼─────────┐  ┌───────────────────┐  ┌───────────────────┐   │   │
│   │  │   Order Service   │  │  Product Service  │  │ Customer Service  │   │   │
│   │  │                   │  │                   │  │                   │   │   │
│   │  │  ┌─────────────┐  │  │  ┌─────────────┐  │  │  ┌─────────────┐  │   │   │
│   │  │  │   Pods      │  │  │  │   Pods      │  │  │  │   Pods      │  │   │   │
│   │  │  │  (Replicas) │  │  │  │  (Replicas) │  │  │  │  (Replicas) │  │   │   │
│   │  │  └─────────────┘  │  │  └─────────────┘  │  │  └─────────────┘  │   │   │
│   │  │        │          │  │        │          │  │        │          │   │   │
│   │  │  ┌─────▼─────┐    │  │  ┌─────▼─────┐    │  │  ┌─────▼─────┐    │   │   │
│   │  │  │ Orders DB │    │  │  │Products DB│    │  │  │Customers  │    │   │   │
│   │  │  │(Cosmos DB)│    │  │  │(Azure SQL)│    │  │  │  DB       │    │   │   │
│   │  │  └───────────┘    │  │  └───────────┘    │  │  │(Cosmos DB)│    │   │   │
│   │  └───────────────────┘  └───────────────────┘  │  └───────────┘    │   │   │
│   │                                                 └───────────────────┘   │   │
│   │                                                                         │   │
│   │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐   │   │
│   │  │ Payment Service   │  │Inventory Service  │  │Notification Svc   │   │   │
│   │  │   (External)      │  │                   │  │                   │   │   │
│   │  └───────────────────┘  └───────────────────┘  └───────────────────┘   │   │
│   │                                                                         │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                    ASYNC COMMUNICATION                                   │   │
│   │  ┌───────────────────────────────────────────────────────────────────┐  │   │
│   │  │              Azure Service Bus / Event Grid                        │  │   │
│   │  │   OrderCreated ──► InventoryReserved ──► PaymentProcessed         │  │   │
│   │  └───────────────────────────────────────────────────────────────────┘  │   │
│   └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Characteristic | Description |
|----------------|-------------|
| **Bounded Context** | Each service owns a specific business domain |
| **Data Ownership** | Each service has its own database |
| **Independent Deployment** | Services deploy independently |
| **Technology Diversity** | Can use different languages/frameworks |
| **Resilience** | Failures are isolated to individual services |

### Anti-Patterns to Avoid

| Anti-Pattern | Problem | Solution |
|--------------|---------|----------|
| Distributed Monolith | Services tightly coupled | True bounded contexts |
| Shared Database | Data coupling | Database per service |
| Synchronous Chains | Cascading failures | Async communication |
| Mega Service | Service too large | Decompose further |

---

## 4. Event-Driven Architecture

### Overview

Uses events as the primary mechanism for communication. Producers emit events without knowledge of consumers. Enables loose coupling and real-time processing.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        EVENT-DRIVEN ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   EVENT PRODUCERS                                                                │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│   │ IoT Hub  │  │  App     │  │ Storage  │  │ Service  │  │ Custom   │         │
│   │ Devices  │  │ Service  │  │ Account  │  │   Bus    │  │  App     │         │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘         │
│        │             │             │             │             │                │
│        │ Events      │ Events      │ Events      │ Events      │ Events        │
│        └─────────────┴─────────────┴──────┬──────┴─────────────┘                │
│                                           │                                      │
│   ┌───────────────────────────────────────▼─────────────────────────────────┐   │
│   │                        AZURE EVENT GRID                                  │   │
│   │                                                                          │   │
│   │   ┌──────────────────────────────────────────────────────────────────┐  │   │
│   │   │                      Event Topics                                 │  │   │
│   │   │   ┌────────────┐  ┌────────────┐  ┌────────────┐                │  │   │
│   │   │   │  Orders    │  │  Inventory │  │  Customers │                │  │   │
│   │   │   │  Topic     │  │   Topic    │  │   Topic    │                │  │   │
│   │   │   └────────────┘  └────────────┘  └────────────┘                │  │   │
│   │   └──────────────────────────────────────────────────────────────────┘  │   │
│   │                                                                          │   │
│   │   Event Filtering:                                                       │   │
│   │   • Event Type: "Order.Created", "Order.Shipped"                        │   │
│   │   • Subject: "/orders/region/west/*"                                    │   │
│   │   • Advanced: data.amount > 1000                                        │   │
│   │                                                                          │   │
│   └───────────────────────────────────┬──────────────────────────────────────┘   │
│                                       │                                          │
│   ┌───────────────┬───────────────────┼───────────────────┬───────────────┐     │
│   │               │                   │                   │               │     │
│   ▼               ▼                   ▼                   ▼               ▼     │
│ ┌─────────┐  ┌─────────┐        ┌─────────┐        ┌─────────┐  ┌─────────┐   │
│ │ Azure   │  │ Logic   │        │ Event   │        │ Service │  │ Webhook │   │
│ │Function │  │  App    │        │  Hub    │        │  Bus    │  │  URL    │   │
│ │         │  │         │        │(Stream) │        │ Queue   │  │         │   │
│ └─────────┘  └─────────┘        └─────────┘        └─────────┘  └─────────┘   │
│                                                                                │
│   EVENT CONSUMERS                                                              │
│                                                                                │
└────────────────────────────────────────────────────────────────────────────────┘

    Event Flow Patterns:
    ════════════════════

    1. Simple Event Processing
       Producer ──► Event Grid ──► Function ──► Action

    2. Event Streaming
       Producer ──► Event Hubs ──► Stream Analytics ──► Output

    3. Complex Event Processing
       Multiple Events ──► Event Grid ──► Logic App ──► Orchestrated Actions
```

### Event vs Message

| Aspect | Events | Messages |
|--------|--------|----------|
| **Purpose** | Notification of state change | Command to do something |
| **Coupling** | Very loose | Loose |
| **Expectation** | Fire and forget | Acknowledgment expected |
| **Example** | "OrderCreated" | "ProcessPayment" |
| **Azure Service** | Event Grid | Service Bus |

### Azure Implementation

```csharp
// Event Producer
public class OrderService
{
    private readonly EventGridPublisherClient _eventGridClient;

    public async Task CreateOrderAsync(Order order)
    {
        // Save order
        await _dbContext.Orders.AddAsync(order);
        await _dbContext.SaveChangesAsync();

        // Publish event
        var cloudEvent = new CloudEvent(
            source: "/orders/service",
            type: "Order.Created",
            data: new OrderCreatedEvent
            {
                OrderId = order.Id,
                CustomerId = order.CustomerId,
                TotalAmount = order.TotalAmount,
                CreatedAt = DateTime.UtcNow
            });

        await _eventGridClient.SendEventAsync(cloudEvent);
    }
}

// Event Consumer (Azure Function)
[FunctionName("HandleOrderCreated")]
public async Task Run(
    [EventGridTrigger] CloudEvent cloudEvent,
    ILogger log)
{
    var orderEvent = cloudEvent.Data.ToObjectFromJson<OrderCreatedEvent>();

    log.LogInformation($"Order {orderEvent.OrderId} created");

    // React to event
    await _inventoryService.ReserveStockAsync(orderEvent.OrderId);
    await _notificationService.SendOrderConfirmationAsync(orderEvent);
}
```

---

## 5. Big Data Architecture

### Overview

Handles massive volumes of data through parallel processing. Combines batch processing for historical analysis with stream processing for real-time insights.

### Lambda Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          BIG DATA ARCHITECTURE (Lambda)                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   DATA SOURCES                                                                   │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                        │
│   │   IoT    │  │   Logs   │  │   Apps   │  │ External │                        │
│   │ Devices  │  │  Streams │  │   Data   │  │  APIs    │                        │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘                        │
│        │             │             │             │                               │
│        └─────────────┴─────────────┴─────────────┘                               │
│                          │                                                       │
│   ┌──────────────────────▼──────────────────────┐                               │
│   │              INGESTION LAYER                 │                               │
│   │         Azure Event Hubs / IoT Hub          │                               │
│   └──────────────────────┬──────────────────────┘                               │
│                          │                                                       │
│          ┌───────────────┼───────────────┐                                      │
│          │               │               │                                      │
│          ▼               ▼               ▼                                      │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                              │
│   │   BATCH     │ │   SPEED     │ │  DATA LAKE  │                              │
│   │   LAYER     │ │   LAYER     │ │  STORAGE    │                              │
│   │             │ │             │ │             │                              │
│   │ ┌─────────┐ │ │ ┌─────────┐ │ │ ┌─────────┐ │                              │
│   │ │Synapse  │ │ │ │ Stream  │ │ │ │  ADLS   │ │                              │
│   │ │Spark    │ │ │ │Analytics│ │ │ │  Gen2   │ │                              │
│   │ │Pools    │ │ │ │         │ │ │ │         │ │                              │
│   │ └────┬────┘ │ │ └────┬────┘ │ │ └─────────┘ │                              │
│   │      │      │ │      │      │ │             │                              │
│   │ Historical │ │ Real-time   │ │  Raw Data   │                              │
│   │ Processing │ │ Processing  │ │  Archive    │                              │
│   └──────┬──────┘ └──────┬──────┘ └─────────────┘                              │
│          │               │                                                      │
│          └───────┬───────┘                                                      │
│                  │                                                              │
│   ┌──────────────▼──────────────┐                                              │
│   │        SERVING LAYER         │                                              │
│   │  ┌───────────────────────┐  │                                              │
│   │  │  Synapse Serverless   │  │  ◄── Ad-hoc queries                         │
│   │  │  Dedicated SQL Pool   │  │  ◄── BI workloads                           │
│   │  │  Azure SQL            │  │  ◄── Operational queries                    │
│   │  └───────────────────────┘  │                                              │
│   └──────────────┬──────────────┘                                              │
│                  │                                                              │
│   ┌──────────────▼──────────────┐                                              │
│   │     PRESENTATION LAYER       │                                              │
│   │  ┌─────────┐  ┌─────────┐   │                                              │
│   │  │Power BI │  │ Grafana │   │                                              │
│   │  │Dashboard│  │Dashboard│   │                                              │
│   │  └─────────┘  └─────────┘   │                                              │
│   └─────────────────────────────┘                                              │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 6. Big Compute Architecture

### Overview

Designed for computationally intensive workloads requiring massive parallel processing. Suited for simulations, rendering, and scientific computing.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          BIG COMPUTE ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   JOB SUBMISSION                                                                 │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                     │  │
│   │   │  Web Portal │  │    API      │  │   CLI       │                     │  │
│   │   │             │  │             │  │             │                     │  │
│   │   └──────┬──────┘  └──────┬──────┘  └──────┬──────┘                     │  │
│   │          │                │                │                             │  │
│   └──────────┴────────────────┴────────────────┴─────────────────────────────┘  │
│                               │                                                  │
│   ┌───────────────────────────▼──────────────────────────────────────────────┐  │
│   │                      JOB SCHEDULER                                        │  │
│   │                                                                           │  │
│   │   ┌─────────────────────────────────────────────────────────────────┐    │  │
│   │   │                    Azure Batch                                   │    │  │
│   │   │   • Job queuing and scheduling                                  │    │  │
│   │   │   • Automatic VM provisioning                                   │    │  │
│   │   │   • Task distribution                                           │    │  │
│   │   │   • Monitoring and retry                                        │    │  │
│   │   └─────────────────────────────────────────────────────────────────┘    │  │
│   │                                                                           │  │
│   └───────────────────────────┬──────────────────────────────────────────────┘  │
│                               │                                                  │
│   ┌───────────────────────────▼──────────────────────────────────────────────┐  │
│   │                      COMPUTE POOL                                         │  │
│   │                                                                           │  │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐       │  │
│   │   │  VM     │  │  VM     │  │  VM     │  │  VM     │  │  VM     │       │  │
│   │   │ (Spot)  │  │ (Spot)  │  │ (Spot)  │  │ (Spot)  │  │ (Spot)  │       │  │
│   │   │  HBv3   │  │  HBv3   │  │  HBv3   │  │  HBv3   │  │  HBv3   │       │  │
│   │   └─────────┘  └─────────┘  └─────────┘  └─────────┘  └─────────┘       │  │
│   │                                                                           │  │
│   │   Scale: 0 ──────────────────────────────────────────────► 10,000 nodes │  │
│   │                                                                           │  │
│   │   VM Types:                                                               │  │
│   │   • HB-series: HPC memory bandwidth                                      │  │
│   │   • HC-series: Compute intensive                                         │  │
│   │   • N-series: GPU (NVIDIA)                                               │  │
│   │   • Spot VMs: 90% discount for interruptible                            │  │
│   │                                                                           │  │
│   └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   ┌───────────────────────────────────────────────────────────────────────────┐  │
│   │                      DATA STORAGE                                          │  │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                       │  │
│   │   │  Blob       │  │  Azure      │  │  Azure      │                       │  │
│   │   │  Storage    │  │  Files      │  │  NetApp     │                       │  │
│   │   │  (Input)    │  │  (Shared)   │  │  (HPC)      │                       │  │
│   │   └─────────────┘  └─────────────┘  └─────────────┘                       │  │
│   └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Use Cases

| Industry | Application |
|----------|-------------|
| Financial Services | Risk modeling, Monte Carlo simulations |
| Life Sciences | Drug discovery, genomic sequencing |
| Engineering | CFD, FEA, structural analysis |
| Media | Rendering, transcoding |
| Oil & Gas | Seismic processing, reservoir simulation |

---

*Continue to [Design Principles](02-design-principles.md)*

*Back to [Architecture Styles Overview](README.md)*
