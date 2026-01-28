# Cost Optimization

## Right-Sizing Resources

### VM Right-Sizing

```
RIGHT-SIZING PROCESS:
─────────────────────

STEP 1: Identify Candidates
┌────────────────────────────────────────────────────────────────────────────┐
│ Metrics to analyze (last 14-30 days):                                      │
│                                                                             │
│ • CPU: Average <5% = definitely oversized                                 │
│        Average <20% = likely oversized                                     │
│ • Memory: Average <30% = potentially oversized                            │
│ • Network: Check if hitting limits                                        │
│ • Disk IOPS: Check if hitting limits                                      │
│                                                                             │
│ Note: Check PEAK usage, not just average                                  │
└────────────────────────────────────────────────────────────────────────────┘

STEP 2: Azure Advisor Recommendations
┌────────────────────────────────────────────────────────────────────────────┐
│ az advisor recommendation list --category Cost                             │
│                                                                             │
│ Types of recommendations:                                                   │
│ • Shutdown idle VMs (no activity detected)                                │
│ • Right-size VMs (low utilization)                                        │
│ • Reserved VM instances (stable usage)                                    │
└────────────────────────────────────────────────────────────────────────────┘

STEP 3: Validate and Resize
┌────────────────────────────────────────────────────────────────────────────┐
│ 1. Confirm with application owner                                          │
│ 2. Test in non-prod first                                                  │
│ 3. Schedule maintenance window                                             │
│ 4. Resize VM (requires restart)                                            │
│ 5. Monitor for 1 week post-change                                          │
└────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Find underutilized VMs using Azure Monitor
az monitor metrics list \
  --resource "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Compute/virtualMachines/{vm}" \
  --metric "Percentage CPU" \
  --interval P7D \
  --aggregation Average \
  --query "value[0].timeseries[0].data[-1].average"

# Get Advisor right-sizing recommendations
az advisor recommendation list \
  --category Cost \
  --query "[?shortDescription.problem contains 'underutilized']"

# Resize VM
az vm resize \
  --resource-group myRG \
  --name myVM \
  --size Standard_D2s_v3
```

### Database Right-Sizing

```
DATABASE OPTIMIZATION:
──────────────────────

AZURE SQL DATABASE:
┌────────────────────────────────────────────────────────────────────────────┐
│ Tier Migration Path:                                                       │
│                                                                             │
│ Premium → Business Critical (vCore)  - Modern, similar perf               │
│ Standard → General Purpose (vCore)   - Better flexibility                 │
│ Basic → Serverless                   - Pay per use                        │
│                                                                             │
│ Serverless benefits:                                                       │
│ • Auto-pause after inactivity (1 hour default)                            │
│ • Min/max vCores configuration                                            │
│ • Pay only for compute used + storage                                     │
│ • Great for: Dev/test, intermittent workloads                             │
└────────────────────────────────────────────────────────────────────────────┘

COSMOS DB:
┌────────────────────────────────────────────────────────────────────────────┐
│ RU optimization:                                                           │
│                                                                             │
│ 1. Enable autoscale (instead of fixed RU/s)                               │
│    • Scales 10% to 100% of max RU/s                                       │
│    • Pay for peak usage, not provisioned                                  │
│                                                                             │
│ 2. Use serverless for sporadic workloads                                  │
│    • No minimum RU commitment                                              │
│    • Pay per request                                                       │
│                                                                             │
│ 3. Review partition strategy                                               │
│    • Hot partitions = wasted RU/s                                         │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Storage Optimization

### Tiered Storage

```
STORAGE TIER OPTIMIZATION:
──────────────────────────

ACCESS TIER SELECTION:
┌────────────────────────────────────────────────────────────────────────────┐
│ Access Pattern        │ Recommended Tier │ Cost Profile                   │
├───────────────────────┼──────────────────┼────────────────────────────────┤
│ Accessed daily        │ Hot              │ High storage, low access       │
│ Accessed monthly      │ Cool             │ Lower storage, higher access   │
│ Accessed quarterly    │ Cold             │ Lower storage, higher access   │
│ Accessed yearly       │ Archive          │ Lowest storage, highest access │
└───────────────────────┴──────────────────┴────────────────────────────────┘

LIFECYCLE MANAGEMENT:
┌────────────────────────────────────────────────────────────────────────────┐
│ Automatically move data between tiers based on age:                        │
│                                                                             │
│ Example policy:                                                            │
│ • 0-30 days: Hot tier                                                      │
│ • 30-90 days: Cool tier                                                    │
│ • 90-365 days: Cold tier                                                   │
│ • >365 days: Archive tier                                                  │
│ • >7 years: Delete (retention policy)                                     │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Set lifecycle management policy
az storage account management-policy create \
  --account-name mystorageaccount \
  --resource-group myRG \
  --policy @lifecycle-policy.json
```

```json
// lifecycle-policy.json
{
  "rules": [
    {
      "name": "tierToCool",
      "type": "Lifecycle",
      "definition": {
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["logs/"]
        },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToCold": { "daysAfterModificationGreaterThan": 90 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 365 },
            "delete": { "daysAfterModificationGreaterThan": 2555 }
          }
        }
      }
    }
  ]
}
```

---

## Orphaned Resources

### Finding Orphaned Resources

```bash
# Unattached Managed Disks
az disk list \
  --query "[?diskState=='Unattached'].{Name:name, RG:resourceGroup, SizeGB:diskSizeGb, Created:timeCreated}" \
  --output table

# Unattached Public IPs
az network public-ip list \
  --query "[?ipConfiguration==null].{Name:name, RG:resourceGroup}" \
  --output table

# Unattached NICs
az network nic list \
  --query "[?virtualMachine==null].{Name:name, RG:resourceGroup}" \
  --output table

