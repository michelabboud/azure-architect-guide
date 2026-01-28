# Application Insights

## What is Application Insights?

Application Insights is Azure's Application Performance Management (APM) service. It provides deep insights into your application's performance, usage, and health through automatic instrumentation, distributed tracing, and smart analytics.

### Application Insights vs AWS X-Ray

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               APPLICATION INSIGHTS vs AWS X-RAY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS X-RAY                             APPLICATION INSIGHTS                 │
│  ─────────                             ─────────────────────                │
│                                                                              │
│  Focus: Tracing only                   Focus: Full APM                      │
│  ┌─────────────────────┐              ┌─────────────────────────────────┐  │
│  │ • Request tracing   │              │ • Request tracing               │  │
│  │ • Service map       │              │ • Service map (App Map)         │  │
│  │ • Latency analysis  │              │ • Performance profiler          │  │
│  └─────────────────────┘              │ • Availability tests            │  │
│                                       │ • User analytics                │  │
│  Need separately:                     │ • Exception tracking            │  │
│  • CloudWatch for logs                │ • Live metrics                  │  │
│  • CloudWatch for metrics             │ • Smart detection               │  │
│  • Separate analytics tools           │ • Usage analytics               │  │
│                                       └─────────────────────────────────┘  │
│                                                                              │
│  Sampling: Fixed                       Sampling: Adaptive (intelligent)    │
│                                                                              │
│  Query: X-Ray console only             Query: KQL (same as all logs)       │
│                                                                              │
│  Integration:                          Integration:                         │
│  • AWS services                        • Azure services                    │
│  • Limited third-party                 • .NET, Java, Node, Python          │
│                                        • OpenTelemetry (cloud-agnostic)    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Application Insights Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    APPLICATION INSIGHTS ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  YOUR APPLICATION                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐             │   │
│  │   │ Web App     │───▶│   API       │───▶│  Database   │             │   │
│  │   │ (SDK)       │    │   (SDK)     │    │             │             │   │
│  │   └──────┬──────┘    └──────┬──────┘    └─────────────┘             │   │
│  │          │                  │                                        │   │
│  │          │   Telemetry      │                                        │   │
│  │          │                  │                                        │   │
│  └──────────┼──────────────────┼────────────────────────────────────────┘   │
│             │                  │                                            │
│             └────────┬─────────┘                                            │
│                      │                                                      │
│                      ▼                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    APPLICATION INSIGHTS                               │   │
│  │                                                                       │   │
│  │   DATA TYPES:                                                        │   │
│  │   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐             │   │
│  │   │   Requests    │ │ Dependencies  │ │  Exceptions   │             │   │
│  │   │   (incoming)  │ │  (outgoing)   │ │   (errors)    │             │   │
│  │   └───────────────┘ └───────────────┘ └───────────────┘             │   │
│  │                                                                       │   │
│  │   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐             │   │
│  │   │    Traces     │ │ Custom Events │ │Custom Metrics │             │   │
│  │   │   (logs)      │ │   (usage)     │ │   (KPIs)      │             │   │
│  │   └───────────────┘ └───────────────┘ └───────────────┘             │   │
│  │                                                                       │   │
│  │   STORAGE: Log Analytics Workspace (workspace-based)                 │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  CONSUMPTION:                                                               │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐      │
│  │ App Map      │ │ Performance  │ │ Failures     │ │ Availability │      │
│  │ (topology)   │ │ (slow ops)   │ │ (errors)     │ │ (health)     │      │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Instrumenting Applications

### .NET Application

```csharp
// 1. Install NuGet package
// dotnet add package Microsoft.ApplicationInsights.AspNetCore

// 2. Configure in Program.cs (.NET 6+)
var builder = WebApplication.CreateBuilder(args);

// Add Application Insights
builder.Services.AddApplicationInsightsTelemetry();

// For connection string from environment or appsettings.json
// APPLICATIONINSIGHTS_CONNECTION_STRING

var app = builder.Build();
// ... rest of app configuration


// 3. Custom telemetry
using Microsoft.ApplicationInsights;

public class MyService
{
    private readonly TelemetryClient _telemetry;

    public MyService(TelemetryClient telemetry)
    {
        _telemetry = telemetry;
    }

    public void ProcessOrder(Order order)
    {
        // Track custom event
        _telemetry.TrackEvent("OrderProcessed", new Dictionary<string, string>
        {
            { "OrderId", order.Id },
            { "CustomerId", order.CustomerId }
        });

        // Track custom metric
        _telemetry.TrackMetric("OrderValue", order.TotalAmount);

        // Track exception
        try
        {
            // ... processing logic
        }
        catch (Exception ex)
        {
            _telemetry.TrackException(ex, new Dictionary<string, string>
            {
                { "OrderId", order.Id }
            });
            throw;
        }
    }
}
```

