# Chapter 07: FinOps & Cost Management

## Overview

Azure Cost Management provides comprehensive tools for monitoring, allocating, and optimizing cloud spend. For AWS engineers, this replaces AWS Cost Explorer, Budgets, and Cost Allocation Tags with a unified platform that also supports multi-cloud visibility.

## AWS to Azure Mapping

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      FINOPS: AWS vs AZURE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS                                   AZURE                                │
│  ───                                   ─────                                │
│                                                                              │
│  Cost Explorer              ──────────▶ Cost Analysis                       │
│  AWS Budgets                ──────────▶ Azure Budgets                       │
│  Cost Allocation Tags       ──────────▶ Cost Allocation / Tags              │
│  Savings Plans              ──────────▶ Azure Savings Plans                 │
│  Reserved Instances         ──────────▶ Azure Reservations                  │
│  Spot Instances             ──────────▶ Azure Spot VMs                      │
│  Compute Optimizer          ──────────▶ Azure Advisor                       │
│  Trusted Advisor (cost)     ──────────▶ Azure Advisor                       │
│  AWS Organizations (billing)──────────▶ Management Groups + Billing Scopes │
│  Cost & Usage Reports       ──────────▶ Cost Management Exports             │
│                                                                              │
│  KEY DIFFERENCES:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  AWS: Cost Explorer is separate from budgets and recommendations    │   │
│  │       Savings Plans and RIs are different purchase mechanisms       │   │
│  │                                                                       │   │
│  │  Azure: Unified Cost Management + Billing portal                    │   │
│  │         Reservations cover compute, storage, database, and more     │   │
│  │         Native multi-cloud support (AWS, GCP visibility)            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Azure Cost Management Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AZURE COST MANAGEMENT ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  BILLING HIERARCHY                    COST MANAGEMENT FEATURES             │
│  ─────────────────                    ────────────────────────             │
│                                                                              │
│  ┌─────────────────────────┐         ┌─────────────────────────────────┐  │
│  │   BILLING ACCOUNT       │         │                                 │  │
│  │   (EA/MCA/CSP)          │         │         COST ANALYSIS           │  │
│  │                         │         │                                 │  │
│  │  ┌───────────────────┐  │         │  • View costs by resource      │  │
│  │  │  Billing Profile  │  │         │  • Group by subscription/tag   │  │
│  │  │  (Invoice section)│  │◀───────▶│  • Time-based trends           │  │
│  │  └───────────────────┘  │         │  • Forecast future spend       │  │
│  │                         │         │  • Export reports              │  │
│  │  ┌───────────────────┐  │         │                                 │  │
│  │  │    Departments    │  │         └─────────────────────────────────┘  │
│  │  │   (EA only)       │  │                                              │
│  │  └───────────────────┘  │         ┌─────────────────────────────────┐  │
│  │                         │         │                                 │  │
│  └─────────────────────────┘         │           BUDGETS               │  │
│            │                         │                                 │  │
│            ▼                         │  • Set spending limits          │  │
│  ┌─────────────────────────┐         │  • Alert at thresholds         │  │
│  │    SUBSCRIPTIONS        │         │  • Trigger automation          │  │
│  │                         │◀───────▶│  • Forecasted alerts           │  │
│  │  ┌───────────────────┐  │         │                                 │  │
│  │  │  Resource Groups  │  │         └─────────────────────────────────┘  │
│  │  │                   │  │                                              │
│  │  │  ┌─────────────┐  │  │         ┌─────────────────────────────────┐  │
│  │  │  │  Resources  │  │  │         │                                 │  │
│  │  │  │  (with tags)│  │  │         │          ADVISOR                │  │
│  │  │  └─────────────┘  │  │         │                                 │  │
│  │  └───────────────────┘  │◀───────▶│  • Right-sizing VMs            │  │
│  │                         │         │  • Reservation recommendations │  │
│  │                         │         │  • Unused resources            │  │
│  └─────────────────────────┘         │  • Savings opportunities       │  │
│                                      │                                 │  │
│                                      └─────────────────────────────────┘  │
│                                                                              │
│  COST ALLOCATION                     COMMITMENT DISCOUNTS                  │
│  ───────────────                     ────────────────────                  │
│                                                                              │
│  ┌─────────────────────────┐         ┌─────────────────────────────────┐  │
│  │                         │         │                                 │  │
│  │  • Tags (cost center,   │         │  RESERVATIONS (1-3 year)       │  │
│  │    project, owner)      │         │  • VMs: up to 72% savings       │  │
│  │                         │         │  • SQL DB: up to 80% savings   │  │
│  │  • Subscription-based   │         │  • Storage: up to 38% savings  │  │
│  │    allocation           │         │  • Cosmos DB, Redis, etc.      │  │
│  │                         │         │                                 │  │
│  │  • Resource group       │         │  SAVINGS PLANS (1-3 year)      │  │
│  │    chargeback           │         │  • Flexible across VM sizes    │  │
│  │                         │         │  • up to 65% savings           │  │
│  │  • Custom allocation    │         │  • Simpler than reservations   │  │
│  │    rules                │         │                                 │  │
│  │                         │         │  SPOT VMS                       │  │
│  │                         │         │  • up to 90% savings            │  │
│  └─────────────────────────┘         │  • Can be evicted              │  │
│                                      │                                 │  │
│                                      └─────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Chapter Contents

