# Alerting & Automation

## Azure Monitor Alerts Overview

Azure Monitor Alerts provide a unified alerting experience across all Azure resources. Think of it as CloudWatch Alarms but with more flexibility and integration options.

### Alert Types

```
AZURE MONITOR ALERT TYPES:
──────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. METRIC ALERTS                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Trigger: When a metric crosses a threshold                           │   │
│  │ Latency: Near real-time (1-5 minutes)                               │   │
│  │ Cost: ~$0.10/alert rule/month                                        │   │
│  │                                                                       │   │
│  │ Examples:                                                            │   │
│  │ • CPU > 90% for 5 minutes                                           │   │
│  │ • Memory < 500 MB                                                    │   │
│  │ • HTTP 5xx errors > 10/minute                                       │   │
│  │                                                                       │   │
│  │ AWS Equivalent: CloudWatch Metric Alarms                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. LOG ALERTS (Scheduled Query)                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Trigger: When a KQL query returns results                            │   │
│  │ Latency: Based on frequency (1 min - 24 hours)                      │   │
│  │ Cost: ~$0.25/alert rule/month + query costs                         │   │
│  │                                                                       │   │
│  │ Examples:                                                            │   │
│  │ • Error count > 50 in last 15 minutes                               │   │
│  │ • Failed logins from same IP > 10                                   │   │
│  │ • Custom application events                                         │   │
│  │                                                                       │   │
│  │ AWS Equivalent: CloudWatch Logs Metric Filters + Alarms             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. ACTIVITY LOG ALERTS                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Trigger: When specific Azure operations occur                        │   │
│  │ Latency: Near real-time (minutes)                                   │   │
│  │ Cost: Free (included)                                                │   │
│  │                                                                       │   │
│  │ Examples:                                                            │   │
│  │ • VM deleted                                                         │   │
│  │ • NSG rule changed                                                   │   │
│  │ • Role assignment created                                           │   │
│  │                                                                       │   │
│  │ AWS Equivalent: CloudTrail + EventBridge + SNS                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  4. SMART DETECTION (App Insights)                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Trigger: ML-based anomaly detection                                  │   │
│  │ Latency: Continuous analysis                                         │   │
│  │ Cost: Included with App Insights                                    │   │
│  │                                                                       │   │
│  │ Auto-detects:                                                        │   │
│  │ • Unusual increase in failures                                      │   │
│  │ • Abnormal response times                                           │   │
│  │ • Memory leaks                                                       │   │
│  │ • Sudden traffic changes                                            │   │
│  │                                                                       │   │
│  │ AWS Equivalent: CloudWatch Anomaly Detection                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  5. SERVICE HEALTH ALERTS                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Trigger: Azure service incidents, maintenance                        │   │
│  │ Latency: As events occur                                            │   │
│  │ Cost: Free                                                           │   │
│  │                                                                       │   │
│  │ Types:                                                               │   │
│  │ • Service issues (outages)                                          │   │
│  │ • Planned maintenance                                               │   │
│  │ • Health advisories                                                 │   │
│  │ • Security advisories                                               │   │
│  │                                                                       │   │
│  │ AWS Equivalent: AWS Health Dashboard + EventBridge                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Creating Alerts

### Metric Alerts

```bash
# Create metric alert via CLI
az monitor metrics alert create \
  --name "High-CPU-Alert" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm} \
  --condition "avg Percentage CPU > 90" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --description "Alert when CPU exceeds 90% for 5 minutes" \
  --action /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/{ag}

# Multiple conditions (AND logic)
az monitor metrics alert create \
  --name "Memory-And-CPU-Alert" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm} \
  --condition "avg Percentage CPU > 80" \
  --condition "avg Available Memory Bytes < 1073741824" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/{ag}

# Dynamic threshold (ML-based)
az monitor metrics alert create \
  --name "Anomaly-Detection-Alert" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{app} \
  --condition "avg Percentage CPU > dynamic medium 4 of 5 since 2024-01-01T00:00:00Z" \
  --window-size 5m \
  --evaluation-frequency 5m \
  --action /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/{ag}
```

### Log Alerts (Scheduled Query)

```bash
# Create log alert
az monitor scheduled-query create \
  --name "High-Error-Rate" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws} \
  --condition "count > 50" \
  --condition-query "AppExceptions | where TimeGenerated > ago(15m) | where SeverityLevel == 3" \
  --window-size 15m \
  --evaluation-frequency 5m \
  --severity 2 \
  --action /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/{ag}

