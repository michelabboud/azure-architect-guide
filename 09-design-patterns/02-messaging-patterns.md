# Messaging Patterns

## Queue-Based Load Leveling

### Problem

Traffic spikes can overwhelm backend services, causing failures or degraded performance.

### Solution

```
QUEUE-BASED LOAD LEVELING:
──────────────────────────

WITHOUT QUEUE (Direct calls):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Clients ═══▶ Service (capacity: 100 req/s)                          │
│                                                                        │
│  Normal:  50 req/s  → Service handles fine                           │
│  Spike:   500 req/s → Service overwhelmed, requests fail             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

WITH QUEUE (Decoupled):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Clients ═══▶ QUEUE ═══▶ Service (processes at 100 req/s)            │
│                                                                        │
│  Normal:  50 req/s  → Queue nearly empty, fast processing            │
│  Spike:   500 req/s → Queue absorbs spike, service processes steady  │
│                                                                        │
│  Trade-off: Eventual consistency (not real-time processing)          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Producer: Send to queue
public class OrderSubmissionService
{
    private readonly ServiceBusSender _sender;

    public OrderSubmissionService(ServiceBusClient client)
    {
        _sender = client.CreateSender("orders");
    }

    public async Task<string> SubmitOrderAsync(Order order)
    {
        var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
        {
            MessageId = order.Id,
            ContentType = "application/json",
            SessionId = order.CustomerId,  // Ensures ordering per customer
            ApplicationProperties =
            {
                ["priority"] = order.IsPremium ? "high" : "normal"
            }
        };

        await _sender.SendMessageAsync(message);

        // Return immediately - order will be processed asynchronously
        return order.Id;
    }
}

// Consumer: Azure Function
public class OrderProcessor
{
    private readonly IOrderService _orderService;

    public OrderProcessor(IOrderService orderService)
    {
        _orderService = orderService;
    }

    [FunctionName("ProcessOrders")]
    public async Task ProcessOrder(
        [ServiceBusTrigger("orders", Connection = "ServiceBusConnection")]
        ServiceBusReceivedMessage message,
        ServiceBusMessageActions messageActions,
        ILogger log)
    {
        try
        {
            var order = JsonSerializer.Deserialize<Order>(message.Body);

            await _orderService.ProcessAsync(order);

            // Complete the message
            await messageActions.CompleteMessageAsync(message);

            log.LogInformation("Processed order {OrderId}", order.Id);
        }
        catch (Exception ex)
        {
            log.LogError(ex, "Failed to process order");

            // Dead letter for investigation
            if (message.DeliveryCount >= 3)
            {
                await messageActions.DeadLetterMessageAsync(message,
                    "MaxRetriesExceeded", ex.Message);
            }
            // Otherwise, message will be retried automatically
        }
    }
}
```

---

## Competing Consumers

### Problem

A single consumer can't keep up with message volume.

### Solution

```
COMPETING CONSUMERS:
────────────────────

┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│                          ┌───────────┐                                │
│                      ┌──▶│ Consumer 1│                                │
│                      │   └───────────┘                                │
│  ┌───────┐           │   ┌───────────┐                                │
│  │ QUEUE │ ──────────┼──▶│ Consumer 2│   (Lock ensures one consumer │
│  │ ▓▓▓▓▓ │           │   └───────────┘    processes each message)    │
│  └───────┘           │   ┌───────────┐                                │
│                      └──▶│ Consumer 3│                                │
│                          └───────────┘                                │
│                                                                        │
│  Each message is processed by exactly ONE consumer                    │
│  Scale consumers based on queue depth                                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Azure Function with KEDA scaling
// host.json
{
  "version": "2.0",
  "extensions": {
    "serviceBus": {
      "prefetchCount": 100,
      "messageHandlerOptions": {
        "maxConcurrentCalls": 16,
        "maxAutoLockRenewalDuration": "00:05:00"
      }
    }
  }
}

// Function scales based on queue depth automatically
[FunctionName("ProcessMessages")]
public async Task Run(
    [ServiceBusTrigger("myqueue", Connection = "ServiceBusConnection")]
    ServiceBusReceivedMessage[] messages,  // Batch processing
    ILogger log)
{
    var tasks = messages.Select(async message =>
    {
        await ProcessMessageAsync(message);
    });

    await Task.WhenAll(tasks);
}
```

