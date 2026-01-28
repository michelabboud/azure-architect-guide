# Azure Event Grid

## What is Azure Event Grid?

Azure Event Grid is a fully managed event routing service that enables event-driven, reactive programming. It uses a publish-subscribe model to allow uniform event consumption using a push-based delivery mechanism.

### Event Grid vs AWS EventBridge

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   EVENT GRID vs AWS EVENTBRIDGE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS EVENTBRIDGE                         AZURE EVENT GRID                   │
│  ───────────────                         ────────────────                   │
│                                                                              │
│  Architecture:                           Architecture:                      │
│  ┌─────────────────────┐                ┌─────────────────────────────────┐ │
│  │   Event Bus         │                │     System Topics               │ │
│  │   (per account)     │                │     (Azure service events)      │ │
│  │                     │                │                                 │ │
│  │   Rules + Targets   │                │     Custom Topics               │ │
│  │                     │                │     (Your application events)   │ │
│  │   Schema Registry   │                │                                 │ │
│  │   (separate)        │                │     Event Domains               │ │
│  │                     │                │     (Multi-tenant)              │ │
│  │   Partner Events    │                │                                 │ │
│  │   (SaaS sources)    │                │     Partner Topics              │ │
│  └─────────────────────┘                └─────────────────────────────────┘ │
│                                                                              │
│  FEATURE COMPARISON:                                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Feature           │ EventBridge          │ Event Grid              │  │
│  ├───────────────────┼──────────────────────┼─────────────────────────┤  │
│  │ Event format      │ CloudEvents +        │ Event Grid schema +     │  │
│  │                   │ Custom               │ CloudEvents            │  │
│  │                   │                      │                         │  │
│  │ Max event size    │ 256 KB               │ 1 MB                    │  │
│  │                   │                      │                         │  │
│  │ Filtering         │ Content-based rules  │ Subject prefix/suffix   │  │
│  │                   │ (JSON path)          │ Event type              │  │
│  │                   │                      │ Advanced filters        │  │
│  │                   │                      │                         │  │
│  │ Targets/Handlers  │ 20+ AWS targets      │ Azure Functions         │  │
│  │                   │                      │ Event Hubs              │  │
│  │                   │                      │ Service Bus             │  │
│  │                   │                      │ Storage Queue           │  │
│  │                   │                      │ Webhooks                │  │
│  │                   │                      │ Logic Apps              │  │
│  │                   │                      │                         │  │
│  │ Replay            │ Archive & Replay     │ No built-in replay     │  │
│  │                   │                      │ (use dead-letter)       │  │
│  │                   │                      │                         │  │
│  │ Schema Registry   │ Yes (integrated)     │ No built-in            │  │
│  │                   │                      │                         │  │
│  │ Pricing           │ Per million events   │ Per million operations  │  │
│  │                   │ ($1.00/million)      │ ($0.60/million)         │  │
│  │                   │                      │                         │  │
│  │ SLA               │ 99.99%               │ 99.99%                  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  KEY DIFFERENCES:                                                           │
│  • Event Grid is push-first (webhooks); EventBridge uses targets           │
│  • Event Grid has native Azure service integration (system topics)         │
│  • EventBridge has built-in archive/replay; Event Grid needs dead-letter   │
│  • Event Grid supports Event Domains for multi-tenant scenarios            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Event Grid Concepts

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EVENT GRID ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  EVENT SOURCES                                                              │
│  (Publishers)                                                               │
│  ─────────────                                                              │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                                                                        │  │
│  │  AZURE SERVICES (System Topics):       CUSTOM (Custom Topics):        │  │
│  │  ┌────────────┐ ┌────────────┐        ┌────────────┐                 │  │
│  │  │  Blob      │ │ Resource   │        │  Your App  │                 │  │
│  │  │  Storage   │ │  Groups    │        │  (HTTP)    │                 │  │
│  │  └─────┬──────┘ └─────┬──────┘        └─────┬──────┘                 │  │
│  │        │              │                     │                         │  │
│  │  ┌────────────┐ ┌────────────┐        ┌────────────┐                 │  │
│  │  │ Event Hubs │ │    IoT     │        │  Partner   │                 │  │
│  │  │            │ │    Hub     │        │  Events    │                 │  │
│  │  └─────┬──────┘ └─────┬──────┘        └─────┬──────┘                 │  │
│  │        │              │                     │                         │  │
│  └────────┴──────────────┴─────────────────────┴─────────────────────────┘  │
│                          │                                                   │
│                          ▼                                                   │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        EVENT GRID                                      │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Topics (Custom or System)                                       │  │  │
│  │  │  ┌───────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │  Event Subscriptions                                       │  │  │  │
│  │  │  │  • Filtering (event type, subject, data)                  │  │  │  │
│  │  │  │  • Delivery (webhook, queue, function, etc.)              │  │  │  │
│  │  │  │  • Retry + Dead-letter                                    │  │  │  │
│  │  │  └───────────────────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                          │                                                   │
│       ┌──────────────────┼──────────────────┐                              │
│       │                  │                  │                              │
│       ▼                  ▼                  ▼                              │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐                         │
│  │  Azure   │      │ Webhook  │      │ Service  │                         │
│  │ Function │      │ (HTTP)   │      │   Bus    │                         │
│  └──────────┘      └──────────┘      └──────────┘                         │
│                                                                              │
│  EVENT HANDLERS                                                             │
│  (Subscribers)                                                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Event Schema