# With dimension grouping (separate alert per dimension)
az monitor scheduled-query create \
  --name "Error-Per-Service" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.OperationalInsights/workspaces/{ws} \
  --condition "count > 10" \
  --condition-query "AppExceptions | summarize count() by cloud_RoleName" \
  --window-size 15m \
  --evaluation-frequency 5m \
  --dimension "cloud_RoleName" "operator=Include" "values=*"
```

### Activity Log Alerts

```bash
# Alert on VM deletion
az monitor activity-log alert create \
  --name "VM-Deletion-Alert" \
  --resource-group myRG \
  --condition "category=Administrative" \
  --condition "operationName=Microsoft.Compute/virtualMachines/delete" \
  --scope /subscriptions/{sub}/resourceGroups/{rg} \
  --action-group /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/{ag}

# Alert on any delete operation
az monitor activity-log alert create \
  --name "Resource-Deletion-Alert" \
  --resource-group myRG \
  --condition "category=Administrative" \
  --condition "operationName contains delete" \
  --condition "level=Warning or level=Error" \
  --scope /subscriptions/{sub} \
  --action-group /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/{ag}
```

---

## Action Groups

### Action Group Configuration

```
ACTION GROUPS:
──────────────

Define who gets notified and how when an alert fires.

NOTIFICATION TYPES:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  EMAIL/SMS/PUSH/VOICE                                                │   │
│  │                                                                       │   │
│  │  • Email: Up to 1000 recipients                                      │   │
│  │  • SMS: International supported                                      │   │
│  │  • Azure app push notification                                       │   │
│  │  • Voice call (automated)                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  AZURE ACTIONS                                                        │   │
│  │                                                                       │   │
│  │  • Automation Runbook: Run PowerShell/Python scripts                │   │
│  │  • Azure Function: Execute custom code                              │   │
│  │  • Logic App: Complex workflow                                      │   │
│  │  • Event Hub: Stream to SIEM/analytics                              │   │
│  │  • Secure Webhook: Call external API with AAD auth                  │   │
│  │  • ITSM Connector: ServiceNow, etc.                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  WEBHOOK                                                              │   │
│  │                                                                       │   │
│  │  • Slack, Teams, PagerDuty, Opsgenie                                │   │
│  │  • Any HTTP endpoint                                                │   │
│  │  • Common Alert Schema for consistency                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Create Action Group

```bash
# Create action group with multiple notification types
az monitor action-group create \
  --resource-group myRG \
  --name "ops-team-critical" \
  --short-name "ops-crit" \
  --email-receiver name="OpsTeam" email="ops@contoso.com" useCommonAlertSchema=true \
  --email-receiver name="OpsLead" email="ops-lead@contoso.com" useCommonAlertSchema=true \
  --sms-receiver name="OnCall" countryCode="1" phoneNumber="5551234567" \
  --webhook-receiver name="Slack" uri="https://hooks.slack.com/services/xxx" useCommonAlertSchema=true \
  --webhook-receiver name="PagerDuty" uri="https://events.pagerduty.com/integration/xxx/enqueue"

# Action group with Azure Function
az monitor action-group create \
  --resource-group myRG \
  --name "auto-remediation" \
  --short-name "auto-rem" \
  --azure-function-receiver name="RestartVM" \
    function-app-resource-id=/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/{functionApp} \
    function-name="RestartVMFunction" \
    http-trigger-url="https://{functionApp}.azurewebsites.net/api/RestartVM"

# Action group with Automation Runbook
az monitor action-group create \
  --resource-group myRG \
  --name "runbook-action" \
  --short-name "runbook" \
  --automation-runbook-receiver name="ScaleUp" \
    automation-account-id=/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Automation/automationAccounts/{account} \
    runbook-name="ScaleUpVM" \
    webhook-resource-id=/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Automation/automationAccounts/{account}/webhooks/{webhook} \
    is-global-runbook=false \
    service-uri="https://{webhook-uri}"
```

---

## Alert Processing Rules

### Managing Alert Noise

