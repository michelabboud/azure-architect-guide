# Azure Service Bus

## What is Azure Service Bus?

Azure Service Bus is a fully managed enterprise message broker with message queues and publish-subscribe topics. It provides reliable message delivery, temporal decoupling, and advanced messaging features like transactions, sessions, and dead-letter handling.

### Service Bus vs AWS SQS/SNS

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   SERVICE BUS vs AWS SQS/SNS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS MODEL                               AZURE SERVICE BUS                  │
│  ─────────                               ──────────────────                  │
│                                                                              │
│  Separate Services:                     Unified Service:                    │
│  ┌─────────────────────┐               ┌─────────────────────────────────┐  │
│  │       SQS           │               │     SERVICE BUS NAMESPACE        │  │
│  │  (Point-to-point)   │               │                                  │  │
│  │  • Standard Queue   │               │  ┌─────────┐  ┌──────────────┐  │  │
│  │  • FIFO Queue       │               │  │ QUEUES  │  │    TOPICS    │  │  │
│  └─────────────────────┘               │  │         │  │              │  │  │
│           +                             │  │ P2P     │  │ Pub/Sub      │  │  │
│  ┌─────────────────────┐               │  │ FIFO    │  │ Subscriptions│  │  │
│  │       SNS           │               │  │ Sessions│  │ Filters      │  │  │
│  │  (Pub/Sub)          │               │  └─────────┘  └──────────────┘  │  │
│  │  • Topics           │               │                                  │  │
│  │  • Subscriptions    │               └─────────────────────────────────┘  │
│  └─────────────────────┘                                                    │
│                                                                              │
│  FEATURE COMPARISON:                                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Feature           │ SQS/SNS              │ Service Bus               │  │
│  ├───────────────────┼──────────────────────┼───────────────────────────┤  │
│  │ Max message size  │ SQS: 256KB           │ Standard: 256KB           │  │
│  │                   │ SNS: 256KB           │ Premium: 100MB            │  │
│  │                   │                      │                           │  │
│  │ Message retention │ SQS: 14 days max     │ Unlimited (with Premium)  │  │
│  │                   │ SNS: Immediate       │                           │  │
│  │                   │                      │                           │  │
│  │ Ordering          │ FIFO queues only     │ Sessions (FIFO)           │  │
│  │                   │                      │ Partitions optional       │  │
│  │                   │                      │                           │  │
│  │ Transactions      │ No                   │ Yes (ACID)                │  │
│  │                   │                      │                           │  │
│  │ Duplicate detect  │ FIFO only (5 min)    │ Yes (configurable window) │  │
│  │                   │                      │                           │  │
│  │ Dead-letter queue │ Yes                  │ Yes (per entity)          │  │
│  │                   │                      │                           │  │
│  │ Scheduled messages│ SQS: Delay up to 15m │ Any future time           │  │
│  │                   │                      │                           │  │
│  │ Message deferral  │ No                   │ Yes                       │  │
│  │                   │                      │                           │  │
│  │ Auto-forwarding   │ No                   │ Yes                       │  │
│  │                   │                      │                           │  │
│  │ Topic filters     │ SNS: Attribute filter│ SQL-like filters          │  │
│  │                   │                      │ Correlation filters       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Service Bus Tiers

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| **Queues** | Yes | Yes | Yes |
| **Topics/Subscriptions** | No | Yes | Yes |
| **Max message size** | 256 KB | 256 KB | 100 MB |
| **Max namespace size** | N/A | N/A | 1-80 TB |
| **Transactions** | No | No | Yes |
| **Duplicate detection** | No | Yes | Yes |
| **Sessions (FIFO)** | No | Yes | Yes |
| **Geo-disaster recovery** | No | Yes | Yes |
| **VNet Integration** | No | No | Yes |
| **Private Link** | No | No | Yes |
| **Dedicated resources** | No | No | Yes |
| **JMS 2.0 Support** | No | No | Yes |
| **Price model** | Per operation | Per operation | Per Messaging Unit |
| **Approx price** | ~$0.05/1M ops | ~$10/mo + ops | ~$668/MU/mo |