```json
// Event Grid Schema
{
    "id": "b8c4f8e0-1234-5678-9abc-def012345678",
    "topic": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{account}",
    "subject": "/blobServices/default/containers/images/blobs/photo.jpg",
    "eventType": "Microsoft.Storage.BlobCreated",
    "eventTime": "2024-01-15T10:30:00.0000000Z",
    "data": {
        "api": "PutBlob",
        "clientRequestId": "6d79dbfb-0e37-4fc4-981f-442c9ca65760",
        "requestId": "831e1650-001e-001b-66ab-eeb76e000000",
        "eTag": "\"0x8D4BCC2E4835CD0\"",
        "contentType": "image/jpeg",
        "contentLength": 524288,
        "blobType": "BlockBlob",
        "url": "https://account.blob.core.windows.net/images/photo.jpg"
    },
    "dataVersion": "1",
    "metadataVersion": "1"
}

// CloudEvents Schema (alternative)
{
    "specversion": "1.0",
    "type": "Microsoft.Storage.BlobCreated",
    "source": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{account}",
    "id": "b8c4f8e0-1234-5678-9abc-def012345678",
    "time": "2024-01-15T10:30:00.0000000Z",
    "subject": "/blobServices/default/containers/images/blobs/photo.jpg",
    "datacontenttype": "application/json",
    "data": {
        // Same as above
    }
}
```

---

## Event Sources and Handlers

### System Topics (Azure Service Events)

