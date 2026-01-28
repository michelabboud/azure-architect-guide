# Observability Quick Reference

## Essential KQL Queries

### KQL Basics (Learn in 5 Minutes)

```kusto
// KQL reads left-to-right (like Unix pipes)
// Each line filters/transforms the data

// Basic structure:
TableName
| where TimeGenerated > ago(1h)      // Filter rows
| where Level == "Error"              // More filters
| project TimeGenerated, Message      // Select columns
| order by TimeGenerated desc         // Sort
| take 100                            // Limit results

// AWS CloudWatch Insights equivalent:
// fields @timestamp, @message
// | filter @message like /Error/
// | sort @timestamp desc
// | limit 100
```

### Top 20 KQL Queries for Daily Operations

```kusto
// 1. Find errors in the last hour
AzureDiagnostics
| where TimeGenerated > ago(1h)
| where Level == "Error" or ResultType == "Failed"
| project TimeGenerated, Resource, OperationName, ResultDescription

// 2. Resource creation/deletion (Activity Log)
AzureActivity
| where TimeGenerated > ago(24h)
| where OperationNameValue has "write" or OperationNameValue has "delete"
| project TimeGenerated, Caller, OperationNameValue, ResourceGroup, _ResourceId

// 3. VM CPU usage over 90%
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where CounterValue > 90
| summarize AvgCPU = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)

// 4. Failed sign-ins
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != 0
| summarize FailedAttempts = count() by UserPrincipalName, AppDisplayName, IPAddress
| order by FailedAttempts desc

// 5. Application exceptions (App Insights)
exceptions
| where timestamp > ago(1h)
| summarize count() by type, outerMessage
| order by count_ desc

// 6. Slow requests (App Insights)
requests
| where timestamp > ago(1h)
| where duration > 3000  // 3 seconds
| project timestamp, name, duration, resultCode, url
| order by duration desc

// 7. Container restarts (AKS)
ContainerInventory
| where TimeGenerated > ago(24h)
| where RestartCount > 0
| summarize TotalRestarts = sum(RestartCount) by ContainerID, Name, Image

// 8. Storage account access
StorageBlobLogs
| where TimeGenerated > ago(1h)
| where OperationName has "Get" or OperationName has "Put"
| summarize Count = count() by CallerIpAddress, OperationName

// 9. Firewall blocks
AzureDiagnostics
| where Category == "AzureFirewallNetworkRule"
| where msg_s has "Deny"
| project TimeGenerated, msg_s
| take 100

// 10. SQL query performance
AzureDiagnostics
| where Category == "QueryStoreRuntimeStatistics"
| where duration_d > 5000  // 5 seconds
| project TimeGenerated, query_hash_s, duration_d, cpu_time_d

// 11. Heartbeat check (VM availability)
Heartbeat
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| where LastHeartbeat < ago(5m)  // Missing heartbeat = potential issue

// 12. Memory pressure
Perf
| where ObjectName == "Memory" and CounterName == "% Committed Bytes In Use"
| where CounterValue > 90
| summarize AvgMemory = avg(CounterValue) by Computer, bin(TimeGenerated, 5m)

// 13. Dependency failures (App Insights)
dependencies
| where timestamp > ago(1h)
| where success == false
| summarize FailureCount = count() by target, type, resultCode

// 14. Security alerts
SecurityAlert
| where TimeGenerated > ago(24h)
| project TimeGenerated, AlertName, Severity, Description
| order by TimeGenerated desc

// 15. Bandwidth by resource
AzureMetrics
| where TimeGenerated > ago(24h)
| where MetricName == "BytesSent" or MetricName == "BytesReceived"
| summarize TotalBytes = sum(Total) by Resource, MetricName

// 16. API Management requests
ApiManagementGatewayLogs
| where TimeGenerated > ago(1h)
| summarize RequestCount = count(),
            AvgDuration = avg(TotalTime),
            ErrorCount = countif(ResponseCode >= 400)
  by OperationId

// 17. Key Vault access
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.KEYVAULT"
| where OperationName == "SecretGet"
| project TimeGenerated, CallerIPAddress, identity_claim_upn_s, id_s

// 18. Cost anomalies (requires Cost Management data)
AzureDiagnostics
| where Category == "CostAnomaly"
| project TimeGenerated, AnomalyType, Cost, ExpectedCost

// 19. Network latency
AzureNetworkAnalytics_CL
| where TimeGenerated > ago(1h)
| summarize AvgLatency = avg(LatencyMs_d) by SourceIP_s, DestIP_s

// 20. Custom app logs
CustomLogs_CL
| where TimeGenerated > ago(1h)
| where Level_s == "Error"
| project TimeGenerated, Message_s, Exception_s
```

