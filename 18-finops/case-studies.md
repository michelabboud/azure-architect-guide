# FinOps Case Studies

## Case Study 1: E-commerce Cost Optimization

### Scenario

**Company**: Mid-size e-commerce retailer
**Challenge**: Azure bill grew 40% YoY without proportional revenue growth
**Monthly Spend**: $180,000 → Target: $120,000 (33% reduction)

### Analysis

```
COST BREAKDOWN (Before):
────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│ Service                    │ Monthly Cost │ % of Total │ Issues Found       │
├────────────────────────────┼──────────────┼────────────┼────────────────────┤
│ Virtual Machines           │ $72,000      │ 40%        │ Oversized, 24/7    │
│ Azure SQL Database         │ $36,000      │ 20%        │ Premium tier       │
│ Storage                    │ $27,000      │ 15%        │ All in Hot tier    │
│ AKS                        │ $18,000      │ 10%        │ No autoscaling     │
│ Log Analytics              │ $14,400      │ 8%         │ 90-day retention   │
│ Other                      │ $12,600      │ 7%         │ Various            │
├────────────────────────────┼──────────────┼────────────┼────────────────────┤
│ TOTAL                      │ $180,000     │ 100%       │                    │
└────────────────────────────┴──────────────┴────────────┴────────────────────┘
```

### Optimization Actions

```
PHASE 1: Quick Wins (Week 1-2)
──────────────────────────────

ACTION 1: Right-size VMs
┌────────────────────────────────────────────────────────────────────────────┐
│ Before: 40 x D8s_v3 (8 vCPU, 32GB) - many at <10% CPU                    │
│ After:  30 x D4s_v3 (4 vCPU, 16GB) + 5 x D8s_v3                          │
│                                                                             │
│ Savings: $28,000/month                                                     │
└────────────────────────────────────────────────────────────────────────────┘

ACTION 2: Implement auto-shutdown for dev/test
┌────────────────────────────────────────────────────────────────────────────┐
│ Before: Dev/Test VMs running 24/7                                         │
│ After:  Auto-shutdown 7PM, auto-start 7AM (weekdays only)                │
│ Runtime: 12hrs/day x 5 days = 60hrs vs 168hrs (64% reduction)            │
│                                                                             │
│ Savings: $8,000/month                                                      │
└────────────────────────────────────────────────────────────────────────────┘

ACTION 3: Delete orphaned resources
┌────────────────────────────────────────────────────────────────────────────┐
│ Found: 50 unattached disks (2TB), 30 unused public IPs                   │
│                                                                             │
│ Savings: $500/month                                                        │
└────────────────────────────────────────────────────────────────────────────┘

PHASE 1 TOTAL SAVINGS: $36,500/month
```

```
PHASE 2: Commitment Purchases (Week 3-4)
────────────────────────────────────────

ACTION 4: Purchase VM reservations
┌────────────────────────────────────────────────────────────────────────────┐
│ Analysis: 35 VMs run 24/7 for production                                  │
│ Purchase: 3-year reservations for production VMs                          │
│ Discount: 62% off pay-as-you-go                                           │
│                                                                             │
│ Before:  $44,000/month (production VMs)                                   │
│ After:   $16,700/month (with reservations)                                │
│                                                                             │
│ Savings: $27,300/month                                                     │
└────────────────────────────────────────────────────────────────────────────┘

ACTION 5: SQL Database reserved capacity
┌────────────────────────────────────────────────────────────────────────────┐
│ Purchase: 3-year reserved capacity for production SQL                     │
│ Discount: 35% off                                                          │
│                                                                             │
│ Savings: $12,600/month                                                     │
└────────────────────────────────────────────────────────────────────────────┘

PHASE 2 TOTAL SAVINGS: $39,900/month
```

```
PHASE 3: Architecture Optimization (Week 5-8)
─────────────────────────────────────────────

ACTION 6: Storage tiering
┌────────────────────────────────────────────────────────────────────────────┐
│ Before: 50TB in Hot tier                                                   │
│ Analysis: 70% of data not accessed in 30+ days                            │
│ After:  15TB Hot, 25TB Cool, 10TB Archive                                 │
│                                                                             │
│ Before: $27,000/month                                                      │
│ After:  $12,000/month                                                      │
│                                                                             │
│ Savings: $15,000/month                                                     │
└────────────────────────────────────────────────────────────────────────────┘

ACTION 7: Enable AKS autoscaling
┌────────────────────────────────────────────────────────────────────────────┐
│ Before: Fixed 20 nodes                                                     │
│ After:  Min 5, Max 25 nodes with cluster autoscaler                       │
│ Average utilization: 8 nodes during off-peak                              │
│                                                                             │
│ Savings: $5,000/month                                                      │
└────────────────────────────────────────────────────────────────────────────┘

ACTION 8: Optimize Log Analytics
┌────────────────────────────────────────────────────────────────────────────┐
│ Before: 90-day retention, all tables                                       │
│ After:  30-day interactive, archive older logs to storage                 │
│ Also:   Moved high-volume tables to Basic logs tier                       │
│                                                                             │
│ Savings: $7,000/month                                                      │
└────────────────────────────────────────────────────────────────────────────┘

PHASE 3 TOTAL SAVINGS: $27,000/month
```

