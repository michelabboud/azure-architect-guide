# Log Analytics & KQL Mastery

## Log Analytics Workspace Architecture

### Workspace Design Considerations

```
WORKSPACE DESIGN DECISION TREE:
───────────────────────────────

                    ┌─────────────────────────────────┐
                    │ How many workspaces do I need?  │
                    └─────────────────┬───────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              │                       │                       │
              ▼                       ▼                       ▼
    ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
    │ Data Sovereignty│    │   Access        │    │    Scale        │
    │ Requirements?   │    │   Separation?   │    │    Limits?      │
    └────────┬────────┘    └────────┬────────┘    └────────┬────────┘
             │                      │                      │
    ┌────────▼────────┐    ┌────────▼────────┐    ┌────────▼────────┐
    │ YES: Regional   │    │ YES: Consider   │    │ Rarely an issue │
    │ workspaces per  │    │ separate for    │    │ Limit: 500 TB   │
    │ data residency  │    │ sensitive data  │    │ per workspace   │
    │ requirement     │    │ (Security/SOC)  │    │                 │
    └─────────────────┘    └─────────────────┘    └─────────────────┘

COMMON PATTERNS:
────────────────

1. SINGLE CENTRALIZED WORKSPACE (Most Common)
   ┌─────────────────────────────────────────────────────────────────────┐
   │                                                                       │
   │              ┌─────────────────────────────┐                         │
   │              │   Central Workspace         │                         │
   │              │   (All subscriptions)       │                         │
   │              └─────────────────────────────┘                         │
   │                           ▲                                          │
   │      ┌────────────────────┼────────────────────┐                    │
   │      │                    │                    │                    │
   │  ┌───┴───┐           ┌────┴────┐          ┌───┴───┐                │
   │  │ Sub 1 │           │  Sub 2  │          │ Sub 3 │                │
   │  └───────┘           └─────────┘          └───────┘                │
   │                                                                       │
   │  Benefits:                                                           │
   │  • Cross-resource queries                                            │
   │  • Simplified management                                             │
   │  • Cost optimization (commitment tiers)                             │
   │  • Single RBAC configuration                                         │
   │                                                                       │
   └─────────────────────────────────────────────────────────────────────┘

2. PRODUCTION + NON-PRODUCTION SPLIT
   ┌─────────────────────────────────────────────────────────────────────┐
   │                                                                       │
   │  ┌─────────────────────┐    ┌─────────────────────┐                 │
   │  │  Production WS      │    │  Non-Prod WS        │                 │
   │  │                     │    │  (Dev, Test, QA)    │                 │
   │  │  • Longer retention │    │  • Shorter retention│                 │
   │  │  • Stricter access  │    │  • Broader access   │                 │
   │  │  • Higher SLA       │    │  • Lower cost       │                 │
   │  └─────────────────────┘    └─────────────────────┘                 │
   │                                                                       │
   │  Benefits:                                                           │
   │  • Access separation                                                 │
   │  • Different retention policies                                      │
   │  • Cost optimization per environment                                │
   │                                                                       │
   └─────────────────────────────────────────────────────────────────────┘

3. DEDICATED SECURITY WORKSPACE
   ┌─────────────────────────────────────────────────────────────────────┐
   │                                                                       │
   │  ┌─────────────────────┐    ┌─────────────────────┐                 │
   │  │  Operations WS      │    │  Security WS        │                 │
   │  │  (IT Ops team)      │    │  (Security/Sentinel)│                 │
   │  │                     │    │                     │                 │
   │  │  • Perf metrics     │    │  • Security events  │                 │
   │  │  • App logs         │    │  • Sign-in logs     │                 │
   │  │  • Diagnostics      │    │  • Audit logs       │                 │
   │  └─────────────────────┘    │  • Threat detection │                 │
   │                             └─────────────────────┘                 │
   │                                                                       │
   │  Benefits:                                                           │
   │  • Security team has dedicated workspace                            │
   │  • Sentinel requires its own workspace                              │
   │  • Different access patterns                                        │
   │                                                                       │
   └─────────────────────────────────────────────────────────────────────┘
```

### Data Tiers (Analytics vs Basic vs Archive)

