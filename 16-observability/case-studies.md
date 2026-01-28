# Observability Case Studies

## Case Study 1: E-Commerce Platform Monitoring

### Scenario

**Company**: Online retailer with 2M daily active users
**Challenge**: Black Friday traffic causes performance issues; no visibility into root cause
**AWS Background**: Using CloudWatch metrics/logs, X-Ray for tracing

### The Problem

```
CURRENT STATE:
──────────────

Peak Traffic Events:
• Regular: 50,000 concurrent users
• Black Friday: 500,000 concurrent users (10x spike)

Previous Black Friday Issues:
1. Homepage slow (8+ seconds)
2. Cart abandonment increased 40%
3. Payment failures tripled
4. No visibility into actual bottleneck
5. 4 hours to identify root cause (database connection pool)

Current Monitoring:
┌────────────────────────────────────────────────────────────────────────────┐
│ CloudWatch Metrics: Basic CPU, memory for EC2                              │
│ CloudWatch Logs: Application logs (unstructured)                          │
│ X-Ray: Limited tracing (only some services)                               │
│ Alerting: CPU > 80% → Page                                                │
│                                                                             │
│ GAPS:                                                                      │
│ • No end-to-end transaction tracing                                       │
│ • No business metrics (cart value, conversion rate)                       │
│ • No dependency performance visibility                                     │
│ • Alerts based on infrastructure, not user experience                     │
└────────────────────────────────────────────────────────────────────────────┘
```

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               E-COMMERCE OBSERVABILITY ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TELEMETRY COLLECTION:                                                      │
│                                                                              │
│  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐           │
│  │    Browser     │    │   Web App      │    │    APIs        │           │
│  │   (App Insights│───▶│  (App Insights │───▶│ (App Insights  │           │
│  │    JS SDK)     │    │    SDK)        │    │    SDK)        │           │
│  └────────┬───────┘    └────────┬───────┘    └────────┬───────┘           │
│           │                     │                     │                    │
│           │   Page views,       │   Requests,         │   Dependencies,   │
│           │   Custom events     │   Traces            │   Custom metrics  │
│           │                     │                     │                    │
│           └─────────────────────┼─────────────────────┘                    │
│                                 ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    LOG ANALYTICS WORKSPACE                            │  │
│  │                                                                       │  │
│  │  Tables:                                                             │  │
│  │  • requests (HTTP requests)                                         │  │
│  │  • dependencies (calls to DB, cache, APIs)                          │  │
│  │  • exceptions (errors)                                              │  │
│  │  • customEvents (business events: cart_add, checkout, purchase)    │  │
│  │  • customMetrics (cart_value, inventory_level, conversion_rate)    │  │
│  │  • pageViews (browser performance)                                  │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  DASHBOARDS & ALERTS:                                                       │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  REAL-TIME DASHBOARD (Live Metrics):                                 │  │
│  │  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐           │  │
│  │  │ Requests/sec   │ │ Avg Response   │ │ Error Rate     │           │  │
│  │  │    1,234       │ │    245ms       │ │    0.5%        │           │  │
│  │  └────────────────┘ └────────────────┘ └────────────────┘           │  │
│  │                                                                       │  │
│  │  BUSINESS DASHBOARD (Workbook):                                      │  │
│  │  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐           │  │
│  │  │ Active Carts   │ │ Conversion %   │ │ Revenue/hr     │           │  │
│  │  │    5,432       │ │    3.2%        │ │   $45,234      │           │  │
│  │  └────────────────┘ └────────────────┘ └────────────────┘           │  │
│  │                                                                       │  │
│  │  ALERTS:                                                             │  │
│  │  • P95 response time > 3s → Page ops                                │  │
│  │  • Error rate > 1% → Page ops + Slack                               │  │
│  │  • Conversion rate drops 20% → Page leadership                      │  │
│  │  • Payment failures > 5% → Page ops + payment team                  │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation Details