---

## Queues vs Topics/Subscriptions

### Queue Pattern (Point-to-Point)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    QUEUE PATTERN (Point-to-Point)                           │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PRODUCER                                              CONSUMER             │
│  ────────                                              ────────             │
│                                                                             │
│  ┌──────────┐         ┌─────────────────────┐        ┌──────────┐         │
│  │  Order   │         │                     │        │  Order   │         │
│  │ Service  │──Send──▶│   ┌───┬───┬───┬───┐ │──Recv──│Processor │         │
│  │          │         │   │ M │ M │ M │ M │ │        │          │         │
│  └──────────┘         │   └───┴───┴───┴───┘ │        └──────────┘         │
│                       │      SERVICE BUS    │                              │
│                       │        QUEUE        │                              │
│                       │                     │                              │
│                       │  ┌───────────────┐  │                              │
│                       │  │  Dead Letter  │  │   Failed messages            │
│                       │  │    Queue      │  │   for investigation          │
│                       │  └───────────────┘  │                              │
│                       └─────────────────────┘                              │
│                                                                             │
│  CHARACTERISTICS:                                                          │
│  • One message → One consumer (competing consumers)                        │
│  • Messages are removed after successful processing                        │
│  • At-least-once or at-most-once delivery                                 │
│  • Good for: Work distribution, command processing                        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Topic/Subscription Pattern (Publish-Subscribe)

```
┌────────────────────────────────────────────────────────────────────────────┐
│                TOPIC/SUBSCRIPTION PATTERN (Pub/Sub)                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PUBLISHER                                           SUBSCRIBERS            │
│  ─────────                                           ───────────            │
│                                                                             │
│  ┌──────────┐         ┌─────────────────────┐                              │
│  │  Order   │         │    SERVICE BUS      │        ┌──────────┐         │
│  │ Service  │──Pub───▶│       TOPIC         │   ┌───▶│  Email   │         │
│  │          │         │                     │   │    │ Service  │         │
│  └──────────┘         │  ┌───┬───┬───┬───┐  │   │    └──────────┘         │
│                       │  │ M │ M │ M │ M │  │   │                          │
│                       │  └───┴───┴───┴───┘  │   │    ┌──────────┐         │
│                       │         │           │   ├───▶│Analytics │         │
│                       │         │           │   │    │ Service  │         │
│                       │    ┌────┴────┐      │   │    └──────────┘         │
│                       │    │ FANOUT  │      │   │                          │
│                       │    └────┬────┘      │   │    ┌──────────┐         │
│                       │    ┌────┼────┐      │   └───▶│Inventory │         │
│                       │    │    │    │      │        │ Service  │         │
│                       │    ▼    ▼    ▼      │        └──────────┘         │
│                       │  ┌───┐┌───┐┌───┐    │                              │
│                       │  │Sub││Sub││Sub│    │   Subscriptions with        │
│                       │  │ 1 ││ 2 ││ 3 │    │   optional filters          │
│                       │  └───┘└───┘└───┘    │                              │
│                       └─────────────────────┘                              │
│                                                                             │
│  CHARACTERISTICS:                                                          │
│  • One message → Multiple subscribers (independent copies)                 │
│  • Each subscription is like a virtual queue                              │
│  • Subscribers can filter messages                                        │
│  • Good for: Event broadcasting, fan-out scenarios                       │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Message Sessions and FIFO

### Sessions Overview

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    MESSAGE SESSIONS (FIFO Ordering)                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WITHOUT SESSIONS:                    WITH SESSIONS:                       │
│  ─────────────────                    ──────────────                       │
│                                                                             │
│  Messages interleaved:               Messages grouped by SessionId:        │
│                                                                             │
│  ┌───┬───┬───┬───┬───┬───┐          ┌───────────────────────────────────┐ │
│  │A1 │B1 │A2 │B2 │A3 │B3 │          │ Session A: A1 → A2 → A3 (ordered) │ │
│  └───┴───┴───┴───┴───┴───┘          │ Session B: B1 → B2 → B3 (ordered) │ │
│                                      └───────────────────────────────────┘ │
│  Any consumer gets any msg           Each session processed by one         │
│                                      consumer at a time                    │
│                                                                             │
│  USE CASES:                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Order processing: All messages for order #123 in sequence        │   │
│  │ • User actions: User's actions processed in order                  │   │
│  │ • Multi-step workflows: Steps processed sequentially               │   │
│  │ • Conversation: Chat messages in order per conversation            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SESSION STATE:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Store up to 256KB of state per session                           │   │
│  │ • Perfect for stateful message processing                          │   │
│  │ • State survives consumer failures                                 │   │
│  │ • Retrieved when accepting session                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Session Example (C#)

```csharp
using Azure.Messaging.ServiceBus;

