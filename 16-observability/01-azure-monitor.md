# Azure Monitor Fundamentals

## What is Azure Monitor?

Azure Monitor is the unified platform for collecting, analyzing, and acting on telemetry from your Azure and on-premises environments. Think of it as CloudWatch, X-Ray, and CloudTrail combined with a single, powerful query language.

### Azure Monitor vs AWS CloudWatch

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   AZURE MONITOR vs AWS CLOUDWATCH                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CLOUDWATCH                            AZURE MONITOR                        │
│  ──────────                            ─────────────                        │
│                                                                              │
│  Architecture:                         Architecture:                        │
│  ┌─────────────────────┐              ┌─────────────────────────────────┐  │
│  │ CW Metrics (separate)│              │                                 │  │
│  │ CW Logs (separate)   │              │    UNIFIED AZURE MONITOR       │  │
│  │ CW Alarms (separate) │              │    ┌───────┬───────┬───────┐   │  │
│  │ CW Insights (separate)              │    │Metrics│ Logs  │Alerts │   │  │
│  │ X-Ray (separate)     │              │    └───────┴───────┴───────┘   │  │
│  └─────────────────────┘              │                                 │  │
│                                       └─────────────────────────────────┘  │
│                                                                              │
│  Query Languages:                      Query Language:                      │
│  • CloudWatch Insights (logs)          • KQL (everything!)                 │
│  • X-Ray query syntax                  • Same syntax for all data          │
│  • Different per service               • Works in: Log Analytics,          │
│                                          App Insights, Sentinel,           │
│                                          Azure Data Explorer               │
│                                                                              │
│  Data Retention:                       Data Retention:                      │
│  • Metrics: 15 months                  • Metrics: 93 days                  │
│  • Logs: Configurable                  • Logs: Up to 2 years interactive  │
│  • X-Ray: 30 days                               12 years archive           │
│                                                                              │
│  Pricing:                              Pricing:                             │
│  • Per metric, per log GB              • Platform metrics: Included        │
│  • Custom metrics extra                • Logs: Per GB ingested             │
│  • Alarms per alarm                    • Commitment tier discounts         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Azure Monitor Data Types

### 1. Metrics (Time-Series Data)

```
METRICS OVERVIEW:
─────────────────

WHAT ARE METRICS?
• Numerical values collected at regular intervals
• Pre-aggregated for fast queries
• 93-day retention (automatic)
• Near real-time (1-minute granularity)

TYPES OF METRICS:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  PLATFORM METRICS (Automatic):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Collected automatically for all Azure resources                    │   │
│  │ • No configuration needed                                            │   │
│  │ • No additional cost                                                 │   │
│  │ • Examples: VM CPU, Storage transactions, App Service requests      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CUSTOM METRICS:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Published via Application Insights SDK                             │   │
│  │ • Or via Azure Monitor API                                          │   │
│  │ • Additional cost (~$0.05/metric time series/month)                 │   │
│  │ • Examples: Business metrics, custom KPIs                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GUEST OS METRICS (via Agent):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Requires Azure Monitor Agent                                       │   │
│  │ • Detailed OS-level metrics                                          │   │
│  │ • Sent to Log Analytics or Metrics database                         │   │
│  │ • Examples: Per-process CPU, memory by application                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

METRIC DIMENSIONS:
──────────────────

Dimensions allow filtering/splitting metrics without creating separate metrics.

Example: Storage Account "Transactions" metric
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric: Transactions                                                        │
│ Dimensions:                                                                 │
│   • ApiName: GetBlob, PutBlob, ListBlobs, etc.                             │
│   • Authentication: AccountKey, SAS, Anonymous                             │
│   • ResponseType: Success, ClientError, ServerError                        │
│                                                                             │
│ Query: Transactions where ApiName == "PutBlob" and ResponseType == "Error" │
└────────────────────────────────────────────────────────────────────────────┘
```

### 2. Logs (Event Data)