```csharp
// Custom telemetry for business events
public class OrderController : Controller
{
    private readonly TelemetryClient _telemetry;

    public async Task<IActionResult> Checkout(CheckoutRequest request)
    {
        var stopwatch = Stopwatch.StartNew();

        // Track cart value as custom metric
        _telemetry.TrackMetric("CartValue", request.CartTotal);

        try
        {
            var result = await _orderService.ProcessOrder(request);

            // Track successful purchase
            _telemetry.TrackEvent("Purchase", new Dictionary<string, string>
            {
                { "OrderId", result.OrderId },
                { "CustomerId", request.CustomerId },
                { "PaymentMethod", request.PaymentMethod }
            }, new Dictionary<string, double>
            {
                { "OrderValue", request.CartTotal },
                { "ItemCount", request.Items.Count },
                { "ProcessingTimeMs", stopwatch.ElapsedMilliseconds }
            });

            return Ok(result);
        }
        catch (PaymentException ex)
        {
            // Track payment failure with context
            _telemetry.TrackEvent("PaymentFailed", new Dictionary<string, string>
            {
                { "CustomerId", request.CustomerId },
                { "PaymentMethod", request.PaymentMethod },
                { "ErrorCode", ex.ErrorCode },
                { "ErrorMessage", ex.Message }
            });

            _telemetry.TrackException(ex);
            throw;
        }
    }
}
```

```kusto
// KQL Queries for E-Commerce Monitoring

// Real-time conversion funnel
let timeRange = 15m;
customEvents
| where timestamp > ago(timeRange)
| where name in ("PageView_Product", "CartAdd", "CheckoutStart", "Purchase")
| summarize Users = dcount(user_Id) by name
| order by case(
    name == "PageView_Product", 1,
    name == "CartAdd", 2,
    name == "CheckoutStart", 3,
    name == "Purchase", 4,
    5)

// Payment success rate by provider
customEvents
| where timestamp > ago(1h)
| where name in ("Purchase", "PaymentFailed")
| extend PaymentMethod = tostring(customDimensions.PaymentMethod)
| summarize
    Attempts = count(),
    Successes = countif(name == "Purchase"),
    SuccessRate = round(100.0 * countif(name == "Purchase") / count(), 2)
by PaymentMethod

// Checkout duration percentiles
customEvents
| where timestamp > ago(1h)
| where name == "Purchase"
| extend ProcessingTime = todouble(customMeasurements.ProcessingTimeMs)
| summarize
    P50 = percentile(ProcessingTime, 50),
    P90 = percentile(ProcessingTime, 90),
    P99 = percentile(ProcessingTime, 99)
by bin(timestamp, 5m)
```

### Results

```
BLACK FRIDAY COMPARISON:
────────────────────────

                          PREVIOUS YEAR    THIS YEAR    IMPROVEMENT
┌─────────────────────────────────────────────────────────────────────────────┐
│ Time to identify issue    │ 4 hours       │ 5 minutes  │ 98% faster        │
│ Homepage P95 latency      │ 8.2 seconds   │ 1.8 seconds│ 78% faster        │
│ Checkout error rate       │ 4.5%          │ 0.3%       │ 93% reduction     │
│ Cart abandonment rate     │ 78%           │ 62%        │ 21% improvement   │
│ Revenue impact            │ -$2.3M lost   │ +$1.8M gain│ $4.1M difference  │
└─────────────────────────────────────────────────────────────────────────────┘

Key Improvements:
• Application Map showed database bottleneck immediately
• Live Metrics enabled real-time deployment monitoring
• Business metric alerts caught conversion drop before customers complained
• Distributed tracing pinpointed slow dependency (payment gateway)
```

---

## Case Study 2: Multi-Region SaaS Application

### Scenario

