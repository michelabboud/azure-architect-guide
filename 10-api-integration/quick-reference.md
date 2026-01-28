# Quick Reference: API Integration

## API Management CLI Commands

### Create APIM Instance

```bash
# Create APIM (Developer tier for testing)
az apim create \
  --resource-group myRG \
  --name my-apim \
  --location eastus \
  --publisher-email admin@company.com \
  --publisher-name "My Company" \
  --sku-name Developer

# Create API from OpenAPI spec
az apim api import \
  --resource-group myRG \
  --service-name my-apim \
  --api-id orders-api \
  --path orders \
  --specification-format OpenApi \
  --specification-url https://raw.githubusercontent.com/org/repo/main/openapi.yaml

# Create product
az apim product create \
  --resource-group myRG \
  --service-name my-apim \
  --product-id premium \
  --display-name "Premium API" \
  --subscription-required true \
  --approval-required false

# Add API to product
az apim product api add \
  --resource-group myRG \
  --service-name my-apim \
  --product-id premium \
  --api-id orders-api
```

## Service Bus CLI Commands

### Queue Operations

```bash
# Create namespace
az servicebus namespace create \
  --resource-group myRG \
  --name my-sb-namespace \
  --location eastus \
  --sku Premium

# Create queue
az servicebus queue create \
  --resource-group myRG \
  --namespace-name my-sb-namespace \
  --name orders \
  --enable-partitioning false \
  --max-size 5120 \
  --default-message-time-to-live P14D \
  --lock-duration PT1M \
  --dead-lettering-on-message-expiration true

# Create topic
az servicebus topic create \
  --resource-group myRG \
  --namespace-name my-sb-namespace \
  --name notifications \
  --enable-partitioning false

# Create subscription
az servicebus topic subscription create \
  --resource-group myRG \
  --namespace-name my-sb-namespace \
  --topic-name notifications \
  --name email-handler
```

## Event Grid CLI Commands

```bash
# Create custom topic
az eventgrid topic create \
  --resource-group myRG \
  --name my-topic \
  --location eastus

# Create event subscription (to Function)
az eventgrid event-subscription create \
  --name my-subscription \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/my-topic \
  --endpoint /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function/functions/ProcessEvent \
  --endpoint-type azurefunction

# Create subscription with filter
az eventgrid event-subscription create \
  --name filtered-subscription \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/my-topic \
  --endpoint https://my-webhook.com/api/events \
  --endpoint-type webhook \
  --advanced-filter data.priority NumberGreaterThan 5
```

## Common APIM Policies

### Rate Limiting

```xml
<policies>
    <inbound>
        <!-- Rate limit by subscription key -->
        <rate-limit-by-key calls="100"
                          renewal-period="60"
                          counter-key="@(context.Subscription.Key)" />

        <!-- Quota by subscription -->
        <quota-by-key calls="10000"
                      renewal-period="86400"
                      counter-key="@(context.Subscription.Key)" />
    </inbound>
</policies>
```

### JWT Validation

```xml
<policies>
    <inbound>
        <validate-jwt header-name="Authorization"
                      require-scheme="Bearer"
                      failed-validation-error-message="Unauthorized">
            <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
            <audiences>
                <audience>api://my-api</audience>
            </audiences>
            <issuers>
                <issuer>https://sts.windows.net/{tenant}/</issuer>
            </issuers>
            <required-claims>
                <claim name="roles" match="any">
                    <value>API.Read</value>
                    <value>API.Write</value>
                </claim>
            </required-claims>
        </validate-jwt>
    </inbound>
</policies>
```

### Request Transformation

```xml
<policies>
    <inbound>
        <!-- Add header -->
        <set-header name="X-Request-ID" exists-action="skip">
            <value>@(Guid.NewGuid().ToString())</value>
        </set-header>

        <!-- Transform body -->
        <set-body>@{
            var body = context.Request.Body.As<JObject>();
            body["timestamp"] = DateTime.UtcNow.ToString("o");
            return body.ToString();
        }</set-body>
    </inbound>

    <outbound>
        <!-- Remove sensitive headers -->
        <set-header name="X-Powered-By" exists-action="delete" />
        <set-header name="X-AspNet-Version" exists-action="delete" />
    </outbound>
</policies>
```

### Caching

```xml
<policies>
    <inbound>
        <cache-lookup vary-by-developer="false"
                      vary-by-developer-groups="false"
                      downstream-caching-type="none">
            <vary-by-header>Accept</vary-by-header>
            <vary-by-query-parameter>version</vary-by-query-parameter>
        </cache-lookup>
    </inbound>
    <outbound>
        <cache-store duration="3600" />
    </outbound>
</policies>
```

## Service Bus Code Snippets

### Send Message (C#)

