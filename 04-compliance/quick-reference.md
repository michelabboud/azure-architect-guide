# Compliance Quick Reference

## Sensitivity Labels

### Built-in Labels (Common Pattern)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SENSITIVITY LABEL HIERARCHY                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PUBLIC                    INTERNAL                 CONFIDENTIAL            │
│  ──────                    ────────                 ────────────            │
│  • No protection           • Header/footer          • Encryption            │
│  • External sharing OK     • No external share      • No external share     │
│  • No encryption           • No encryption          • Watermark             │
│                                                                              │
│  ┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐     │
│  │ Marketing       │      │ Internal Only   │      │ Confidential    │     │
│  │ materials       │      │                 │      │                 │     │
│  └─────────────────┘      └─────────────────┘      └────────┬────────┘     │
│                                                             │               │
│                                             ┌───────────────┼───────────┐   │
│                                             │               │           │   │
│                                             ▼               ▼           ▼   │
│                                      ┌───────────┐  ┌───────────┐ ┌───────┐│
│                                      │Confidential│  │Confidential│ │Highly ││
│                                      │- All Emp  │  │- Finance  │ │Conf.  ││
│                                      └───────────┘  └───────────┘ └───────┘│
│                                                                              │
│  SUBLABEL PERMISSIONS:                                                      │
│  • Confidential - All Employees: All authenticated users can view          │
│  • Confidential - Finance: Only Finance group can view                     │
│  • Highly Confidential: Named users only, no copy/print/forward            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Label Actions

| Label Setting | What It Does | Use Case |
|---------------|--------------|----------|
| Encryption | Encrypts file, controls access | Sensitive documents |
| Content marking | Adds header/footer/watermark | Visual classification |
| Auto-labeling | Applies label based on content | Automated classification |
| Site/Group settings | Configures SharePoint/Teams | Container protection |
| External sharing | Controls sharing outside org | Data loss prevention |

---

## DLP Policy Quick Reference

### Policy Components

```
DLP POLICY STRUCTURE:
─────────────────────

┌──────────────────────────────────────────────────────────────────────────┐
│                           DLP POLICY                                      │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  LOCATIONS (Where to apply):                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │
│  │ Exchange    │ │ SharePoint  │ │ OneDrive    │ │ Teams       │        │
│  │ (Email)     │ │ (Sites)     │ │ (Personal)  │ │ (Chat/Files)│        │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘        │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                        │
│  │ Endpoints   │ │ Power BI    │ │ On-premises │                        │
│  │ (Devices)   │ │             │ │ (File shares)│                       │
│  └─────────────┘ └─────────────┘ └─────────────┘                        │
│                                                                           │
│  CONDITIONS (What to detect):                                            │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ Sensitive Info Types:                                               │ │
│  │ • Credit Card Number (built-in)                                    │ │
│  │ • SSN (built-in)                                                   │ │
│  │ • Custom regex patterns                                            │ │
│  │ • Trainable classifiers (ML-based)                                 │ │
│  │                                                                     │ │
│  │ Content contains:                                                   │ │
│  │ • Sensitivity labels                                               │ │
│  │ • Keywords                                                         │ │
│  │ • Document properties                                              │ │
│  └────────────────────────────────────────────────────────────────────┘ │
│                                                                           │
│  ACTIONS (What to do):                                                   │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │
│  │ Block       │ │ Warn        │ │ Audit only  │ │ Notify      │        │
│  │ (Prevent)   │ │ (Override)  │ │ (Log)       │ │ (Alert)     │        │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘        │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

### Common DLP Rules

| Rule | Detects | Action | Priority |
|------|---------|--------|----------|
| Block credit cards external | Credit card + external recipient | Block + notify | High |
| Warn on SSN sharing | SSN in email/file | Warn, allow override | Medium |
| Audit PII | Any PII pattern | Audit only | Low |
| Block confidential external | Confidential label + external | Block | High |

---

## Sensitive Information Types

### Built-in Types (Most Used)

| Category | Examples | Region |
|----------|----------|--------|
| Financial | Credit Card, Bank Account, SWIFT | Global |
| Health | Medical terms, Drug names | Global |
| PII - US | SSN, Driver's License, Passport | US |
| PII - EU | National ID, IBAN | EU |
| Credentials | Azure keys, AWS keys, passwords | Global |

### Custom Sensitive Info Type

```xml
<!-- Example: Employee ID pattern -->
<Entity id="custom-employee-id"
        patternsProximity="300"
        recommendedConfidence="75">
  <Pattern confidenceLevel="85">
    <IdMatch idRef="Regex_employee_id"/>
    <Match idRef="Keyword_employee"/>
  </Pattern>