### Node.js Application

```javascript
// 1. Install package
// npm install applicationinsights

// 2. Initialize at the very top of your main file
const appInsights = require("applicationinsights");
appInsights.setup(process.env.APPLICATIONINSIGHTS_CONNECTION_STRING)
    .setAutoDependencyCorrelation(true)
    .setAutoCollectRequests(true)
    .setAutoCollectPerformance(true)
    .setAutoCollectExceptions(true)
    .setAutoCollectDependencies(true)
    .setAutoCollectConsole(true, true) // Include console.log
    .setUseDiskRetryCaching(true)
    .start();

const client = appInsights.defaultClient;

// 3. Custom telemetry
// Track event
client.trackEvent({
    name: "OrderProcessed",
    properties: { orderId: "123", customerId: "456" }
});

// Track metric
client.trackMetric({
    name: "OrderValue",
    value: 99.99
});

// Track exception
try {
    // ... code
} catch (err) {
    client.trackException({ exception: err });
}

// Track custom trace
client.trackTrace({
    message: "Processing order started",
    severity: appInsights.Contracts.SeverityLevel.Information
});
```

### Python Application

```python
# 1. Install package
# pip install opencensus-ext-azure

# 2. Configure
from opencensus.ext.azure.trace_exporter import AzureExporter
from opencensus.ext.azure import metrics_exporter
from opencensus.ext.azure.log_exporter import AzureLogHandler
from opencensus.trace.samplers import ProbabilitySampler
from opencensus.trace.tracer import Tracer
import logging

connection_string = "InstrumentationKey=xxx;IngestionEndpoint=xxx"

# Set up tracing
tracer = Tracer(
    exporter=AzureExporter(connection_string=connection_string),
    sampler=ProbabilitySampler(1.0)
)

# Set up logging
logger = logging.getLogger(__name__)
logger.addHandler(AzureLogHandler(connection_string=connection_string))
logger.setLevel(logging.INFO)

# Set up metrics
exporter = metrics_exporter.new_metrics_exporter(
    connection_string=connection_string
)

# 3. Use in your application
def process_order(order):
    with tracer.span(name="ProcessOrder"):
        logger.info(f"Processing order {order.id}")

        # Custom properties
        tracer.add_attribute_to_current_span("order_id", order.id)
        tracer.add_attribute_to_current_span("customer_id", order.customer_id)

        try:
            # ... processing
            pass
        except Exception as e:
            logger.exception(f"Error processing order: {e}")
            raise
```

### OpenTelemetry (Cloud-Agnostic)

```javascript
// OpenTelemetry is the future-proof, vendor-neutral approach

// 1. Install packages
// npm install @opentelemetry/api @opentelemetry/sdk-node
// npm install @azure/monitor-opentelemetry-exporter

// 2. Configure
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { AzureMonitorTraceExporter } = require('@azure/monitor-opentelemetry-exporter');

const sdk = new NodeSDK({
    traceExporter: new AzureMonitorTraceExporter({
        connectionString: process.env.APPLICATIONINSIGHTS_CONNECTION_STRING
    })
});

sdk.start();

// 3. Use OpenTelemetry APIs - works with any backend!
const { trace } = require('@opentelemetry/api');
const tracer = trace.getTracer('my-service');

const span = tracer.startSpan('processOrder');
span.setAttribute('orderId', '123');
// ... do work
span.end();
```

---

## Application Insights Features

### Application Map