### Results

```
FINAL RESULTS:
──────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  BEFORE: $180,000/month                                                     │
│  AFTER:  $76,600/month                                                      │
│                                                                              │
│  TOTAL SAVINGS: $103,400/month (57% reduction)                             │
│  ANNUAL SAVINGS: $1,240,800                                                 │
│                                                                              │
│  TIMELINE: 8 weeks to full implementation                                   │
│  UPFRONT INVESTMENT: $450,000 (3-year reservations)                        │
│  ROI ON RESERVATIONS: 4.3 months                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

BREAKDOWN BY ACTION:
┌─────────────────────────────────────────────────────────────────────────────┐
│ Action                    │ Monthly Savings │ % of Total                   │
├───────────────────────────┼─────────────────┼──────────────────────────────┤
│ VM Right-sizing           │ $28,000         │ 27%                          │
│ VM Reservations           │ $27,300         │ 26%                          │
│ Storage Tiering           │ $15,000         │ 15%                          │
│ SQL Reservations          │ $12,600         │ 12%                          │
│ Dev/Test Auto-shutdown    │ $8,000          │ 8%                           │
│ Log Analytics Optimization│ $7,000          │ 7%                           │
│ AKS Autoscaling           │ $5,000          │ 5%                           │
│ Orphaned Resources        │ $500            │ <1%                          │
└───────────────────────────┴─────────────────┴──────────────────────────────┘
```

---

## Case Study 2: Enterprise FinOps Program

### Scenario

**Company**: Fortune 500 financial services firm
**Challenge**: Cloud spend growing 60% YoY, no visibility or accountability
**Annual Cloud Spend**: $15M → Need governance and optimization

### FinOps Implementation

```
FINOPS MATURITY MODEL JOURNEY:
──────────────────────────────

STAGE 1: CRAWL (Months 1-3)
┌────────────────────────────────────────────────────────────────────────────┐
│ Goals:                                                                      │
│ • Establish cost visibility                                                │
│ • Implement basic tagging                                                  │
│ • Set up budgets and alerts                                                │
│                                                                             │
│ Actions:                                                                    │
│ 1. Created central FinOps team (3 people)                                 │
│ 2. Implemented mandatory tagging policy:                                   │
│    - CostCenter, Owner, Environment, Application                          │
│ 3. Created Power BI dashboard for executives                               │
│ 4. Set up budgets at subscription level                                    │
│ 5. Enabled Azure Advisor recommendations review                           │
│                                                                             │
│ Investment: $300K (tools, team, process)                                   │
│ Immediate Savings: $500K (low-hanging fruit)                              │
└────────────────────────────────────────────────────────────────────────────┘

STAGE 2: WALK (Months 4-8)
┌────────────────────────────────────────────────────────────────────────────┐
│ Goals:                                                                      │
│ • Enable cost allocation and showback                                      │
│ • Implement optimization processes                                         │
│ • Create accountability framework                                          │
│                                                                             │
│ Actions:                                                                    │
│ 1. Implemented chargeback to business units                               │
│ 2. Created monthly cost review process                                     │
│ 3. Trained 50 cloud engineers on cost optimization                        │
│ 4. Purchased $3M in reservations                                           │
│ 5. Automated orphaned resource cleanup                                     │
│ 6. Implemented right-sizing recommendations                               │
│                                                                             │
│ Investment: $200K (training, automation)                                   │
│ Savings: $2.5M/year (reservations + optimization)                         │
└────────────────────────────────────────────────────────────────────────────┘

STAGE 3: RUN (Months 9-12+)
┌────────────────────────────────────────────────────────────────────────────┐
│ Goals:                                                                      │
│ • Continuous optimization                                                   │
│ • Predictive forecasting                                                   │
│ • Cloud-native cost efficiency                                             │
│                                                                             │
│ Actions:                                                                    │
│ 1. Integrated cost into CI/CD (terraform plan cost estimate)              │
│ 2. Implemented FinOps KPIs for engineering teams                          │
│ 3. Created architecture review board for cost                              │
│ 4. Automated commitment management                                         │
│ 5. Multi-cloud cost visibility (Azure + AWS)                              │
│                                                                             │
│ Ongoing Investment: $150K/year                                             │
│ Ongoing Savings: $3M/year                                                  │
└────────────────────────────────────────────────────────────────────────────┘
```

### Governance Structure