```
LOG ANALYTICS DATA TIERS:
─────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ANALYTICS LOGS (Default):                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Cost:        ~$2.76/GB ingestion                                     │   │
│  │ Retention:   31 days free, then ~$0.12/GB/month                     │   │
│  │ Query:       Full KQL capabilities                                   │   │
│  │ Use for:     Frequently queried operational data                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  BASIC LOGS (Cost-optimized):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Cost:        ~$0.65/GB ingestion (76% cheaper!)                     │   │
│  │ Retention:   8 days fixed (cannot extend)                           │   │
│  │ Query:       Limited KQL (simple filters, no joins/aggregations)   │   │
│  │ Use for:     High-volume, debug-only data                           │   │
│  │                                                                       │   │
│  │ Good candidates:                                                     │   │
│  │ • Container logs (ContainerLogV2)                                   │   │
│  │ • Debug/trace level logs                                            │   │
│  │ • High-volume firewall logs                                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ARCHIVE (Long-term storage):                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Cost:        ~$0.026/GB/month storage                               │   │
│  │ Retention:   Up to 12 years                                          │   │
│  │ Query:       Must restore to Analytics tier first                   │   │
│  │ Restore:     Takes time, costs per query                            │   │
│  │ Use for:     Compliance/audit data                                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DATA FLOW:                                                                │
│                                                                             │
│  Ingestion → Analytics Logs → (after retention) → Archive                  │
│              (31+ days)                             (up to 12 years)       │
│                                                                             │
│           → Basic Logs → (auto-delete after 8 days)                        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## KQL (Kusto Query Language) Deep Dive

### KQL Fundamentals

```kusto
// KQL BASICS:
// - Read left to right (like Unix pipes)
// - Each operator transforms the data
// - Case sensitive for operators, often for column names

// STRUCTURE:
TableName
| where <condition>           // Filter rows
| extend <new_column>         // Add calculated column
| project <columns>           // Select columns
| summarize <aggregation>     // Group and aggregate
| order by <column>           // Sort
| take <n>                    // Limit results

// COMPARISON TO SQL:
// SQL:    SELECT TOP 10 * FROM Table WHERE Column = 'Value' ORDER BY Time DESC
// KQL:    Table | where Column == 'Value' | order by Time desc | take 10
```

### Essential KQL Operators

```kusto
// ========== FILTERING ==========

// where - Filter rows
AzureDiagnostics
| where TimeGenerated > ago(1h)
| where Level == "Error"

// has - Contains substring (case-insensitive, faster)
AzureDiagnostics
| where Message has "connection failed"

// contains - Contains substring (case-insensitive)
AzureDiagnostics
| where Message contains "error"

// startswith, endswith
SigninLogs
| where UserPrincipalName startswith "admin"

// in - Match any value in list
SecurityEvent
| where EventID in (4624, 4625, 4626)

// between - Range
Perf
| where TimeGenerated between (ago(1h) .. now())

// ========== SELECTING & TRANSFORMING ==========

// project - Select specific columns
SigninLogs
| project TimeGenerated, UserPrincipalName, ResultType, Location

// project-away - Remove columns
AzureDiagnostics
| project-away TenantId, SubscriptionId

// extend - Add calculated columns
requests
| extend DurationSeconds = duration / 1000
| extend IsError = resultCode >= 400

// parse - Extract from string
AzureDiagnostics
| parse msg_s with * "client " ClientIP ":" ClientPort " " *

// ========== AGGREGATING ==========

// summarize - Group and aggregate
AzureDiagnostics
| summarize Count = count() by Category

// Common aggregations: count(), sum(), avg(), min(), max(), dcount()
requests
| summarize
    TotalRequests = count(),
    AvgDuration = avg(duration),
    MaxDuration = max(duration),
    UniqueUsers = dcount(user_Id)
    by bin(timestamp, 1h)

// bin - Time buckets
Perf
| summarize avg(CounterValue) by bin(TimeGenerated, 5m), Computer

// ========== JOINING ==========

// join - Combine tables
Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| join kind=leftouter (
    Perf
    | where TimeGenerated > ago(1h)
    | where CounterName == "% Processor Time"
    | summarize AvgCPU = avg(CounterValue) by Computer
) on Computer

// union - Combine rows from multiple tables
union SecurityEvent, WindowsEvent
| where TimeGenerated > ago(1h)

// ========== SORTING & LIMITING ==========

// order by (or sort by)
requests
| order by duration desc

// take (or limit)
requests
| take 100

// top - Sort and limit combined
requests
| top 10 by duration desc
```

### Advanced KQL Patterns

```kusto
// ========== TIME SERIES ANALYSIS ==========

// Trend over time
requests
| make-series RequestCount = count() on timestamp from ago(7d) to now() step 1h
| render timechart

// Compare periods
let thisWeek = requests | where timestamp > ago(7d) | summarize ThisWeek = count();
let lastWeek = requests | where timestamp between (ago(14d) .. ago(7d)) | summarize LastWeek = count();
thisWeek | join lastWeek on $left.$result == $right.$result

// ========== PATTERN DETECTION ==========

// Find anomalies
let baseline = Perf
    | where TimeGenerated > ago(7d) and TimeGenerated < ago(1d)
    | where CounterName == "% Processor Time"
    | summarize AvgCPU = avg(CounterValue), StdDev = stdev(CounterValue) by Computer;