```
SYSTEM TOPICS - BUILT-IN AZURE SERVICE EVENTS:
──────────────────────────────────────────────

STORAGE ACCOUNTS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Event Type                        │ When Triggered                         │
├───────────────────────────────────┼────────────────────────────────────────┤
│ Microsoft.Storage.BlobCreated     │ Blob created or updated                │
│ Microsoft.Storage.BlobDeleted     │ Blob deleted                           │
│ Microsoft.Storage.BlobRenamed     │ Blob renamed (ADLS Gen2)               │
│ Microsoft.Storage.BlobTierChanged │ Blob tier changed                      │
│ Microsoft.Storage.DirectoryCreated│ Directory created (ADLS Gen2)          │
│ Microsoft.Storage.DirectoryDeleted│ Directory deleted (ADLS Gen2)          │
└────────────────────────────────────────────────────────────────────────────┘

RESOURCE GROUPS / SUBSCRIPTIONS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Event Type                              │ When Triggered                   │
├─────────────────────────────────────────┼──────────────────────────────────┤
│ Microsoft.Resources.ResourceWriteSuccess│ Resource created/updated         │
│ Microsoft.Resources.ResourceWriteFailure│ Resource creation failed         │
│ Microsoft.Resources.ResourceDeleteSuccess│ Resource deleted                │
│ Microsoft.Resources.ResourceDeleteFailure│ Resource deletion failed        │
│ Microsoft.Resources.ResourceActionSuccess│ POST action succeeded           │
│ Microsoft.Resources.ResourceActionFailure│ POST action failed              │
└────────────────────────────────────────────────────────────────────────────┘

CONTAINER REGISTRY:
┌────────────────────────────────────────────────────────────────────────────┐
│ Microsoft.ContainerRegistry.ImagePushed  │ Image pushed to registry        │
│ Microsoft.ContainerRegistry.ImageDeleted │ Image deleted from registry     │
│ Microsoft.ContainerRegistry.ChartPushed  │ Helm chart pushed               │
│ Microsoft.ContainerRegistry.ChartDeleted │ Helm chart deleted              │
└────────────────────────────────────────────────────────────────────────────┘

KEY VAULT:
┌────────────────────────────────────────────────────────────────────────────┐
│ Microsoft.KeyVault.SecretNewVersionCreated │ New secret version           │
│ Microsoft.KeyVault.SecretNearExpiry        │ Secret near expiry           │
│ Microsoft.KeyVault.SecretExpired           │ Secret expired               │
│ Microsoft.KeyVault.CertificateNewVersionCreated │ New cert version        │
│ Microsoft.KeyVault.CertificateNearExpiry   │ Certificate near expiry      │
│ Microsoft.KeyVault.CertificateExpired      │ Certificate expired          │
└────────────────────────────────────────────────────────────────────────────┘

IOT HUB:
┌────────────────────────────────────────────────────────────────────────────┐
│ Microsoft.Devices.DeviceCreated     │ Device registered                    │
│ Microsoft.Devices.DeviceDeleted     │ Device deleted                       │
│ Microsoft.Devices.DeviceConnected   │ Device connected                     │
│ Microsoft.Devices.DeviceDisconnected│ Device disconnected                  │
│ Microsoft.Devices.DeviceTelemetry   │ Telemetry received                   │
└────────────────────────────────────────────────────────────────────────────┘

OTHER SERVICES:
• App Configuration (key-value changes)
• Azure SignalR (client connections)
• Azure Maps (geofence events)
• Azure Policy (compliance changes)
• Azure Communication Services (SMS, email)
• Microsoft Entra ID (user events) - via Microsoft Graph
```

### Event Handlers

```
EVENT HANDLERS (Subscribers):
─────────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│ Handler Type         │ Use Case                    │ Configuration         │
├──────────────────────┼─────────────────────────────┼───────────────────────┤
│                      │                             │                       │
│ AZURE FUNCTIONS      │ Serverless event processing │ Function endpoint     │
│ [Recommended]        │ Real-time, scalable         │ System/User assigned  │
│                      │                             │ managed identity      │
│                      │                             │                       │
├──────────────────────┼─────────────────────────────┼───────────────────────┤
│                      │                             │                       │
│ WEBHOOKS             │ Custom HTTP endpoints       │ HTTPS URL             │
│                      │ Third-party services        │ Validation required   │
│                      │ On-premises systems         │ AAD auth supported    │
│                      │                             │                       │
├──────────────────────┼─────────────────────────────┼───────────────────────┤
│                      │                             │                       │
│ EVENT HUBS           │ High-throughput streaming   │ Event Hub + namespace │
│                      │ Telemetry aggregation       │                       │
│                      │ Analytics pipelines         │                       │
│                      │                             │                       │
├──────────────────────┼─────────────────────────────┼───────────────────────┤
│                      │                             │                       │
│ SERVICE BUS          │ Reliable message delivery   │ Queue or Topic        │
│ (Queue/Topic)        │ Ordered processing          │                       │
│                      │ Transactional handling      │                       │
│                      │                             │                       │
├──────────────────────┼─────────────────────────────┼───────────────────────┤
│                      │                             │                       │
│ STORAGE QUEUE        │ Simple queuing              │ Queue name            │
│                      │ Cost-effective              │                       │
│                      │ Large backlogs              │                       │
│                      │                             │                       │
├──────────────────────┼─────────────────────────────┼───────────────────────┤
│                      │                             │                       │
│ LOGIC APPS           │ Workflow automation         │ Trigger endpoint      │
│                      │ Integration scenarios       │                       │
│                      │ Low-code processing         │                       │
│                      │                             │                       │
├──────────────────────┼─────────────────────────────┼───────────────────────┤
│                      │                             │                       │
│ HYBRID CONNECTIONS   │ On-premises delivery        │ Relay namespace       │
│                      │ Without public endpoint     │                       │
│                      │                             │                       │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Custom Topics vs System Topics

### Comparison

```
┌────────────────────────────────────────────────────────────────────────────┐
│              CUSTOM TOPICS vs SYSTEM TOPICS                                 │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SYSTEM TOPICS:                        CUSTOM TOPICS:                      │
│  ──────────────                        ──────────────                      │
│                                                                             │
│  ┌─────────────────────────────┐      ┌─────────────────────────────┐     │
│  │ • Created automatically     │      │ • You create and manage     │     │
│  │   for Azure resources       │      │                             │     │
│  │                             │      │ • Publish from your apps    │     │
│  │ • Events from Azure         │      │                             │     │
│  │   services                  │      │ • Full control over         │     │
│  │                             │      │   event schema              │     │
│  │ • No publishing required    │      │                             │     │
│  │   (automatic)               │      │ • Endpoint + access key     │     │
│  │                             │      │   for publishing            │     │
│  │ • Scoped to source          │      │                             │     │
│  │   resource                  │      │ • Regional or global        │     │
│  │                             │      │   (with domains)            │     │
│  │ • No additional cost        │      │                             │     │
│  │   for topic itself          │      │ • Charged for topic +       │     │
│  │                             │      │   operations                │     │
│  └─────────────────────────────┘      └─────────────────────────────┘     │
│                                                                             │
│  WHEN TO USE EACH:                                                         │
│                                                                             │
│  System Topics:                        Custom Topics:                      │
│  • React to Azure resource changes    • Application-generated events       │
│  • Storage blob events                • Domain events (OrderCreated)       │
│  • Key Vault secret rotation          • Integration with external systems  │
│  • Container image pushes             • Custom event schema requirements   │
│  • IoT device telemetry               • Multi-tenant event routing        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Creating System Topics