```
APPLICATION MAP:
────────────────

Visual representation of your distributed application topology.

┌─────────────────────────────────────────────────────────────────────────────┐
│                         APPLICATION MAP                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌─────────────────┐                                      │
│                    │     Browser     │                                      │
│                    │   (2.3k calls)  │                                      │
│                    │   avg: 1.2s     │                                      │
│                    └────────┬────────┘                                      │
│                             │                                                │
│                             ▼                                                │
│                    ┌─────────────────┐                                      │
│                    │   Web Frontend  │                                      │
│      ┌────────────▶│   (contoso-web) │◀────────────┐                       │
│      │             │   1.2k req/min  │             │                       │
│      │             │   0.5% failed   │             │                       │
│      │             └────────┬────────┘             │                       │
│      │                      │                      │                       │
│      │         ┌────────────┼────────────┐        │                       │
│      │         │            │            │        │                       │
│      │         ▼            ▼            ▼        │                       │
│  ┌───┴──────────────┐ ┌─────────────┐ ┌─────────────────┐                 │
│  │   Product API    │ │  Order API  │ │    Auth API     │                 │
│  │   (contoso-prod) │ │ (contoso-   │ │   (Entra ID)    │                 │
│  │   800 req/min    │ │    order)   │ │                 │                 │
│  │   0.1% failed    │ │ 400 req/min │ │   99.9% avail   │                 │
│  └────────┬─────────┘ └──────┬──────┘ └─────────────────┘                 │
│           │                  │                                             │
│           ▼                  ▼                                             │
│    ┌─────────────┐    ┌─────────────┐                                     │
│    │  Cosmos DB  │    │  Azure SQL  │                                     │
│    │  (catalog)  │    │  (orders)   │                                     │
│    │  5ms avg    │    │  12ms avg   │                                     │
│    │  🟢 Healthy │    │  🟡 Warn    │  ← Performance issue detected       │
│    └─────────────┘    └─────────────┘                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

Features:
• Auto-discovered from telemetry
• Click any node to see details
• Color indicates health (green/yellow/red)
• Shows request rates and failure rates
• Click edges to see dependency call details
```

### Live Metrics Stream

```
LIVE METRICS:
─────────────

Real-time view of application performance (no storage delay).

┌─────────────────────────────────────────────────────────────────────────────┐
│                          LIVE METRICS STREAM                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  INCOMING REQUESTS                    OUTGOING REQUESTS                     │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │  Rate: 1,234 /sec               │  │  Rate: 2,456 /sec               │  │
│  │  ████████████████████▓▓▓▓▓     │  │  █████████████████████████████  │  │
│  │                                 │  │                                 │  │
│  │  Failed: 12 /sec (0.9%)         │  │  Failed: 3 /sec (0.1%)          │  │
│  │  ██▓                            │  │  ▓                              │  │
│  └─────────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                              │
│  DURATION (percentiles)               SERVERS ONLINE                        │
│  ┌─────────────────────────────────┐  ┌─────────────────────────────────┐  │
│  │  P50: 45ms                      │  │  web-prod-eastus-1    🟢 OK     │  │
│  │  P95: 234ms                     │  │  web-prod-eastus-2    🟢 OK     │  │
│  │  P99: 892ms                     │  │  web-prod-westus-1    🟢 OK     │  │
│  │  ███████████████▓▓▓▓████       │  │  web-prod-westus-2    🟡 Warn   │  │
│  └─────────────────────────────────┘  └─────────────────────────────────┘  │
│                                                                              │
│  LIVE SAMPLE TELEMETRY:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 10:23:45.123 | GET /api/products | 200 | 45ms | web-prod-eastus-1   │   │
│  │ 10:23:45.124 | POST /api/orders  | 201 | 123ms| web-prod-eastus-2   │   │
│  │ 10:23:45.125 | GET /api/users    | 500 | 234ms| web-prod-westus-2   │   │
│  │ 10:23:45.126 | GET /api/products | 200 | 38ms | web-prod-eastus-1   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

Use cases:
• Deployment monitoring (watch during release)
• Incident response (real-time debugging)
• Load testing observation
• Demo and presentations
```

### Availability Tests

