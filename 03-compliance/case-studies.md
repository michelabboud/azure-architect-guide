# Compliance Case Studies

## Case Study 1: Healthcare Provider HIPAA Compliance

### Scenario

**Company**: Regional healthcare provider with 5,000 employees
**Challenge**: Protected Health Information (PHI) scattered across systems, no visibility into data flows, regulatory audit approaching
**AWS Background**: Previously used Macie for S3 scanning, custom Lambda for email monitoring

### The Problem

```
CURRENT STATE (Problematic):
────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  PHI SCATTERED EVERYWHERE:                                                  │
│                                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │ EMR System     │  │ File Shares    │  │ Email          │                │
│  │ (On-premises)  │  │ (SharePoint)   │  │ (Exchange)     │                │
│  │                │  │                │  │                │                │
│  │ ✓ Known PHI    │  │ ? Unknown PHI  │  │ ? Unknown PHI  │                │
│  │ ✓ Access logs  │  │ ✗ No scanning  │  │ ✗ No DLP       │                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                              │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                │
│  │ OneDrive       │  │ Teams Chats    │  │ Third-party    │                │
│  │ (Personal)     │  │                │  │ Apps           │                │
│  │                │  │                │  │                │                │
│  │ ? Unknown PHI  │  │ ? Unknown PHI  │  │ ? Unknown PHI  │                │
│  │ ✗ No control   │  │ ✗ No control   │  │ ✗ No visibility│                │
│  └────────────────┘  └────────────────┘  └────────────────┘                │
│                                                                              │
│  RISKS:                                                                     │
│  • HIPAA fines: $100-$50,000 per violation (up to $1.5M/year per category)│
│  • PHI in personal OneDrive folders                                        │
│  • Staff sharing patient info via Teams without encryption                 │
│  • No audit trail for data access                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               HIPAA COMPLIANCE WITH MICROSOFT PURVIEW                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                     ┌─────────────────────────────┐                         │
│                     │   MICROSOFT PURVIEW         │                         │
│                     │   COMPLIANCE CENTER         │                         │
│                     │                             │                         │
│                     │  • DLP Policies (HIPAA)    │                         │
│                     │  • Sensitivity Labels       │                         │
│                     │  • Data Map & Catalog      │                         │
│                     │  • Compliance Manager      │                         │
│                     └──────────────┬──────────────┘                         │
│                                    │                                         │
│         ┌──────────────────────────┼──────────────────────────┐             │
│         │                          │                          │             │
│         ▼                          ▼                          ▼             │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐    │
│  │  DISCOVERY      │      │  PROTECTION     │      │  GOVERNANCE     │    │
│  │                 │      │                 │      │                 │    │
│  │ Data Map scans: │      │ DLP Policies:   │      │ Compliance:     │    │
│  │ • SharePoint    │      │ • Block PHI     │      │ • HIPAA template│    │
│  │ • OneDrive      │      │   external      │      │ • Assessments   │    │
│  │ • Exchange      │      │ • Warn on PHI   │      │ • Evidence      │    │
│  │ • SQL databases │      │   in Teams      │      │                 │    │
│  │ • File shares   │      │ • Audit all PHI │      │ Retention:      │    │
│  │   (via SHIR)    │      │   movement      │      │ • 7-year hold   │    │
│  │                 │      │                 │      │ • Legal hold    │    │
│  │ Classifications:│      │ Labels:         │      │   capability    │    │
│  │ • SSN           │      │ • PHI-Protected │      │                 │    │
│  │ • Medical terms │      │ • PHI-Encrypted │      │ Audit:          │    │
│  │ • Patient IDs   │      │ • PHI-Research  │      │ • Full audit    │    │
│  │   (custom)      │      │   (de-ID'd)     │      │   log           │    │
│  └─────────────────┘      └─────────────────┘      └─────────────────┘    │
│                                                                              │
│  INTEGRATION WITH CLINICAL SYSTEMS:                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  EMR System (Epic/Cerner)    Azure SQL (Analytics)    Power BI       │   │
│  │  ┌─────────────────┐         ┌─────────────────┐     ┌─────────┐    │   │
│  │  │ Self-hosted     │─────────│ PHI columns     │─────│ Reports │    │   │
│  │  │ Integration     │  ADF    │ auto-labeled    │     │ with    │    │   │
│  │  │ Runtime scans   │         │ column-level    │     │ label   │    │   │
│  │  │                 │         │ classification  │     │ inherit │    │   │
│  │  └─────────────────┘         └─────────────────┘     └─────────┘    │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```
PHASE 1: DISCOVERY (Week 1-2)
─────────────────────────────

