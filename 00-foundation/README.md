# Chapter 00: Foundation & Learning Path

## Purpose of This Guide

This guide is designed for **AWS-experienced Cloud Architects** preparing for Azure-focused roles that require:

- Azure reference architecture ownership
- AI-in-the-enterprise (Azure AI Foundry + Copilot Studio)
- Identity-first security (Entra ID + Zero Trust)
- Security & compliance standardization (Defender + Purview)
- Infrastructure as Code (Terraform/Bicep)
- Observability & FinOps

---

## How to Use This Guide

### Quick Sync (5-10 minutes per topic)
Each chapter contains a **`quick-reference.md`** file with:
- One-page cheat sheets
- AWS-to-Azure mapping tables
- Key commands and configurations
- Decision flowcharts

### Deep Dive (30-60 minutes per topic)
Each chapter contains **`deep-dive.md`** files with:
- Comprehensive explanations
- Architecture diagrams
- Three-tier examples (Beginner → Intermediate → Expert)
- Case studies with decision debates

---

## Recommended Learning Path

Based on the job requirements and Azure's architecture model, follow this sequence:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        LEARNING PATH                                     │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  WEEK 1-2: IDENTITY FOUNDATION                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 01: Identity                                             │   │
│  │  • Entra ID fundamentals                                          │   │
│  │  • Conditional Access policies                                    │   │
│  │  • Governance & lifecycle                                         │   │
│  │  WHY FIRST: Everything in Azure depends on identity               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  WEEK 3-4: SECURITY LAYER                                               │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 02: Security                                             │   │
│  │  • Zero Trust architecture                                        │   │
│  │  • Microsoft Defender for Cloud                                   │   │
│  │  • Network security patterns                                      │   │
│  │  WHY SECOND: Security builds on identity foundation               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  WEEK 5-6: COMPLIANCE & DATA GOVERNANCE                                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 03: Compliance                                           │   │
│  │  • Microsoft Purview suite                                        │   │
│  │  • DLP policies                                                   │   │
│  │  • Information Protection                                         │   │
│  │  WHY THIRD: Compliance enforces security/identity policies        │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  WEEK 7-8: OBSERVABILITY                                                │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 04: Observability                                        │   │
│  │  • Azure Monitor                                                  │   │
│  │  • Log Analytics (KQL)                                            │   │
│  │  • Managed Grafana                                                │   │
│  │  WHY HERE: Need visibility into identity/security/compliance      │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  WEEK 9-10: AI PLATFORM                                                 │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 05: AI Platform                                          │   │
│  │  • Azure AI Foundry                                               │   │
│  │  • Copilot Studio                                                 │   │
│  │  • M365 integration                                               │   │
│  │  WHY HERE: AI requires identity, security, compliance foundation  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                           │
│  WEEK 11-12: INFRASTRUCTURE & FINOPS                                    │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 06: Infrastructure as Code                               │   │
│  │  Chapter 07: FinOps                                               │   │
│  │  • Bicep/Terraform patterns                                       │   │
│  │  • Cost management                                                │   │
│  │  WHY LAST: Implement everything above as code with cost awareness │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## AWS to Azure Mental Model Shift

### The Big Difference: Identity-First vs Resource-First

```
AWS Mental Model:                    Azure Mental Model:
─────────────────                    ──────────────────
     Resources                            Identity
         │                                    │
    IAM Policies                         Entra ID
         │                                    │
    Security Groups                   Conditional Access
         │                                    │
      Identity                            Resources
                                              │
                                          RBAC + Policy

AWS: "What can this resource do?"    Azure: "Who are you, and should
                                            you even be here?"
```

### Key Mindset Changes

| AWS Thinking | Azure Thinking |
|--------------|----------------|
| IAM is for permissions | Entra ID is the security perimeter |
| Security groups protect resources | Identity + Conditional Access protect everything |
| GuardDuty detects threats | Defender is a comprehensive security platform |
| Macie classifies data | Purview governs the entire data estate |
| CloudWatch monitors | Azure Monitor + Log Analytics is the telemetry backbone |

---

## Repository Structure