```
ALERT PROCESSING RULES:
───────────────────────

Control how alerts are processed AFTER they fire.

USE CASES:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. SUPPRESSION (Maintenance Windows)                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Suppress alerts during planned maintenance                           │   │
│  │                                                                       │   │
│  │ Example:                                                             │   │
│  │ • Every Sunday 2-4 AM: Suppress all alerts in "prod" RG             │   │
│  │ • One-time: 2024-01-15 1-3 PM for patching                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  2. ADD ACTION GROUPS                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Add extra notifications based on criteria                            │   │
│  │                                                                       │   │
│  │ Example:                                                             │   │
│  │ • Severity 0-1 alerts → Also notify executives                      │   │
│  │ • After-hours alerts → Page on-call                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  3. REMOVE ACTION GROUPS                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Remove notifications based on criteria                               │   │
│  │                                                                       │   │
│  │ Example:                                                             │   │
│  │ • Dev/test resources → Remove page, only email                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Create suppression rule for maintenance window
az monitor alert-processing-rule create \
  --name "Sunday-Maintenance" \
  --resource-group myRG \
  --scopes /subscriptions/{sub}/resourceGroups/prod-rg \
  --rule-type RemoveAllActionGroups \
  --schedule-recurrence-type Weekly \
  --schedule-recurrence "Sunday" \
  --schedule-start-date-time "2024-01-01 02:00:00" \
  --schedule-end-date-time "2024-12-31 04:00:00" \
  --schedule-recurrence-start-time "02:00:00" \
  --schedule-recurrence-end-time "04:00:00" \
  --schedule-time-zone "America/New_York"

# Add additional notifications for critical alerts
az monitor alert-processing-rule create \
  --name "Critical-Alert-Escalation" \
  --resource-group myRG \
  --scopes /subscriptions/{sub} \
  --filter-severity "Equals" "Sev0" "Sev1" \
  --rule-type AddActionGroups \
  --action-groups /subscriptions/{sub}/resourceGroups/{rg}/providers/microsoft.insights/actionGroups/executives
```

---

## Auto-Remediation

### Common Auto-Remediation Patterns

```
AUTO-REMEDIATION PATTERNS:
──────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  PATTERN 1: VM RESTART ON HIGH CPU                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  Alert: CPU > 95% for 15 minutes                                     │   │
│  │        ↓                                                              │   │
│  │  Action Group: Azure Function                                        │   │
│  │        ↓                                                              │   │
│  │  Function: Restart VM using Azure SDK                               │   │
│  │        ↓                                                              │   │
│  │  Notification: Email ops team                                        │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PATTERN 2: AUTO-SCALE ON QUEUE DEPTH                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  Alert: Queue depth > 1000 messages                                  │   │
│  │        ↓                                                              │   │
│  │  Action Group: Logic App                                             │   │
│  │        ↓                                                              │   │
│  │  Logic App: Scale VMSS/AKS replicas                                 │   │
│  │        ↓                                                              │   │
│  │  Notification: Teams channel                                        │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PATTERN 3: DISK CLEANUP ON LOW SPACE                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  Alert: Disk free space < 10%                                        │   │
│  │        ↓                                                              │   │
│  │  Action Group: Automation Runbook                                   │   │
│  │        ↓                                                              │   │
│  │  Runbook: Clean temp files, compress logs                           │   │
│  │        ↓                                                              │   │
│  │  Notification: Email if still low after cleanup                     │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  PATTERN 4: SECURITY RESPONSE                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  Alert: Failed logins > 10 from same IP                             │   │
│  │        ↓                                                              │   │
│  │  Action Group: Azure Function                                        │   │
│  │        ↓                                                              │   │
│  │  Function: Add IP to NSG deny rule                                  │   │
│  │        ↓                                                              │   │
│  │  Notification: Security team + Create ticket                        │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Azure Function for Auto-Remediation

```csharp
// Azure Function to restart VM on alert
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Azure.Identity;
using Azure.ResourceManager;
using Azure.ResourceManager.Compute;

public class RestartVmFunction
{
    [Function("RestartVM")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequestData req,
        FunctionContext context)
    {
        var logger = context.GetLogger("RestartVM");

        // Parse the alert payload
        var body = await req.ReadAsStringAsync();
        var alert = JsonSerializer.Deserialize<AlertPayload>(body);

        // Extract VM resource ID from alert context
        var vmResourceId = alert.Data.Essentials.AlertTargetIDs[0];

        logger.LogInformation($"Restarting VM: {vmResourceId}");

        // Use managed identity to authenticate
        var credential = new DefaultAzureCredential();
        var client = new ArmClient(credential);

        // Get the VM resource
        var vmResource = client.GetVirtualMachineResource(new ResourceIdentifier(vmResourceId));

        // Restart the VM
        await vmResource.RestartAsync(WaitUntil.Started);

        logger.LogInformation("VM restart initiated successfully");

        var response = req.CreateResponse(HttpStatusCode.OK);
        return response;
    }
}