1. Create Purview account
2. Register data sources:
   - SharePoint Online (all sites)
   - Exchange Online (all mailboxes)
   - OneDrive (all accounts)
   - Azure SQL databases
   - On-premises file shares (via Self-hosted IR)

3. Configure custom sensitive info types:
   - Patient Medical Record Number: "MRN-[0-9]{8}"
   - Internal Provider ID: "PRV-[A-Z]{2}[0-9]{4}"

4. Run initial scans (full scan all sources)

RESULTS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Source              │ Assets  │ PHI Found │ High Risk                      │
├─────────────────────┼─────────┼───────────┼────────────────────────────────┤
│ SharePoint          │ 45,000  │ 2,341     │ 156 in personal sites          │
│ OneDrive            │ 120,000 │ 892       │ 892 (all personal!)            │
│ Exchange            │ 2.1M    │ 15,234    │ 234 to external                │
│ Azure SQL           │ 45 DBs  │ 12 tables │ 3 unencrypted                  │
│ File Shares         │ 890,000 │ 5,672     │ 1,234 in shared drives         │
└────────────────────────────────────────────────────────────────────────────┘


PHASE 2: PROTECTION (Week 3-4)
──────────────────────────────

1. Create sensitivity labels:
   - PHI - Standard (encryption, watermark)
   - PHI - Research (de-identified, less restrictive)
   - PHI - External (authorized external sharing)

2. Create DLP policies:
   - Block PHI in external email (no exceptions)
   - Warn on PHI in Teams (with override for care coordination)
   - Block PHI to USB devices
   - Audit all PHI access

3. Enable auto-labeling:
   - All documents with SSN + medical terms = PHI - Standard
   - Apply retroactively to discovered content

PHASE 3: GOVERNANCE (Week 5-6)
──────────────────────────────

1. Configure Compliance Manager:
   - Add HIPAA assessment
   - Assign control owners
   - Upload evidence for completed controls

2. Set up retention:
   - 7-year retention for all PHI
   - Legal hold capability for litigation

3. Enable audit logging:
   - All label changes logged
   - All PHI access logged
   - Weekly report to Compliance Officer
```

### Architectural Decision

**Question**: Should we use Endpoint DLP or rely on cloud-only DLP?

```
DECISION MATRIX:
────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  OPTION A: Cloud-Only DLP             OPTION B: Cloud + Endpoint DLP        │
│  ──────────────────────               ───────────────────────────           │
│                                                                              │
│  Pros:                                Pros:                                  │
│  ✓ Faster deployment                  ✓ Complete coverage                   │
│  ✓ No endpoint agent needed           ✓ USB/print protection                │
│  ✓ Lower complexity                   ✓ Offline protection                  │
│                                       ✓ Clipboard monitoring                │
│  Cons:                                                                       │
│  ✗ No USB protection                  Cons:                                  │
│  ✗ No print control                   ✗ Requires Defender for Endpoint      │
│  ✗ Data can leave via device          ✗ Agent deployment effort             │
│                                       ✗ E5 license required                  │
│                                                                              │
│  CHOSEN: Option B (Cloud + Endpoint DLP)                                    │
│                                                                              │
│  RATIONALE:                                                                 │
│  • HIPAA requires protection of PHI in ALL forms                           │
│  • Significant risk of PHI on USB drives                                   │
│  • Already deploying Defender for Endpoint for security                    │
│  • Investment justified by regulatory risk                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Results

```
6-MONTH OUTCOMES:
─────────────────

COMPLIANCE:
• Compliance Manager score: 45% → 89%
• HIPAA audit: Passed with minor findings
• PHI incidents: 23 → 2 (91% reduction)

VISIBILITY:
• 100% of data sources scanned
• 45,000+ documents auto-labeled
• Complete data lineage for analytics

USER ADOPTION:
• 85% of users trained on labels
• Override requests reviewed within 4 hours
• False positive rate < 5%

COST:
• Purview: ~$15,000/month
• Licensing upgrade (E5): ~$20/user/month premium
• Avoided HIPAA fine: Potentially $1M+
```

---

## Case Study 2: Financial Services Multi-Cloud Governance

### Scenario