**Company**: B2B SaaS platform with customers in NA, EU, APAC
**Challenge**: Inconsistent performance across regions, difficult to compare
**Goal**: Unified monitoring with regional insights

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│             MULTI-REGION OBSERVABILITY ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                     ┌─────────────────────────────┐                         │
│                     │   GLOBAL LOG ANALYTICS      │                         │
│                     │   WORKSPACE (East US 2)     │                         │
│                     │                             │                         │
│                     │  Aggregated view of all     │                         │
│                     │  regions for global ops     │                         │
│                     └──────────────┬──────────────┘                         │
│                                    │                                         │
│     ┌──────────────────────────────┼──────────────────────────┐             │
│     │                              │                          │             │
│     ▼                              ▼                          ▼             │
│  ┌───────────────────┐   ┌───────────────────┐   ┌───────────────────┐    │
│  │   EAST US         │   │   WEST EUROPE     │   │   SOUTHEAST ASIA  │    │
│  │                   │   │                   │   │                   │    │
│  │ App Insights      │   │ App Insights      │   │ App Insights      │    │
│  │ (Regional)        │   │ (Regional)        │   │ (Regional)        │    │
│  │                   │   │                   │   │                   │    │
│  │ Availability:     │   │ Availability:     │   │ Availability:     │    │
│  │ • Test from EU    │   │ • Test from US    │   │ • Test from US    │    │
│  │ • Test from APAC  │   │ • Test from APAC  │   │ • Test from EU    │    │
│  │                   │   │                   │   │                   │    │
│  │ Data Residency:   │   │ Data Residency:   │   │ Data Residency:   │    │
│  │ US data stays     │   │ EU data stays     │   │ APAC data stays   │    │
│  │ in US workspace   │   │ in EU workspace   │   │ in APAC workspace │    │
│  └───────────────────┘   └───────────────────┘   └───────────────────┘    │
│                                                                              │
│  CROSS-REGION QUERIES:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  // Query across all workspaces                                      │   │
│  │  union                                                                │   │
│  │      workspace("eastus-workspace").requests,                         │   │
│  │      workspace("westeu-workspace").requests,                         │   │
│  │      workspace("seasia-workspace").requests                          │   │
│  │  | summarize AvgDuration = avg(duration) by cloud_RoleInstance      │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Regional Comparison Dashboard

```kusto
// Cross-region performance comparison
let allRegions = union
    workspace("eastus-workspace").requests,
    workspace("westeu-workspace").requests,
    workspace("seasia-workspace").requests;
allRegions
| where timestamp > ago(1h)
| extend Region = case(
    cloud_RoleInstance has "eastus", "East US",
    cloud_RoleInstance has "westeu", "West Europe",
    cloud_RoleInstance has "seasia", "Southeast Asia",
    "Unknown"
)
| summarize
    RequestCount = count(),
    AvgDuration = avg(duration),
    P95Duration = percentile(duration, 95),
    ErrorRate = round(100.0 * countif(success == false) / count(), 2)
by Region, bin(timestamp, 5m)

// Regional availability comparison
availabilityResults
| where timestamp > ago(24h)
| extend TestRegion = tostring(split(location, "-")[0])
| extend TargetRegion = tostring(customDimensions.TargetRegion)
| summarize
    SuccessRate = round(100.0 * countif(success == true) / count(), 2),
    AvgDuration = avg(duration)
by TestRegion, TargetRegion
| order by TargetRegion, TestRegion
```

### Architectural Decision