```
FINOPS ORGANIZATION:
────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                           CFO / CTO Sponsorship                            │
│                                    │                                        │
│                    ┌───────────────┴───────────────┐                       │
│                    │                               │                        │
│            FinOps Team (Central)          Cloud CoE (Engineering)          │
│            ───────────────────            ───────────────────────          │
│            • Cost analysis                • Architecture standards         │
│            • Budget management            • Tooling & automation           │
│            • Showback/chargeback          • Training                       │
│            • Vendor management            • Best practices                 │
│            • Reporting                    • Policy enforcement             │
│                    │                               │                        │
│                    └───────────────┬───────────────┘                       │
│                                    │                                        │
│                    Business Unit Cloud Champions                           │
│                    ─────────────────────────────                           │
│                    • Local optimization                                    │
│                    • Budget ownership                                      │
│                    • Requirements gathering                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Results

```
3-YEAR OUTCOMES:
────────────────

FINANCIAL IMPACT:
┌────────────────────────────────────────────────────────────────────────────┐
│ Year 1:                                                                     │
│ • Spend: $15M → $13M (14% reduction)                                       │
│ • Savings: $2M                                                              │
│ • Unit cost reduction: 20% (cost per transaction)                         │
│                                                                             │
│ Year 2:                                                                     │
│ • Spend: $13M → $14M (8% growth vs projected 25%)                         │
│ • Avoided cost: $4M                                                         │
│ • Unit cost reduction: Additional 15%                                      │
│                                                                             │
│ Year 3:                                                                     │
│ • Spend: $14M → $16M (14% growth with 40% more workloads)                 │
│ • Cost efficiency: Best in industry benchmark                              │
└────────────────────────────────────────────────────────────────────────────┘

CULTURAL IMPACT:
┌────────────────────────────────────────────────────────────────────────────┐
│ • 85% of engineers consider cost in design decisions                       │
│ • Cost is discussed in sprint planning                                     │
│ • Business units own their cloud budgets                                   │
│ • Monthly FinOps review is standard practice                               │
│ • Cost optimization is a promotion criterion for architects                │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Case Study 3: Startup Cost Control

### Scenario

**Company**: Series B SaaS startup (50 employees)
**Challenge**: Burn rate too high, need to extend runway
**Monthly Cloud Spend**: $85,000 → Target: $50,000

### Approach

```
STARTUP-SPECIFIC OPTIMIZATIONS:
───────────────────────────────

PRIORITY 1: Eliminate Waste (Immediate)
┌────────────────────────────────────────────────────────────────────────────┐
│ Finding: 3 complete staging environments, only 1 needed                   │
│ Action: Consolidated to 1 staging + on-demand environments                │
│ Savings: $15,000/month                                                     │
│                                                                             │
│ Finding: Production database Premium P6, using 5% capacity                │
│ Action: Downgraded to P2, implemented read replicas                       │
│ Savings: $8,000/month                                                      │
│                                                                             │
│ Finding: Dev VMs running 24/7                                              │
│ Action: Auto-shutdown + use Codespaces for development                    │
│ Savings: $5,000/month                                                      │
└────────────────────────────────────────────────────────────────────────────┘

PRIORITY 2: Architecture Changes (1-2 months)
┌────────────────────────────────────────────────────────────────────────────┐
│ Change: Moved batch processing to Spot VMs                                 │
│ Savings: $4,000/month (80% discount on batch compute)                     │
│                                                                             │
│ Change: Implemented Azure CDN for static assets                            │
│ Savings: $2,000/month (reduced origin egress)                             │
│                                                                             │
│ Change: Moved to serverless where appropriate                              │
│ - Azure Functions for background jobs                                      │
│ - Cosmos DB serverless for low-traffic collections                        │
│ Savings: $3,000/month                                                      │
└────────────────────────────────────────────────────────────────────────────┘

PRIORITY 3: Startup Programs (Ongoing)
┌────────────────────────────────────────────────────────────────────────────┐
│ Microsoft for Startups: $150K in Azure credits                            │
│ (Applied and accepted - extended runway by 3 months)                      │
└────────────────────────────────────────────────────────────────────────────┘
```

### Results

```
OUTCOME:
────────

Before: $85,000/month
After:  $48,000/month (44% reduction)

Runway extended: 8 months → 14 months (with credits)

Key decisions:
• Did NOT buy reservations (uncertain future scale)
• Focused on elimination and right-sizing
• Used startup credits strategically
• Built cost awareness into engineering culture early
```

---

## Key Takeaways

```
FINOPS SUCCESS FACTORS:
───────────────────────

1. EXECUTIVE SPONSORSHIP
   • CFO and CTO alignment essential
   • Cost optimization must be a business priority

2. VISIBILITY FIRST
   • Can't optimize what you can't see
   • Tagging and reporting before optimization

3. SHARED ACCOUNTABILITY
   • Engineering owns efficiency
   • Finance owns budgets
   • FinOps facilitates

4. CONTINUOUS PROCESS
   • Not a one-time project
   • Regular reviews and optimization cycles

5. RIGHT-SIZE COMMITMENTS
   • Reservations for stable workloads
   • Start conservative, expand based on data
   • Monitor utilization weekly

6. ARCHITECTURE MATTERS
   • Biggest savings come from design decisions
   • Include cost in architecture reviews
   • Consider serverless and managed services
```

---

*Back to [Chapter Overview](README.md)* | *Next Chapter: [Architecture Case Studies](../08-case-studies/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
