# Budgets & Alerts

## Budget Types

### Cost Budgets

```
COST BUDGET CONFIGURATION:
──────────────────────────

SCOPE OPTIONS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Scope              │ Use Case                                              │
├────────────────────┼──────────────────────────────────────────────────────┤
│ Billing Account    │ Enterprise-wide spending cap                         │
│ Management Group   │ Business unit or department budget                   │
│ Subscription       │ Application or team budget                           │
│ Resource Group     │ Project-specific budget                              │
└────────────────────────────────────────────────────────────────────────────┘

AMOUNT OPTIONS:
• Fixed: Same amount every period (e.g., $10,000/month)
• Reset: Budget resets each period
• Rolling: Accumulates unused budget

TIME GRAIN:
• Monthly (most common)
• Quarterly
• Annually

FILTERS:
• Resource group
• Resource type (service)
• Tag values
• Meter category
```

### Creating Budgets

```bash
# CLI - Create subscription budget
az consumption budget create \
  --budget-name "prod-monthly-budget" \
  --amount 50000 \
  --category Cost \
  --time-grain Monthly \
  --start-date "2024-01-01" \
  --end-date "2024-12-31" \
  --resource-group "production-rg"

# Create budget with filter
az consumption budget create \
  --budget-name "vm-budget" \
  --amount 20000 \
  --category Cost \
  --time-grain Monthly \
  --start-date "2024-01-01" \
  --end-date "2024-12-31" \
  --meter-filter "virtual machines"
```

```bicep
// Bicep - Subscription budget
targetScope = 'subscription'

param budgetName string = 'monthly-budget'
param amount int = 50000
param startDate string = '2024-01-01'
param contactEmails array = ['finance@company.com']

resource budget 'Microsoft.Consumption/budgets@2023-05-01' = {
  name: budgetName
  properties: {
    category: 'Cost'
    amount: amount
    timeGrain: 'Monthly'
    timePeriod: {
      startDate: startDate
      endDate: '2024-12-31'
    }
    notifications: {
      Actual_GreaterThan_80_Percent: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 80
        thresholdType: 'Actual'
        contactEmails: contactEmails
      }
    }
  }
}
```

---

## Alert Configuration

### Threshold Types

```
BUDGET THRESHOLD TYPES:
───────────────────────

ACTUAL COST ALERTS:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Triggers when actual spend reaches threshold                              │
│                                                                             │
│  Budget: $10,000                                                           │
│  Threshold: 80%                                                            │
│  Trigger: When actual spend reaches $8,000                                 │
│                                                                             │
│  Typical thresholds: 50%, 75%, 90%, 100%, 110%                            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

FORECASTED COST ALERTS:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Triggers when projected spend will exceed threshold                       │
│                                                                             │
│  Budget: $10,000                                                           │
│  Threshold: 100%                                                           │
│  Trigger: When Azure forecasts you'll exceed $10,000 this period          │
│                                                                             │
│  Benefit: Early warning before you actually exceed budget                  │
│                                                                             │
│  Note: Forecasts available after ~10 days of historical data              │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Notification Channels

```
ALERT NOTIFICATION OPTIONS:
───────────────────────────

1. EMAIL:
   • Up to 5 email addresses per notification
   • Sends budget status and details
   • Can include resource owners via RBAC roles

2. ACTION GROUPS:
   • Email/SMS/Push/Voice
   • Azure Functions
   • Logic Apps
   • Webhooks
   • ITSM (ServiceNow, etc.)
   • Automation Runbooks

3. RBAC ROLES:
   • Notify Owner role holders
   • Notify Contributor role holders
   • Useful for dynamic team membership
```

### Action Group Configuration

```bash
# Create action group for budget alerts
az monitor action-group create \
  --name "FinOps-Alerts" \
  --resource-group "monitoring-rg" \
  --short-name "FinOps" \
  --action email finance-team finance@company.com \
  --action email ops-team ops@company.com

# Create action group with Logic App
az monitor action-group create \
  --name "Budget-Automation" \
  --resource-group "monitoring-rg" \
  --short-name "BudgetAuto" \
  --action logic-app "budget-logic-app" \
    "/subscriptions/{sub}/resourceGroups/automation-rg/providers/Microsoft.Logic/workflows/budget-exceeded"