**Company**: Investment firm with data across Azure, AWS, and on-premises
**Challenge**: No unified view of sensitive data across clouds, PCI-DSS compliance gaps
**AWS Background**: Macie for S3, custom classification for Redshift

### The Problem

```
CURRENT STATE (Multi-Cloud Chaos):
──────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  THREE SEPARATE GOVERNANCE APPROACHES:                                      │
│                                                                              │
│  AWS                         AZURE                        ON-PREMISES       │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐ │
│  │ Macie           │        │ (Nothing)       │        │ Manual          │ │
│  │ - S3 only       │        │                 │        │ spreadsheets    │ │
│  │ - No DLP        │        │ Azure SQL       │        │                 │ │
│  │ - Separate      │        │ unclassified    │        │ SQL Server      │ │
│  │   console       │        │                 │        │ unscanned       │ │
│  │                 │        │ SharePoint      │        │                 │ │
│  │ Redshift        │        │ no labels       │        │ File shares     │ │
│  │ - Custom tags   │        │                 │        │ unknown content │ │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘ │
│                                                                              │
│  PROBLEMS:                                                                  │
│  • PCI auditor asks "where is cardholder data?" → No single answer         │
│  • Different classification schemes per cloud                               │
│  • No unified DLP (data can leak between clouds)                           │
│  • Manual audit evidence collection                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              UNIFIED MULTI-CLOUD GOVERNANCE WITH PURVIEW                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                     ┌─────────────────────────────┐                         │
│                     │   MICROSOFT PURVIEW         │                         │
│                     │   (Unified Data Governance) │                         │
│                     │                             │                         │
│                     │  Single Data Map            │                         │
│                     │  Unified Classifications    │                         │
│                     │  Cross-cloud Lineage        │                         │
│                     └──────────────┬──────────────┘                         │
│                                    │                                         │
│     ┌──────────────────────────────┼──────────────────────────┐             │
│     │                              │                          │             │
│     ▼                              ▼                          ▼             │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐  │
│  │      AZURE        │    │       AWS         │    │   ON-PREMISES     │  │
│  │                   │    │                   │    │                   │  │
│  │  Scanned:         │    │  Scanned:         │    │  Scanned:         │  │
│  │  • Azure SQL      │    │  • S3 Buckets     │    │  • SQL Server     │  │
│  │  • Synapse        │    │  • Redshift       │    │  • Oracle         │  │
│  │  • Cosmos DB      │    │  • Glue Catalog   │    │  • File Shares    │  │
│  │  • Blob Storage   │    │  • RDS            │    │                   │  │
│  │  • SharePoint     │    │                   │    │  Connection:      │  │
│  │  • Power BI       │    │  Connection:      │    │  Self-hosted      │  │
│  │                   │    │  Cross-account    │    │  Integration      │  │
│  │  Connection:      │    │  IAM role         │    │  Runtime          │  │
│  │  Managed Identity │    │                   │    │                   │  │
│  └───────────────────┘    └───────────────────┘    └───────────────────┘  │
│                                                                              │
│  UNIFIED CLASSIFICATION RESULTS:                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  Classification: Credit Card Number                                  │   │
│  │  ┌────────────────────────────────────────────────────────────────┐ │   │
│  │  │ Location              │ Assets │ Tables/Columns │ Risk         │ │   │
│  │  ├───────────────────────┼────────┼────────────────┼──────────────┤ │   │
│  │  │ Azure SQL             │ 12     │ 34 columns     │ Encrypted ✓  │ │   │
│  │  │ AWS S3                │ 45     │ N/A (files)    │ Unencrypted! │ │   │
│  │  │ AWS Redshift          │ 8      │ 23 columns     │ Encrypted ✓  │ │   │
│  │  │ On-prem SQL Server    │ 5      │ 12 columns     │ Unencrypted! │ │   │
│  │  │ SharePoint            │ 234    │ N/A (files)    │ No DLP!      │ │   │
│  │  └────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                       │   │
│  │  TOTAL: 304 assets with credit card data across 3 clouds             │   │
│  │  ACTION REQUIRED: 279 assets need remediation                        │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```
AWS INTEGRATION SETUP:
──────────────────────

# 1. Create IAM role in AWS for Purview
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::PURVIEW_ACCOUNT:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "PURVIEW_EXTERNAL_ID"
        }
      }
    }
  ]
}

# 2. Attach policies for S3, Glue, Redshift access
aws iam attach-role-policy \
  --role-name PurviewScanner \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# 3. Register in Purview