```
DECISION: Regional vs Global Workspaces?
────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  OPTION A: Single Global Workspace                                          │
│  ─────────────────────────────────                                          │
│                                                                              │
│  Pros:                                                                       │
│  ✓ Simplest to manage                                                       │
│  ✓ Cross-region queries without union                                       │
│  ✓ Single dashboard                                                         │
│                                                                              │
│  Cons:                                                                       │
│  ✗ Data residency violations (GDPR)                                        │
│  ✗ High egress costs from remote regions                                   │
│  ✗ Higher latency for local queries                                        │
│                                                                              │
│  OPTION B: Regional Workspaces with Global Aggregation (CHOSEN)            │
│  ───────────────────────────────────────────────────────────                │
│                                                                              │
│  Pros:                                                                       │
│  ✓ GDPR compliant (EU data in EU)                                          │
│  ✓ Lower egress costs                                                       │
│  ✓ Fast regional queries                                                    │
│  ✓ Global view via cross-workspace queries                                 │
│                                                                              │
│  Cons:                                                                       │
│  ✗ More complex management                                                  │
│  ✗ Need to use union for global queries                                    │
│  ✗ Separate alert configuration per region                                 │
│                                                                              │
│  CHOSEN: Option B - Regional with global aggregation                       │
│                                                                              │
│  RATIONALE:                                                                 │
│  • GDPR compliance is non-negotiable for EU customers                      │
│  • Regional teams benefit from fast local queries                          │
│  • Global ops can still query across regions when needed                   │
│  • Cost savings from reduced cross-region egress                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Case Study 3: Microservices Migration Monitoring

### Scenario

**Company**: Insurance company migrating monolith to microservices
**Challenge**: Need visibility during migration when both systems run in parallel
**Goal**: Compare performance, track migration progress, ensure no regression

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              MIGRATION OBSERVABILITY ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TRAFFIC ROUTING (Feature Flags):                                           │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         API GATEWAY                                   │   │
│  │                                                                       │   │
│  │  Rule: If user in "new-system" cohort AND feature "claims-v2"       │   │
│  │        Route to microservices                                        │   │
│  │        Else route to monolith                                        │   │
│  │                                                                       │   │
│  └────────────────────────────────┬──────────────────────────────────────┘   │
│                                   │                                         │
│              ┌────────────────────┴────────────────────┐                   │
│              │                                         │                   │
│              ▼                                         ▼                   │
│  ┌─────────────────────────┐           ┌─────────────────────────────────┐│
│  │      MONOLITH           │           │       MICROSERVICES             ││
│  │   (Legacy System)       │           │       (New System)              ││
│  │                         │           │                                  ││
│  │  App Insights:          │           │  App Insights:                  ││
│  │  cloud_RoleName =       │           │  cloud_RoleName =               ││
│  │    "claims-monolith"    │           │    "claims-api",               ││
│  │                         │           │    "policy-api",               ││
│  │                         │           │    "pricing-api"               ││
│  └──────────────┬──────────┘           └───────────────┬─────────────────┘│
│                 │                                       │                  │
│                 └───────────────┬───────────────────────┘                  │
│                                 │                                          │
│                                 ▼                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                    LOG ANALYTICS WORKSPACE                            │  │
│  │                                                                       │  │
│  │  COMPARISON QUERIES:                                                 │  │
│  │  • Same operation, different systems                                 │  │
│  │  • Error rates by system                                            │  │
│  │  • Performance percentiles side-by-side                             │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  MIGRATION DASHBOARD:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │  TRAFFIC SPLIT:                    PERFORMANCE COMPARISON:           │  │
│  │  ┌───────────────────────┐        ┌───────────────────────────────┐ │  │
│  │  │ Monolith: 70% ████████│        │ Operation    │ Mono │ Micro  │ │  │
│  │  │ Micro:    30% ████    │        │ GetClaim     │ 450ms│ 120ms  │ │  │
│  │  └───────────────────────┘        │ CreateClaim  │ 800ms│ 230ms  │ │  │
│  │                                    │ UpdateClaim  │ 650ms│ 180ms  │ │  │
│  │  ERROR RATE:                      └───────────────────────────────┘ │  │
│  │  ┌───────────────────────┐                                          │  │
│  │  │ Monolith: 0.8%        │        MIGRATION PROGRESS:               │  │
│  │  │ Micro:    0.3%        │        ██████████░░░░░░░░░░ 50%         │  │
│  │  └───────────────────────┘                                          │  │
│  │                                                                       │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Comparison Queries

```kusto
// Side-by-side performance comparison
let monolith = requests
| where cloud_RoleName == "claims-monolith"
| where name has "GetClaim"
| summarize Mono_P50 = percentile(duration, 50),
            Mono_P95 = percentile(duration, 95),
            Mono_Errors = countif(success == false) * 100.0 / count()
  by bin(timestamp, 5m);