```
AVAILABILITY TESTS:
───────────────────

Proactive monitoring of application availability from global locations.

TEST TYPES:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  URL PING TEST (Simple):                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Simple HTTP GET request                                            │   │
│  │ • Check response code and content                                    │   │
│  │ • Multiple global locations                                          │   │
│  │ • Free tier available                                                │   │
│  │                                                                       │   │
│  │ Configuration:                                                       │   │
│  │ URL: https://myapp.azurewebsites.net/health                         │   │
│  │ Frequency: Every 5 minutes                                          │   │
│  │ Locations: East US, West Europe, Southeast Asia, etc.               │   │
│  │ Success criteria: HTTP 200, response contains "healthy"             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  STANDARD TEST (Multi-step):                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Multiple requests in sequence                                      │   │
│  │ • Authentication flows                                               │   │
│  │ • Form submissions                                                   │   │
│  │ • Record and playback from browser                                   │   │
│  │                                                                       │   │
│  │ Example flow:                                                        │   │
│  │ 1. GET /login                                                        │   │
│  │ 2. POST /login (with credentials)                                   │   │
│  │ 3. GET /dashboard (verify authenticated)                            │   │
│  │ 4. POST /api/action (verify functionality)                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CUSTOM TRACK AVAILABILITY (Code-based):                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Run custom availability logic                                      │   │
│  │ • Azure Function or any compute                                     │   │
│  │ • Complex business validations                                       │   │
│  │                                                                       │   │
│  │ telemetryClient.TrackAvailability(                                  │   │
│  │     "ComplexCheck",                                                  │   │
│  │     DateTimeOffset.UtcNow,                                          │   │
│  │     TimeSpan.FromSeconds(1.5),                                      │   │
│  │     "eastus",                                                        │   │
│  │     success: true,                                                   │   │
│  │     message: "All systems operational"                              │   │
│  │ );                                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

CLI Configuration:
──────────────────

# Create availability test
az monitor app-insights web-test create \
  --resource-group myRG \
  --name "HealthEndpointTest" \
  --location eastus \
  --defined-web-test-name "HealthCheck" \
  --web-test-kind "ping" \
  --synthetic-monitor-id "healthcheck1" \
  --request-url "https://myapp.azurewebsites.net/health" \
  --expected-http-status-code 200 \
  --frequency 300 \
  --timeout 120 \
  --geo-locations '[{"Id": "us-tx-sn1-azr"}, {"Id": "emea-nl-ams-azr"}, {"Id": "apac-sg-sin-azr"}]'
```

---

## Distributed Tracing

### Understanding Trace Context

```
DISTRIBUTED TRACING:
────────────────────

Track a single request across multiple services.

REQUEST FLOW:
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  User Request                                                                │
│       │                                                                     │
│       │ Correlation ID: abc-123-def (traceparent header)                   │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────┐                                                       │
│  │   API Gateway   │  operation_Id: abc-123-def                            │
│  │   Span 1        │  id: 1111                                             │
│  │   Duration: 5ms │  parentId: null                                       │
│  └────────┬────────┘                                                       │
│           │                                                                 │
│           │ traceparent: 00-abc123def-1111-01                              │
│           │                                                                 │
│           ▼                                                                 │
│  ┌─────────────────┐                                                       │
│  │   Order Service │  operation_Id: abc-123-def (same!)                    │
│  │   Span 2        │  id: 2222                                             │
│  │   Duration: 45ms│  parentId: 1111 (links to gateway)                   │
│  └────────┬────────┘                                                       │
│           │                                                                 │
│     ┌─────┴─────┐                                                          │
│     │           │                                                          │
│     ▼           ▼                                                          │
│  ┌─────────┐ ┌─────────┐                                                  │
│  │Inventory│ │ Payment │  Both have operation_Id: abc-123-def             │
│  │Span 3   │ │ Span 4  │  Both have parentId: 2222                        │
│  │ 20ms    │ │ 100ms   │                                                  │
│  └─────────┘ └─────────┘                                                  │
│                                                                              │
│  TOTAL DURATION: 5 + 45 + max(20, 100) = 150ms                             │
│  (excluding parallel operations)                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘


KQL QUERY FOR END-TO-END TRACE:
───────────────────────────────

// Find all operations for a specific request
union requests, dependencies, exceptions
| where operation_Id == "abc-123-def"
| project timestamp, itemType, name, duration, success, operation_ParentId
| order by timestamp asc

// Visualize trace timeline
requests
| where timestamp > ago(1h)
| where name == "POST /api/orders"
| take 1
| project operation_Id
| join kind=inner (
    union requests, dependencies
    | project timestamp, operation_Id, itemType, name, duration, operation_ParentId
) on operation_Id
| order by timestamp asc
```

### Profiler and Snapshot Debugger

```
PERFORMANCE PROFILER:
─────────────────────

Capture production code execution traces without affecting performance.

What it captures:
• Method-level execution times
• CPU hotspots
• Memory allocations
• Blocking calls

Enable:
1. Azure Portal → App Insights → Performance → Profiler
2. Configure triggers (CPU %, manual, always)
3. Analyze flame graphs

SNAPSHOT DEBUGGER:
──────────────────

Capture debug snapshots when exceptions occur in production.

What it captures:
• Local variables at exception point
• Call stack with values
• Heap snapshot

Enable:
1. Add NuGet: Microsoft.ApplicationInsights.SnapshotCollector
2. Configure in Startup.cs:
   services.AddSnapshotCollector();
3. View snapshots in Portal when exceptions occur
```

