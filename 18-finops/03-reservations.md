# Reservations & Savings Plans

## Azure Reservations

### What Can Be Reserved

```
RESERVABLE AZURE SERVICES:
──────────────────────────

COMPUTE:
┌────────────────────────────────────────────────────────────────────────────┐
│ Service              │ Discount    │ Term Options │ Scope Options          │
├──────────────────────┼─────────────┼──────────────┼────────────────────────┤
│ Virtual Machines     │ Up to 72%   │ 1yr, 3yr     │ Shared, Sub, RG        │
│ Azure VMware Solution│ Up to 70%   │ 1yr, 3yr     │ Shared, Sub            │
│ Dedicated Host       │ Up to 65%   │ 1yr, 3yr     │ Shared, Sub            │
│ App Service Premium  │ Up to 55%   │ 1yr, 3yr     │ Shared, Sub            │
└──────────────────────┴─────────────┴──────────────┴────────────────────────┘

DATABASE:
┌────────────────────────────────────────────────────────────────────────────┐
│ Azure SQL Database   │ Up to 80%   │ 1yr, 3yr     │ Shared, Sub            │
│ SQL Managed Instance │ Up to 80%   │ 1yr, 3yr     │ Shared, Sub            │
│ Cosmos DB            │ Up to 65%   │ 1yr, 3yr     │ Shared, Sub, RG        │
│ PostgreSQL           │ Up to 64%   │ 1yr, 3yr     │ Shared, Sub            │
│ MySQL                │ Up to 64%   │ 1yr, 3yr     │ Shared, Sub            │
│ MariaDB              │ Up to 64%   │ 1yr, 3yr     │ Shared, Sub            │
└──────────────────────┴─────────────┴──────────────┴────────────────────────┘

STORAGE & DATA:
┌────────────────────────────────────────────────────────────────────────────┐
│ Blob Storage         │ Up to 38%   │ 1yr, 3yr     │ Account-specific       │
│ Azure Files          │ Up to 36%   │ 1yr, 3yr     │ Account-specific       │
│ Data Lake Storage    │ Up to 38%   │ 1yr, 3yr     │ Account-specific       │
│ Azure Data Explorer  │ Up to 65%   │ 1yr, 3yr     │ Shared, Sub            │
│ Synapse Analytics    │ Up to 65%   │ 1yr, 3yr     │ Shared, Sub            │
│ Databricks           │ Up to 38%   │ 1yr, 3yr     │ Shared, Sub            │
└──────────────────────┴─────────────┴──────────────┴────────────────────────┘

OTHER:
┌────────────────────────────────────────────────────────────────────────────┐
│ Redis Cache          │ Up to 55%   │ 1yr, 3yr     │ Shared, Sub            │
│ Azure Backup         │ Up to 35%   │ 1yr, 3yr     │ Shared, Sub            │
│ Log Analytics        │ Up to 30%   │ 1yr, 3yr     │ Cluster-specific       │
│ Azure Monitor        │ Up to 30%   │ 1yr, 3yr     │ Shared                 │
└──────────────────────┴─────────────┴──────────────┴────────────────────────┘
```

### Reservation Scope

```
RESERVATION SCOPE OPTIONS:
──────────────────────────

SHARED (Recommended for flexibility):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  • Applies to ALL subscriptions in billing account                        │
│  • Automatically finds matching resources                                  │
│  • Best utilization - unused capacity helps other subs                    │
│  • Recommended for most scenarios                                          │
│                                                                             │
│  Example: 10 D4s_v3 reservations, shared scope                            │
│  - Sub A has 6 D4s_v3 VMs → uses 6 reservations                           │
│  - Sub B has 4 D4s_v3 VMs → uses 4 reservations                           │
│  - All 10 utilized, even though split across subs                         │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

SINGLE SUBSCRIPTION:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  • Applies only to specified subscription                                  │
│  • Good for chargeback/showback scenarios                                  │
│  • May result in lower utilization                                         │
│                                                                             │
│  Example: 10 D4s_v3 reservations, Sub A only                              │
│  - Sub A has 6 D4s_v3 VMs → uses 6 reservations                           │
│  - 4 reservations unused (even if Sub B has matching VMs)                 │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

RESOURCE GROUP (Limited services):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  • Applies to specific resource group                                      │
│  • Most restrictive - use only when required                              │
│  • Useful for dedicated project budgets                                    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Purchasing Reservations

```bash
# View reservation recommendations
az advisor recommendation list \
  --category Cost \
  --query "[?contains(shortDescription.problem, 'reserved')]"

# Purchase reservation via CLI (limited)
# Most purchases done via Portal or REST API

# Check existing reservations
az reservations reservation-order list --output table

# View reservation details
az reservations reservation list \
  --reservation-order-id "order-id" \
  --output table
```

### Instance Size Flexibility

```
INSTANCE SIZE FLEXIBILITY:
──────────────────────────

