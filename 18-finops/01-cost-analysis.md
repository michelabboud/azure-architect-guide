# Cost Analysis & Visibility

## Accessing Cost Management

### Portal Navigation

```
COST MANAGEMENT LOCATIONS:
──────────────────────────

1. SUBSCRIPTION LEVEL:
   Azure Portal → Subscriptions → [Your Sub] → Cost Management

2. RESOURCE GROUP LEVEL:
   Azure Portal → Resource Groups → [Your RG] → Cost analysis

3. MANAGEMENT GROUP LEVEL:
   Azure Portal → Management Groups → [Your MG] → Cost Management

4. BILLING ACCOUNT LEVEL (EA/MCA):
   Azure Portal → Cost Management + Billing → Cost Management

ACCESS REQUIREMENTS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Scope              │ Required Role                                         │
├────────────────────┼───────────────────────────────────────────────────────┤
│ Subscription       │ Cost Management Reader, Contributor, or Owner        │
│ Resource Group     │ Cost Management Reader on RG                         │
│ Management Group   │ Cost Management Reader on MG                         │
│ Billing Account    │ Billing Reader, Billing Account Reader               │
│ EA Enrollment      │ Enterprise Administrator (read-only or full)        │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Cost Analysis Views

### Built-in Views

```
DEFAULT COST VIEWS:
───────────────────

ACCUMULATED COSTS:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Running total over time                                                   │
│                                                                             │
│  $50,000 ─────────────────────────────────────────────────────────── ●    │
│  $40,000 ──────────────────────────────────────────────────● · · · ·      │
│  $30,000 ────────────────────────────────────● · · · · · · ·              │
│  $20,000 ──────────────────────● · · · · · · ·                            │
│  $10,000 ────────● · · · · · · ·                                          │
│       $0 ┼───────┼───────┼───────┼───────┼───────┼───────┼                │
│          Jan 1   Jan 7   Jan 14  Jan 21  Jan 28  Feb 4   Feb 11           │
│                                                                             │
│  ● Actual cost    · · · Forecast                                          │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

DAILY COSTS:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Day-by-day spending                                                       │
│                                                                             │
│  $2,000 ─────────────────────────────────────────────────────────────     │
│  $1,500 ────────█─────────────────█───────────────────────────────────    │
│  $1,000 █───█───█───█───█───█───█───█───█───█───█───█                     │
│    $500 █───█───█───█───█───█───█───█───█───█───█───█───█───█─────────    │
│      $0 ┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼───┼─────────    │
│         1   3   5   7   9   11  13  15  17  19  21  23  25  27            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

COST BY SERVICE:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Virtual Machines        ████████████████████████████████  $15,000 (40%)  │
│  Azure SQL Database      ██████████████████                 $8,000 (21%)  │
│  Storage                 ████████████                       $5,500 (15%)  │
│  App Service             ████████                           $4,000 (11%)  │
│  Networking              ████                               $2,000 (5%)   │
│  Other                   ████                               $3,000 (8%)   │
│                                                                             │
│  Total: $37,500                                                            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Custom Views

```
CREATING CUSTOM COST VIEWS:
───────────────────────────

1. GROUP BY OPTIONS:
   • Resource (individual resource)
   • Resource group
   • Resource type (service)
   • Location (region)
   • Tag (CostCenter, Project, etc.)
   • Subscription
   • Meter (detailed usage)

2. FILTER OPTIONS:
   • Subscription
   • Resource group
   • Resource type
   • Location
   • Tag
   • Service name
   • Service tier

3. GRANULARITY:
   • Daily
   • Monthly
   • Accumulated

4. TIME RANGE:
   • Last 7 days
   • Last 30 days
   • This month
   • Last month
   • Last 3 months
   • Last 12 months
   • Custom range
```

### Save and Share Views

```bash
# Export current view configuration
# (Done via Portal - Save view button)

# API: Get cost details
az rest --method post \
  --uri "https://management.azure.com/subscriptions/{subscriptionId}/providers/Microsoft.CostManagement/query?api-version=2023-03-01" \
  --body '{
    "type": "ActualCost",
    "timeframe": "MonthToDate",
    "dataset": {
      "granularity": "Daily",
      "aggregation": {
        "totalCost": {
          "name": "Cost",
          "function": "Sum"
        }
      },
      "grouping": [
        {
          "type": "Dimension",
          "name": "ResourceGroup"
        }
      ]
    }
  }'
```