// Send messages with session ID
await using var client = new ServiceBusClient(connectionString);
await using var sender = client.CreateSender("orders-queue");

// All messages for order "ORD-123" go to same session
var sessionId = "ORD-123";

await sender.SendMessageAsync(new ServiceBusMessage("Step 1: Validate")
{
    SessionId = sessionId,
    Subject = "validate"
});

await sender.SendMessageAsync(new ServiceBusMessage("Step 2: Process Payment")
{
    SessionId = sessionId,
    Subject = "payment"
});

await sender.SendMessageAsync(new ServiceBusMessage("Step 3: Ship")
{
    SessionId = sessionId,
    Subject = "ship"
});

// Receive with session processor
var processor = client.CreateSessionProcessor("orders-queue", new ServiceBusSessionProcessorOptions
{
    MaxConcurrentSessions = 10,
    MaxConcurrentCallsPerSession = 1, // Process one at a time per session
    AutoCompleteMessages = false
});

processor.ProcessMessageAsync += async args =>
{
    var sessionId = args.SessionId;
    var body = args.Message.Body.ToString();

    // Get/set session state
    var stateBytes = await args.GetSessionStateAsync();
    var state = stateBytes != null
        ? JsonSerializer.Deserialize<OrderState>(stateBytes.ToArray())
        : new OrderState();

    // Process message
    Console.WriteLine($"Session {sessionId}: {body}");
    state.CompletedSteps.Add(args.Message.Subject);

    // Save state
    await args.SetSessionStateAsync(BinaryData.FromObjectAsJson(state));

    await args.CompleteMessageAsync(args.Message);
};

processor.ProcessErrorAsync += async args =>
{
    Console.WriteLine($"Error: {args.Exception}");
};