### Quick Reference
- [Quick Reference](quick-reference.md) - Essential commands, cost queries, and optimization checklist

### Deep Dive Topics
1. [Cost Analysis & Visibility](01-cost-analysis.md) - Understanding and analyzing Azure costs
2. [Budgets & Alerts](02-budgets-alerts.md) - Setting budgets and automated responses
3. [Reservations & Savings Plans](03-reservations.md) - Commitment-based discounts
4. [Cost Optimization](04-optimization.md) - Right-sizing, cleanup, and best practices
5. [Case Studies](case-studies.md) - Real-world FinOps implementations

## Key Concepts for AWS Engineers

### Billing Account Types

```
AZURE BILLING ACCOUNT TYPES:
────────────────────────────

ENTERPRISE AGREEMENT (EA):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  • Large organizations (>$100K/year commitment)                            │
│  • Upfront commitment, discounted rates                                    │
│  • Departments and enrollment accounts                                     │
│  • Similar to: AWS Enterprise Discount Program                             │
│                                                                             │
│  Hierarchy:                                                                 │
│  Billing Account → Departments → Enrollment Accounts → Subscriptions      │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

MICROSOFT CUSTOMER AGREEMENT (MCA):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  • Modern billing for direct customers                                     │
│  • No upfront commitment required                                          │
│  • Billing profiles and invoice sections                                   │
│  • Similar to: AWS regular accounts                                        │
│                                                                             │
│  Hierarchy:                                                                 │
│  Billing Account → Billing Profiles → Invoice Sections → Subscriptions    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘

CLOUD SOLUTION PROVIDER (CSP):
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  • Purchased through Microsoft partner                                     │
│  • Partner manages billing relationship                                    │
│  • May include value-added services                                        │
│  • Similar to: AWS Marketplace reseller                                    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Cost Visibility Scopes

```
COST MANAGEMENT SCOPES:
───────────────────────

You can view costs at different levels:

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  SCOPE                    ACCESS NEEDED              USE CASE               │
│  ─────                    ─────────────              ────────               │
│                                                                              │
│  Billing Account          Billing Admin             Total enterprise spend │
│  (EA/MCA)                                           across all subs         │
│                                                                              │
│  Management Group         Reader on MG              Business unit costs    │
│                                                     across child subs       │
│                                                                              │
│  Subscription             Reader on sub             Application/team costs │
│                                                                              │
│  Resource Group           Reader on RG              Project-level costs    │
│                                                                              │
│  Individual Resource      Reader on resource        Specific service cost  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

AWS COMPARISON:
• AWS Cost Explorer also supports account-level filtering
• Azure Management Groups = AWS Organizations OUs
• Azure provides native multi-cloud cost visibility (AWS, GCP)
```

### Tagging Strategy for Cost Allocation

```
RECOMMENDED TAGGING STRATEGY:
─────────────────────────────