```

```bicep
// Action group for budget alerts
resource actionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: 'FinOps-Alerts'
  location: 'global'
  properties: {
    groupShortName: 'FinOps'
    enabled: true
    emailReceivers: [
      {
        name: 'Finance Team'
        emailAddress: 'finance@company.com'
        useCommonAlertSchema: true
      }
      {
        name: 'Operations'
        emailAddress: 'ops@company.com'
        useCommonAlertSchema: true
      }
    ]
    smsReceivers: [
      {
        name: 'On-Call'
        countryCode: '1'
        phoneNumber: '5551234567'
      }
    ]
    logicAppReceivers: [
      {
        name: 'Budget Automation'
        resourceId: logicApp.id
        callbackUrl: listCallbackUrl(logicApp.id, '2019-05-01').value
        useCommonAlertSchema: true
      }
    ]
  }
}
```

---

## Budget Automation

### Auto-Remediation with Logic Apps

```
BUDGET AUTOMATION SCENARIOS:
────────────────────────────

SCENARIO 1: Notification Escalation
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  50% threshold → Email to team                                             │
│  75% threshold → Email to team + manager                                   │
│  90% threshold → Email + Slack + Teams notification                        │
│  100% threshold → Email leadership + create Jira ticket                    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

SCENARIO 2: Cost Control Actions
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  90% threshold → Disable auto-scaling (prevent new VMs)                   │
│  100% threshold → Stop non-critical VMs                                    │
│  110% threshold → Alert + require VP approval for new resources           │
│                                                                             │
│  ⚠️  CAUTION: Automated shutdown can cause outages                         │
│      Only use for dev/test or non-critical workloads                       │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

SCENARIO 3: Reporting
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Monthly: Generate cost report → Store in SharePoint → Email CFO          │
│  Weekly: Check budget status → Update Slack channel                        │
│  Daily: Compare forecast vs budget → Alert if trending over               │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Logic App Example

```json
// Logic App - Budget alert handler
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "contentVersion": "1.0.0.0",
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "schemaId": { "type": "string" },
              "data": {
                "type": "object",
                "properties": {
                  "subscriptionName": { "type": "string" },
                  "budgetName": { "type": "string" },
                  "budgetType": { "type": "string" },
                  "notificationThresholdAmount": { "type": "number" },
                  "unit": { "type": "string" }
                }
              }
            }
          }
        }
      }
    },
    "actions": {
      "Parse_Budget_Alert": {
        "type": "ParseJson",
        "inputs": {
          "content": "@triggerBody()",
          "schema": {}
        }
      },
      "Condition_-_Check_Threshold": {
        "type": "If",
        "expression": {
          "and": [
            {
              "greater": [
                "@body('Parse_Budget_Alert')?['data']?['notificationThresholdAmount']",
                90
              ]
            }
          ]
        },
        "actions": {
          "Stop_Dev_VMs": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connection": {
                  "name": "@parameters('$connections')['azurevm']['connectionId']"
                }
              },
              "method": "post",
              "path": "/subscriptions/@{body('Parse_Budget_Alert')?['data']?['subscriptionId']}/resourceGroups/dev-rg/providers/Microsoft.Compute/virtualMachines/stop",
              "queries": {
                "api-version": "2023-03-01"
              }
            }
          },
          "Send_Teams_Alert": {
            "type": "ApiConnection",
            "inputs": {
              "host": {
                "connection": {
                  "name": "@parameters('$connections')['teams']['connectionId']"
                }
              },
              "method": "post",
              "path": "/v3/beta/teams/@{parameters('teamsId')}/channels/@{parameters('channelId')}/messages",
              "body": {
                "body": {
                  "content": "⚠️ Budget Alert: @{body('Parse_Budget_Alert')?['data']?['budgetName']} has exceeded 90% threshold. Dev VMs have been stopped."
                }
              }
            }
          }
        },
        "else": {
          "actions": {
            "Send_Email_Notification": {
              "type": "ApiConnection",
              "inputs": {
                "host": {
                  "connection": {
                    "name": "@parameters('$connections')['office365']['connectionId']"
                  }
                },
                "method": "post",
                "path": "/v2/Mail",
                "body": {
                  "To": "finance@company.com",
                  "Subject": "Budget Alert: @{body('Parse_Budget_Alert')?['data']?['budgetName']}",
                  "Body": "Budget @{body('Parse_Budget_Alert')?['data']?['budgetName']} has reached @{body('Parse_Budget_Alert')?['data']?['notificationThresholdAmount']}% of allocated amount."
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### Azure Automation Runbook

```powershell
# Runbook: Stop-NonCriticalVMs.ps1
# Triggered by budget alert via webhook