Perf
| where TimeGenerated > ago(1h)
| where CounterName == "% Processor Time"
| summarize CurrentCPU = avg(CounterValue) by Computer
| join baseline on Computer
| extend Zscore = (CurrentCPU - AvgCPU) / StdDev
| where Zscore > 2  // More than 2 standard deviations

// ========== SESSION ANALYSIS ==========

// User sessions
requests
| where timestamp > ago(24h)
| summarize
    SessionStart = min(timestamp),
    SessionEnd = max(timestamp),
    PageViews = count()
    by session_Id, user_Id
| extend SessionDuration = SessionEnd - SessionStart

// ========== PERCENTILES ==========

// Response time percentiles
requests
| where timestamp > ago(1h)
| summarize
    P50 = percentile(duration, 50),
    P90 = percentile(duration, 90),
    P95 = percentile(duration, 95),
    P99 = percentile(duration, 99)
    by bin(timestamp, 5m)

// ========== DYNAMIC PARSING ==========

// Parse JSON
AzureDiagnostics
| extend parsed = parse_json(properties_s)
| extend userName = parsed.userName
| extend action = parsed.action

// Parse key-value pairs
traces
| parse message with * "user=" User " action=" Action " result=" Result
| where Result == "failed"

// ========== WORKING WITH ARRAYS ==========

// mv-expand - Expand array to rows
let data = datatable(Computer:string, Processes:dynamic) [
    "VM1", dynamic(["process1", "process2"]),
    "VM2", dynamic(["process1", "process3"])
];
data
| mv-expand Processes

// ========== CONDITIONAL LOGIC ==========

// case statement
SecurityEvent
| extend Severity = case(
    EventID in (4624, 4634), "Info",
    EventID in (4625, 4648), "Warning",
    EventID in (4720, 4722), "Alert",
    "Unknown"
)

// iff (if-then-else)
requests
| extend Status = iff(resultCode >= 400, "Error", "Success")

// ========== USEFUL FUNCTIONS ==========

// String functions
| extend lower_name = tolower(Name)
| extend trimmed = trim(" ", Message)
| extend parts = split(Path, "/")

// Date functions
| extend DayOfWeek = dayofweek(timestamp)
| extend Hour = hourofday(timestamp)
| extend StartOfDay = startofday(timestamp)

// ago() - Relative time
| where timestamp > ago(1h)
| where timestamp > ago(7d)
| where timestamp > ago(30d)

// datetime comparison
| where timestamp > datetime(2024-01-01)
```

### KQL Query Examples by Scenario

```kusto
// ========== SCENARIO 1: TROUBLESHOOTING APP ERRORS ==========

// Find all errors in the last hour with context
let timeRange = 1h;
exceptions
| where timestamp > ago(timeRange)
| project timestamp, type, outerMessage, innermostMessage,
         operation_Name, cloud_RoleName
| order by timestamp desc

// Error trend
exceptions
| where timestamp > ago(24h)
| summarize ErrorCount = count() by bin(timestamp, 1h), type
| render timechart

// Correlate errors with requests
exceptions
| where timestamp > ago(1h)
| project timestamp, operation_Id, type, outerMessage
| join kind=inner (
    requests
    | where timestamp > ago(1h)
    | project operation_Id, name, url, duration
) on operation_Id


// ========== SCENARIO 2: SECURITY INVESTIGATION ==========

// Failed login attempts by user
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != 0  // Non-zero = failure
| summarize FailedAttempts = count(),
            IPs = make_set(IPAddress),
            LastAttempt = max(TimeGenerated)
    by UserPrincipalName
| where FailedAttempts > 5
| order by FailedAttempts desc

// Suspicious activity: Multiple locations
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType == 0  // Successful
| summarize
    Locations = make_set(Location),
    LocationCount = dcount(Location)
    by UserPrincipalName, bin(TimeGenerated, 1h)
| where LocationCount > 2


// ========== SCENARIO 3: PERFORMANCE ANALYSIS ==========

// Slow requests by endpoint
requests
| where timestamp > ago(1h)
| where duration > 3000  // > 3 seconds
| summarize
    SlowRequests = count(),
    AvgDuration = avg(duration),
    P95Duration = percentile(duration, 95)
    by name
| order by SlowRequests desc

// Dependency performance
dependencies
| where timestamp > ago(1h)
| summarize
    Calls = count(),
    AvgDuration = avg(duration),
    FailureRate = countif(success == false) * 100.0 / count()
    by target, type
| order by AvgDuration desc


// ========== SCENARIO 4: INFRASTRUCTURE MONITORING ==========

// VMs with high CPU
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where InstanceName == "_Total"
| summarize AvgCPU = avg(CounterValue), MaxCPU = max(CounterValue) by Computer
| where AvgCPU > 80