// Alert payload structure
public class AlertPayload
{
    public AlertData Data { get; set; }
}

public class AlertData
{
    public AlertEssentials Essentials { get; set; }
}

public class AlertEssentials
{
    public string[] AlertTargetIDs { get; set; }
    public string AlertRule { get; set; }
    public string Severity { get; set; }
}
```

### Logic App for Complex Workflows

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/schemas/2016-06-01/workflowdefinition.json#",
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "schemaId": { "type": "string" },
              "data": { "type": "object" }
            }
          }
        }
      }
    },
    "actions": {
      "Parse_Alert": {
        "type": "ParseJson",
        "inputs": {
          "content": "@triggerBody()",
          "schema": { /* Common Alert Schema */ }
        }
      },
      "Get_VM_Details": {
        "type": "Http",
        "inputs": {
          "method": "GET",
          "uri": "@{body('Parse_Alert')?['data']?['essentials']?['alertTargetIDs'][0]}?api-version=2023-07-01",
          "authentication": { "type": "ManagedServiceIdentity" }
        }
      },
      "Condition_Check_If_Already_Restarted": {
        "type": "If",
        "expression": {
          "and": [
            {
              "less": [
                "@body('Get_VM_Details')?['properties']?['instanceView']?['statuses'][1]?['time']",
                "@addMinutes(utcNow(), -30)"
              ]
            }
          ]
        },
        "actions": {
          "Restart_VM": {
            "type": "Http",
            "inputs": {
              "method": "POST",
              "uri": "@{body('Parse_Alert')?['data']?['essentials']?['alertTargetIDs'][0]}/restart?api-version=2023-07-01",
              "authentication": { "type": "ManagedServiceIdentity" }
            }
          },
          "Post_To_Teams": {
            "type": "Http",
            "inputs": {
              "method": "POST",
              "uri": "https://outlook.office.com/webhook/xxx",
              "body": {
                "text": "VM @{body('Get_VM_Details')?['name']} was automatically restarted due to high CPU"
              }
            }
          }
        }
      }
    }
  }
}
```

---

## Best Practices

```
ALERTING BEST PRACTICES:
────────────────────────

1. USE APPROPRIATE SEVERITY LEVELS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Sev 0: Critical - Service down, immediate action required           │
   │ Sev 1: Error - Significant impact, urgent attention                 │
   │ Sev 2: Warning - Potential issue, investigate soon                  │
   │ Sev 3: Informational - Awareness, no immediate action              │
   │ Sev 4: Verbose - Detailed logging, typically suppressed            │
   └─────────────────────────────────────────────────────────────────────┘

2. AVOID ALERT FATIGUE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Set thresholds based on real impact, not arbitrary numbers        │
   │ • Use longer evaluation windows for noisy metrics                   │
   │ • Implement auto-resolution to clear transient issues              │
   │ • Group related alerts (one alert for "database issues")           │
   │ • Use alert processing rules for maintenance windows               │
   └─────────────────────────────────────────────────────────────────────┘

3. ENSURE ALERTS ARE ACTIONABLE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Every alert should answer:                                          │
   │ • What happened?                                                    │
   │ • What is the impact?                                               │
   │ • What action should be taken?                                      │
   │ • Who should act on it?                                             │
   │                                                                       │
   │ Include runbook links in alert descriptions!                        │
   └─────────────────────────────────────────────────────────────────────┘

4. TEST YOUR ALERTS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Verify alerts fire when expected                                  │
   │ • Verify notifications reach the right people                       │
   │ • Test auto-remediation in non-prod first                          │
   │ • Review alerts monthly for relevance                               │
   └─────────────────────────────────────────────────────────────────────┘

5. IMPLEMENT ESCALATION PATHS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ Level 1: Ops team (email + Teams)                                  │
   │ Level 2: On-call (PagerDuty) after 15 min no response              │
   │ Level 3: Management after 30 min no response                       │
   │                                                                       │
   │ Use alert processing rules to add escalation action groups         │
   └─────────────────────────────────────────────────────────────────────┘

6. DOCUMENT EVERYTHING
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Alert purpose and business impact                                 │
   │ • Threshold rationale                                               │
   │ • Runbook for response                                              │
   │ • Owner and escalation path                                         │
   │ • Last review date                                                   │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Case Studies](case-studies.md)* | *Back to [Application Insights](03-application-insights.md)*