# Data Map → Sources → Register → Amazon S3
# Provide: Role ARN, External ID, Region


DLP POLICY FOR PCI DATA:
────────────────────────

Policy: PCI-DSS Cardholder Data Protection
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│ Locations:                                                                  │
│ ☑ Exchange Online        ☑ SharePoint Online                               │
│ ☑ OneDrive              ☑ Teams                                            │
│ ☑ Endpoint DLP          ☑ Defender for Cloud Apps (for AWS console)       │
│                                                                             │
│ Rules:                                                                      │
│                                                                             │
│ Rule 1: Block external sharing of card data                                │
│ ├── IF: Credit Card Number detected (High confidence, 1+)                  │
│ ├── AND: Shared externally                                                 │
│ └── THEN: Block + Notify compliance team                                   │
│                                                                             │
│ Rule 2: Audit all card data movement                                       │
│ ├── IF: Credit Card Number detected (any confidence)                       │
│ └── THEN: Log activity (no block)                                          │
│                                                                             │
│ Rule 3: Block card data to unauthorized cloud apps                         │
│ ├── IF: Credit Card Number detected                                        │
│ ├── AND: Upload to unsanctioned cloud storage                             │
│ └── THEN: Block                                                            │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Architectural Decision

**Question**: Should we use Purview for AWS or keep Macie?

```
ANALYSIS:
─────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  OPTION A: Purview + Macie        OPTION B: Purview Only                   │
│  (Dual systems)                   (Unified)                                 │
│                                                                              │
│  Purview for:                     Purview for:                              │
│  - Azure assets                   - ALL assets (Azure, AWS, on-prem)       │
│  - Unified view                   - Unified classification                 │
│  - M365 DLP                       - Single data catalog                    │
│                                   - Cross-cloud lineage                    │
│  Macie for:                                                                 │
│  - S3 detailed scanning           No Macie:                                │
│  - AWS-native alerts              - Retire Macie                           │
│  - GuardDuty integration                                                   │
│                                                                              │
│  Pros:                            Pros:                                     │
│  ✓ Best-of-breed per cloud        ✓ Single pane of glass                  │
│  ✓ AWS-native integrations        ✓ Unified taxonomy                      │
│  ✓ No migration effort            ✓ Simplified operations                 │
│                                   ✓ Single audit evidence source          │
│  Cons:                            ✓ Cost savings (retire Macie)           │
│  ✗ Two dashboards                                                          │
│  ✗ Two classification schemes     Cons:                                    │
│  ✗ Manual correlation             ✗ Less AWS-native features              │
│  ✗ Double cost                    ✗ Migration effort                       │
│                                                                              │
│  CHOSEN: Option B (Purview Only)                                           │
│                                                                              │
│  RATIONALE:                                                                 │
│  • PCI auditors want ONE answer to "where is card data?"                  │
│  • Operational simplicity > marginal feature differences                   │
│  • Cross-cloud lineage critical for data flow understanding               │
│  • Cost reduction by eliminating Macie (~$2/GB scanned)                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Results

```
OUTCOMES:
─────────

PCI AUDIT:
• Question: "Where is cardholder data stored?"
• Before: "We think S3, Azure SQL, and some file shares" (3 days to compile)
• After: Single Purview report, real-time, complete inventory

METRICS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric                      │ Before        │ After         │ Change      │
├─────────────────────────────┼───────────────┼───────────────┼─────────────┤
│ Time to locate card data    │ 3 days        │ 5 minutes     │ -99%        │
│ Classification coverage     │ 35%           │ 98%           │ +63%        │
│ DLP incidents (card data)   │ 45/month      │ 3/month       │ -93%        │
│ Audit prep time             │ 2 weeks       │ 2 days        │ -85%        │
│ Monthly tooling cost        │ $18,000       │ $12,000       │ -33%        │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Case Study 3: Global Enterprise GDPR/Privacy Compliance

### Scenario

