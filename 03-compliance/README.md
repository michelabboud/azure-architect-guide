# Chapter 03: Compliance - Microsoft Purview

## Why Compliance After Security?

Compliance builds on the security foundation. While security protects resources, compliance ensures you're meeting regulatory and organizational requirements for data governance.

```
Chapter 01: Identity      Chapter 02: Security      Chapter 03: Compliance
─────────────────────     ──────────────────────    ─────────────────────────

"Who are you?"            "How do we protect?"      "Are we following rules?"

Entra ID                  Defender for Cloud        Purview
Conditional Access   →    Zero Trust           →    Data Classification
Governance                Network Security          DLP Policies
                                                    Information Protection
```

## AWS to Azure Compliance Mapping

| AWS Service | Azure Equivalent | Key Difference |
|-------------|------------------|----------------|
| Macie | Purview Data Map | Purview is broader data governance |
| Config Rules | Azure Policy | Policy has more effects (deny, modify) |
| Audit Manager | Purview Compliance Manager | Integrated compliance scoring |
| Lake Formation | Purview Data Catalog | Unified data governance |
| Security Hub (compliance) | Defender for Cloud + Purview | Split between security and data |
| CloudTrail | Activity Log + Purview Audit | Audit spans M365 + Azure |

## Microsoft Purview Suite

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MICROSOFT PURVIEW                                    │
│                    (Unified Data Governance)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DATA SECURITY                         DATA GOVERNANCE                      │
│  ─────────────                         ───────────────                      │
│  ┌─────────────────────────┐          ┌─────────────────────────┐          │
│  │ Information Protection  │          │ Data Catalog            │          │
│  │ • Sensitivity labels    │          │ • Asset discovery       │          │
│  │ • Encryption            │          │ • Business glossary     │          │
│  │ • Rights management     │          │ • Data lineage          │          │
│  └─────────────────────────┘          └─────────────────────────┘          │
│  ┌─────────────────────────┐          ┌─────────────────────────┐          │
│  │ Data Loss Prevention    │          │ Data Map                │          │
│  │ • Policy enforcement    │          │ • Automated scanning    │          │
│  │ • Block/warn/audit      │          │ • Classification        │          │
│  │ • Endpoint DLP          │          │ • Sensitive data types  │          │
│  └─────────────────────────┘          └─────────────────────────┘          │
│  ┌─────────────────────────┐          ┌─────────────────────────┐          │
│  │ Insider Risk Management │          │ Data Estate Insights    │          │
│  │ • Behavior analytics    │          │ • Health scores         │          │
│  │ • Policy violations     │          │ • Data quality          │          │
│  └─────────────────────────┘          └─────────────────────────┘          │
│                                                                              │
│  RISK & COMPLIANCE                                                          │
│  ─────────────────                                                          │
│  ┌─────────────────────────┐          ┌─────────────────────────┐          │
│  │ Compliance Manager      │          │ Audit                   │          │
│  │ • Assessment templates  │          │ • Search & investigate  │          │
│  │ • Compliance score      │          │ • Retention policies    │          │
│  │ • Improvement actions   │          │ • eDiscovery            │          │
│  └─────────────────────────┘          └─────────────────────────┘          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Chapter Contents

| Document | Purpose | Time |
|----------|---------|------|
| [Quick Reference](quick-reference.md) | Compliance cheat sheet | 5 min |
| [01 - Purview Overview](01-purview-overview.md) | Data governance platform | 25 min |
| [02 - DLP Policies](02-dlp-policies.md) | Data Loss Prevention | 25 min |
| [03 - Information Protection](03-information-protection.md) | Labels and encryption | 25 min |
| [Case Studies](case-studies.md) | Compliance scenarios | 20 min |

## Job Relevance: NICE Cloud Architect

This chapter addresses:

✅ **Security + Compliance (Defender/Purview)**
- Configure data classification
- Implement DLP policies
- Design information protection strategy

✅ **Data Governance**
- Understand data estate
- Implement data lineage
- Meet regulatory requirements

---

*Start with: [Quick Reference](quick-reference.md)*