</Entity>

<Regex id="Regex_employee_id">EMP-[0-9]{6}</Regex>

<Keyword id="Keyword_employee">
  <Group matchStyle="word">
    <Term>employee</Term>
    <Term>staff</Term>
    <Term>worker</Term>
  </Group>
</Keyword>
```

---

## Compliance Manager

### Compliance Score

```
COMPLIANCE SCORE CALCULATION:
─────────────────────────────

Score = (Points Achieved / Total Points) × 100

┌────────────────────────────────────────────────────────────────────────┐
│                    COMPLIANCE SCORE: 72%                                │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  BY CONTROL FAMILY:                                                    │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ Access Control          ████████████████████░░░░░░  78%           │ │
│  │ Data Protection         ██████████████████░░░░░░░░  70%           │ │
│  │ Incident Response       ████████████████████████░░  85%           │ │
│  │ Risk Management         ████████████████░░░░░░░░░░  65%           │ │
│  │ Audit & Accountability  ██████████████████████░░░░  80%           │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
│  IMPROVEMENT ACTIONS:                                                  │
│  ┌───────────────────────────────────────────────────────────────────┐ │
│  │ 1. Enable MFA for all users            +5 pts   [In Progress]    │ │
│  │ 2. Configure DLP for PII               +8 pts   [Not Started]    │ │
│  │ 3. Enable audit logging                +3 pts   [Completed]      │ │
│  │ 4. Implement data classification       +10 pts  [Not Started]    │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### Assessment Templates

| Template | Regulation | Regions |
|----------|-----------|---------|
| ISO 27001 | Information Security | Global |
| SOC 2 | Service Organization Controls | US |
| GDPR | Data Protection | EU |
| HIPAA | Healthcare | US |
| PCI DSS | Payment Card | Global |
| NIST 800-53 | Federal Security | US |
| FedRAMP | Federal Cloud | US |

---

## Audit Log Search

### Common Searches

```powershell
# PowerShell: Search audit logs
Search-UnifiedAuditLog -StartDate (Get-Date).AddDays(-7) `
  -EndDate (Get-Date) `
  -RecordType SharePointFileOperation `
  -Operations FileDownloaded `
  -ResultSize 5000

# Filter for external sharing
Search-UnifiedAuditLog -StartDate (Get-Date).AddDays(-7) `
  -EndDate (Get-Date) `
  -RecordType SharingSet `
  -ResultSize 5000 |
  Where-Object { $_.AuditData -like "*external*" }
```

### KQL in Log Analytics

```kusto
// Files with sensitivity labels accessed
OfficeActivity
| where TimeGenerated > ago(7d)
| where Operation == "FileAccessed"
| where SensitivityLabel != ""
| summarize AccessCount = count() by UserId, SensitivityLabel
| order by AccessCount desc

// DLP policy matches
DLPRuleMatch
| where TimeGenerated > ago(24h)
| summarize MatchCount = count() by PolicyName, RuleName
| order by MatchCount desc

// External sharing events
OfficeActivity
| where TimeGenerated > ago(7d)
| where Operation == "SharingSet"
| where TargetUserOrGroupType == "Guest"
| project TimeGenerated, UserId, ObjectId, TargetUserOrGroupName
```

---

## Quick Decision Guide

### When to Use What

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLIANCE TOOL SELECTION                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  "I need to..."                        Use this:                            │
│  ──────────────                        ─────────                            │
│                                                                              │
│  Classify documents                    → Sensitivity Labels                 │
│  Encrypt sensitive files               → Sensitivity Labels + Encryption   │
│  Prevent data leakage                  → DLP Policies                       │
│  Detect sensitive data in storage      → Purview Data Map                   │
│  Track data lineage                    → Purview Data Catalog               │
│  Meet regulatory requirements          → Compliance Manager                 │
│  Investigate user activity             → Audit Log Search                   │
│  Detect insider threats                → Insider Risk Management            │
│  Retain/delete data by policy          → Retention Policies                 │
│  Legal hold for investigation          → eDiscovery                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## AWS to Azure Compliance Commands

| Action | AWS | Azure/M365 |
|--------|-----|------------|
| Scan for PII | Macie job | Purview scan |
| Create DLP policy | (Custom) | Compliance portal → DLP |
| View compliance status | Security Hub | Compliance Manager |
| Search audit logs | CloudTrail | Audit log search |
| Classify data | Macie | Sensitivity labels |

---

*Deep Dive: [Purview Overview](01-purview-overview.md)* | *Back to [Chapter Overview](README.md)*