```bash
# Create system topic for storage account
az eventgrid system-topic create \
  --resource-group myRG \
  --name storage-events \
  --location eastus \
  --topic-type Microsoft.Storage.StorageAccounts \
  --source /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount

# Create subscription for blob events
az eventgrid system-topic event-subscription create \
  --resource-group myRG \
  --system-topic-name storage-events \
  --name blob-created-handler \
  --endpoint /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function/functions/ProcessBlob \
  --endpoint-type azurefunction \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/uploads/
```

### Creating Custom Topics

```bash
# Create custom topic
az eventgrid topic create \
  --resource-group myRG \
  --name order-events \
  --location eastus \
  --input-schema eventgridschema

# Get topic endpoint and key
az eventgrid topic show \
  --resource-group myRG \
  --name order-events \
  --query "endpoint" --output tsv

az eventgrid topic key list \
  --resource-group myRG \
  --name order-events \
  --query "key1" --output tsv
```

---

## Event Domains for Multi-Tenant

### Event Domain Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    EVENT DOMAINS                                            │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  PROBLEM: Managing thousands of topics for multi-tenant SaaS              │
│                                                                             │
│  WITHOUT DOMAINS:                      WITH DOMAINS:                       │
│  ────────────────                      ─────────────                       │
│                                                                             │
│  ┌─────────────────────┐              ┌─────────────────────────────────┐  │
│  │ Topic: tenant-001   │              │         EVENT DOMAIN            │  │
│  │ Topic: tenant-002   │              │                                 │  │
│  │ Topic: tenant-003   │              │  ┌─────────────────────────┐   │  │
│  │ Topic: tenant-004   │              │  │ Domain Topic: tenant-001│   │  │
│  │ ...                 │              │  │ Domain Topic: tenant-002│   │  │
│  │ Topic: tenant-999   │              │  │ Domain Topic: tenant-003│   │  │
│  └─────────────────────┘              │  │ ...                     │   │  │
│                                        │  │ Domain Topic: tenant-999│   │  │
│  • Manage 999 separate topics         │  └─────────────────────────┘   │  │
│  • 999 separate endpoints             │                                 │  │
│  • Complex permission management      │  • Single endpoint              │  │
│  • Hitting topic limits              │  • Up to 100,000 domain topics  │  │
│                                        │  • Unified management           │  │
│                                        │  • Per-topic permissions        │  │
│                                        └─────────────────────────────────┘  │
│                                                                             │
│  PUBLISHING TO DOMAIN:                                                     │
│  ─────────────────────                                                     │
│                                                                             │
│  POST https://{domain-name}.{region}.eventgrid.azure.net/api/events       │
│                                                                             │
│  [                                                                          │
│    {                                                                        │
│      "id": "event-1",                                                      │
│      "topic": "tenant-001",        ← Specifies which domain topic         │
│      "eventType": "Order.Created",                                         │
│      "subject": "/orders/12345",                                           │
│      "data": { "orderId": "12345" },                                       │
│      "eventTime": "2024-01-15T10:30:00Z"                                   │
│    }                                                                        │
│  ]                                                                          │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Event Domain Example