```csharp
using Azure.Messaging.ServiceBus;

// Create client
await using var client = new ServiceBusClient(connectionString);
await using var sender = client.CreateSender("orders");

// Send single message
var message = new ServiceBusMessage(JsonSerializer.Serialize(order))
{
    ContentType = "application/json",
    MessageId = order.Id,
    CorrelationId = correlationId,
    Subject = "NewOrder",
    TimeToLive = TimeSpan.FromDays(1)
};

await sender.SendMessageAsync(message);

// Send batch
var batch = await sender.CreateMessageBatchAsync();
foreach (var item in orders)
{
    if (!batch.TryAddMessage(new ServiceBusMessage(JsonSerializer.Serialize(item))))
    {
        await sender.SendMessagesAsync(batch);
        batch = await sender.CreateMessageBatchAsync();
        batch.TryAddMessage(new ServiceBusMessage(JsonSerializer.Serialize(item)));
    }
}
await sender.SendMessagesAsync(batch);
```

### Receive Messages (C#)

```csharp
// Create processor
await using var processor = client.CreateProcessor("orders", new ServiceBusProcessorOptions
{
    MaxConcurrentCalls = 10,
    AutoCompleteMessages = false,
    PrefetchCount = 100
});

// Message handler
processor.ProcessMessageAsync += async args =>
{
    var order = JsonSerializer.Deserialize<Order>(args.Message.Body);

    try
    {
        await ProcessOrderAsync(order);
        await args.CompleteMessageAsync(args.Message);
    }
    catch (Exception ex)
    {
        // Will retry or go to dead letter
        await args.AbandonMessageAsync(args.Message);
    }
};

// Error handler
processor.ProcessErrorAsync += async args =>
{
    _logger.LogError(args.Exception, "Service Bus processing error");
};

await processor.StartProcessingAsync();
```

## Event Grid Code Snippets

### Publish Events (C#)

```csharp
using Azure.Messaging.EventGrid;

var client = new EventGridPublisherClient(
    new Uri("https://my-topic.eastus-1.eventgrid.azure.net/api/events"),
    new AzureKeyCredential(topicKey));

// Publish CloudEvent
var cloudEvent = new CloudEvent(
    source: "/orders/processor",
    type: "Order.Created",
    data: new { OrderId = "12345", Amount = 99.99 });

await client.SendEventAsync(cloudEvent);

// Publish batch
var events = orders.Select(o => new CloudEvent(
    source: "/orders/processor",
    type: "Order.Created",
    data: o)).ToList();

await client.SendEventsAsync(events);
```

### Handle Events (Azure Function)

```csharp
[FunctionName("ProcessOrderEvent")]
public async Task Run(
    [EventGridTrigger] CloudEvent cloudEvent,
    ILogger log)
{
    var order = cloudEvent.Data.ToObjectFromJson<Order>();
    log.LogInformation($"Processing order: {order.Id}");

    // Process event
    await ProcessOrderAsync(order);
}
```

## Logic App Common Patterns

### HTTP Trigger Workflow (JSON)

```json
{
    "definition": {
        "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
        "actions": {
            "Parse_JSON": {
                "inputs": {
                    "content": "@triggerBody()",
                    "schema": {
                        "properties": {
                            "orderId": { "type": "string" },
                            "amount": { "type": "number" }
                        }
                    }
                },
                "runAfter": {},
                "type": "ParseJson"
            },
            "Send_Email": {
                "inputs": {
                    "body": {
                        "Body": "Order @{body('Parse_JSON')?['orderId']} received",
                        "Subject": "New Order",
                        "To": "orders@company.com"
                    },
                    "host": {
                        "connection": {
                            "name": "@parameters('$connections')['office365']['connectionId']"
                        }
                    },
                    "method": "post",
                    "path": "/v2/Mail"
                },
                "runAfter": {
                    "Parse_JSON": ["Succeeded"]
                },
                "type": "ApiConnection"
            }
        },
        "triggers": {
            "manual": {
                "inputs": {
                    "schema": {}
                },
                "kind": "Http",
                "type": "Request"
            }
        }
    }
}
```

## Bicep Snippets

### API Management

```bicep
resource apim 'Microsoft.ApiManagement/service@2023-03-01-preview' = {
  name: 'apim-${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Developer'
    capacity: 1
  }
  properties: {
    publisherEmail: 'admin@company.com'
    publisherName: 'My Company'
  }
}

resource api 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  parent: apim
  name: 'orders-api'
  properties: {
    displayName: 'Orders API'
    path: 'orders'
    protocols: ['https']
    subscriptionRequired: true
  }
}
```

### Service Bus

```bicep
resource sbNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Premium'
    tier: 'Premium'
    capacity: 1
  }
}

resource queue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: sbNamespace
  name: 'orders'
  properties: {
    maxSizeInMegabytes: 5120
    defaultMessageTimeToLive: 'P14D'
    lockDuration: 'PT1M'
    deadLetteringOnMessageExpiration: true
    requiresDuplicateDetection: true
    duplicateDetectionHistoryTimeWindow: 'PT10M'
  }
}
```

---

*Back to [API Integration Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