---

## Cost Exports

### Configure Scheduled Exports

```bash
# Create storage account for exports
az storage account create \
  --name costreportstorage \
  --resource-group finops-rg \
  --location eastus \
  --sku Standard_LRS

# Create container
az storage container create \
  --name cost-exports \
  --account-name costreportstorage

# Create cost export (monthly)
az costmanagement export create \
  --name "monthly-cost-export" \
  --scope "/subscriptions/{subscription-id}" \
  --type "ActualCost" \
  --storage-account-id "/subscriptions/{subscription-id}/resourceGroups/finops-rg/providers/Microsoft.Storage/storageAccounts/costreportstorage" \
  --storage-container "cost-exports" \
  --timeframe "MonthToDate" \
  --recurrence "Monthly" \
  --recurrence-period-from "2024-01-01T00:00:00Z" \
  --recurrence-period-to "2024-12-31T00:00:00Z" \
  --schedule-status "Active"

# List exports
az costmanagement export list \
  --scope "/subscriptions/{subscription-id}" \
  --output table

# Run export manually
az costmanagement export execute \
  --name "monthly-cost-export" \
  --scope "/subscriptions/{subscription-id}"
```

### Export Data Format

```
EXPORT FILE STRUCTURE:
──────────────────────

storage-account/
└── cost-exports/
    └── monthly-cost-export/
        └── 20240101-20240131/
            └── monthly-cost-export_00000000-0000-0000-0000-000000000000.csv

CSV COLUMNS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Column              │ Description                                          │
├─────────────────────┼──────────────────────────────────────────────────────┤
│ InvoiceSectionName  │ Invoice section (MCA) or department (EA)            │
│ AccountName         │ Billing account name                                 │
│ SubscriptionId      │ Azure subscription GUID                              │
│ SubscriptionName    │ Subscription display name                            │
│ ResourceGroup       │ Resource group name                                  │
│ ResourceLocation    │ Azure region                                         │
│ Date                │ Usage date                                           │
│ MeterCategory       │ Service category (e.g., Virtual Machines)           │
│ MeterSubCategory    │ Service subcategory (e.g., Dv3 Series)              │
│ MeterId             │ Unique meter identifier                              │
│ MeterName           │ Meter description                                    │
│ MeterRegion         │ Meter pricing region                                 │
│ UnitOfMeasure       │ Billing unit (Hours, GB, etc.)                       │
│ Quantity            │ Consumed quantity                                    │
│ EffectivePrice      │ Actual price per unit                               │
│ CostInBillingCurrency│ Cost in billing currency                           │
│ ResourceId          │ Full Azure resource ID                               │
│ Tags                │ Resource tags as JSON                                │
│ CostCenter          │ Tag: CostCenter value                               │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Power BI Integration

### Cost Management Connector

```
POWER BI SETUP:
───────────────

1. Install Azure Cost Management connector
   Get Data → Azure → Azure Cost Management

2. Select scope:
   • Enterprise Agreement enrollment number
   • Billing account ID (MCA)
   • Subscription ID

3. Choose data type:
   • Usage Details - Actual consumption
   • Balance Summary - Commitment balance
   • Budgets - Budget vs actual
   • Price Sheets - Your negotiated prices
   • Reservations - Reservation details

4. Set date range:
   • Number of months to load (1-36)