await processor.StartProcessingAsync();
```

---

## Dead-Letter Handling

### Dead-Letter Queue (DLQ) Overview

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    DEAD-LETTER QUEUE                                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  WHEN MESSAGES GO TO DLQ:                                                  │
│  ────────────────────────                                                  │
│                                                                             │
│  1. Max delivery count exceeded                                            │
│     └── Message received and abandoned too many times (default: 10)       │
│                                                                             │
│  2. Message TTL expired                                                    │
│     └── Message stayed in queue longer than TimeToLive                    │
│     └── Requires: deadLetteringOnMessageExpiration = true                 │
│                                                                             │
│  3. Subscription filter evaluation fails                                   │
│     └── Topic subscription's filter throws exception                      │
│                                                                             │
│  4. Explicit dead-lettering                                                │
│     └── Application calls DeadLetterMessageAsync()                        │
│                                                                             │
│  DLQ ARCHITECTURE:                                                         │
│  ─────────────────                                                         │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                        MAIN QUEUE                                    │   │
│  │                                                                      │   │
│  │   ┌───┬───┬───┬───┬───┐                                            │   │
│  │   │ M │ M │ M │ M │ M │◄─── Messages arrive here                   │   │
│  │   └───┴───┴───┴───┴───┘                                            │   │
│  │                │                                                     │   │
│  │                │ Failed processing / Expired / Explicit             │   │
│  │                ▼                                                     │   │
│  │   ┌─────────────────────────────────────────────┐                   │   │
│  │   │            DEAD-LETTER QUEUE                │                   │   │
│  │   │   Path: {queue}/$deadletterqueue           │                   │   │
│  │   │                                            │                   │   │
│  │   │   ┌───┬───┬───┐                            │                   │   │
│  │   │   │ D │ D │ D │◄─── Failed messages        │                   │   │
│  │   │   └───┴───┴───┘                            │                   │   │
│  │   │                                            │                   │   │
│  │   │   Properties added:                        │                   │   │
│  │   │   • DeadLetterReason                       │                   │   │
│  │   │   • DeadLetterErrorDescription             │                   │   │
│  │   │   • Original message properties preserved  │                   │   │
│  │   └─────────────────────────────────────────────┘                   │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### DLQ Processing Pattern

```csharp
// Process dead-letter queue
var dlqPath = EntityNameHelper.FormatDeadLetterPath("orders-queue");
await using var dlqReceiver = client.CreateReceiver(dlqPath);

// Receive dead-lettered messages
var messages = await dlqReceiver.ReceiveMessagesAsync(maxMessages: 10);

foreach (var message in messages)
{
    // Get dead-letter information
    var reason = message.DeadLetterReason;
    var description = message.DeadLetterErrorDescription;
    var originalBody = message.Body.ToString();

    Console.WriteLine($"Dead-lettered: {reason} - {description}");
    Console.WriteLine($"Original body: {originalBody}");
    Console.WriteLine($"Enqueued: {message.EnqueuedTime}");
    Console.WriteLine($"Delivery count: {message.DeliveryCount}");

    // Options:
    // 1. Log and complete (remove from DLQ)
    await dlqReceiver.CompleteMessageAsync(message);

    // 2. Fix and resubmit to main queue
    // var fixedMessage = new ServiceBusMessage(fixedBody);
    // await sender.SendMessageAsync(fixedMessage);
    // await dlqReceiver.CompleteMessageAsync(message);

    // 3. Send to manual review queue
    // 4. Store in database for analysis
}
```

### Explicit Dead-Lettering

```csharp
processor.ProcessMessageAsync += async args =>
{
    try
    {
        var order = JsonSerializer.Deserialize<Order>(args.Message.Body);

        if (!order.IsValid())
        {
            // Explicit dead-letter with custom reason
            await args.DeadLetterMessageAsync(
                args.Message,
                deadLetterReason: "ValidationFailed",
                deadLetterErrorDescription: $"Order {order.Id} failed validation: {order.ValidationErrors}");
            return;
        }

        await ProcessOrderAsync(order);
        await args.CompleteMessageAsync(args.Message);
    }
    catch (TransientException ex)
    {
        // Abandon - will retry (up to max delivery count)
        await args.AbandonMessageAsync(args.Message);
    }
    catch (PermanentException ex)
    {
        // Dead-letter - won't retry
        await args.DeadLetterMessageAsync(
            args.Message,
            deadLetterReason: "ProcessingFailed",
            deadLetterErrorDescription: ex.Message);
    }
};
```

---

## Geo-Disaster Recovery

### Geo-DR Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                GEO-DISASTER RECOVERY                                        │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ACTIVE-PASSIVE PAIRING:                                                   │
│  ───────────────────────                                                   │
│                                                                             │
│  PRIMARY REGION (Active)          SECONDARY REGION (Passive)               │
│  ┌────────────────────────┐      ┌────────────────────────┐               │
│  │    East US Namespace    │      │   West US Namespace    │               │
│  │                        │      │                        │               │
│  │  ┌──────────────────┐  │      │  ┌──────────────────┐  │               │
│  │  │  orders-queue    │  │─────▶│  │  orders-queue    │  │               │
│  │  └──────────────────┘  │ Sync │  └──────────────────┘  │               │
│  │  ┌──────────────────┐  │ Meta │  ┌──────────────────┐  │               │
│  │  │  events-topic    │  │─────▶│  │  events-topic    │  │               │
│  │  └──────────────────┘  │ data │  └──────────────────┘  │               │
│  │                        │      │                        │               │
│  │  ✓ Accepting messages  │      │  ✗ Read-only          │               │
│  └────────────────────────┘      └────────────────────────┘               │
│           │                                  │                              │
│           │     ALIAS (DNS): sb-prod.servicebus.windows.net                │
│           │               │                                                 │
│           └───────────────┴──────────────────┘                             │
│                                                                             │
│  WHAT'S SYNCHRONIZED:                      NOT SYNCHRONIZED:               │
│  ┌───────────────────────────────┐        ┌───────────────────────────┐   │
│  │ ✓ Entity metadata             │        │ ✗ Messages in queues      │   │
│  │ ✓ Queues, topics, subscriptions│       │ ✗ Dead-letter messages    │   │
│  │ ✓ Filters and rules           │        │ ✗ Message state          │   │
│  │ ✓ Authorization rules         │        │ ✗ Session state          │   │
│  └───────────────────────────────┘        └───────────────────────────┘   │
│                                                                             │
│  FAILOVER PROCESS:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. Initiate failover (manual or automatic)                          │   │
│  │ 2. Alias DNS updated to point to secondary                          │   │
│  │ 3. Secondary becomes primary (read-write)                           │   │
│  │ 4. Applications reconnect via alias (no code change)                │   │
│  │ 5. Old primary becomes secondary                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Geo-DR Setup (CLI)

```bash
# Create primary namespace (Premium required for Geo-DR)
az servicebus namespace create \
  --resource-group myRG \
  --name sb-primary-eastus \
  --location eastus \
  --sku Premium \
  --capacity 1