```bicep
// Container Apps with KEDA scaler
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'message-processor'
  properties: {
    configuration: {
      secrets: [
        {
          name: 'sb-connection'
          value: serviceBusConnectionString
        }
      ]
    }
    template: {
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [
          {
            name: 'queue-based-scaling'
            custom: {
              type: 'azure-servicebus'
              metadata: {
                queueName: 'orders'
                messageCount: '100'  // Scale up when > 100 messages
              }
              auth: [
                {
                  secretRef: 'sb-connection'
                  triggerParameter: 'connection'
                }
              ]
            }
          }
        ]
      }
    }
  }
}
```

---

## Publisher/Subscriber Pattern

### Problem

Multiple services need to react to the same event, but you don't want tight coupling.

### Solution

```
PUBLISHER/SUBSCRIBER:
─────────────────────

┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│                                  ┌─────────────────┐                  │
│                              ┌──▶│ Email Service   │                  │
│                              │   │ (send confirm)  │                  │
│                              │   └─────────────────┘                  │
│  ┌─────────────┐    ┌───────┴───┐                                    │
│  │   Order     │───▶│  EVENT    │   ┌─────────────────┐              │
│  │   Service   │    │   GRID    │──▶│ Inventory Svc   │              │
│  └─────────────┘    └───────┬───┘   │ (reserve stock) │              │
│                              │       └─────────────────┘              │
│        publishes             │   ┌─────────────────┐                  │
│     "OrderCreated"           └──▶│ Analytics Svc   │                  │
│                                  │ (track metrics) │                  │
│                                  └─────────────────┘                  │
│                                                                        │
│  Publisher doesn't know (or care) about subscribers                   │
│  Easy to add new subscribers without changing publisher              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Event Grid Implementation

```csharp
// Publisher
public class OrderEventPublisher
{
    private readonly EventGridPublisherClient _client;

    public OrderEventPublisher(string endpoint, string key)
    {
        _client = new EventGridPublisherClient(
            new Uri(endpoint),
            new AzureKeyCredential(key));
    }

    public async Task PublishOrderCreatedAsync(Order order)
    {
        var events = new List<EventGridEvent>
        {
            new EventGridEvent(
                subject: $"orders/{order.Id}",
                eventType: "Contoso.Orders.OrderCreated",
                dataVersion: "1.0",
                data: new OrderCreatedEventData
                {
                    OrderId = order.Id,
                    CustomerId = order.CustomerId,
                    Items = order.Items,
                    Total = order.Total
                })
        };

        await _client.SendEventsAsync(events);
    }
}

// Subscriber 1: Email Service (Azure Function)
[FunctionName("SendOrderConfirmation")]
public async Task SendConfirmation(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    ILogger log)
{
    if (eventGridEvent.EventType != "Contoso.Orders.OrderCreated")
        return;

    var data = eventGridEvent.Data.ToObjectFromJson<OrderCreatedEventData>();
    await _emailService.SendOrderConfirmationAsync(data);
}

// Subscriber 2: Inventory Service (Azure Function)
[FunctionName("ReserveInventory")]
public async Task Reserve(
    [EventGridTrigger] EventGridEvent eventGridEvent,
    ILogger log)
{
    if (eventGridEvent.EventType != "Contoso.Orders.OrderCreated")
        return;

    var data = eventGridEvent.Data.ToObjectFromJson<OrderCreatedEventData>();
    await _inventoryService.ReserveAsync(data.Items);
}
```

```bicep
// Event Grid topic and subscriptions
resource eventGridTopic 'Microsoft.EventGrid/topics@2022-06-15' = {
  name: 'order-events'
  location: location
}

resource emailSubscription 'Microsoft.EventGrid/eventSubscriptions@2022-06-15' = {
  name: 'email-subscription'
  scope: eventGridTopic
  properties: {
    destination: {
      endpointType: 'AzureFunction'
      properties: {
        resourceId: emailFunction.id
      }
    }
    filter: {
      includedEventTypes: [
        'Contoso.Orders.OrderCreated'
      ]
    }
  }
}
```

---

## Saga Pattern

### Problem

You need to maintain data consistency across multiple services without distributed transactions.

### Solution

```
SAGA PATTERN - CHOREOGRAPHY VS ORCHESTRATION:
─────────────────────────────────────────────

CHOREOGRAPHY (Event-driven):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Order ──▶ Inventory ──▶ Payment ──▶ Shipping                        │
│  Svc       Svc          Svc         Svc                              │
│   │         │            │           │                                │
│   │ OrderCreated         │           │                                │
│   └────────▶│            │           │                                │
│             │ InventoryReserved      │                                │
│             └───────────▶│           │                                │
│                          │ PaymentProcessed                          │
│                          └──────────▶│                                │
│                                                                        │
│  Each service reacts to events from previous step                    │
│  Decentralized - no single point of failure                          │
│  Harder to understand and debug                                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

