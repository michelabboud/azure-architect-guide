# FinOps Quick Reference

## Essential CLI Commands

### Cost Management

```bash
# View current month's cost
az consumption usage list \
  --subscription "sub-name" \
  --start-date "2024-01-01" \
  --end-date "2024-01-31" \
  --query "[].{Resource:instanceName, Cost:pretaxCost, Currency:currency}" \
  --output table

# Get cost by resource group
az consumption usage list \
  --subscription "sub-name" \
  --start-date "2024-01-01" \
  --end-date "2024-01-31" \
  --query "[].{RG:instanceId, Cost:pretaxCost}" \
  --output table | sort -t$'\t' -k2 -rn

# List all budgets
az consumption budget list --output table

# Create a budget
az consumption budget create \
  --budget-name "Monthly-Production-Budget" \
  --amount 10000 \
  --category Cost \
  --time-grain Monthly \
  --start-date "2024-01-01" \
  --end-date "2024-12-31"

# Get budget status
az consumption budget show --budget-name "Monthly-Production-Budget"

# List reservations
az reservations reservation-order list --output table

# Get reservation utilization
az consumption reservation summary list \
  --reservation-order-id "order-id" \
  --grain daily \
  --start-date "2024-01-01" \
  --end-date "2024-01-31"
```

### Azure Advisor (Cost Recommendations)

```bash
# Get all cost recommendations
az advisor recommendation list \
  --category Cost \
  --output table

# Get recommendations with details
az advisor recommendation list \
  --category Cost \
  --query "[].{Impact:impact, Problem:shortDescription.problem, Solution:shortDescription.solution}"

# Get specific recommendation types
az advisor recommendation list \
  --category Cost \
  --query "[?contains(shortDescription.problem, 'underutilized')]"

# Dismiss a recommendation (if not applicable)
az advisor recommendation disable \
  --ids "/subscriptions/xxx/resourceGroups/rg/providers/Microsoft.Advisor/recommendations/rec-id"
```

### Resource Cleanup

```bash
# Find unattached managed disks
az disk list \
  --query "[?diskState=='Unattached'].{Name:name, RG:resourceGroup, Size:diskSizeGb}" \
  --output table

# Find unattached public IPs
az network public-ip list \
  --query "[?ipConfiguration==null].{Name:name, RG:resourceGroup}" \
  --output table

# Find unused NICs
az network nic list \
  --query "[?virtualMachine==null].{Name:name, RG:resourceGroup}" \
  --output table

# Find empty resource groups
az group list --query "[?properties.provisioningState=='Succeeded']" -o json | \
  jq -r '.[].name' | while read rg; do
    count=$(az resource list -g "$rg" --query "length(@)")
    if [ "$count" == "0" ]; then
      echo "Empty: $rg"
    fi
  done

# Find VMs with low CPU utilization (requires metrics)
az monitor metrics list \
  --resource "/subscriptions/xxx/resourceGroups/rg/providers/Microsoft.Compute/virtualMachines/vm-name" \
  --metric "Percentage CPU" \
  --interval PT1H \
  --aggregation Average \
  --query "value[0].timeseries[0].data[].average"
```

---

## Cost Analysis Queries

### Azure Resource Graph (KQL)

```kusto
// Total resources by type
Resources
| summarize Count=count() by type
| order by Count desc
| take 20

// Resources by location
Resources
| summarize Count=count() by location
| order by Count desc

// Resources without required tags
Resources
| where tags['CostCenter'] == '' or isnull(tags['CostCenter'])
| project name, type, resourceGroup, subscriptionId
| take 100

// VMs by size and count
Resources
| where type =~ 'microsoft.compute/virtualmachines'
| extend vmSize = properties.hardwareProfile.vmSize
| summarize Count=count() by tostring(vmSize)
| order by Count desc

// Storage accounts by tier
Resources
| where type =~ 'microsoft.storage/storageaccounts'
| extend tier = properties.accessTier
| summarize Count=count() by tostring(tier)

// Resources created in last 7 days
Resources
| extend createdTime = todatetime(tags['CreatedDate'])
| where createdTime > ago(7d)
| project name, type, resourceGroup, createdTime
| order by createdTime desc
```