```

### Sample Power BI Queries

```m
// Power Query M - Cost by Department
let
    Source = AzureCostManagement.Tables("enrollment-number", "Usage Details"),
    FilteredRows = Table.SelectRows(Source, each [Date] >= #date(2024, 1, 1)),
    GroupedRows = Table.Group(FilteredRows, {"DepartmentName"},
        {{"TotalCost", each List.Sum([CostInBillingCurrency]), type number}}),
    SortedRows = Table.Sort(GroupedRows,{{"TotalCost", Order.Descending}})
in
    SortedRows

// Cost Trend by Month
let
    Source = AzureCostManagement.Tables("enrollment-number", "Usage Details"),
    AddedMonth = Table.AddColumn(Source, "Month",
        each Date.StartOfMonth([Date]), type date),
    GroupedRows = Table.Group(AddedMonth, {"Month"},
        {{"TotalCost", each List.Sum([CostInBillingCurrency]), type number}})
in
    GroupedRows
```

### Cost Dashboard Template

```
EXECUTIVE COST DASHBOARD COMPONENTS:
────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                        AZURE COST DASHBOARD - January 2024                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐             │
│  │   MTD SPEND     │  │   FORECAST      │  │   BUDGET        │             │
│  │                 │  │                 │  │   REMAINING     │             │
│  │   $47,500       │  │   $52,000       │  │   $2,500        │             │
│  │   ▲ 5% vs LM    │  │   104% of $50K  │  │   (95% used)    │             │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘             │
│                                                                              │
│  SPEND BY DEPARTMENT           │  MONTHLY TREND                            │
│  ─────────────────────────────│────────────────────────────────────────── │
│                                │                                            │
│  Engineering    $25,000 ████  │  $60K ─────────────────────────────────── │
│  Sales          $12,000 ██    │  $50K ──────────────────────● ─ ─ ─ ●     │
│  Marketing       $6,000 █     │  $40K ────────────● ───● ────              │
│  Finance         $4,500 █     │  $30K ────● ──● ───                        │
│                                │  $20K ● ───                                │
│                                │       Oct  Nov  Dec  Jan  Feb  Mar        │
│                                │                                            │
│  TOP 10 RESOURCES              │  OPTIMIZATION OPPORTUNITIES              │
│  ─────────────────────────────│────────────────────────────────────────── │
│                                │                                            │
│  1. sql-prod-db    $8,500     │  ⚠ 5 underutilized VMs       Save $1,200 │
│  2. vm-api-prod    $4,200     │  ⚠ 12 unattached disks       Save $300   │
│  3. aks-cluster    $3,800     │  ✓ Reservation opportunity   Save $5,000 │
│  4. storage-logs   $2,100     │                                            │
│  5. vm-web-01      $1,900     │  Total Potential Savings: $6,500/month   │
│                                │                                            │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Anomaly Detection

### Configure Anomaly Alerts

```
COST ANOMALY DETECTION:
───────────────────────

Azure Cost Management includes built-in anomaly detection that identifies
unusual spending patterns automatically.

HOW IT WORKS:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. Machine learning analyzes your historical spending patterns            │
│  2. Establishes baseline for "normal" spending                             │
│  3. Alerts when spending deviates significantly from baseline              │
│  4. Considers seasonality (weekends, month-end, etc.)                      │
│                                                                             │
│  DETECTION SCENARIOS:                                                       │
│  • Unexpected spike (new resource, misconfiguration)                       │
│  • Gradual increase (data growth, scaling)                                 │
│  • Category change (new service adoption)                                  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

ENABLE ANOMALY ALERTS:
1. Cost Management → Cost alerts → Add
2. Select "Anomaly" alert type
3. Configure email recipients
4. Set minimum threshold (e.g., alert only if anomaly > $100)
```

### Manual Anomaly Investigation

```kusto
// Azure Resource Graph - Find recently created resources
Resources
| where properties.createdTime > ago(7d)
| project name, type, resourceGroup, subscriptionId,
    createdTime = properties.createdTime
| order by createdTime desc

// Find large resources
Resources
| where type =~ 'microsoft.compute/virtualmachines'
| extend vmSize = properties.hardwareProfile.vmSize
| where vmSize contains 'E' or vmSize contains 'M' // Large VM families
| project name, resourceGroup, vmSize
```

```bash
# CLI - Find high-cost resources in last week
az consumption usage list \
  --start-date $(date -d '7 days ago' +%Y-%m-%d) \
  --end-date $(date +%Y-%m-%d) \
  --query "sort_by(@, &pretaxCost)[-10:].{Resource:instanceName, Cost:pretaxCost}" \
  --output table
```

---

## Multi-Cloud Cost Visibility

### AWS Cost Integration

```
VIEWING AWS COSTS IN AZURE:
───────────────────────────

Azure Cost Management can import AWS costs for unified visibility.

SETUP STEPS:
1. Create AWS Cost and Usage Report (CUR) in AWS
   • Enable resource IDs
   • Choose hourly granularity
   • Enable Athena integration

2. Create cross-account role in AWS
   • Grant S3 read access to CUR bucket
   • Grant Cost Explorer read access

3. Configure connector in Azure
   • Cost Management → AWS → Create connector
   • Provide AWS role ARN
   • Specify S3 bucket details

4. View unified costs
   • Cost Analysis shows both Azure and AWS
   • Filter by cloud provider
   • Group by tag across clouds

LIMITATIONS:
• 24-48 hour delay for AWS data
• Tags must be enabled in AWS CUR
• Some AWS services have limited detail
```

### GCP Cost Integration

```
GCP COST VISIBILITY:
────────────────────

Similar process for GCP:
1. Enable billing export to BigQuery in GCP
2. Create service account with BigQuery read access
3. Configure connector in Azure Cost Management
4. View unified multi-cloud costs

BENEFIT: Single pane of glass for hybrid/multi-cloud costs
```

---

## Cost Allocation

### Tag-Based Allocation

```
IMPLEMENTING COST ALLOCATION:
─────────────────────────────

STEP 1: Define Allocation Strategy
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  DIRECT COSTS: Resources with clear ownership                              │
│  • Tag with CostCenter, Project, Owner                                     │
│  • 100% allocated to the tagged entity                                     │
│                                                                             │
│  SHARED COSTS: Resources used by multiple teams                            │
│  • Networking (hub VNet, firewall)                                         │
│  • Shared services (AD DS, monitoring)                                     │
│  • Management plane (Azure AD P2, Defender)                                │
│                                                                             │
│  Options for shared costs:                                                  │
│  A. Split evenly across teams                                              │
│  B. Split by usage (e.g., data transferred)                                │
│  C. Split by headcount                                                      │
│  D. Allocate to central IT/platform team                                   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

STEP 2: Enforce Tagging
• Use Azure Policy to require tags on resources
• Deny deployment if mandatory tags missing
• Inherit tags from resource group/subscription
```

### Cost Allocation Rules

```
COST ALLOCATION RULES (Preview):
────────────────────────────────

Azure Cost Management supports allocation rules to distribute
shared costs automatically.

RULE EXAMPLE: Distribute Log Analytics costs
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Source: Log Analytics workspace (shared)                                  │
│  Total Cost: $5,000/month                                                  │
│                                                                             │
│  Allocation Rule: Split by tag "BusinessUnit"                              │
│                                                                             │
│  Target Breakdown:                                                          │
│  • BusinessUnit: Engineering  → 60% → $3,000                               │
│  • BusinessUnit: Sales        → 25% → $1,250                               │
│  • BusinessUnit: Marketing    → 15% → $750                                 │
│                                                                             │
│  Result: Costs appear under each business unit's view                      │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

CREATE RULE:
1. Cost Management → Settings → Cost allocation (preview)
2. Create allocation rule
3. Select source (shared resource/subscription)
4. Define targets (subscriptions, resource groups, or tags)
5. Set allocation percentages
```

---

## Best Practices

### Cost Visibility Checklist

```
COST VISIBILITY BEST PRACTICES:
───────────────────────────────

TAGGING:
☐ Define mandatory tags (CostCenter, Owner, Environment)
☐ Enforce via Azure Policy
☐ Audit tag compliance weekly
☐ Document tag values and meanings

REPORTING:
☐ Set up automated cost exports
☐ Create executive dashboard in Power BI
☐ Configure weekly cost email reports
☐ Enable anomaly detection alerts

ACCESS:
☐ Grant Cost Management Reader to finance team
☐ Grant cost access to resource owners
☐ Set up billing alerts for budget owners
☐ Create management group structure for rollup

PROCESS:
☐ Weekly cost review meeting
☐ Monthly cost report to leadership
☐ Quarterly optimization review
☐ Annual budget planning process
```

---

*Next: [Budgets & Alerts](02-budgets-alerts.md)* | *Back to [Quick Reference](quick-reference.md)*