VM reservations apply to the INSTANCE SIZE FAMILY, not exact size.

EXAMPLE - Dv3 Family:
┌────────────────────────────────────────────────────────────────────────────┐
│ VM Size    │ Ratio │ Reservation Coverage                                  │
├────────────┼───────┼───────────────────────────────────────────────────────┤
│ D2_v3      │ 1     │ 1 reservation = 1x D2_v3                             │
│ D4_v3      │ 2     │ 1 reservation = 0.5x D4_v3                           │
│ D8_v3      │ 4     │ 1 reservation = 0.25x D8_v3                          │
│ D16_v3     │ 8     │ 1 reservation = 0.125x D16_v3                        │
│ D32_v3     │ 16    │ 1 reservation = 0.0625x D32_v3                       │
└────────────┴───────┴───────────────────────────────────────────────────────┘

If you have 8 D2_v3 reservations:
• Can cover: 8x D2_v3, OR 4x D4_v3, OR 2x D8_v3, OR 1x D16_v3
• Mix and match: 4x D2_v3 + 2x D4_v3 (4 + 4 = 8 ratio units)

BENEFIT: Flexibility to resize VMs without losing reservation value
```

---

## Azure Savings Plans

### Savings Plans Overview

```
SAVINGS PLANS vs RESERVATIONS:
──────────────────────────────

┌─────────────────────┬──────────────────────┬──────────────────────────────┐
│ Feature             │ Reservations         │ Savings Plans                │
├─────────────────────┼──────────────────────┼──────────────────────────────┤
│ Commitment          │ Specific SKU/Region  │ Hourly spend amount          │
│ Flexibility         │ Instance family only │ Any VM size/region/OS        │
│ Max Discount        │ Up to 72%            │ Up to 65%                    │
│ Covered Services    │ VMs, DBs, Storage... │ Compute only (VMs, AKS...)   │
│ Best For            │ Stable workloads     │ Variable/dynamic workloads   │
│ Management          │ Per-SKU management   │ Single commitment            │
└─────────────────────┴──────────────────────┴──────────────────────────────┘

WHEN TO USE SAVINGS PLANS:
• Frequently change VM sizes
• Multi-region deployments
• Dynamic autoscaling workloads
• Want simpler management
• Modernizing VM fleet

WHEN TO USE RESERVATIONS:
• Stable, predictable workloads
• Need maximum discount
• Reserved capacity guarantee
• Non-compute services (SQL, Cosmos, Storage)
```

### Savings Plan Types

```
COMPUTE SAVINGS PLAN:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Commitment: Hourly spend (e.g., $10/hour)                                │
│  Covers: ALL compute across ALL regions                                   │
│                                                                             │
│  Included services:                                                        │
│  • Virtual Machines (all sizes)                                           │
│  • Azure Kubernetes Service                                                │
│  • Azure Container Instances                                               │
│  • Azure Functions Premium                                                 │
│  • Azure App Service Premium                                               │
│  • Azure Dedicated Host                                                    │
│                                                                             │
│  Discount applied automatically to highest-value usage first              │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

EXAMPLE:
$10/hour commitment = $7,300/month commitment
If you have $12,000/month in compute:
• $7,300 covered at savings plan rate (up to 65% discount)
• $4,700 charged at pay-as-you-go rate
```

### Combining Discounts

```
DISCOUNT STACKING:
──────────────────

Reservations and Savings Plans can be COMBINED with:

AZURE HYBRID BENEFIT (AHUB):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Bring your existing Windows Server or SQL Server licenses                │
│                                                                             │
│  Savings:                                                                   │
│  • Windows Server: ~40% savings                                            │
│  • SQL Server: ~55% savings                                                │
│                                                                             │
│  Combined with reservation:                                                 │
│  • VM reservation (72%) + AHUB (40%) = Up to 80% total savings            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

DEV/TEST PRICING:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  Reduced rates for non-production workloads                               │
│  Requires Visual Studio subscription                                       │
│                                                                             │
│  Note: Cannot combine with reservations                                    │
│  Best for: True dev/test where reservations don't apply                   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

PRIORITY ORDER:
1. Reservations (applied first - most specific)
2. Savings Plans (applied to remaining compute)
3. Azure Hybrid Benefit (applied on top)
4. Pay-as-you-go (remaining usage)
```

---

## Spot VMs

### Spot VM Overview

```
AZURE SPOT VMS:
───────────────

CONCEPT:
• Use Azure's spare capacity at up to 90% discount
• Can be evicted when Azure needs capacity back
• Get 30-second warning before eviction
• Best for interruptible workloads

EVICTION TYPES:
┌────────────────────────────────────────────────────────────────────────────┐
│ Policy              │ Behavior                                            │
├─────────────────────┼─────────────────────────────────────────────────────┤
│ Capacity Only       │ Evicted only when Azure needs capacity             │
│ Price or Capacity   │ Evicted if price exceeds your max or capacity need │
└─────────────────────┴─────────────────────────────────────────────────────┘