```
LOGS OVERVIEW:
──────────────

WHAT ARE LOGS?
• Text or structured event data
• Stored in Log Analytics workspace
• Queried with KQL (Kusto Query Language)
• Configurable retention (31 days to 2 years)

LOG SOURCES:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ACTIVITY LOGS (Azure Control Plane):                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Who did what, when, and where                                      │   │
│  │ • Resource creation, deletion, modification                          │   │
│  │ • RBAC changes, policy events                                        │   │
│  │ • 90-day retention (free, in Activity Log blade)                    │   │
│  │ • Can export to Log Analytics for longer retention/queries          │   │
│  │                                                                       │   │
│  │ AWS Equivalent: CloudTrail                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  RESOURCE LOGS (Diagnostic Logs):                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Data plane operations                                              │   │
│  │ • Internal operations of resources                                   │   │
│  │ • Must be enabled via Diagnostic Settings                           │   │
│  │ • Destination: Log Analytics, Storage, Event Hub                    │   │
│  │                                                                       │   │
│  │ Examples:                                                             │   │
│  │ • Azure SQL: Query performance, errors, deadlocks                   │   │
│  │ • App Service: HTTP logs, application logs                          │   │
│  │ • Key Vault: Access logs                                            │   │
│  │                                                                       │   │
│  │ AWS Equivalent: VPC Flow Logs, S3 access logs, RDS logs             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  GUEST OS LOGS (via Agent):                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Windows Event Logs                                                 │   │
│  │ • Linux Syslog                                                       │   │
│  │ • Text files (custom logs)                                          │   │
│  │ • Requires Azure Monitor Agent                                       │   │
│  │                                                                       │   │
│  │ AWS Equivalent: CloudWatch Agent                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  APPLICATION LOGS (via SDK):                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Request/response traces                                            │   │
│  │ • Exceptions and errors                                              │   │
│  │ • Custom events and metrics                                         │   │
│  │ • Via Application Insights SDK                                      │   │
│  │                                                                       │   │
│  │ AWS Equivalent: X-Ray SDK + CloudWatch Logs SDK                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Configuring Data Collection

### Diagnostic Settings

```
DIAGNOSTIC SETTINGS ARCHITECTURE:
─────────────────────────────────

                    ┌─────────────────────┐
                    │   Azure Resource    │
                    │   (e.g., App Service)│
                    └──────────┬──────────┘
                               │
                    Diagnostic Settings
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
         ▼                     ▼                     ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ Log Analytics   │  │ Storage Account │  │   Event Hub     │
│ Workspace       │  │                 │  │                 │
│                 │  │ For:            │  │ For:            │
│ For:            │  │ • Long-term     │  │ • SIEM export   │
│ • Query/analyze │  │   archive       │  │ • Stream        │
│ • Alerts        │  │ • Compliance    │  │   processing    │
│ • Workbooks     │  │ • Cost savings  │  │ • Third-party   │
│ • Sentinel      │  │                 │  │   tools         │
└─────────────────┘  └─────────────────┘  └─────────────────┘


CONFIGURATION EXAMPLE (Azure CLI):
──────────────────────────────────

# Enable diagnostic settings for an App Service
az monitor diagnostic-settings create \
  --name "app-diagnostics" \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{app} \
  --workspace /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws} \
  --logs '[
    {"category": "AppServiceHTTPLogs", "enabled": true, "retentionPolicy": {"enabled": false}},
    {"category": "AppServiceConsoleLogs", "enabled": true, "retentionPolicy": {"enabled": false}},
    {"category": "AppServiceAppLogs", "enabled": true, "retentionPolicy": {"enabled": false}},
    {"category": "AppServiceAuditLogs", "enabled": true, "retentionPolicy": {"enabled": false}},
    {"category": "AppServiceIPSecAuditLogs", "enabled": true, "retentionPolicy": {"enabled": false}},
    {"category": "AppServicePlatformLogs", "enabled": true, "retentionPolicy": {"enabled": false}}
  ]' \
  --metrics '[
    {"category": "AllMetrics", "enabled": true, "retentionPolicy": {"enabled": false}}
  ]'
```

### Azure Monitor Agent (AMA)

```
AZURE MONITOR AGENT:
────────────────────

The modern agent for collecting guest OS data (replaces legacy agents).

LEGACY AGENTS (Deprecated):
• Log Analytics Agent (MMA/OMS)
• Diagnostics Extension (WAD/LAD)
• Telegraf

MODERN AGENT (Use This):
• Azure Monitor Agent (AMA)
• Single agent for all data collection
• Data Collection Rules for configuration

ARCHITECTURE:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌─────────────────────────────────┐                                       │
│  │         DATA COLLECTION         │                                       │
│  │             RULE                │                                       │
│  │                                 │                                       │
│  │  Defines:                       │                                       │
│  │  • What to collect              │                                       │
│  │  • From which VMs               │                                       │
│  │  • Where to send                │                                       │
│  └─────────────────┬───────────────┘                                       │
│                    │                                                        │
│       ┌────────────┼────────────┐                                          │
│       │            │            │                                          │
│       ▼            ▼            ▼                                          │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                                    │
│  │  VM 1   │  │  VM 2   │  │  VM 3   │     VMs with AMA installed         │
│  │  (AMA)  │  │  (AMA)  │  │  (AMA)  │                                    │
│  └────┬────┘  └────┬────┘  └────┬────┘                                    │
│       │            │            │                                          │
│       └────────────┼────────────┘                                          │
│                    │                                                        │
│                    ▼                                                        │
│           ┌───────────────────┐                                            │
│           │  Log Analytics    │                                            │
│           │  Workspace        │                                            │
│           └───────────────────┘                                            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘


DATA COLLECTION RULE EXAMPLE:
────────────────────────────

# Create Data Collection Rule
az monitor data-collection rule create \
  --name "WindowsPerformanceRule" \
  --resource-group myRG \
  --location eastus \
  --data-flows '[{
    "streams": ["Microsoft-Perf"],
    "destinations": ["myWorkspace"]
  }]' \
  --destinations '{
    "logAnalytics": [{
      "name": "myWorkspace",
      "workspaceResourceId": "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws}"
    }]
  }' \
  --data-sources '{
    "performanceCounters": [{
      "name": "WindowsPerfCounters",
      "streams": ["Microsoft-Perf"],
      "scheduledTransferPeriod": "PT1M",
      "samplingFrequencyInSeconds": 60,
      "counterSpecifiers": [
        "\\Processor(_Total)\\% Processor Time",
        "\\Memory\\Available MBytes",
        "\\LogicalDisk(_Total)\\% Free Space"
      ]
    }]
  }'

# Associate DCR with VM
az monitor data-collection rule association create \
  --name "vmAssociation" \
  --rule-id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Insights/dataCollectionRules/WindowsPerformanceRule \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/myVM
```

---

## Azure Monitor Insights

### Built-in Insights

```
AZURE MONITOR INSIGHTS:
───────────────────────

Pre-built monitoring solutions for common scenarios.

VM INSIGHTS:
┌────────────────────────────────────────────────────────────────────────────┐
│ • Performance: CPU, memory, disk, network over time                        │
│ • Map: Visualize connections between VMs and dependencies                 │
│ • Health: Track VM health state                                            │
│                                                                             │
│ Enable:                                                                     │
│ az monitor vm-insights enable --resource-group myRG --vm-name myVM        │
│                                --workspace myWorkspace                     │
└────────────────────────────────────────────────────────────────────────────┘

CONTAINER INSIGHTS:
┌────────────────────────────────────────────────────────────────────────────┐
│ • Node and pod metrics                                                      │
│ • Container logs (stdout/stderr)                                           │
│ • Live data view                                                           │
│ • Recommended alerts                                                        │
│                                                                             │
│ Enable for AKS:                                                            │
│ az aks enable-addons --resource-group myRG --name myAKS                    │
│                      --addons monitoring                                   │
│                      --workspace-resource-id /subscriptions/.../workspaces │
└────────────────────────────────────────────────────────────────────────────┘

NETWORK INSIGHTS:
┌────────────────────────────────────────────────────────────────────────────┐
│ • Topology view of network resources                                       │
│ • Connection monitor for endpoint health                                   │
│ • NSG flow logs analysis                                                   │
│ • Traffic Analytics                                                        │
│                                                                             │
│ Access via: Azure Portal → Monitor → Networks                              │
└────────────────────────────────────────────────────────────────────────────┘

STORAGE INSIGHTS:
┌────────────────────────────────────────────────────────────────────────────┐
│ • Capacity trends                                                          │
│ • Transaction analysis                                                     │
│ • Latency metrics                                                          │
│ • Failure analysis                                                         │
│                                                                             │
│ Access via: Storage Account → Insights                                     │
└────────────────────────────────────────────────────────────────────────────┘

SQL INSIGHTS:
┌────────────────────────────────────────────────────────────────────────────┐
│ • Query performance                                                        │
│ • Wait statistics                                                          │
│ • Resource utilization                                                     │
│ • Intelligent insights (auto-tuning)                                      │
│                                                                             │
│ Access via: SQL Database → Intelligent Performance                        │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Metrics Explorer

### Using Metrics Explorer

```
METRICS EXPLORER:
─────────────────

Portal-based tool for visualizing and analyzing metrics.

HOW TO ACCESS:
Azure Portal → Monitor → Metrics
(or from any resource blade → Monitoring → Metrics)

QUERY STRUCTURE:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. SCOPE (Which resource?)                                                │
│     └── Select resource(s) or resource group                               │
│                                                                             │
│  2. METRIC NAMESPACE (Which category?)                                     │
│     └── e.g., "Virtual Machine Host" vs "Virtual Machine Guest"           │
│                                                                             │
│  3. METRIC (What to measure?)                                              │
│     └── e.g., "Percentage CPU", "Network In Total"                        │
│                                                                             │
│  4. AGGREGATION (How to calculate?)                                        │
│     └── Avg, Min, Max, Sum, Count                                          │
│                                                                             │
│  5. TIME RANGE & GRANULARITY                                               │
│     └── Last 24 hours, 1-minute granularity                                │
│                                                                             │
│  6. SPLIT (Optional - by dimension)                                        │
│     └── e.g., Split by "Instance" for per-CPU view                        │
│                                                                             │
│  7. FILTER (Optional - by dimension value)                                 │
│     └── e.g., Where Instance = "cpu0"                                      │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

EXAMPLE QUERIES:
────────────────

# Query metrics via CLI
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm} \
  --metric "Percentage CPU" \
  --interval PT1M \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --aggregation Average

# Multiple metrics
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{app} \
  --metric "Requests" "AverageResponseTime" "Http5xx" \
  --interval PT5M \
  --aggregation Total Average Count
```