```
azure-architect-guide/
├── README.md                          # Main navigation
├── 00-foundation/
│   ├── README.md                      # This file
│   ├── aws-azure-mapping.md           # Complete service mapping
│   └── azure-fundamentals.md          # Core Azure concepts
│
├── 01-identity/
│   ├── README.md                      # Chapter overview
│   ├── quick-reference.md             # Cheat sheet
│   ├── 01-entra-id.md                 # Deep dive
│   ├── 02-conditional-access.md       # Deep dive
│   ├── 03-governance.md               # Deep dive
│   └── case-studies.md                # Real scenarios
│
├── 02-security/
│   ├── README.md
│   ├── quick-reference.md
│   ├── 01-zero-trust.md
│   ├── 02-defender-for-cloud.md
│   ├── 03-network-security.md
│   └── case-studies.md
│
├── 03-compliance/
│   ├── README.md
│   ├── quick-reference.md
│   ├── 01-purview-overview.md
│   ├── 02-dlp-policies.md
│   ├── 03-information-protection.md
│   └── case-studies.md
│
├── 04-observability/
│   ├── README.md
│   ├── quick-reference.md
│   ├── 01-azure-monitor.md
│   ├── 02-log-analytics-kql.md
│   ├── 03-grafana-dashboards.md
│   └── case-studies.md
│
├── 05-ai-platform/
│   ├── README.md
│   ├── quick-reference.md
│   ├── 01-azure-ai-foundry.md
│   ├── 02-copilot-studio.md
│   ├── 03-m365-integration.md
│   └── case-studies.md
│
├── 06-infrastructure-as-code/
│   ├── README.md
│   ├── quick-reference.md
│   ├── 01-bicep.md
│   ├── 02-terraform-azure.md
│   ├── 03-arm-templates.md
│   └── case-studies.md
│
├── 07-finops/
│   ├── README.md
│   ├── quick-reference.md
│   ├── 01-cost-management.md
│   ├── 02-optimization-strategies.md
│   └── case-studies.md
│
├── 08-case-studies/
│   ├── README.md
│   ├── enterprise-identity-migration.md
│   ├── multi-cloud-security.md
│   ├── ai-platform-deployment.md
│   └── finops-transformation.md
│
├── 09-certifications/
│   ├── az-305-study-guide.md
│   ├── az-104-study-guide.md
│   └── sc-100-study-guide.md
│
└── quick-reference/
    ├── aws-azure-services.md
    ├── cli-commands.md
    ├── decision-trees.md
    └── interview-questions.md
```

---

## Chapter Navigation

| Chapter | Topic | Quick Sync | Deep Dive |
|---------|-------|------------|-----------|
| 00 | Foundation | [AWS-Azure Mapping](aws-azure-mapping.md) | [Azure Fundamentals](azure-fundamentals.md) |
| 01 | Identity | [Quick Reference](../01-identity/quick-reference.md) | [Entra ID](../01-identity/01-entra-id.md) |
| 02 | Security | [Quick Reference](../02-security/quick-reference.md) | [Zero Trust](../02-security/01-zero-trust.md) |
| 03 | Compliance | [Quick Reference](../03-compliance/quick-reference.md) | [Purview](../03-compliance/01-purview-overview.md) |
| 04 | Observability | [Quick Reference](../04-observability/quick-reference.md) | [Azure Monitor](../04-observability/01-azure-monitor.md) |
| 05 | AI Platform | [Quick Reference](../05-ai-platform/quick-reference.md) | [AI Foundry](../05-ai-platform/01-azure-ai-foundry.md) |
| 06 | IaC | [Quick Reference](../06-infrastructure-as-code/quick-reference.md) | [Bicep](../06-infrastructure-as-code/01-bicep.md) |
| 07 | FinOps | [Quick Reference](../07-finops/quick-reference.md) | [Cost Management](../07-finops/01-cost-management.md) |
| 08 | Case Studies | — | [All Studies](../08-case-studies/README.md) |

---

## Skills Coverage Mapping

Here's how each chapter maps to key Cloud Architect skills:

| Skill Area | Primary Chapter | Supporting Chapters |
|-----------------|-----------------|---------------------|
| Azure reference architecture | 06, 08 | All |
| AI-in-the-enterprise (Foundry/Copilot) | 05 | 01, 02, 03 |
| Identity-first security (Entra/Zero Trust) | 01, 02 | 03 |
| Security + compliance (Defender/Purview) | 02, 03 | 04 |
| IaC (Terraform/Bicep/ARM) | 06 | All |
| Observability (Monitor/Grafana) | 04 | 07 |
| FinOps | 07 | 04, 06 |

---

*Next: [AWS-Azure Service Mapping](aws-azure-mapping.md)*