```bash
# Create event domain
az eventgrid domain create \
  --resource-group myRG \
  --name saas-events \
  --location eastus \
  --input-schema eventgridschema

# Domain topics are created automatically when events are published
# But you can also create them explicitly:
az eventgrid domain topic create \
  --resource-group myRG \
  --domain-name saas-events \
  --name tenant-contoso

# Create subscription for specific tenant's topic
az eventgrid event-subscription create \
  --name contoso-handler \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/domains/saas-events/topics/tenant-contoso \
  --endpoint https://contoso-app.azurewebsites.net/api/events
```

---

## Filtering and Routing

### Filter Types

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    EVENT FILTERING                                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  SUBJECT FILTERING (Simple, fast):                                         │
│  ─────────────────────────────────                                         │
│                                                                             │
│  Subject: /blobServices/default/containers/images/blobs/photo.jpg         │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ subjectBeginsWith: "/blobServices/default/containers/images/"       │   │
│  │ subjectEndsWith: ".jpg"                                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  EVENT TYPE FILTERING:                                                     │
│  ─────────────────────                                                     │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ includedEventTypes:                                                  │   │
│  │   - Microsoft.Storage.BlobCreated                                   │   │
│  │   - Microsoft.Storage.BlobDeleted                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ADVANCED FILTERING (Data field filtering):                                │
│  ──────────────────────────────────────────                                │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  OPERATOR         │ DESCRIPTION                │ EXAMPLE            │   │
│  │  ─────────────────┼────────────────────────────┼────────────────────│   │
│  │  NumberIn         │ Value in list of numbers   │ priority NumberIn  │   │
│  │                   │                            │   [1, 2, 3]        │   │
│  │                   │                            │                    │   │
│  │  NumberNotIn      │ Value not in list          │ status NumberNotIn │   │
│  │                   │                            │   [0]              │   │
│  │                   │                            │                    │   │
│  │  NumberLessThan   │ Value less than            │ amount Number      │   │
│  │                   │                            │   LessThan 100     │   │
│  │                   │                            │                    │   │
│  │  NumberGreaterThan│ Value greater than         │ priority Number    │   │
│  │                   │                            │   GreaterThan 5    │   │
│  │                   │                            │                    │   │
│  │  StringContains   │ String contains            │ subject String     │   │
│  │                   │                            │   Contains "error" │   │
│  │                   │                            │                    │   │
│  │  StringBeginsWith │ String starts with         │ data.url String    │   │
│  │                   │                            │   BeginsWith "https│   │
│  │                   │                            │                    │   │
│  │  StringEndsWith   │ String ends with           │ data.filename      │   │
│  │                   │                            │   StringEndsWith   │   │
│  │                   │                            │   ".pdf"           │   │
│  │                   │                            │                    │   │
│  │  BoolEquals       │ Boolean equals             │ data.isUrgent      │   │
│  │                   │                            │   BoolEquals true  │   │
│  │                   │                            │                    │   │
│  │  IsNullOrUndefined│ Field is null/missing      │ data.errorCode     │   │
│  │                   │                            │   IsNotNull        │   │
│  │                   │                            │                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Advanced Filtering Examples

```bash
# Create subscription with advanced filters
az eventgrid event-subscription create \
  --name high-priority-orders \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events \
  --endpoint /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function/functions/ProcessHighPriority \
  --endpoint-type azurefunction \
  --advanced-filter data.priority NumberGreaterThan 7 \
  --advanced-filter data.amount NumberGreaterThan 1000 \
  --advanced-filter data.region StringIn US EU

# Multiple filters = AND condition
# Above: priority > 7 AND amount > 1000 AND region IN ('US', 'EU')

# Create subscription with subject filter
az eventgrid event-subscription create \
  --name image-processor \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --endpoint /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function/functions/ProcessImage \
  --endpoint-type azurefunction \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/uploads/ \
  --subject-ends-with .jpg
```