---

## Azure Workbooks

### Creating Interactive Reports

```
AZURE WORKBOOKS:
────────────────

Interactive, customizable reports combining metrics, logs, and text.

AWS Equivalent: CloudWatch Dashboards (but more powerful)

WORKBOOK COMPONENTS:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  TEXT BLOCKS:                                                               │
│  └── Markdown-formatted explanations and context                           │
│                                                                             │
│  PARAMETERS:                                                                │
│  └── Dynamic dropdowns for filtering (subscription, resource, time)       │
│                                                                             │
│  QUERY BLOCKS:                                                              │
│  └── KQL queries against Log Analytics or Metrics                         │
│                                                                             │
│  VISUALIZATIONS:                                                            │
│  └── Charts, grids, tiles, maps                                            │
│                                                                             │
│  LINKS/TABS:                                                                │
│  └── Navigation between sections                                           │
│                                                                             │
│  GROUPS:                                                                    │
│  └── Collapsible sections for organization                                 │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

EXAMPLE WORKBOOK STRUCTURE:
───────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                     APPLICATION HEALTH WORKBOOK                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  [Parameters: Subscription ▼] [Resource Group ▼] [Time Range ▼]            │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  OVERVIEW TAB                                                         │   │
│  │                                                                       │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐            │   │
│  │  │ Requests/min  │  │ Error Rate    │  │ Avg Response  │            │   │
│  │  │    1,234      │  │    0.5%       │  │    245ms      │            │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘            │   │
│  │                                                                       │   │
│  │  [Request Trend Chart - Line graph over time]                        │   │
│  │                                                                       │   │
│  │  [Error Breakdown - Pie chart by status code]                        │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ERRORS TAB                                                           │   │
│  │                                                                       │   │
│  │  [KQL Query: exceptions | summarize count() by type, outerMessage]   │   │
│  │                                                                       │   │
│  │  ┌────────────────────────────────────────────────────────────────┐ │   │
│  │  │ Exception Type          │ Count │ Last Seen                    │ │   │
│  │  ├─────────────────────────┼───────┼──────────────────────────────┤ │   │
│  │  │ NullReferenceException  │ 45    │ 2024-01-15 10:23:45          │ │   │
│  │  │ TimeoutException        │ 23    │ 2024-01-15 10:20:12          │ │   │
│  │  │ SqlException            │ 12    │ 2024-01-15 10:15:33          │ │   │
│  │  └────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

```
AZURE MONITOR BEST PRACTICES:
─────────────────────────────

1. ENABLE DIAGNOSTIC SETTINGS EVERYWHERE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use Azure Policy to enforce diagnostic settings                    │
   │ • "Deploy if not exists" policy auto-enables for new resources      │
   │ • Minimum: Send to Log Analytics workspace                          │
   └─────────────────────────────────────────────────────────────────────┘

2. CENTRALIZE LOGS IN ONE WORKSPACE (Usually)
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Enables cross-resource queries                                     │
   │ • Simplifies access management                                       │
   │ • Qualifies for commitment tier discounts                           │
   │ • Exception: Data sovereignty requirements                          │
   └─────────────────────────────────────────────────────────────────────┘

3. SET UP ALERTS FOR CRITICAL METRICS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • CPU, memory, disk for VMs                                          │
   │ • Error rates for applications                                       │
   │ • Availability for web apps                                          │
   │ • Security alerts from Defender                                      │
   └─────────────────────────────────────────────────────────────────────┘

4. USE APPROPRIATE DATA TIERS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Analytics logs: Frequently queried data                           │
   │ • Basic logs: High-volume, rarely queried                           │
   │ • Archive: Long-term compliance storage                             │
   └─────────────────────────────────────────────────────────────────────┘

5. IMPLEMENT RESOURCE TAGGING
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Tags flow to metrics and logs                                      │
   │ • Enable cost allocation                                             │
   │ • Filter dashboards and alerts by tag                               │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Log Analytics & KQL](02-log-analytics-kql.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