### Log Analytics Cost Queries

```kusto
// Data ingestion by table (last 30 days)
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize TotalGB = sum(Quantity) / 1000 by DataType
| order by TotalGB desc

// Top 10 data sources by volume
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize TotalGB = sum(Quantity) / 1000 by Solution
| top 10 by TotalGB desc

// Daily ingestion trend
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize DailyGB = sum(Quantity) / 1000 by bin(TimeGenerated, 1d)
| render timechart

// Estimated monthly cost (at $2.76/GB)
Usage
| where TimeGenerated > ago(30d)
| where IsBillable == true
| summarize TotalGB = sum(Quantity) / 1000
| extend EstimatedCost = TotalGB * 2.76
```

---

## Cost Comparison Tables

### VM Pricing Tiers

```
VM PRICING OPTIONS:
───────────────────

┌─────────────────┬──────────────┬────────────┬─────────────────────────────┐
│ Option          │ Discount     │ Commitment │ Best For                    │
├─────────────────┼──────────────┼────────────┼─────────────────────────────┤
│ Pay-as-you-go   │ 0%           │ None       │ Short-term, variable        │
│ Spot VMs        │ Up to 90%    │ None       │ Interruptible workloads    │
│ Savings Plan    │ Up to 65%    │ 1 or 3 yr  │ Flexible, variable sizes   │
│ Reservation     │ Up to 72%    │ 1 or 3 yr  │ Predictable, fixed size    │
│ Hybrid Benefit  │ Up to 49%    │ License    │ Existing Windows licenses  │
│ Dev/Test        │ ~55%         │ None       │ Non-prod with VS sub       │
└─────────────────┴──────────────┴────────────┴─────────────────────────────┘

COMBINING DISCOUNTS:
• Reservation + Hybrid Benefit = Up to 80% savings
• Savings Plan + Hybrid Benefit = Up to 75% savings
• Dev/Test pricing cannot combine with reservations
```

### Storage Pricing Tiers

```
STORAGE ACCOUNT PRICING (per GB/month, LRS, East US):
─────────────────────────────────────────────────────

┌─────────────────┬────────────┬─────────────┬────────────────────────────┐
│ Tier            │ Storage    │ Access      │ Best For                   │
├─────────────────┼────────────┼─────────────┼────────────────────────────┤
│ Hot             │ $0.018/GB  │ $0.0004/10K │ Frequently accessed data   │
│ Cool            │ $0.010/GB  │ $0.01/10K   │ Infrequent (30+ days)     │
│ Cold            │ $0.0045/GB │ $0.018/10K  │ Rarely accessed (90+ days)│
│ Archive         │ $0.00099/GB│ $5.00/10K   │ Long-term backup (180+)   │
└─────────────────┴────────────┴─────────────┴────────────────────────────┘

NOTE: Access costs include read operations. Early deletion fees apply.
```

### Database Pricing

```
AZURE SQL DATABASE PRICING (vCore model):
─────────────────────────────────────────

┌─────────────────┬────────────────┬────────────────┬─────────────────────┐
│ Tier            │ 4 vCores/month │ Max Size       │ Use Case            │
├─────────────────┼────────────────┼────────────────┼─────────────────────┤
│ Basic DTU       │ ~$5            │ 2 GB           │ Dev/test            │
│ Standard DTU    │ ~$75           │ 250 GB         │ Small production    │
│ Premium DTU     │ ~$465          │ 1 TB           │ High performance    │
│ GP vCore        │ ~$400          │ 4 TB           │ General workloads   │
│ BC vCore        │ ~$850          │ 4 TB           │ Mission critical    │
│ Hyperscale      │ ~$600 + storage│ 100 TB         │ Large databases     │
│ Serverless      │ Per-second     │ 4 TB           │ Intermittent usage  │
└─────────────────┴────────────────┴────────────────┴─────────────────────┘

RESERVATION SAVINGS:
• 1-year: ~20% discount
• 3-year: ~35% discount
```