---

## Delivery and Retry

### Retry Policy

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    DELIVERY AND RETRY                                       │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  DEFAULT RETRY POLICY:                                                     │
│  ─────────────────────                                                     │
│                                                                             │
│  • Maximum retries: 30 attempts                                            │
│  • Time-to-live: 24 hours                                                  │
│  • Exponential backoff with jitter                                         │
│                                                                             │
│  RETRY SCHEDULE (Approximate):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Attempt 1:  Immediate                                                │   │
│  │ Attempt 2:  10 seconds                                               │   │
│  │ Attempt 3:  30 seconds                                               │   │
│  │ Attempt 4:  1 minute                                                 │   │
│  │ Attempt 5:  5 minutes                                                │   │
│  │ Attempt 6+: 10 minutes (max interval)                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SUCCESSFUL DELIVERY:                                                      │
│  ────────────────────                                                      │
│  HTTP Status 200-299 = Success                                            │
│                                                                             │
│  RETRY TRIGGERS:                                                           │
│  ───────────────                                                           │
│  • HTTP 400, 401, 403, 404, 413 = Dead-letter (no retry)                  │
│  • HTTP 408, 429 = Retry with backoff                                      │
│  • HTTP 500, 502, 503, 504 = Retry with backoff                           │
│  • Timeout = Retry                                                         │
│                                                                             │
│  DEAD-LETTER:                                                              │
│  ────────────                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Events that can't be delivered after all retries:                    │   │
│  │                                                                      │   │
│  │ • Sent to configured dead-letter destination (Storage blob)         │   │
│  │ • Includes: Original event + delivery attempt info + error          │   │
│  │ • Retention: Based on storage account settings                      │   │
│  │                                                                      │   │
│  │ Dead-letter event format:                                           │   │
│  │ {                                                                    │   │
│  │   "deadLetterReason": "MaxDeliveryAttemptsExceeded",               │   │
│  │   "deliveryAttempts": 30,                                           │   │
│  │   "lastDeliveryOutcome": "ServiceUnavailable",                     │   │
│  │   "lastHttpStatusCode": 503,                                        │   │
│  │   "event": { ... original event ... }                               │   │
│  │ }                                                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Configuring Dead-Letter

```bash
# Create subscription with dead-letter destination
az eventgrid event-subscription create \
  --name order-processor \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events \
  --endpoint https://my-api.azurewebsites.net/api/events \
  --deadletter-endpoint /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount/blobServices/default/containers/deadletter \
  --max-delivery-attempts 10 \
  --event-ttl 1440
```

---

## Bicep/CLI Deployment Examples

### Complete Event Grid Deployment (Bicep)