**Company**: Global manufacturing company with EU, US, and APAC operations
**Challenge**: Different privacy regulations per region, data residency requirements
**AWS Background**: Minimal - mostly on-premises and Azure

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│           MULTI-REGION PRIVACY COMPLIANCE ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  REGULATORY REQUIREMENTS BY REGION:                                         │
│  ┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐ │
│  │ EU (GDPR)       │ US (Various)    │ APAC            │ GLOBAL          │ │
│  ├─────────────────┼─────────────────┼─────────────────┼─────────────────┤ │
│  │ Data residency  │ CCPA (CA)       │ PDPA (Singapore)│ Corporate       │ │
│  │ Right to delete │ State laws      │ Privacy Act (AU)│ policy          │ │
│  │ Consent mgmt    │ Industry regs   │ PIPL (China)    │                 │ │
│  │ 72-hr breach    │                 │                 │                 │ │
│  └─────────────────┴─────────────────┴─────────────────┴─────────────────┘ │
│                                                                              │
│  ARCHITECTURE:                                                              │
│                                                                              │
│              ┌────────────────────────────────────┐                         │
│              │     GLOBAL PURVIEW INSTANCE        │                         │
│              │     (Centralized Governance)       │                         │
│              │                                    │                         │
│              │  • Unified data catalog            │                         │
│              │  • Global label taxonomy           │                         │
│              │  • Cross-region lineage            │                         │
│              │  • Compliance Manager              │                         │
│              └─────────────────┬──────────────────┘                         │
│                                │                                            │
│      ┌─────────────────────────┼─────────────────────────┐                 │
│      │                         │                         │                 │
│      ▼                         ▼                         ▼                 │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐            │
│  │   EU REGION   │     │   US REGION   │     │  APAC REGION  │            │
│  │   (West EU)   │     │   (East US)   │     │  (SE Asia)    │            │
│  │               │     │               │     │               │            │
│  │ Data Sources: │     │ Data Sources: │     │ Data Sources: │            │
│  │ • Azure SQL   │     │ • Azure SQL   │     │ • Azure SQL   │            │
│  │ • SharePoint  │     │ • SharePoint  │     │ • SharePoint  │            │
│  │   (EU site)   │     │   (US site)   │     │   (APAC site) │            │
│  │ • On-prem     │     │ • On-prem     │     │ • On-prem     │            │
│  │   Germany     │     │   US DCs      │     │   Singapore   │            │
│  │               │     │               │     │               │
│  │ DLP Policy:   │     │ DLP Policy:   │     │ DLP Policy:   │            │
│  │ GDPR template │     │ CCPA + Corp   │     │ PDPA + Corp   │            │
│  │               │     │               │     │               │            │
│  │ Labels:       │     │ Labels:       │     │ Labels:       │            │
│  │ EU Personal   │     │ US Personal   │     │ APAC Personal │            │
│  │ EU Sensitive  │     │ US Sensitive  │     │ APAC Sensitive│            │
│  └───────────────┘     └───────────────┘     └───────────────┘            │
│                                                                              │
│  DATA SUBJECT REQUEST WORKFLOW:                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │  1. DSR received    2. Purview search    3. Generate report          │   │
│  │     (via portal)       (all regions)        (within 30 days)         │   │
│  │                                                                       │   │
│  │  ┌─────────┐       ┌─────────────────┐   ┌─────────────────────┐    │   │
│  │  │ Subject │──────▶│ Content Search  │──▶│ Export personal     │    │   │
│  │  │ Request │       │ for email:      │   │ data or delete      │    │   │
│  │  │ Portal  │       │ john@example.eu │   │ per request type    │    │   │
│  │  └─────────┘       └─────────────────┘   └─────────────────────┘    │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Regional Label Strategy

```
LABEL HIERARCHY FOR MULTI-REGION:
─────────────────────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  GLOBAL LABELS (Apply everywhere):                                          │
│  ├── Public                                                                 │
│  ├── Internal                                                               │
│  └── Confidential                                                           │
│      └── Confidential\Corporate                                            │
│                                                                             │
│  REGIONAL LABELS (Scoped to regional users):                                │
│  │                                                                          │
│  ├── EU Labels:                                                             │
│  │   ├── EU Personal Data                                                  │
│  │   │   └── Encryption: Required                                          │
│  │   │   └── Residency: EU only                                            │
│  │   └── EU Sensitive Data (Art. 9)                                        │
│  │       └── Encryption: Required + additional controls                    │
│  │       └── Processing: Explicit consent required                         │
│  │                                                                          │
│  ├── US Labels:                                                             │
│  │   ├── US Personal Information                                           │
│  │   │   └── CCPA: Sale opt-out tracking                                  │
│  │   └── US Sensitive (HIPAA/Financial)                                    │
│  │       └── Industry-specific protections                                 │
│  │                                                                          │
│  └── APAC Labels:                                                           │
│      ├── APAC Personal Data                                                │
│      │   └── Cross-border: Requires assessment                            │
│      └── China Personal Information                                        │
│          └── PIPL: Data localization required                             │
│          └── Cross-border: Security assessment required                   │
│                                                                             │
│  LABEL SCOPING:                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Label                  │ Published To                               │   │
│  ├────────────────────────┼────────────────────────────────────────────┤   │
│  │ Global labels          │ All users                                   │   │
│  │ EU Personal Data       │ EU-Employees security group                │   │
│  │ US Personal Information│ US-Employees security group                │   │
│  │ APAC Personal Data     │ APAC-Employees security group              │   │
│  │ China Personal Info    │ China-Employees security group             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Key Implementation Decisions

```
DECISION 1: Single vs Multiple Purview Instances?
─────────────────────────────────────────────────