---

## Sampling and Cost Control

```
SAMPLING STRATEGIES:
────────────────────

App Insights can generate massive data volumes. Use sampling to control costs.

1. ADAPTIVE SAMPLING (Default - Recommended)
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Automatically adjusts sampling rate based on traffic volume         │
   │                                                                       │
   │ Configuration (.NET):                                                │
   │ services.Configure<TelemetryConfiguration>((config) =>              │
   │ {                                                                    │
   │     var processor = config.DefaultTelemetrySink.TelemetryProcessors│
   │         .OfType<AdaptiveSamplingTelemetryProcessor>().FirstOrDefault();│
   │     processor.MaxTelemetryItemsPerSecond = 5; // Adjust as needed   │
   │ });                                                                  │
   │                                                                       │
   │ Low traffic: 100% sampled                                           │
   │ High traffic: Automatically reduces (e.g., 10% at 1000 req/sec)    │
   └─────────────────────────────────────────────────────────────────────┘

2. FIXED-RATE SAMPLING
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Always sample at a fixed percentage                                  │
   │                                                                       │
   │ Configuration:                                                       │
   │ services.Configure<TelemetryConfiguration>((config) =>              │
   │ {                                                                    │
   │     config.DefaultTelemetrySink.TelemetryProcessorChainBuilder      │
   │         .UseSampling(25); // 25% sampled                            │
   │ });                                                                  │
   │                                                                       │
   │ Predictable but may miss rare events                                │
   └─────────────────────────────────────────────────────────────────────┘

3. INGESTION SAMPLING (Server-side)
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Configured in Application Insights resource                         │
   │                                                                       │
   │ Azure Portal → App Insights → Usage and estimated costs →          │
   │ Data sampling → Set percentage                                      │
   │                                                                       │
   │ Applied after data reaches Azure (still charged for ingestion!)    │
   └─────────────────────────────────────────────────────────────────────┘


COST OPTIMIZATION TIPS:
───────────────────────

1. Use sampling for high-volume applications
2. Filter unnecessary telemetry at the SDK level
3. Use workspace-based App Insights (shares Log Analytics costs)
4. Set appropriate retention (don't keep data longer than needed)
5. Use Basic logs tier for verbose trace data
6. Monitor your spending in Usage and estimated costs
```

---

## Best Practices

```
APPLICATION INSIGHTS BEST PRACTICES:
────────────────────────────────────

1. USE CONNECTION STRING (Not Instrumentation Key)
   ┌─────────────────────────────────────────────────────────────────────┐
   │ // Old way (deprecated)                                              │
   │ InstrumentationKey=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx            │
   │                                                                       │
   │ // New way (recommended)                                             │
   │ APPLICATIONINSIGHTS_CONNECTION_STRING=InstrumentationKey=xxx;...    │
   │                                                                       │
   │ Benefits: Regional endpoints, future-proof                          │
   └─────────────────────────────────────────────────────────────────────┘

2. USE WORKSPACE-BASED CONFIGURATION
   ┌─────────────────────────────────────────────────────────────────────┐
   │ All new App Insights should be workspace-based:                     │
   │ • Data stored in Log Analytics workspace                            │
   │ • Cross-resource queries                                            │
   │ • Unified retention settings                                        │
   │ • Works with Sentinel                                               │
   └─────────────────────────────────────────────────────────────────────┘

3. ADD CUSTOM PROPERTIES TO ALL TELEMETRY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Use TelemetryInitializer to add common properties:                  │
   │ • Environment (prod, staging, dev)                                  │
   │ • Version number                                                    │
   │ • User context                                                      │
   │ • Request correlation IDs                                           │
   └─────────────────────────────────────────────────────────────────────┘

4. SET UP AVAILABILITY TESTS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Test from multiple geographic locations                           │
   │ • Test critical user flows, not just health endpoints              │
   │ • Set up alerts on availability failures                           │
   └─────────────────────────────────────────────────────────────────────┘

5. USE LIVE METRICS FOR DEPLOYMENTS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Watch live during deployments                                     │
   │ • Catch issues immediately                                          │
   │ • Zero storage cost (streaming only)                               │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Alerting & Automation](04-alerting.md)* | *Back to [Log Analytics & KQL](02-log-analytics-kql.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