# Empty Network Security Groups
az network nsg list \
  --query "[?length(networkInterfaces)==\`0\` && length(subnets)==\`0\`].{Name:name, RG:resourceGroup}" \
  --output table

# Stopped VMs (still incurring disk costs)
az vm list \
  --query "[?powerState!='VM running'].{Name:name, RG:resourceGroup, State:powerState}" \
  --output table

# Old snapshots (>90 days)
az snapshot list \
  --query "[?timeCreated < '$(date -d '90 days ago' --iso-8601)'].{Name:name, RG:resourceGroup, Created:timeCreated}" \
  --output table
```

### Automated Cleanup

```bicep
// Azure Policy - Deny unattached disks after 30 days
resource policy 'Microsoft.Authorization/policyDefinitions@2021-06-01' = {
  name: 'deny-old-unattached-disks'
  properties: {
    policyType: 'Custom'
    mode: 'All'
    description: 'Deny managed disks that have been unattached for more than 30 days'
    policyRule: {
      if: {
        allOf: [
          {
            field: 'type'
            equals: 'Microsoft.Compute/disks'
          }
          {
            field: 'Microsoft.Compute/disks/diskState'
            equals: 'Unattached'
          }
          {
            field: 'tags[\'LastAttached\']'
            less: '[addDays(utcNow(), -30)]'
          }
        ]
      }
      then: {
        effect: 'audit'  // Use 'deny' to prevent, 'audit' to report
      }
    }
  }
}
```

---

## Dev/Test Optimization

### Auto-Shutdown

```bash
# Configure auto-shutdown for VM
az vm auto-shutdown \
  --resource-group dev-rg \
  --name dev-vm \
  --time 1900 \
  --timezone "Eastern Standard Time" \
  --email "dev-team@company.com"

# Create schedule for VMSS (via Azure Automation)
# Or use Azure DevTest Labs for managed dev environments
```

```bicep
// Auto-shutdown schedule
resource shutdownSchedule 'Microsoft.DevTestLab/schedules@2018-09-15' = {
  name: 'shutdown-computevm-${vm.name}'
  location: location
  properties: {
    status: 'Enabled'
    taskType: 'ComputeVmShutdownTask'
    dailyRecurrence: {
      time: '1900'
    }
    timeZoneId: 'Eastern Standard Time'
    notificationSettings: {
      status: 'Enabled'
      emailRecipient: 'dev-team@company.com'
      notificationLocale: 'en'
      timeInMinutes: 30
    }
    targetResourceId: vm.id
  }
}
```

### Dev/Test Pricing

```
DEV/TEST SUBSCRIPTION BENEFITS:
───────────────────────────────

REQUIREMENTS:
• Visual Studio subscription (Professional, Enterprise, or MSDN Platforms)
• Mark subscription as Dev/Test in EA portal

BENEFITS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Service               │ Savings                                           │
├───────────────────────┼───────────────────────────────────────────────────┤
│ Windows VMs           │ Linux pricing (no Windows license cost)          │
│ Azure SQL Database    │ Discounted rates                                  │
│ App Service           │ Discounted rates                                  │
│ Logic Apps            │ Discounted rates                                  │
│ BizTalk               │ Significant discounts                             │
└───────────────────────┴───────────────────────────────────────────────────┘

RESTRICTIONS:
• No production workloads allowed
• No SLA provided
• For development and testing only
```

---

## Network Optimization

### Reduce Egress Costs

```
NETWORK COST OPTIMIZATION:
──────────────────────────

DATA EGRESS CHARGES:
┌────────────────────────────────────────────────────────────────────────────┐
│ First 5 GB/month     │ Free                                               │
│ 5 GB - 10 TB/month   │ ~$0.087/GB                                         │
│ 10 TB - 50 TB/month  │ ~$0.083/GB                                         │
│ 50 TB - 150 TB/month │ ~$0.07/GB                                          │
│ >150 TB/month        │ ~$0.05/GB                                          │
└────────────────────────────────────────────────────────────────────────────┘

OPTIMIZATION STRATEGIES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Strategy              │ Implementation                                     │
├───────────────────────┼────────────────────────────────────────────────────┤
│ Use CDN               │ Cache static content closer to users              │
│ Compress data         │ gzip/brotli compression reduces transfer          │
│ Regional deployment   │ Deploy near users to reduce cross-region traffic │
│ Private endpoints     │ Use Azure backbone, not internet                  │
│ VNet peering          │ Lower cost than VPN/ExpressRoute for Azure-Azure │
│ Azure Front Door      │ Global load balancing with edge caching          │
└───────────────────────┴────────────────────────────────────────────────────┘
```

---

## Optimization Checklist

```
IMMEDIATE (This Week):
☐ Review Azure Advisor cost recommendations
☐ Delete orphaned disks, IPs, NICs
☐ Stop or deallocate unused VMs
☐ Enable auto-shutdown for dev/test VMs
☐ Review and remove unused snapshots

SHORT-TERM (This Month):
☐ Right-size underutilized VMs
☐ Implement storage lifecycle policies
☐ Review database SKUs and tiers
☐ Enable autoscale where appropriate
☐ Purchase reservations for stable workloads

MEDIUM-TERM (This Quarter):
☐ Implement tagging for cost allocation
☐ Set up budgets and alerts
☐ Review hybrid benefit eligibility
☐ Consolidate Log Analytics workspaces
☐ Evaluate serverless options

ONGOING:
☐ Weekly Advisor review
☐ Monthly cost review meeting
☐ Quarterly commitment analysis
☐ Annual architecture review
```

---

*Next: [Case Studies](case-studies.md)* | *Back to [Reservations](03-reservations.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