let micro = requests
| where cloud_RoleName == "claims-api"
| where name has "GetClaim"
| summarize Micro_P50 = percentile(duration, 50),
            Micro_P95 = percentile(duration, 95),
            Micro_Errors = countif(success == false) * 100.0 / count()
  by bin(timestamp, 5m);

monolith
| join micro on timestamp
| project timestamp, Mono_P50, Micro_P50, Mono_P95, Micro_P95, Mono_Errors, Micro_Errors

// Migration progress tracking
let operations = datatable(Operation:string) [
    "GetClaim", "CreateClaim", "UpdateClaim", "DeleteClaim",
    "GetPolicy", "CreatePolicy", "UpdatePolicy", "DeletePolicy"
];
let microTraffic = requests
| where cloud_RoleName !has "monolith"
| where timestamp > ago(1h)
| summarize MicroRequests = count() by Operation = name;
let monoTraffic = requests
| where cloud_RoleName has "monolith"
| where timestamp > ago(1h)
| summarize MonoRequests = count() by Operation = name;
microTraffic
| join kind=fullouter monoTraffic on Operation
| extend MigrationPercent = round(100.0 * MicroRequests / (MicroRequests + MonoRequests), 1)
| project Operation, MonoRequests, MicroRequests, MigrationPercent
```

### Migration Go/No-Go Alerts

```bash
# Alert: Microservices error rate higher than monolith
az monitor scheduled-query create \
  --name "Migration-Error-Regression" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws} \
  --condition "count > 0" \
  --condition-query "
    let micro = requests
    | where cloud_RoleName !has 'monolith'
    | summarize MicroErrors = countif(success == false) * 100.0 / count();
    let mono = requests
    | where cloud_RoleName has 'monolith'
    | summarize MonoErrors = countif(success == false) * 100.0 / count();
    micro | join mono on \$left.x == \$right.x
    | where MicroErrors > MonoErrors * 1.5  // 50% higher error rate
    | project Alert = 'Microservices error rate significantly higher than monolith'
  " \
  --window-size 15m \
  --evaluation-frequency 5m \
  --severity 1 \
  --action /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/migration-team
```

---

## Key Takeaways

```
OBSERVABILITY ARCHITECTURE PRINCIPLES:
──────────────────────────────────────

1. INSTRUMENT FOR BUSINESS OUTCOMES
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Don't just monitor infrastructure - track business metrics          │
   │ • Conversion rates                                                  │
   │ • Cart value                                                        │
   │ • User satisfaction (APDEX)                                        │
   │ • Revenue impact                                                    │
   └─────────────────────────────────────────────────────────────────────┘

2. USE DISTRIBUTED TRACING
   ┌─────────────────────────────────────────────────────────────────────┐
   │ In microservices, individual service metrics aren't enough         │
   │ • Trace entire user journeys                                       │
   │ • Use correlation IDs consistently                                 │
   │ • Include business context in traces                               │
   └─────────────────────────────────────────────────────────────────────┘

3. ALERT ON USER EXPERIENCE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • P95 latency, not average                                         │
   │ • Error rate from user perspective                                 │
   │ • Availability from multiple locations                             │
   │ • Business KPI degradation                                         │
   └─────────────────────────────────────────────────────────────────────┘

4. PREPARE FOR INCIDENTS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Have pre-built dashboards for common scenarios                   │
   │ • Save useful KQL queries as functions                             │
   │ • Document runbooks linked from alerts                             │
   │ • Practice incident response with chaos engineering                │
   └─────────────────────────────────────────────────────────────────────┘

5. CONTINUOUSLY IMPROVE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Review alerts monthly (are they actionable?)                     │
   │ • Post-incident: What telemetry was missing?                       │
   │ • Measure MTTD and MTTR, work to improve                          │
   │ • Share learnings across teams                                     │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Back to [Chapter Overview](README.md)* | *Next Chapter: [AI Platform](../05-ai-platform/README.md)*