---

## Azure CLI Commands

### Workspace Management

```bash
# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group myRG \
  --workspace-name myWorkspace \
  --location eastus \
  --retention-time 90

# Get workspace ID (needed for many operations)
az monitor log-analytics workspace show \
  --resource-group myRG \
  --workspace-name myWorkspace \
  --query customerId -o tsv

# Get workspace key
az monitor log-analytics workspace get-shared-keys \
  --resource-group myRG \
  --workspace-name myWorkspace

# List tables in workspace
az monitor log-analytics workspace table list \
  --resource-group myRG \
  --workspace-name myWorkspace \
  --query "[].name" -o table
```

### Diagnostic Settings

```bash
# Enable diagnostic settings for a resource
az monitor diagnostic-settings create \
  --name "send-to-log-analytics" \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{app} \
  --workspace /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws} \
  --logs '[{"category": "AppServiceHTTPLogs", "enabled": true}]' \
  --metrics '[{"category": "AllMetrics", "enabled": true}]'

# List diagnostic settings
az monitor diagnostic-settings list \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{app}

# Delete diagnostic settings
az monitor diagnostic-settings delete \
  --name "send-to-log-analytics" \
  --resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{app}
```

### Application Insights

```bash
# Create Application Insights
az monitor app-insights component create \
  --app myAppInsights \
  --location eastus \
  --resource-group myRG \
  --application-type web \
  --workspace /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws}

# Get instrumentation key
az monitor app-insights component show \
  --app myAppInsights \
  --resource-group myRG \
  --query instrumentationKey -o tsv

# Get connection string (preferred over instrumentation key)
az monitor app-insights component show \
  --app myAppInsights \
  --resource-group myRG \
  --query connectionString -o tsv

# Create availability test
az monitor app-insights web-test create \
  --resource-group myRG \
  --name "HealthCheck" \
  --location eastus \
  --web-test-kind "ping" \
  --defined-web-test-name "HealthCheck" \
  --url "https://myapp.azurewebsites.net/health"
```

### Alerts

```bash
# Create metric alert
az monitor metrics alert create \
  --name "High CPU Alert" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm} \
  --condition "avg Percentage CPU > 90" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/{ag}

# Create log alert
az monitor scheduled-query create \
  --name "Error Alert" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws} \
  --condition "count > 10" \
  --condition-query "AzureDiagnostics | where Level == 'Error'" \
  --action /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/{ag}

# Create action group
az monitor action-group create \
  --resource-group myRG \
  --name myActionGroup \
  --short-name myAG \
  --email-receiver name=admin email=admin@contoso.com \
  --sms-receiver name=oncall phone=1234567890 \
  --webhook-receiver name=slack uri=https://hooks.slack.com/xxx
```

### Run KQL Query

```bash
# Run query against Log Analytics
az monitor log-analytics query \
  --workspace {workspace-id} \
  --analytics-query "AzureDiagnostics | take 10" \
  --timespan P1D

# Run query against Application Insights
az monitor app-insights query \
  --app {app-id} \
  --analytics-query "requests | take 10"
```

---

## AWS to Azure Command Mapping

| Operation | AWS CLI | Azure CLI |
|-----------|---------|-----------|
| Create log group/workspace | `aws logs create-log-group` | `az monitor log-analytics workspace create` |
| Query logs | `aws logs start-query` | `az monitor log-analytics query` |
| Put metric alarm | `aws cloudwatch put-metric-alarm` | `az monitor metrics alert create` |
| Get metrics | `aws cloudwatch get-metric-data` | `az monitor metrics list` |
| Create dashboard | `aws cloudwatch put-dashboard` | `az portal dashboard create` |
| Enable X-Ray/App Insights | SDK configuration | `az monitor app-insights component create` |

---

## Diagnostic Settings Categories

### Common Resources