GOOD USE CASES:
✅ Batch processing jobs
✅ CI/CD build agents
✅ Dev/test environments
✅ Big data processing (Spark, Hadoop)
✅ Rendering/transcoding
✅ Stateless web workers (with load balancer)

BAD USE CASES:
❌ Production databases
❌ Stateful applications
❌ Long-running single jobs
❌ Applications that can't handle interruption
```

### Spot VM Configuration

```bash
# Create Spot VM
az vm create \
  --name spot-vm \
  --resource-group spot-rg \
  --image Ubuntu2204 \
  --size Standard_D4s_v3 \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price 0.05

# Create Spot VMSS
az vmss create \
  --name spot-vmss \
  --resource-group spot-rg \
  --image Ubuntu2204 \
  --vm-sku Standard_D4s_v3 \
  --instance-count 10 \
  --priority Spot \
  --eviction-policy Delete \
  --max-price -1  # Use current spot price

# Check spot prices
az vm list-skus \
  --location eastus \
  --size Standard_D4s_v3 \
  --query "[].{Name:name, SpotPrice:capabilities[?name=='MaxSpotPricePercentage'].value}" \
  --output table
```

```bicep
// Spot VM in Bicep
resource spotVM 'Microsoft.Compute/virtualMachines@2023-03-01' = {
  name: 'spot-vm'
  location: location
  properties: {
    hardwareProfile: {
      vmSize: 'Standard_D4s_v3'
    }
    priority: 'Spot'
    evictionPolicy: 'Deallocate'
    billingProfile: {
      maxPrice: -1  // Current spot price
    }
    // ... other properties
  }
}
```

---

## Managing Commitments

### Utilization Monitoring

```bash
# Check reservation utilization
az consumption reservation summary list \
  --reservation-order-id "order-id" \
  --reservation-id "reservation-id" \
  --grain daily \
  --start-date "2024-01-01" \
  --end-date "2024-01-31"

# Get underutilized reservations
az advisor recommendation list \
  --category Cost \
  --query "[?contains(shortDescription.problem, 'utilization')]"
```

```
UTILIZATION TARGETS:
────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│ Utilization %  │ Status      │ Action                                     │
├────────────────┼─────────────┼────────────────────────────────────────────┤
│ 95-100%        │ Optimal     │ Consider additional reservations           │
│ 80-95%         │ Good        │ Monitor for improvement opportunities     │
│ 60-80%         │ Needs Work  │ Review scope, exchange/return options     │
│ <60%           │ Critical    │ Exchange or return reservation            │
└────────────────┴─────────────┴────────────────────────────────────────────┘

LOW UTILIZATION CAUSES:
• Wrong scope (too restrictive)
• Workload decommissioned
• VM sizes changed outside family
• Seasonal variation
```

### Exchange and Refund

```
RESERVATION EXCHANGE & REFUND POLICY:
─────────────────────────────────────

EXCHANGES (Allowed):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  • Exchange for same service type (VM for VM, SQL for SQL)                │
│  • New reservation must be EQUAL OR GREATER value                         │
│  • Unlimited exchanges allowed                                             │
│  • Self-service in Azure Portal                                            │
│                                                                             │
│  Example: Exchange D4_v3 reservation for D8_v3 reservation                │
│           (pay difference or use credit for larger)                        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

REFUNDS (Limited):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  • $50,000 rolling 12-month refund limit                                  │
│  • 12% early termination fee                                               │
│  • Prorated refund for remaining term                                      │
│  • Refund = (Remaining months × monthly rate) - 12% fee                   │
│                                                                             │
│  Example: 3-year reservation, cancel after 1 year                         │
│           Paid: $3,600 upfront                                             │
│           Remaining: $2,400 (24 months)                                    │
│           Fee: $288 (12%)                                                  │
│           Refund: $2,112                                                   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

### Commitment Strategy

```
COMMITMENT PURCHASE STRATEGY:
─────────────────────────────

1. ANALYZE BEFORE BUYING:
   • Review 30-60 days of usage data
   • Use Advisor recommendations as starting point
   • Calculate break-even point

2. START CONSERVATIVE:
   • Buy 60-70% of recommended quantity first
   • Monitor utilization for 2-4 weeks
   • Add more if utilization stays >95%

3. USE APPROPRIATE SCOPE:
   • Shared scope for maximum flexibility
   • Single subscription only for chargeback requirements

4. COMBINE STRATEGICALLY:
   • Reservations for stable, predictable workloads
   • Savings Plans for variable compute
   • Spot VMs for interruptible workloads
   • AHUB for Windows/SQL workloads

5. MONITOR CONTINUOUSLY:
   • Weekly utilization review
   • Monthly commitment analysis
   • Quarterly strategy adjustment
```

---

*Next: [Cost Optimization](04-optimization.md)* | *Back to [Budgets & Alerts](02-budgets-alerts.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