ORCHESTRATION (Centralized coordinator):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│         ┌──────────────────────────┐                                  │
│         │       ORCHESTRATOR       │                                  │
│         │   (Durable Function)     │                                  │
│         └───────────┬──────────────┘                                  │
│              ┌──────┼──────┬────────┐                                │
│              ▼      ▼      ▼        ▼                                │
│         Inventory Payment Shipping  (compensations)                  │
│         Svc      Svc      Svc                                        │
│                                                                        │
│  Orchestrator controls the flow                                       │
│  Easier to understand and debug                                       │
│  Single point of coordination (but can be durable)                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Durable Functions Implementation

```csharp
// Saga Orchestrator
[FunctionName("OrderSaga")]
public static async Task<SagaResult> OrderSagaOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var order = context.GetInput<OrderRequest>();
    var sagaLog = new List<string>();

    try
    {
        // Step 1: Reserve inventory
        sagaLog.Add("Reserving inventory...");
        var inventoryResult = await context.CallActivityAsync<InventoryResult>(
            nameof(ReserveInventory), order);

        if (!inventoryResult.Success)
        {
            return SagaResult.Failed("Inventory unavailable", sagaLog);
        }
        sagaLog.Add($"Inventory reserved: {inventoryResult.ReservationId}");

        // Step 2: Process payment
        sagaLog.Add("Processing payment...");
        var paymentResult = await context.CallActivityAsync<PaymentResult>(
            nameof(ProcessPayment), new PaymentRequest(order));

        if (!paymentResult.Success)
        {
            // Compensate: Release inventory
            sagaLog.Add("Payment failed, releasing inventory...");
            await context.CallActivityAsync(
                nameof(ReleaseInventory), inventoryResult.ReservationId);
            sagaLog.Add("Inventory released");
            return SagaResult.Failed("Payment failed", sagaLog);
        }
        sagaLog.Add($"Payment processed: {paymentResult.TransactionId}");

        // Step 3: Create shipment
        sagaLog.Add("Creating shipment...");
        var shipmentResult = await context.CallActivityAsync<ShipmentResult>(
            nameof(CreateShipment), new ShipmentRequest(order, inventoryResult));

        if (!shipmentResult.Success)
        {
            // Compensate: Refund payment, release inventory
            sagaLog.Add("Shipment failed, compensating...");
            await context.CallActivityAsync(
                nameof(RefundPayment), paymentResult.TransactionId);
            await context.CallActivityAsync(
                nameof(ReleaseInventory), inventoryResult.ReservationId);
            sagaLog.Add("Compensation complete");
            return SagaResult.Failed("Shipment failed", sagaLog);
        }
        sagaLog.Add($"Shipment created: {shipmentResult.TrackingNumber}");

        // Step 4: Send confirmation
        await context.CallActivityAsync(nameof(SendConfirmation), new ConfirmationRequest
        {
            OrderId = order.Id,
            TrackingNumber = shipmentResult.TrackingNumber
        });
        sagaLog.Add("Confirmation sent");

        return SagaResult.Succeeded(shipmentResult.TrackingNumber, sagaLog);
    }
    catch (Exception ex)
    {
        sagaLog.Add($"Unexpected error: {ex.Message}");
        // Would need more sophisticated compensation here
        return SagaResult.Failed($"Unexpected error: {ex.Message}", sagaLog);
    }
}

// Activity functions
[FunctionName(nameof(ReserveInventory))]
public static async Task<InventoryResult> ReserveInventory(
    [ActivityTrigger] OrderRequest order)
{
    // Call inventory service API
    return await _inventoryClient.ReserveAsync(order.Items);
}

[FunctionName(nameof(ReleaseInventory))]
public static async Task ReleaseInventory(
    [ActivityTrigger] string reservationId)
{
    await _inventoryClient.ReleaseAsync(reservationId);
}

[FunctionName(nameof(ProcessPayment))]
public static async Task<PaymentResult> ProcessPayment(
    [ActivityTrigger] PaymentRequest request)
{
    return await _paymentClient.ProcessAsync(request);
}

[FunctionName(nameof(RefundPayment))]
public static async Task RefundPayment(
    [ActivityTrigger] string transactionId)
{
    await _paymentClient.RefundAsync(transactionId);
}
```

---

## Claim Check Pattern

### Problem

Large messages exceed message broker limits or add unnecessary overhead.

### Solution