param (
    [Parameter(Mandatory=$false)]
    [object] $WebhookData
)

# Parse webhook data
$RequestBody = $WebhookData.RequestBody | ConvertFrom-Json
$BudgetName = $RequestBody.data.budgetName
$Threshold = $RequestBody.data.notificationThresholdAmount

Write-Output "Budget Alert: $BudgetName at $Threshold%"

# Only act if threshold is >= 100%
if ($Threshold -lt 100) {
    Write-Output "Threshold below 100%, no action taken"
    exit
}

# Connect using Managed Identity
Connect-AzAccount -Identity

# Get VMs with "AutoShutdown:Enabled" tag
$TargetRG = "dev-rg"
$VMs = Get-AzVM -ResourceGroupName $TargetRG |
    Where-Object { $_.Tags['AutoShutdown'] -eq 'Enabled' }

foreach ($VM in $VMs) {
    Write-Output "Stopping VM: $($VM.Name)"
    Stop-AzVM -ResourceGroupName $VM.ResourceGroupName -Name $VM.Name -Force -NoWait
}

Write-Output "Stopped $($VMs.Count) VMs due to budget threshold exceeded"

# Send notification
$EmailBody = @"
Budget '$BudgetName' has exceeded 100% threshold.
Automated action: Stopped $($VMs.Count) non-critical VMs in $TargetRG.

VMs stopped:
$($VMs.Name -join "`n")
"@

# Would integrate with SendGrid/Office365 for actual email
Write-Output $EmailBody
```

---

## Budget Best Practices

### Budget Hierarchy

```
RECOMMENDED BUDGET STRUCTURE:
─────────────────────────────

ENTERPRISE LEVEL:
┌────────────────────────────────────────────────────────────────────────────┐
│  Billing Account Budget: $500,000/month                                    │
│  • Overall company cloud spending cap                                      │
│  • Alert at 90%, 100%                                                      │
│  • Notify: CFO, CTO                                                        │
└────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
DEPARTMENT LEVEL:
┌────────────────────────────────────────────────────────────────────────────┐
│  Management Group Budgets:                                                 │
│                                                                             │
│  • Engineering: $300,000/month (60%)                                       │
│  • Sales: $100,000/month (20%)                                             │
│  • Marketing: $50,000/month (10%)                                          │
│  • Corporate: $50,000/month (10%)                                          │
│                                                                             │
│  Alert at 75%, 90%, 100%                                                   │
│  Notify: Department heads, FinOps team                                     │
└────────────────────────────────────────────────────────────────────────────┘
         │
         ▼
APPLICATION LEVEL:
┌────────────────────────────────────────────────────────────────────────────┐
│  Subscription/Resource Group Budgets:                                      │
│                                                                             │
│  • Production App A: $50,000/month                                         │
│  • Production App B: $30,000/month                                         │
│  • Development: $20,000/month                                              │
│  • Shared Services: $25,000/month                                          │
│                                                                             │
│  Alert at 50%, 80%, 100%                                                   │
│  Notify: Application owners, team leads                                    │
└────────────────────────────────────────────────────────────────────────────┘
```

### Budget Monitoring Process

```
BUDGET MONITORING WORKFLOW:
───────────────────────────

DAILY:
┌────────────────────────────────────────────────────────────────────────────┐
│  ☐ Check for anomaly alerts                                                │
│  ☐ Review any forecast warnings                                           │
│  ☐ Investigate significant spikes (>10% daily increase)                   │
└────────────────────────────────────────────────────────────────────────────┘