MANDATORY TAGS (enforce via Azure Policy):
┌────────────────────────────────────────────────────────────────────────────┐
│ Tag Name          Example Value        Purpose                             │
│ ────────          ─────────────        ───────                             │
│ CostCenter        CC-12345             Financial allocation                │
│ Environment       prod/dev/test        Environment identification         │
│ Owner             team@company.com     Accountability                      │
│ Project           ProjectAlpha         Project tracking                    │
│ Application       inventory-api        Application identification         │
└────────────────────────────────────────────────────────────────────────────┘

OPTIONAL TAGS (recommended):
┌────────────────────────────────────────────────────────────────────────────┐
│ Tag Name          Example Value        Purpose                             │
│ ────────          ─────────────        ───────                             │
│ Department        Engineering          Organizational unit                 │
│ BusinessUnit      Retail               Business alignment                  │
│ Compliance        HIPAA/PCI            Regulatory tracking                 │
│ DataClass         Confidential         Data classification                 │
│ EndDate           2024-12-31           Temporary resource cleanup          │
└────────────────────────────────────────────────────────────────────────────┘

ENFORCEMENT VIA POLICY:
{
  "if": {
    "allOf": [
      { "field": "tags['CostCenter']", "exists": "false" },
      { "field": "type", "notEquals": "Microsoft.Resources/subscriptions" }
    ]
  },
  "then": {
    "effect": "deny"
  }
}
```

## Learning Path

```
RECOMMENDED STUDY ORDER:
────────────────────────

Week 1: Foundations
├── Read: Quick Reference (cost tools and comparisons)
├── Do: Access Cost Management for your subscription
├── Do: Create a basic budget with email alerts
└── Practice: Analyze last month's spend by resource type

Week 2: Deep Analysis
├── Read: Cost Analysis & Visibility deep dive
├── Do: Build custom cost views by tag
├── Do: Set up cost exports to storage account
└── Practice: Create Power BI dashboard from exports

Week 3: Optimization
├── Read: Cost Optimization deep dive
├── Do: Review Azure Advisor recommendations
├── Do: Identify and clean up unused resources
└── Practice: Calculate potential reservation savings

Week 4: Automation & Governance
├── Read: Budgets & Alerts deep dive
├── Read: Reservations & Savings Plans
├── Do: Create budget with automation (Logic App)
├── Review: Case studies
└── Practice: Design FinOps process for organization
```

## Key Services Summary

| Feature | Description | AWS Equivalent |
|---------|-------------|----------------|
| Cost Analysis | Interactive cost exploration | Cost Explorer |
| Budgets | Spending limits and alerts | AWS Budgets |
| Advisor (Cost) | Optimization recommendations | Trusted Advisor + Compute Optimizer |
| Reservations | 1-3 year capacity commitments | Reserved Instances |
| Savings Plans | Flexible compute commitment | Savings Plans |
| Cost Exports | Scheduled data exports | Cost & Usage Reports |
| Cost Allocation | Tag-based cost distribution | Cost Allocation Tags |

## Quick Cost Optimization Checklist

```
IMMEDIATE WINS (Do this week):
──────────────────────────────
☐ Review Azure Advisor cost recommendations
☐ Identify and delete unused resources (orphaned disks, IPs)
☐ Right-size underutilized VMs (<5% CPU)
☐ Enable auto-shutdown for dev/test VMs
☐ Review storage tiers (move cold data to Cool/Archive)

SHORT-TERM (This month):
────────────────────────
☐ Purchase reservations for stable workloads
☐ Set up budgets with alerts at 80%, 100%
☐ Implement tagging policy for new resources
☐ Review hybrid benefit eligibility (Windows/SQL licenses)
☐ Consolidate Log Analytics workspaces

LONG-TERM (This quarter):
─────────────────────────
☐ Implement FinOps culture (cost reviews, showback)
☐ Automate resource cleanup for dev environments
☐ Evaluate Savings Plans for variable workloads
☐ Set up cross-team cost allocation
☐ Create executive cost dashboards
```

---

*Next: [Quick Reference](quick-reference.md)* | *Back to [Chapter 06: Infrastructure as Code](../06-infrastructure-as-code/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