---

## Budget Templates

### Bicep Budget Definition

```bicep
resource budget 'Microsoft.Consumption/budgets@2023-05-01' = {
  name: 'monthly-budget-${subscription().subscriptionId}'
  properties: {
    category: 'Cost'
    amount: 10000
    timeGrain: 'Monthly'
    timePeriod: {
      startDate: '2024-01-01'
      endDate: '2024-12-31'
    }
    filter: {
      dimensions: {
        name: 'ResourceGroupName'
        operator: 'In'
        values: [
          'production-rg'
          'shared-services-rg'
        ]
      }
    }
    notifications: {
      NotificationForExceeded80Percent: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 80
        thresholdType: 'Actual'
        contactEmails: [
          'finance@company.com'
          'ops@company.com'
        ]
        contactRoles: [
          'Owner'
          'Contributor'
        ]
      }
      NotificationForExceeded100Percent: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 100
        thresholdType: 'Actual'
        contactEmails: [
          'finance@company.com'
          'leadership@company.com'
        ]
      }
      NotificationForForecasted100Percent: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 100
        thresholdType: 'Forecasted'
        contactEmails: [
          'finance@company.com'
        ]
      }
    }
  }
}
```

### Terraform Budget

```hcl
resource "azurerm_consumption_budget_subscription" "monthly" {
  name            = "monthly-budget"
  subscription_id = data.azurerm_subscription.current.id

  amount     = 10000
  time_grain = "Monthly"

  time_period {
    start_date = "2024-01-01T00:00:00Z"
    end_date   = "2024-12-31T00:00:00Z"
  }

  filter {
    dimension {
      name = "ResourceGroupName"
      values = [
        "production-rg",
        "shared-services-rg"
      ]
    }
  }

  notification {
    enabled        = true
    threshold      = 80.0
    operator       = "GreaterThan"
    threshold_type = "Actual"

    contact_emails = [
      "finance@company.com",
      "ops@company.com"
    ]

    contact_roles = [
      "Owner",
      "Contributor"
    ]
  }

  notification {
    enabled        = true
    threshold      = 100.0
    operator       = "GreaterThan"
    threshold_type = "Actual"

    contact_emails = [
      "finance@company.com",
      "leadership@company.com"
    ]
  }

  notification {
    enabled        = true
    threshold      = 100.0
    operator       = "GreaterThan"
    threshold_type = "Forecasted"

    contact_emails = [
      "finance@company.com"
    ]
  }
}
```

---

## Reservation Recommendations

### When to Reserve

```
RESERVATION DECISION FRAMEWORK:
───────────────────────────────

RESERVE WHEN:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│ ✅ Workload runs 24/7 (production databases, always-on VMs)               │
│ ✅ Predictable, stable resource usage                                      │
│ ✅ Same VM size/SKU for 1+ years                                          │
│ ✅ Utilization will exceed 60% of commitment                              │
│ ✅ Budget available for upfront or monthly payment                         │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

USE SAVINGS PLAN INSTEAD WHEN:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│ ✅ VM sizes may change (modernization, scaling)                           │
│ ✅ Regions may change (DR, expansion)                                     │
│ ✅ Mix of different VM families                                           │
│ ✅ Want flexibility with similar savings                                   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

USE PAY-AS-YOU-GO WHEN:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│ ✅ Short-term or temporary workloads                                       │
│ ✅ Unpredictable usage patterns                                            │
│ ✅ Testing and development                                                 │
│ ✅ Resources that scale to zero                                            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Calculate Reservation Savings

```bash
# Get VM recommendations
az advisor recommendation list \
  --category Cost \
  --query "[?contains(shortDescription.problem, 'reserved')]" \
  --output table