# Create secondary namespace
az servicebus namespace create \
  --resource-group myRG \
  --name sb-secondary-westus \
  --location westus \
  --sku Premium \
  --capacity 1

# Create Geo-DR pairing with alias
az servicebus georecovery-alias set \
  --resource-group myRG \
  --namespace-name sb-primary-eastus \
  --alias sb-prod \
  --partner-namespace /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.ServiceBus/namespaces/sb-secondary-westus

# Check pairing status
az servicebus georecovery-alias show \
  --resource-group myRG \
  --namespace-name sb-primary-eastus \
  --alias sb-prod

# Initiate failover (when disaster occurs)
az servicebus georecovery-alias fail-over \
  --resource-group myRG \
  --namespace-name sb-secondary-westus \
  --alias sb-prod

# Break pairing (for maintenance or to set up new pair)
az servicebus georecovery-alias break-pair \
  --resource-group myRG \
  --namespace-name sb-primary-eastus \
  --alias sb-prod
```

---

## Topic Subscriptions and Filters

### Filter Types

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    SUBSCRIPTION FILTERS                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SQL FILTER (Boolean expression):                                          │
│  ────────────────────────────────                                          │
│                                                                             │
│  • Full SQL-like WHERE clause                                              │
│  • Access system and user properties                                       │
│  • Supports: AND, OR, NOT, comparison, LIKE, IN, IS NULL                  │
│                                                                             │
│  Examples:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ "priority > 5"                                                      │   │
│  │ "region = 'US' AND amount > 100"                                    │   │
│  │ "eventType LIKE 'Order.%'"                                          │   │
│  │ "customerId IN ('C001', 'C002', 'C003')"                           │   │
│  │ "sys.Label = 'urgent' AND user.department = 'sales'"               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CORRELATION FILTER (Exact match):                                         │
│  ─────────────────────────────────                                         │
│                                                                             │
│  • Exact property matching only                                            │
│  • More efficient than SQL (hash-based)                                   │
│  • Matches: CorrelationId, MessageId, To, Subject, etc.                  │
│  • Can match user-defined properties                                      │
│                                                                             │
│  Examples:                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ CorrelationId = "order-123"                                         │   │
│  │ Subject = "NewOrder" AND properties["region"] = "EU"               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  TRUE/FALSE FILTER:                                                        │
│  ──────────────────                                                        │
│                                                                             │
│  • TrueFilter: Match all messages (default)                               │
│  • FalseFilter: Match no messages (useful for pause)                      │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Subscription Example

```csharp
// Create topic with subscriptions using filters
var adminClient = new ServiceBusAdministrationClient(connectionString);