CHOSEN: Single Global Instance

Rationale:
• Unified data catalog across all regions
• Single source of truth for auditors
• Simplified management
• Cross-region data lineage visibility

Data residency addressed by:
• Regional data stays in regional Azure subscriptions
• Purview metadata only (not actual data) is global
• DLP policies enforced at regional boundaries


DECISION 2: How to handle data subject requests efficiently?
────────────────────────────────────────────────────────────

CHOSEN: Purview Content Search + Automated Workflow

Implementation:
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  1. Custom Power Apps portal for DSR intake                                │
│  2. Power Automate triggers Purview Content Search                         │
│  3. Search scoped to subject's identity across all regions                 │
│  4. Results compiled into report                                            │
│  5. Privacy team reviews and executes (export/delete)                      │
│  6. Audit trail maintained in Compliance Manager                           │
│                                                                             │
│  SLA Met: 30-day response (GDPR) / 45-day response (CCPA)                 │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘


DECISION 3: How to prevent accidental cross-border transfers?
─────────────────────────────────────────────────────────────

CHOSEN: DLP + Conditional Access + Container Labels

Implementation:
• DLP Policy: Block sharing "EU Personal Data" outside EU tenant
• Conditional Access: EU sites only accessible from EU or approved countries
• Container Labels: EU SharePoint sites labeled, external sharing blocked
• Alerts: Any cross-border attempt generates Security Alert
```

### Results

```
COMPLIANCE OUTCOMES:
────────────────────

GDPR:
• Article 30 (Records of Processing): Automated via Purview catalog
• Article 17 (Right to Erasure): DSR process < 15 days average
• Article 33 (Breach Notification): Automated detection + alerting

CCPA:
• "Do Not Sell" requests: Tracked via custom SIT
• Access requests: Fulfilled within 20 days average

METRICS:
┌────────────────────────────────────────────────────────────────────────────┐
│ Metric                          │ Before       │ After        │ Change    │
├─────────────────────────────────┼──────────────┼──────────────┼───────────┤
│ DSR response time (avg)         │ 28 days      │ 12 days      │ -57%      │
│ Cross-border data incidents     │ 15/quarter   │ 2/quarter    │ -87%      │
│ Compliance audit prep           │ 6 weeks      │ 1 week       │ -83%      │
│ Privacy team headcount needed   │ 12 FTE       │ 8 FTE        │ -33%      │
│ Regulatory fines                │ €50,000/year │ €0           │ -100%     │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Key Takeaways

```
COMPLIANCE ARCHITECTURE PRINCIPLES:
───────────────────────────────────

1. UNIFIED VISIBILITY
   • Single pane of glass across clouds and on-premises
   • One classification taxonomy applied everywhere
   • Cross-platform data lineage

2. AUTOMATE EVERYTHING
   • Auto-labeling reduces human error
   • Automated DSR workflows meet SLAs
   • Continuous scanning vs. point-in-time audits

3. START WITH DISCOVERY
   • You can't protect what you can't see
   • Scan first, then apply policies
   • Address shadow data before it becomes a breach

4. LAYER YOUR CONTROLS
   • Labels for classification
   • DLP for enforcement
   • Encryption for protection
   • Audit logs for accountability

5. DESIGN FOR COMPLIANCE FROM DAY ONE
   • Easier to build compliant than to retrofit
   • Include compliance team in architecture decisions
   • Document decisions for auditors
```

---

*Back to [Chapter Overview](README.md)* | *Next Chapter: [Observability](../04-observability/README.md)*