// Disk space warning
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space"
| where InstanceName !in ("_Total", "HarddiskVolume1")
| summarize FreePercent = avg(CounterValue) by Computer, InstanceName
| where FreePercent < 20

// Network throughput
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Network Adapter"
| where CounterName in ("Bytes Received/sec", "Bytes Sent/sec")
| summarize AvgBytesPerSec = avg(CounterValue) by Computer, CounterName, bin(TimeGenerated, 5m)
| render timechart
```

---

## Custom Tables and Data Ingestion

### Creating Custom Log Tables

```bash
# Create custom table via Data Collection Rule

# 1. Define table schema
cat > custom-table-schema.json << 'EOF'
{
  "properties": {
    "schema": {
      "name": "CustomAppLogs_CL",
      "columns": [
        {"name": "TimeGenerated", "type": "datetime"},
        {"name": "Computer", "type": "string"},
        {"name": "Application", "type": "string"},
        {"name": "Level", "type": "string"},
        {"name": "Message", "type": "string"},
        {"name": "CorrelationId", "type": "string"}
      ]
    }
  }
}
EOF

# 2. Create the table
az monitor log-analytics workspace table create \
  --resource-group myRG \
  --workspace-name myWorkspace \
  --name CustomAppLogs_CL \
  --columns TimeGenerated=datetime Computer=string Application=string Level=string Message=string CorrelationId=string
```

### Ingesting Data via API

```bash
# Using the Logs Ingestion API

# 1. Get DCE endpoint and DCR immutable ID
DCE_ENDPOINT=$(az monitor data-collection endpoint show \
  --resource-group myRG \
  --name myDCE \
  --query logsIngestion.endpoint -o tsv)

DCR_ID=$(az monitor data-collection rule show \
  --resource-group myRG \
  --name myDCR \
  --query immutableId -o tsv)

# 2. Get access token
ACCESS_TOKEN=$(az account get-access-token \
  --resource https://monitor.azure.com \
  --query accessToken -o tsv)

# 3. Send data
curl -X POST "${DCE_ENDPOINT}/dataCollectionRules/${DCR_ID}/streams/Custom-CustomAppLogs_CL?api-version=2023-01-01" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '[
    {
      "TimeGenerated": "2024-01-15T10:00:00Z",
      "Computer": "WebServer01",
      "Application": "MyApp",
      "Level": "Error",
      "Message": "Connection timeout to database",
      "CorrelationId": "abc-123"
    }
  ]'
```

---

## Best Practices

```
KQL & LOG ANALYTICS BEST PRACTICES:
───────────────────────────────────

1. ALWAYS FILTER BY TIME FIRST
   ┌─────────────────────────────────────────────────────────────────────┐
   │ // Good: Time filter first, reduces data scanned                     │
   │ AzureDiagnostics                                                     │
   │ | where TimeGenerated > ago(1h)                                      │
   │ | where Category == "SQLInsights"                                    │
   │                                                                       │
   │ // Bad: Full table scan before filtering                             │
   │ AzureDiagnostics                                                     │
   │ | where Category == "SQLInsights"                                    │
   │ | where TimeGenerated > ago(1h)                                      │
   └─────────────────────────────────────────────────────────────────────┘

2. USE 'has' INSTEAD OF 'contains' FOR PERFORMANCE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ // Faster: Uses term index                                           │
   │ | where Message has "error"                                          │
   │                                                                       │
   │ // Slower: Full string scan                                          │
   │ | where Message contains "error"                                     │
   └─────────────────────────────────────────────────────────────────────┘

3. PROJECT ONLY NEEDED COLUMNS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ // Good: Only return what you need                                   │
   │ | project TimeGenerated, Computer, Message                          │
   │                                                                       │
   │ // Bad: Returns all columns (wastes bandwidth)                       │
   │ | project *                                                          │
   └─────────────────────────────────────────────────────────────────────┘

4. USE MATERIALIZED VIEWS FOR FREQUENT QUERIES
   ┌─────────────────────────────────────────────────────────────────────┐
   │ // Pre-aggregate common queries                                      │
   │ .create materialized-view HourlyErrors on exceptions                │
   │ {                                                                    │
   │     exceptions                                                       │
   │     | summarize count() by bin(timestamp, 1h), type                 │
   │ }                                                                    │
   └─────────────────────────────────────────────────────────────────────┘

5. SET APPROPRIATE RETENTION PER TABLE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Security logs: 2 years (compliance)                               │
   │ • Application logs: 90 days (debugging)                             │
   │ • Debug logs: Use Basic logs (8 days)                               │
   │ • Performance counters: 30 days (trending)                          │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Application Insights](03-application-insights.md)* | *Back to [Azure Monitor](01-azure-monitor.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