// Create topic
await adminClient.CreateTopicAsync("order-events");

// Subscription 1: All high-priority orders
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("order-events", "high-priority")
    {
        MaxDeliveryCount = 10,
        LockDuration = TimeSpan.FromMinutes(1)
    });

await adminClient.CreateRuleAsync("order-events", "high-priority",
    new CreateRuleOptions("high-priority-rule", new SqlRuleFilter("priority >= 8")));

// Delete default rule
await adminClient.DeleteRuleAsync("order-events", "high-priority", "$Default");

// Subscription 2: US region orders using correlation filter (faster)
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("order-events", "us-orders"));

await adminClient.CreateRuleAsync("order-events", "us-orders",
    new CreateRuleOptions("us-orders-rule", new CorrelationRuleFilter
    {
        ApplicationProperties =
        {
            { "region", "US" }
        }
    }));

await adminClient.DeleteRuleAsync("order-events", "us-orders", "$Default");

// Subscription 3: All orders (audit log)
await adminClient.CreateSubscriptionAsync(
    new CreateSubscriptionOptions("order-events", "audit-all")
    {
        // Keep default TrueFilter - receives all messages
    });

// Publish message
await using var sender = client.CreateSender("order-events");

var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
{
    Subject = "NewOrder",
    ApplicationProperties =
    {
        { "priority", 9 },
        { "region", "US" },
        { "orderId", order.Id }
    }
};

await sender.SendMessageAsync(message);
// This message will be received by: high-priority, us-orders, and audit-all
```

---

## Bicep/CLI Deployment Examples

### Complete Service Bus Deployment (Bicep)

```bicep
// main.bicep - Complete Service Bus deployment

@description('The name of the Service Bus namespace')
param namespaceName string = 'sb-${uniqueString(resourceGroup().id)}'

@description('Location for all resources')
param location string = resourceGroup().location

@description('The pricing tier of the Service Bus namespace')
@allowed(['Basic', 'Standard', 'Premium'])
param sku string = 'Standard'

@description('Messaging units for Premium tier (1, 2, 4, 8, 16)')
param capacity int = 1

// Service Bus Namespace
resource sbNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: namespaceName
  location: location
  sku: {
    name: sku
    tier: sku
    capacity: sku == 'Premium' ? capacity : 0
  }
  properties: {
    minimumTlsVersion: '1.2'
    publicNetworkAccess: 'Enabled'
    disableLocalAuth: false
    zoneRedundant: sku == 'Premium'
  }
}

// Queue for orders
resource ordersQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: sbNamespace
  name: 'orders'
  properties: {
    maxSizeInMegabytes: 5120
    defaultMessageTimeToLive: 'P14D'
    lockDuration: 'PT1M'
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT10M'
    deadLetteringOnMessageExpiration: true
    maxDeliveryCount: 10
    enablePartitioning: false
    requiresSession: false
  }
}