```bicep
// main.bicep - Complete Event Grid deployment

@description('Location for all resources')
param location string = resourceGroup().location

@description('Name of the custom topic')
param topicName string = 'orders-${uniqueString(resourceGroup().id)}'

@description('Name of the Function App for event handling')
param functionAppName string

@description('Storage account for dead-lettering')
param storageAccountName string

// Custom Topic
resource eventGridTopic 'Microsoft.EventGrid/topics@2023-12-15-preview' = {
  name: topicName
  location: location
  properties: {
    inputSchema: 'EventGridSchema'
    publicNetworkAccess: 'Enabled'
    inboundIpRules: []
    disableLocalAuth: false
  }
  identity: {
    type: 'SystemAssigned'
  }
}

// Reference existing Function App
resource functionApp 'Microsoft.Web/sites@2023-01-01' existing = {
  name: functionAppName
}

// Reference existing Storage Account for dead-letter
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: storageAccountName
}

// Dead-letter container
resource deadLetterContainer 'Microsoft.Storage/storageAccounts/blobServices/containers@2023-01-01' = {
  name: '${storageAccountName}/default/eventgrid-deadletter'
  properties: {
    publicAccess: 'None'
  }
}

// Event Subscription - High Priority Orders to Function
resource highPrioritySubscription 'Microsoft.EventGrid/topics/eventSubscriptions@2023-12-15-preview' = {
  parent: eventGridTopic
  name: 'high-priority-orders'
  properties: {
    destination: {
      endpointType: 'AzureFunction'
      properties: {
        resourceId: '${functionApp.id}/functions/ProcessHighPriorityOrder'
        maxEventsPerBatch: 1
        preferredBatchSizeInKilobytes: 64
      }
    }
    filter: {
      includedEventTypes: [
        'Order.Created'
        'Order.Updated'
      ]
      advancedFilters: [
        {
          operatorType: 'NumberGreaterThan'
          key: 'data.priority'
          value: 7
        }
      ]
    }
    eventDeliverySchema: 'EventGridSchema'
    retryPolicy: {
      maxDeliveryAttempts: 30
      eventTimeToLiveInMinutes: 1440
    }
    deadLetterDestination: {
      endpointType: 'StorageBlob'
      properties: {
        resourceId: storageAccount.id
        blobContainerName: 'eventgrid-deadletter'
      }
    }
  }
}

// Event Subscription - All Orders to Service Bus
resource allOrdersSubscription 'Microsoft.EventGrid/topics/eventSubscriptions@2023-12-15-preview' = {
  parent: eventGridTopic
  name: 'all-orders-to-servicebus'
  properties: {
    destination: {
      endpointType: 'ServiceBusQueue'
      properties: {
        resourceId: serviceBusQueue.id
      }
    }
    filter: {
      includedEventTypes: [
        'Order.Created'
        'Order.Updated'
        'Order.Completed'
        'Order.Cancelled'
      ]
      subjectBeginsWith: '/orders/'
    }
    eventDeliverySchema: 'CloudEventSchemaV1_0'
  }
}

// Service Bus Queue for events
resource serviceBusNamespace 'Microsoft.ServiceBus/namespaces@2022-10-01-preview' = {
  name: 'sb-${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Standard'
    tier: 'Standard'
  }
}

resource serviceBusQueue 'Microsoft.ServiceBus/namespaces/queues@2022-10-01-preview' = {
  parent: serviceBusNamespace
  name: 'order-events'
  properties: {
    maxSizeInMegabytes: 5120
    defaultMessageTimeToLive: 'P14D'
  }
}

// System Topic for Storage Events
resource storageSystemTopic 'Microsoft.EventGrid/systemTopics@2023-12-15-preview' = {
  name: 'storage-events'
  location: location
  properties: {
    source: storageAccount.id
    topicType: 'Microsoft.Storage.StorageAccounts'
  }
}

resource blobCreatedSubscription 'Microsoft.EventGrid/systemTopics/eventSubscriptions@2023-12-15-preview' = {
  parent: storageSystemTopic
  name: 'blob-created'
  properties: {
    destination: {
      endpointType: 'AzureFunction'
      properties: {
        resourceId: '${functionApp.id}/functions/ProcessBlobCreated'
      }
    }
    filter: {
      includedEventTypes: [
        'Microsoft.Storage.BlobCreated'
      ]
      subjectBeginsWith: '/blobServices/default/containers/uploads/'
      advancedFilters: [
        {
          operatorType: 'StringEndsWith'
          key: 'subject'
          values: ['.pdf', '.doc', '.docx']
        }
      ]
    }
  }
}

// Outputs
output topicEndpoint string = eventGridTopic.properties.endpoint
output topicId string = eventGridTopic.id
output systemTopicId string = storageSystemTopic.id
```

### Azure CLI Commands

```bash
# Create custom topic
az eventgrid topic create \
  --resource-group myRG \
  --name order-events \
  --location eastus \
  --input-schema eventgridschema

# Create system topic for storage
az eventgrid system-topic create \
  --resource-group myRG \
  --name storage-events \
  --location eastus \
  --topic-type Microsoft.Storage.StorageAccounts \
  --source /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount

# Create subscription to Azure Function
az eventgrid event-subscription create \
  --name process-orders \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events \
  --endpoint /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function-app/functions/ProcessOrder \
  --endpoint-type azurefunction

# Create subscription to webhook with validation
az eventgrid event-subscription create \
  --name webhook-handler \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events \
  --endpoint https://my-api.azurewebsites.net/api/events \
  --endpoint-type webhook \
  --azure-active-directory-tenant-id {tenant-id} \
  --azure-active-directory-application-id-or-uri api://my-api

# Create subscription to Service Bus queue
az eventgrid event-subscription create \
  --name servicebus-handler \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events \
  --endpoint /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.ServiceBus/namespaces/my-sb/queues/order-events \
  --endpoint-type servicebusqueue

# Create subscription with advanced filters
az eventgrid event-subscription create \
  --name premium-orders \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events \
  --endpoint /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Web/sites/my-function/functions/ProcessPremium \
  --endpoint-type azurefunction \
  --advanced-filter data.customerTier StringIn Premium Gold \
  --advanced-filter data.amount NumberGreaterThanOrEquals 500

# List subscriptions
az eventgrid event-subscription list \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events

# Show delivery metrics
az eventgrid event-subscription show \
  --name process-orders \
  --source-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.EventGrid/topics/order-events \
  --include-full-endpoint-url
```