# Calculate break-even point
# Formula: Reservation Cost / (PAYG Rate - Reservation Rate) = Break-even hours

# Example: D4s_v3 in East US
# PAYG: $0.192/hour = $140.16/month
# 1-year reservation: $96/month
# 3-year reservation: $61/month

# 1-year break-even:
# If using >68% of the month (500 hours), reservation saves money

# 3-year break-even:
# If using >43% of the month (318 hours), reservation saves money
```

---

## Optimization Checklist

### Weekly Review

```
WEEKLY COST REVIEW CHECKLIST:
─────────────────────────────

□ Check this week's spend vs budget
□ Review any new Advisor cost recommendations
□ Investigate any cost spikes (+20% from normal)
□ Check reservation utilization (target: >95%)
□ Review orphaned resources report
□ Check dev/test auto-shutdown compliance
```

### Monthly Review

```
MONTHLY COST REVIEW CHECKLIST:
──────────────────────────────

□ Compare actual vs budgeted spend
□ Review cost by department/project (tags)
□ Analyze cost trends (month-over-month)
□ Review and action all Advisor recommendations
□ Check commitment discount utilization
□ Update forecasts for next quarter
□ Review and clean up unused resources
□ Validate tag compliance
□ Review reservation purchase opportunities
□ Present cost report to stakeholders
```

### Quarterly Review

```
QUARTERLY COST REVIEW CHECKLIST:
────────────────────────────────

□ Review enterprise agreement utilization
□ Assess reservation/savings plan coverage
□ Conduct architecture review for optimization
□ Review pricing tier appropriateness
□ Evaluate commitment discount renewals
□ Update tagging strategy if needed
□ Review FinOps process maturity
□ Set cost targets for next quarter
□ Review and update budgets
□ Conduct cost anomaly retrospective
```

---

## AWS to Azure Cost Mapping

```
COST TOOL COMPARISON:
─────────────────────

┌───────────────────────────┬──────────────────────────────────────────────┐
│ AWS                       │ Azure                                        │
├───────────────────────────┼──────────────────────────────────────────────┤
│ Cost Explorer             │ Cost Analysis                                │
│ - Filter by service       │ - Filter by resource type                    │
│ - Group by tag            │ - Group by tag                              │
│ - Forecast                │ - Forecast                                   │
│                           │                                              │
│ AWS Budgets               │ Azure Budgets                                │
│ - Cost budgets            │ - Cost budgets                              │
│ - Usage budgets           │ - Usage budgets (via metrics)               │
│ - Budget actions          │ - Action groups (more flexible)             │
│                           │                                              │
│ Savings Plans             │ Savings Plans                                │
│ - Compute SP              │ - Compute SP                                │
│ - EC2 Instance SP         │ - (No equivalent - use reservations)        │
│                           │                                              │
│ Reserved Instances        │ Azure Reservations                           │
│ - EC2                     │ - VMs                                        │
│ - RDS                     │ - SQL, PostgreSQL, MySQL                    │
│ - ElastiCache             │ - Redis Cache                               │
│                           │ - Storage, Cosmos DB, Databricks, etc.      │
│                           │                                              │
│ Spot Instances            │ Azure Spot VMs                               │
│ - Spot Fleet              │ - Spot Priority VMSS                        │
│ - Spot blocks             │ - (Not available)                           │
│                           │                                              │
│ Compute Optimizer         │ Azure Advisor                                │
│ Trusted Advisor           │ Azure Advisor                                │
│                           │                                              │
│ Cost & Usage Reports      │ Cost Management Exports                      │
│ - S3 delivery             │ - Storage Account delivery                  │
│ - Athena queries          │ - Power BI / Synapse Analytics              │
└───────────────────────────┴──────────────────────────────────────────────┘
```

---

*Next: [Cost Analysis & Visibility](01-cost-analysis.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