// Queue with sessions for order processing workflow
resource orderProcessingQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: sbNamespace
  name: 'order-processing'
  properties: {
    maxSizeInMegabytes: 5120
    defaultMessageTimeToLive: 'P7D'
    lockDuration: 'PT5M'
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT10M'
    deadLetteringOnMessageExpiration: true
    maxDeliveryCount: 5
    enablePartitioning: false
    requiresSession: true  // Enable sessions for FIFO
  }
}

// Topic for order events
resource orderEventsTopic 'Microsoft.ServiceBus/namespaces/topics@2022-10-01-preview' = {
  parent: sbNamespace
  name: 'order-events'
  properties: {
    maxSizeInMegabytes: 5120
    defaultMessageTimeToLive: 'P14D'
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT10M'
    enablePartitioning: false
    supportOrdering: true
  }
}

// Subscription: Email notifications
resource emailSubscription 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: orderEventsTopic
  name: 'email-notifications'
  properties: {
    maxDeliveryCount: 10
    lockDuration: 'PT1M'
    deadLetteringOnMessageExpiration: true
    deadLetteringOnFilterEvaluationExceptions: true
  }
}

resource emailRule 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: emailSubscription
  name: 'high-value-orders'
  properties: {
    filterType: 'SqlFilter'
    sqlFilter: {
      sqlExpression: 'amount > 100 OR priority >= 8'
    }
    action: {
      sqlExpression: 'SET sys.Label = \'HighValue\''
    }
  }
}

// Subscription: Analytics (all events)
resource analyticsSubscription 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: orderEventsTopic
  name: 'analytics'
  properties: {
    maxDeliveryCount: 3
    lockDuration: 'PT30S'
    deadLetteringOnMessageExpiration: false
    // Default TrueFilter - receives all messages
  }
}

// Subscription: Inventory updates (correlation filter)
resource inventorySubscription 'Microsoft.ServiceBus/namespaces/topics/subscriptions@2022-10-01-preview' = {
  parent: orderEventsTopic
  name: 'inventory-updates'
  properties: {
    maxDeliveryCount: 10
    lockDuration: 'PT1M'
    deadLetteringOnMessageExpiration: true
  }
}

resource inventoryRule 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: inventorySubscription
  name: 'inventory-events'
  properties: {
    filterType: 'CorrelationFilter'
    correlationFilter: {
      label: 'InventoryChange'
      properties: {
        eventType: 'OrderCreated'
      }
    }
  }
}

// Delete default rule for inventory subscription
resource deleteDefaultRule 'Microsoft.ServiceBus/namespaces/topics/subscriptions/rules@2022-10-01-preview' = {
  parent: inventorySubscription
  name: '$Default'
  properties: {
    filterType: 'SqlFilter'
    sqlFilter: {
      sqlExpression: '1=0'  // Never match - effectively disabled
    }
  }
  dependsOn: [inventoryRule]
}

// Authorization rules
resource sendListenRule 'Microsoft.ServiceBus/namespaces/AuthorizationRules@2022-10-01-preview' = {
  parent: sbNamespace
  name: 'app-send-listen'
  properties: {
    rights: ['Send', 'Listen']
  }
}

resource listenOnlyRule 'Microsoft.ServiceBus/namespaces/queues/authorizationRules@2022-10-01-preview' = {
  parent: ordersQueue
  name: 'consumer-listen'
  properties: {
    rights: ['Listen']
  }
}

// Outputs
output namespaceId string = sbNamespace.id
output namespaceName string = sbNamespace.name
output ordersQueueName string = ordersQueue.name
output orderEventsTopicName string = orderEventsTopic.name
output connectionString string = listKeys(sendListenRule.id, sendListenRule.apiVersion).primaryConnectionString
```

### Azure CLI Commands

```bash
# Create namespace
az servicebus namespace create \
  --resource-group myRG \
  --name sb-myapp \
  --location eastus \
  --sku Standard