WEEKLY:
┌────────────────────────────────────────────────────────────────────────────┐
│  ☐ Review week-to-date vs budget                                          │
│  ☐ Check all budgets approaching thresholds                               │
│  ☐ Action Advisor recommendations                                          │
│  ☐ Update forecast if significant changes                                 │
└────────────────────────────────────────────────────────────────────────────┘

MONTHLY:
┌────────────────────────────────────────────────────────────────────────────┐
│  ☐ Close out month - actual vs budget report                              │
│  ☐ Analyze variances (+/- 10%)                                            │
│  ☐ Adjust next month budget if needed                                     │
│  ☐ Review budget alerts triggered                                          │
│  ☐ Update stakeholder cost report                                         │
└────────────────────────────────────────────────────────────────────────────┘

QUARTERLY:
┌────────────────────────────────────────────────────────────────────────────┐
│  ☐ Review annual budget trajectory                                         │
│  ☐ Adjust budgets for growth/reduction                                    │
│  ☐ Update automation rules                                                 │
│  ☐ Present to leadership                                                   │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Alerting Patterns

### Progressive Alerting

```bicep
// Progressive alert thresholds
resource budget 'Microsoft.Consumption/budgets@2023-05-01' = {
  name: 'progressive-budget'
  properties: {
    category: 'Cost'
    amount: 10000
    timeGrain: 'Monthly'
    timePeriod: {
      startDate: '2024-01-01'
      endDate: '2024-12-31'
    }
    notifications: {
      // 50% - Information only
      Actual_50: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 50
        thresholdType: 'Actual'
        contactEmails: ['team@company.com']
      }
      // 75% - Warning
      Actual_75: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 75
        thresholdType: 'Actual'
        contactEmails: ['team@company.com', 'manager@company.com']
      }
      // 90% - Action required
      Actual_90: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 90
        thresholdType: 'Actual'
        contactEmails: ['team@company.com', 'manager@company.com']
        contactGroups: [actionGroup.id]
      }
      // 100% - Critical
      Actual_100: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 100
        thresholdType: 'Actual'
        contactEmails: ['team@company.com', 'manager@company.com', 'director@company.com']
        contactGroups: [actionGroup.id]
      }
      // Forecasted 100% - Early warning
      Forecasted_100: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 100
        thresholdType: 'Forecasted'
        contactEmails: ['team@company.com', 'manager@company.com']
      }
    }
  }
}
```

### Environment-Specific Alerts

```
ALERT STRATEGY BY ENVIRONMENT:
──────────────────────────────

PRODUCTION:
┌────────────────────────────────────────────────────────────────────────────┐
│  • Alert only - NO automated shutdown                                      │
│  • Lower thresholds for early warning (50%, 75%, 90%, 100%)               │
│  • Notify ops team + management                                            │
│  • Forecasted alerts enabled                                               │
└────────────────────────────────────────────────────────────────────────────┘

STAGING/TEST:
┌────────────────────────────────────────────────────────────────────────────┐
│  • Higher threshold for alerts (75%, 100%)                                 │
│  • Can automate shutdown at 110% with warning                             │
│  • Notify team only                                                        │
│  • Focus on preventing runaway costs                                       │
└────────────────────────────────────────────────────────────────────────────┘

DEVELOPMENT:
┌────────────────────────────────────────────────────────────────────────────┐
│  • Aggressive thresholds (100%)                                            │
│  • Automate shutdown at 100% threshold                                     │
│  • Notify developer only                                                   │
│  • Auto-shutdown after hours anyway                                        │
└────────────────────────────────────────────────────────────────────────────┘

SANDBOX/TRAINING:
┌────────────────────────────────────────────────────────────────────────────┐
│  • Hard cap - automate cleanup at 100%                                     │
│  • Short time window (daily or weekly reset)                               │
│  • Self-service budget requests                                            │
│  • Delete resources after period ends                                      │
└────────────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Reservations & Savings Plans](03-reservations.md)* | *Back to [Cost Analysis](01-cost-analysis.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
