# Microsoft Purview Overview

## What is Microsoft Purview?

Microsoft Purview is a unified data governance platform that helps you:
- **Discover** sensitive data across your entire data estate
- **Classify** data automatically using built-in or custom rules
- **Protect** data with labels, encryption, and DLP
- **Govern** data with policies, lineage, and compliance tools

### AWS Macie vs Microsoft Purview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AWS MACIE vs MICROSOFT PURVIEW                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS MACIE                             MICROSOFT PURVIEW                    │
│  ─────────                             ─────────────────                    │
│                                                                              │
│  Scope: S3 buckets                     Scope: Everything                    │
│  ┌─────────────────────┐              ┌─────────────────────────────────┐   │
│  │ • S3 data discovery │              │ • Azure Storage                  │   │
│  │ • PII detection     │              │ • Azure SQL/Synapse             │   │
│  │ • Classification    │              │ • Power BI                       │   │
│  └─────────────────────┘              │ • Microsoft 365                  │   │
│                                       │ • On-premises SQL                │   │
│  No data protection                   │ • AWS S3 (yes, AWS!)             │   │
│  No DLP                               │ • GCP Storage                    │   │
│  Separate from AWS Config             │ • Teradata, SAP, Oracle...       │   │
│                                       └─────────────────────────────────┘   │
│                                                                              │
│                                       Plus:                                 │
│                                       • DLP policies                        │
│                                       • Sensitivity labels                 │
│                                       • Encryption                         │
│                                       • Compliance Manager                 │
│                                       • Insider Risk                       │
│                                       • eDiscovery                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Purview Components

### 1. Data Map (Discovery & Classification)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PURVIEW DATA MAP                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DATA SOURCES (What you can scan):                                          │
│                                                                              │
│  Azure:                    Multi-cloud:              On-premises:           │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐     │
│  │ Azure SQL       │      │ AWS S3          │      │ SQL Server      │     │
│  │ Synapse         │      │ AWS Glue        │      │ Oracle          │     │
│  │ Cosmos DB       │      │ GCP BigQuery    │      │ SAP HANA        │     │
│  │ Data Lake       │      │ GCP Storage     │      │ Teradata        │     │
│  │ Blob Storage    │      │                 │      │ File Shares     │     │
│  │ Power BI        │      │                 │      │                 │     │
│  └─────────────────┘      └─────────────────┘      └─────────────────┘     │
│                                                                              │
│  SCAN RESULTS:                                                              │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │                                                                         ││
│  │  Asset: customers_table (Azure SQL)                                    ││
│  │  ├── Schema: sales.customers                                           ││
│  │  ├── Columns: 15                                                       ││
│  │  │                                                                      ││
│  │  │  Column Classifications:                                            ││
│  │  │  ┌─────────────────┬─────────────────────────────────────────────┐ ││
│  │  │  │ first_name      │ Person's Name                                │ ││
│  │  │  │ last_name       │ Person's Name                                │ ││
│  │  │  │ email           │ Email Address                                │ ││
│  │  │  │ ssn             │ U.S. Social Security Number (SSN) ⚠️        │ ││
│  │  │  │ credit_card     │ Credit Card Number ⚠️                       │ ││
│  │  │  │ phone           │ U.S. Phone Number                           │ ││
│  │  │  └─────────────────┴─────────────────────────────────────────────┘ ││
│  │  │                                                                      ││
│  │  └── Sensitivity: Confidential (auto-applied based on SSN/CC)         ││
│  │                                                                         ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. Data Catalog (Business Context)

```
BUSINESS GLOSSARY:
──────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│  Term: Customer Lifetime Value (CLV)                                        │
│  ─────────────────────────────────────                                      │
│  Definition: The total revenue expected from a customer over their         │
│              entire relationship with the company.                         │
│                                                                             │
│  Owner: Finance Team                                                        │
│  Steward: John Smith                                                        │
│                                                                             │
│  Related Assets:                                                            │
│  ├── sales.customer_metrics (Azure SQL)                                    │
│  ├── CustomerAnalytics (Power BI Dataset)                                  │
│  └── CLV_Report.xlsx (SharePoint)                                          │
│                                                                             │
│  Related Terms:                                                             │
│  ├── Customer Acquisition Cost (CAC)                                       │
│  └── Monthly Recurring Revenue (MRR)                                       │
└────────────────────────────────────────────────────────────────────────────┘
```

### 3. Data Lineage (Where Data Flows)

```
DATA LINEAGE VISUALIZATION:
───────────────────────────

┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Source:    │     │  Transform: │     │  Warehouse: │     │  Report:    │
│  CRM API    │────▶│  Azure Data │────▶│  Synapse    │────▶│  Power BI   │
│             │     │  Factory    │     │  Analytics  │     │  Dashboard  │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
                           │
                           ▼
                    ┌─────────────┐
                    │  Transform: │
                    │  Databricks │────▶ ML Model
                    └─────────────┘

Benefits:
• Trace data from source to consumption
• Impact analysis (what breaks if source changes?)
• Compliance (where does PII flow?)
• Debugging (where did bad data originate?)
```

---

## Setting Up Purview

### Create Purview Account