### Publishing Events (C#)

```csharp
using Azure;
using Azure.Messaging.EventGrid;

// Using EventGridPublisherClient
var client = new EventGridPublisherClient(
    new Uri("https://order-events.eastus-1.eventgrid.azure.net/api/events"),
    new AzureKeyCredential(topicKey));

// Publish Event Grid event
var egEvent = new EventGridEvent(
    subject: "/orders/12345",
    eventType: "Order.Created",
    dataVersion: "1.0",
    data: new
    {
        OrderId = "12345",
        CustomerId = "C001",
        Amount = 299.99,
        Priority = 8,
        Items = new[] { "SKU-001", "SKU-002" }
    });

await client.SendEventAsync(egEvent);

// Publish CloudEvent
var cloudEvent = new CloudEvent(
    source: "/orders/processor",
    type: "Order.Created",
    data: new
    {
        OrderId = "12345",
        CustomerId = "C001",
        Amount = 299.99
    });

await client.SendEventAsync(cloudEvent);

// Publish batch
var events = orders.Select(order => new EventGridEvent(
    subject: $"/orders/{order.Id}",
    eventType: "Order.Created",
    dataVersion: "1.0",
    data: order)).ToList();

await client.SendEventsAsync(events);
```

---

## Best Practices

```
EVENT GRID BEST PRACTICES:
──────────────────────────

1. EVENT DESIGN
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use meaningful subject paths for filtering (/orders/{id})        │
   │ • Keep event data small (< 64KB recommended)                       │
   │ • Include correlation IDs for tracing                              │
   │ • Use CloudEvents schema for interoperability                     │
   │ • Version your event types (Order.Created.v2)                     │
   │ • Don't include sensitive data in events                          │
   └─────────────────────────────────────────────────────────────────────┘

2. RELIABILITY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Always configure dead-letter destinations                        │
   │ • Implement idempotent event handlers                             │
   │ • Set appropriate retry policies                                   │
   │ • Monitor dead-letter container regularly                         │
   │ • Return appropriate HTTP status codes from handlers              │
   │ • Use 200 for success, 400 for non-retryable errors              │
   └─────────────────────────────────────────────────────────────────────┘

3. SECURITY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use managed identity for Function endpoints                      │
   │ • Enable AAD authentication for webhooks                          │
   │ • Implement webhook validation handshake                          │
   │ • Use Private Link for sensitive workloads                        │
   │ • Rotate access keys regularly                                     │
   │ • Use SAS tokens with limited scope                               │
   └─────────────────────────────────────────────────────────────────────┘

4. PERFORMANCE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use filters to reduce unnecessary deliveries                     │
   │ • Prefer Azure Function handlers for scale                        │
   │ • Route high-volume events through Event Hubs                     │
   │ • Use Service Bus for ordered processing requirements             │
   │ • Batch events when publishing (up to 1MB total)                  │
   │ • Set maxEventsPerBatch for Function handlers                     │
   └─────────────────────────────────────────────────────────────────────┘

5. OPERATIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Enable diagnostic logs to Log Analytics                          │
   │ • Set up alerts for delivery failures                             │
   │ • Monitor subscription health metrics                              │
   │ • Use Event Domains for multi-tenant scenarios                    │
   │ • Test event handlers with Event Grid Viewer                      │
   │ • Document event schemas for consumers                            │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Logic Apps](04-logic-apps.md)* | *Previous: [Service Bus](02-service-bus.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