```
WHAT TO ENABLE PER RESOURCE TYPE:
─────────────────────────────────

APP SERVICE:
┌────────────────────────────────────────────────────────────────────────────┐
│ Category              │ Content                  │ Recommended             │
├───────────────────────┼──────────────────────────┼─────────────────────────┤
│ AppServiceHTTPLogs    │ HTTP request logs        │ ✓ Always                │
│ AppServiceConsoleLogs │ Console output           │ ✓ For debugging         │
│ AppServiceAppLogs     │ Application logs         │ ✓ Always                │
│ AppServiceAuditLogs   │ Audit events             │ ✓ For compliance        │
│ AppServicePlatformLogs│ Platform events          │ ✓ For debugging         │
└────────────────────────────────────────────────────────────────────────────┘

AZURE SQL:
┌────────────────────────────────────────────────────────────────────────────┐
│ Category                      │ Content                │ Recommended        │
├───────────────────────────────┼────────────────────────┼────────────────────┤
│ SQLInsights                   │ Query performance      │ ✓ Always           │
│ QueryStoreRuntimeStatistics   │ Query stats            │ ✓ Always           │
│ QueryStoreWaitStatistics      │ Wait stats             │ ✓ For perf tuning  │
│ Errors                        │ Error events           │ ✓ Always           │
│ DatabaseWaitStatistics        │ Database waits         │ ✓ For perf tuning  │
│ Timeouts                      │ Timeout events         │ ✓ Always           │
│ Blocks                        │ Blocking events        │ ✓ For perf tuning  │
│ Deadlocks                     │ Deadlock events        │ ✓ Always           │
└────────────────────────────────────────────────────────────────────────────┘

STORAGE ACCOUNT:
┌────────────────────────────────────────────────────────────────────────────┐
│ Category              │ Content                  │ Recommended             │
├───────────────────────┼──────────────────────────┼─────────────────────────┤
│ StorageRead           │ Read operations          │ For audit (high volume) │
│ StorageWrite          │ Write operations         │ ✓ For audit             │
│ StorageDelete         │ Delete operations        │ ✓ Always                │
│ Transaction           │ All transactions         │ ✓ Metrics always        │
└────────────────────────────────────────────────────────────────────────────┘

KEY VAULT:
┌────────────────────────────────────────────────────────────────────────────┐
│ Category              │ Content                  │ Recommended             │
├───────────────────────┼──────────────────────────┼─────────────────────────┤
│ AuditEvent            │ All operations           │ ✓ Always (compliance)   │
│ AzurePolicyEvaluated  │ Policy checks            │ For governance          │
└────────────────────────────────────────────────────────────────────────────┘

AKS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Category              │ Content                  │ Recommended             │
├───────────────────────┼──────────────────────────┼─────────────────────────┤
│ kube-apiserver        │ API server logs          │ ✓ For debugging         │
│ kube-controller-manager│ Controller logs         │ For debugging           │
│ kube-scheduler        │ Scheduler logs           │ For debugging           │
│ kube-audit            │ Audit logs               │ ✓ For compliance        │
│ kube-audit-admin      │ Admin audit logs         │ ✓ For security          │
│ guard                 │ AAD integration          │ ✓ If using AAD          │
│ cluster-autoscaler    │ Autoscaler logs          │ For scaling issues      │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Alert Configuration Reference

### Metric Alert Thresholds

```
RECOMMENDED ALERT THRESHOLDS:
─────────────────────────────

VIRTUAL MACHINES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric                    │ Warning    │ Critical  │ Window   │ Frequency │
├───────────────────────────┼────────────┼───────────┼──────────┼───────────┤
│ Percentage CPU            │ > 80%      │ > 95%     │ 5 min    │ 1 min     │
│ Available Memory Bytes    │ < 1 GB     │ < 500 MB  │ 5 min    │ 1 min     │
│ Disk Read/Write Bytes/sec │ Context    │ Context   │ 5 min    │ 5 min     │
│ Network In/Out            │ Context    │ Context   │ 5 min    │ 5 min     │
└────────────────────────────────────────────────────────────────────────────┘

APP SERVICE:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric                    │ Warning    │ Critical  │ Window   │ Frequency │
├───────────────────────────┼────────────┼───────────┼──────────┼───────────┤
│ Http Server Errors (5xx)  │ > 5        │ > 20      │ 5 min    │ 1 min     │
│ Response Time             │ > 3s       │ > 10s     │ 5 min    │ 1 min     │
│ CPU Percentage            │ > 80%      │ > 95%     │ 5 min    │ 1 min     │
│ Memory Percentage         │ > 80%      │ > 95%     │ 5 min    │ 1 min     │
│ Http 4xx                  │ > 50       │ > 200     │ 5 min    │ 5 min     │
└────────────────────────────────────────────────────────────────────────────┘

AZURE SQL:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric                    │ Warning    │ Critical  │ Window   │ Frequency │
├───────────────────────────┼────────────┼───────────┼──────────┼───────────┤
│ DTU Percentage            │ > 80%      │ > 95%     │ 15 min   │ 5 min     │
│ Storage Percent           │ > 80%      │ > 90%     │ 1 hour   │ 15 min    │
│ Deadlocks                 │ > 1        │ > 5       │ 5 min    │ 1 min     │
│ Failed Connections        │ > 10       │ > 50      │ 5 min    │ 1 min     │
│ Workers Percentage        │ > 80%      │ > 95%     │ 5 min    │ 1 min     │
└────────────────────────────────────────────────────────────────────────────┘