# Create queue with sessions
az servicebus queue create \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --name order-processing \
  --enable-session true \
  --max-delivery-count 5 \
  --default-message-time-to-live P7D \
  --lock-duration PT5M \
  --dead-lettering-on-message-expiration true \
  --enable-duplicate-detection true \
  --duplicate-detection-history-time-window PT10M

# Create topic
az servicebus topic create \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --name order-events \
  --default-message-time-to-live P14D \
  --enable-duplicate-detection true

# Create subscription with SQL filter
az servicebus topic subscription create \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --topic-name order-events \
  --name high-priority \
  --max-delivery-count 10 \
  --lock-duration PT1M

az servicebus topic subscription rule create \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --topic-name order-events \
  --subscription-name high-priority \
  --name priority-filter \
  --filter-type SqlFilter \
  --filter-sql-expression "priority >= 8"

# Delete default rule
az servicebus topic subscription rule delete \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --topic-name order-events \
  --subscription-name high-priority \
  --name '$Default'

# Create subscription with correlation filter
az servicebus topic subscription create \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --topic-name order-events \
  --name us-orders

az servicebus topic subscription rule create \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --topic-name order-events \
  --subscription-name us-orders \
  --name region-filter \
  --filter-type CorrelationFilter \
  --correlation-filter application-properties='{"region":"US"}'

# Get connection string
az servicebus namespace authorization-rule keys list \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --name RootManageSharedAccessKey \
  --query primaryConnectionString \
  --output tsv

# Monitor queue metrics
az servicebus queue show \
  --resource-group myRG \
  --namespace-name sb-myapp \
  --name order-processing \
  --query "{ActiveMessages:countDetails.activeMessageCount, DeadLettered:countDetails.deadLetterMessageCount, Scheduled:countDetails.scheduledMessageCount}"
```

---

## Best Practices

```
SERVICE BUS BEST PRACTICES:
───────────────────────────

1. MESSAGE DESIGN
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Keep messages small (payload in external storage if > 64KB)       │
   │ • Use ContentType to indicate payload format                        │
   │ • Set MessageId for idempotency checking                           │
   │ • Use CorrelationId for request/response patterns                  │
   │ • Add meaningful Subject/Label for filtering                        │
   │ • Include metadata in ApplicationProperties                        │
   └─────────────────────────────────────────────────────────────────────┘

2. RELIABILITY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Enable duplicate detection for critical queues                    │
   │ • Set appropriate MaxDeliveryCount (default 10)                    │
   │ • Monitor and process dead-letter queues                           │
   │ • Use PeekLock mode (not ReceiveAndDelete)                        │
   │ • Implement idempotent message handlers                            │
   │ • Use transactions for atomic operations (Premium)                 │
   └─────────────────────────────────────────────────────────────────────┘

3. PERFORMANCE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use batching for high throughput (SendMessagesAsync)             │
   │ • Enable prefetch on receivers (PrefetchCount)                     │
   │ • Use Premium tier for predictable performance                     │
   │ • Avoid sessions unless ordering required (adds overhead)          │
   │ • Use correlation filters over SQL filters when possible          │
   │ • Consider partitioning for scale (Standard tier)                 │
   └─────────────────────────────────────────────────────────────────────┘

4. SECURITY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use Managed Identity instead of connection strings               │
   │ • Apply least-privilege with SAS policies                         │
   │ • Use separate SAS keys for send/receive                          │
   │ • Enable Private Link for Premium tier                            │
   │ • Disable public network access if not needed                     │
   │ • Rotate SAS keys regularly                                        │
   └─────────────────────────────────────────────────────────────────────┘

5. OPERATIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Set up alerts for queue depth and DLQ count                      │
   │ • Enable diagnostic logs to Log Analytics                          │
   │ • Implement health checks in applications                          │
   │ • Plan for Geo-DR with Premium tier                               │
   │ • Use auto-forwarding for message routing                         │
   │ • Monitor message age (time in queue)                             │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Event Grid](03-event-grid.md)* | *Previous: [API Management](01-api-management.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