```
CLAIM CHECK PATTERN:
────────────────────

┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  SENDER:                                                               │
│  ┌─────────────┐    1. Store payload    ┌─────────────┐              │
│  │ Large       │ ────────────────────▶ │ Blob        │              │
│  │ Payload     │    2. Get reference    │ Storage     │              │
│  │ (50MB)      │ ◀──────────────────── │             │              │
│  └─────────────┘                        └─────────────┘              │
│         │                                                             │
│         │ 3. Send claim check (reference only)                       │
│         ▼                                                             │
│  ┌─────────────┐                                                      │
│  │ Service Bus │  Message: { "claimCheck": "blob://container/id" }   │
│  │ (256KB max) │                                                      │
│  └─────────────┘                                                      │
│         │                                                             │
│         │ 4. Receive claim check                                      │
│         ▼                                                             │
│  RECEIVER:                                                            │
│  ┌─────────────┐    5. Retrieve payload ┌─────────────┐              │
│  │ Consumer    │ ────────────────────▶ │ Blob        │              │
│  │             │ ◀──────────────────── │ Storage     │              │
│  │             │    6. Large payload    │             │              │
│  └─────────────┘                        └─────────────┘              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
public class ClaimCheckService
{
    private readonly BlobContainerClient _blobClient;
    private readonly ServiceBusSender _sender;
    private readonly ServiceBusReceiver _receiver;

    public async Task SendLargeMessageAsync<T>(T payload, string correlationId)
    {
        // 1. Store payload in blob
        var blobName = $"messages/{correlationId}/{Guid.NewGuid()}";
        var blob = _blobClient.GetBlobClient(blobName);

        var json = JsonSerializer.Serialize(payload);
        await blob.UploadAsync(BinaryData.FromString(json));

        // 2. Send claim check via Service Bus
        var claimCheck = new ClaimCheckMessage
        {
            BlobUri = blob.Uri.ToString(),
            ContentType = typeof(T).FullName,
            CreatedAt = DateTimeOffset.UtcNow
        };

        var message = new ServiceBusMessage(JsonSerializer.Serialize(claimCheck))
        {
            CorrelationId = correlationId,
            ContentType = "application/x-claim-check+json",
            TimeToLive = TimeSpan.FromHours(24)
        };

        await _sender.SendMessageAsync(message);
    }

    public async Task<T> ReceiveLargeMessageAsync<T>()
    {
        // 3. Receive claim check
        var message = await _receiver.ReceiveMessageAsync();
        var claimCheck = JsonSerializer.Deserialize<ClaimCheckMessage>(
            message.Body.ToString());

        // 4. Retrieve payload from blob
        var blobClient = new BlobClient(new Uri(claimCheck.BlobUri));
        var response = await blobClient.DownloadContentAsync();
        var payload = JsonSerializer.Deserialize<T>(response.Value.Content);

        // 5. Complete message and optionally delete blob
        await _receiver.CompleteMessageAsync(message);
        await blobClient.DeleteAsync();

        return payload;
    }
}

public class ClaimCheckMessage
{
    public string BlobUri { get; set; }
    public string ContentType { get; set; }
    public DateTimeOffset CreatedAt { get; set; }
}
```

---

## Priority Queue Pattern

### Problem

Some messages are more important than others and should be processed first.

### Azure Implementation

```csharp
// Option 1: Separate queues with different consumers
public class PriorityMessageSender
{
    private readonly ServiceBusSender _highPrioritySender;
    private readonly ServiceBusSender _normalPrioritySender;
    private readonly ServiceBusSender _lowPrioritySender;

    public async Task SendAsync(Order order)
    {
        var message = new ServiceBusMessage(JsonSerializer.Serialize(order));

        var sender = order.Priority switch
        {
            Priority.High => _highPrioritySender,
            Priority.Normal => _normalPrioritySender,
            Priority.Low => _lowPrioritySender,
            _ => _normalPrioritySender
        };

        await sender.SendMessageAsync(message);
    }
}

// Scale consumers differently:
// High priority: 5 instances, fast processing
// Normal priority: 3 instances
// Low priority: 1 instance, process when capacity available
```

```csharp
// Option 2: Single queue with priority sorting (Service Bus sessions)
public class PriorityProcessor
{
    [FunctionName("ProcessHighPriority")]
    public async Task ProcessHigh(
        [ServiceBusTrigger("orders", "high-priority-subscription")]
        ServiceBusReceivedMessage message) { }

    [FunctionName("ProcessNormalPriority")]
    public async Task ProcessNormal(
        [ServiceBusTrigger("orders", "normal-priority-subscription")]
        ServiceBusReceivedMessage message) { }
}
```

---

*Next: [Data Patterns](03-data-patterns.md)* | *Back to [Reliability Patterns](01-reliability-patterns.md)*