STORAGE ACCOUNT:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric                    │ Warning    │ Critical  │ Window   │ Frequency │
├───────────────────────────┼────────────┼───────────┼──────────┼───────────┤
│ Availability              │ < 99.9%    │ < 99%     │ 5 min    │ 1 min     │
│ E2E Latency               │ > 100ms    │ > 500ms   │ 5 min    │ 5 min     │
│ Throttling Errors         │ > 0        │ > 10      │ 5 min    │ 1 min     │
│ Used Capacity             │ > 80%      │ > 90%     │ 1 hour   │ 1 hour    │
└────────────────────────────────────────────────────────────────────────────┘

AKS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric                    │ Warning    │ Critical  │ Window   │ Frequency │
├───────────────────────────┼────────────┼───────────┼──────────┼───────────┤
│ Node CPU %                │ > 80%      │ > 95%     │ 5 min    │ 1 min     │
│ Node Memory %             │ > 80%      │ > 95%     │ 5 min    │ 1 min     │
│ Pod Count (vs limit)      │ > 80%      │ > 95%     │ 5 min    │ 5 min     │
│ Cluster Health            │ != Healthy │ N/A       │ 5 min    │ 1 min     │
│ Container Restarts        │ > 3        │ > 10      │ 15 min   │ 5 min     │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Common Log Analytics Tables

```
FREQUENTLY USED TABLES:
───────────────────────

INFRASTRUCTURE:
• Perf                  - Performance counters (CPU, Memory, Disk, Network)
• Heartbeat             - VM availability (1-minute heartbeat)
• Event                 - Windows event logs
• Syslog                - Linux syslog
• VMConnection          - Network connection data
• InsightsMetrics       - Container and VM insights metrics

AZURE PLATFORM:
• AzureActivity         - Control plane operations (ARM)
• AzureDiagnostics      - Resource diagnostic logs
• AzureMetrics          - Platform metrics
• ResourceHealth        - Resource health events

SECURITY:
• SigninLogs            - Azure AD sign-ins
• AuditLogs             - Azure AD audit events
• SecurityAlert         - Defender alerts
• SecurityEvent         - Security events from VMs

APPLICATION INSIGHTS:
• requests              - HTTP requests
• dependencies          - Calls to external services
• exceptions            - Application exceptions
• traces                - Custom trace logs
• customMetrics         - Custom metrics
• customEvents          - Custom events
• pageViews             - Browser page views
• browserTimings        - Browser performance

CONTAINERS:
• ContainerLog          - Container stdout/stderr
• ContainerInventory    - Container metadata
• KubeEvents            - Kubernetes events
• KubePodInventory      - Pod metadata
• KubeNodeInventory     - Node metadata
```

---

## Workbook Templates

### Quick Workbook JSON

```json
{
  "version": "Notebook/1.0",
  "items": [
    {
      "type": 1,
      "content": {
        "json": "## Resource Health Overview"
      }
    },
    {
      "type": 3,
      "content": {
        "version": "KqlItem/1.0",
        "query": "Heartbeat | summarize LastHeartbeat = max(TimeGenerated) by Computer | extend Status = iff(LastHeartbeat < ago(5m), 'Offline', 'Online')",
        "size": 1,
        "timeContext": {"durationMs": 3600000},
        "queryType": 0,
        "visualization": "table"
      }
    }
  ]
}
```

---

## Retention and Costs

```
LOG ANALYTICS PRICING TIERS:
────────────────────────────

PAY-AS-YOU-GO:
• Ingestion: ~$2.76/GB
• First 31 days retention: Free
• Extended retention: ~$0.12/GB/month

COMMITMENT TIERS (daily commitment):
┌────────────────────────────────────────────────────────────────────────────┐
│ Tier         │ Daily Commitment │ Effective Price │ Savings vs PAYG       │
├──────────────┼──────────────────┼─────────────────┼───────────────────────┤
│ 100 GB/day   │ ~$196/day        │ ~$1.96/GB       │ ~29%                  │
│ 200 GB/day   │ ~$368/day        │ ~$1.84/GB       │ ~33%                  │
│ 300 GB/day   │ ~$522/day        │ ~$1.74/GB       │ ~37%                  │
│ 400 GB/day   │ ~$664/day        │ ~$1.66/GB       │ ~40%                  │
│ 500 GB/day   │ ~$790/day        │ ~$1.58/GB       │ ~43%                  │
└────────────────────────────────────────────────────────────────────────────┘

BASIC LOGS (Limited functionality):
• Ingestion: ~$0.65/GB (76% cheaper)
• 8-day retention only
• Limited query capability
• Best for: High-volume, rarely-queried data
```

---

*Next: [Azure Monitor Fundamentals](01-azure-monitor.md)* | *Back to [Chapter Overview](README.md)*