```bash
# Create Purview account
az purview account create \
  --account-name mypurview \
  --resource-group myRG \
  --location eastus \
  --managed-group-name purview-managed-rg

# Get Purview endpoint
az purview account show \
  --account-name mypurview \
  --resource-group myRG \
  --query "endpoints.catalog" -o tsv
```

### Register Data Sources

```bash
# Register Azure SQL (via Portal or REST API typically)
# In Purview Portal:
# 1. Data Map → Sources → Register
# 2. Select "Azure SQL Database"
# 3. Provide connection details
# 4. Choose authentication (Managed Identity recommended)
```

### Create and Run Scan

```
SCAN CONFIGURATION:
───────────────────

Source: Azure SQL Database
├── Server: myserver.database.windows.net
├── Database: salesdb
├── Authentication: Purview Managed Identity
│
├── Scan Rule Set: System default (Azure SQL)
│   ├── Detect: Credit Card Numbers
│   ├── Detect: SSN
│   ├── Detect: Email Addresses
│   └── Detect: Custom patterns...
│
├── Scan Trigger:
│   ├── Once (manual)
│   └── Recurring (weekly recommended)
│
└── Scope:
    ├── All schemas
    └── Exclude: [temp%, staging%]
```

---

## Case Study: Multi-Cloud Data Governance

### Scenario

**Company**: Retail company with data in Azure, AWS, and on-premises
**Challenge**: No visibility into where sensitive data exists
**Goal**: Unified data governance across all platforms

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MULTI-CLOUD DATA GOVERNANCE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                          MICROSOFT PURVIEW                                   │
│                    ┌────────────────────────────┐                           │
│                    │  Unified Data Map          │                           │
│                    │  • All assets cataloged    │                           │
│                    │  • Classifications applied │                           │
│                    │  • Lineage tracked         │                           │
│                    └─────────────┬──────────────┘                           │
│                                  │                                           │
│         ┌────────────────────────┼────────────────────────┐                 │
│         │                        │                        │                 │
│         ▼                        ▼                        ▼                 │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐        │
│  │     AZURE       │    │      AWS        │    │  ON-PREMISES    │        │
│  │                 │    │                 │    │                 │        │
│  │ • SQL Database  │    │ • S3 Buckets    │    │ • SQL Server    │        │
│  │ • Synapse       │    │ • Redshift      │    │ • Oracle        │        │
│  │ • Data Lake     │    │ • Glue Catalog  │    │ • File Shares   │        │
│  │ • Power BI      │    │                 │    │                 │        │
│  │                 │    │                 │    │                 │        │
│  │ Scan via:       │    │ Scan via:       │    │ Scan via:       │        │
│  │ Managed Identity│    │ AWS Role +      │    │ Self-hosted     │        │
│  │                 │    │ Purview creds   │    │ Integration     │        │
│  │                 │    │                 │    │ Runtime         │        │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘        │
│                                                                              │
│  RESULTS:                                                                   │
│  ────────                                                                   │
│  • 1,500 data assets discovered                                            │
│  • 45 contain PII (SSN, credit cards)                                      │
│  • 12 have PII in unexpected locations (shadow data!)                      │
│  • Data lineage shows PII flowing to unprotected reports                   │
│                                                                              │
│  ACTIONS TAKEN:                                                             │
│  ──────────────                                                             │
│  • Applied sensitivity labels to PII sources                               │
│  • Created DLP policies to prevent PII leakage                            │
│  • Updated data pipelines to mask PII before reporting                    │
│  • Implemented column-level encryption for sensitive fields               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Purview Roles and Permissions

| Role | What They Can Do |
|------|------------------|
| Purview Data Reader | View assets, classifications, glossary |
| Purview Data Curator | Edit assets, manage glossary terms |
| Purview Data Source Admin | Register sources, manage scans |
| Purview Data Share Contributor | Create and manage data shares |

### Assign Roles

```bash
# Assign Purview role (via REST API or Portal)
# These are Purview-specific roles, not Azure RBAC

# In Purview Portal:
# Management → Role assignments → Add
# Select: Collection (root or specific)
# Select: Role (Data Reader, Data Curator, etc.)
# Select: User/Group
```

---

## Integration with Other Services

### Purview + Defender for Cloud

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  DEFENDER FOR CLOUD                    PURVIEW                             │
│  ──────────────────                    ───────                             │
│                                                                             │
│  Security Posture                      Data Governance                     │
│  • Recommendations    ◄───────────────▶ • Classifications                  │
│  • Secure Score                        • Sensitivity labels                │
│  • Threat detection                    • Data lineage                      │
│                                                                             │
│  Integration:                                                               │
│  • Purview classifications show in Defender                                │
│  • "Sensitive data at risk" recommendations                                │
│  • Unified view of security + data governance                              │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Purview + Azure Policy

```bash
# Example: Require resources to be scanned by Purview
# (Custom policy definition)

{
  "mode": "All",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "type",
          "equals": "Microsoft.Sql/servers/databases"
        },
        {
          "field": "tags['purview-scanned']",
          "exists": "false"
        }
      ]
    },
    "then": {
      "effect": "audit"
    }
  }
}
```

---

*Next: [DLP Policies](02-dlp-policies.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
