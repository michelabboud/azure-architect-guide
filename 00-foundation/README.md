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

## Chapter Contents

| Chapter | Topic | Quick Sync | Deep Dive |
|---------|-------|------------|-----------|
| 00 | Foundation | [AWS-Azure Mapping](aws-azure-mapping.md) | [Azure Fundamentals](azure-fundamentals.md) |
| 01 | Identity | [Quick Reference](../01-cloud-adoption-framework/quick-reference.md) | [Entra ID](../02-identity/README.md) |
| 02 | Security | [Quick Reference](../03-security/quick-reference.md) | [Zero Trust](../03-security/README.md) |
| 03 | Compliance | [Quick Reference](../04-compliance/quick-reference.md) | [Purview](../04-compliance/README.md) |
| 04 | Observability | [Quick Reference](../16-observability/quick-reference.md) | [Azure Monitor](../16-observability/README.md) |
| 05 | AI Platform | [Quick Reference](../14-ai-platform/quick-reference.md) | [AI Foundry](../14-ai-platform/README.md) |
| 06 | IaC | [Quick Reference](../17-infrastructure-as-code/quick-reference.md) | [Bicep](../17-infrastructure-as-code/README.md) |
| 07 | FinOps | [Quick Reference](../18-finops/quick-reference.md) | [Cost Management](../18-finops/README.md) |
| 08 | Case Studies | — | [All Studies](../19-case-studies/README.md) |

### Additional Resources

| File | Description | Time |
|------|-------------|------|
| [Quick Reference](quick-reference.md) | AWS-Azure service mapping, terminology, CLI commands | 5-10 min |
| [AWS-Azure Mapping](aws-azure-mapping.md) | Complete service comparison tables | 15 min |
| [Azure Fundamentals](azure-fundamentals.md) | Core concepts for AWS architects | 30 min |
| [Extra Resources](extra-resources.md) | Official docs, videos, labs | Reference |

---

## How to Use This Guide

### Quick Sync (5-10 minutes per topic)
Each chapter contains a **`quick-reference.md`** file with:
- One-page cheat sheets
- AWS-to-Azure mapping tables
- Key commands and configurations
- Decision flowcharts

### Deep Dive (30-60 minutes per topic)
Each chapter contains detailed **topic files** with:
- Comprehensive explanations
- Architecture diagrams
- Bicep/Terraform code examples
- Case studies with architectural decisions

---

## Recommended Learning Path

Based on the job requirements and Azure's architecture model, follow this sequence:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     COMPREHENSIVE LEARNING PATH                         │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  WEEK 1-2: FOUNDATION & STRATEGY                                        │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 00: Foundation (this chapter)                           │   │
│  │  Chapter 01: Cloud Adoption Framework                            │   │
│  │  • Azure fundamentals, resource hierarchy                        │   │
│  │  • CAF phases, Landing Zones                                     │   │
│  │  WHY FIRST: Understand Azure's organizational model              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  WEEK 3-4: IDENTITY & SECURITY                                          │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 02: Identity (Entra ID, RBAC)                           │   │
│  │  Chapter 03: Security (Zero Trust, Defender)                     │   │
│  │  Chapter 04: Compliance (Purview, DLP)                           │   │
│  │  WHY HERE: Identity-first approach is core to Azure              │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  WEEK 5-6: NETWORKING                                                   │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 05: Networking (VNets, VPN, ExpressRoute)               │   │
│  │  Chapter 06: Network Security (NSG, Firewall, Private Link)      │   │
│  │  Chapter 07: Load Balancing & CDN (App GW, Front Door)           │   │
│  │  WHY HERE: Network foundation for all workloads                  │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  WEEK 7-8: COMPUTE, DATA & INTEGRATION                                  │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 08: Containers & AKS                                    │   │
│  │  Chapter 09: Databases (SQL, Cosmos DB)                          │   │
│  │  Chapter 10: API & Integration (APIM, Service Bus)               │   │
│  │  WHY HERE: Core services for application workloads               │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  WEEK 9-10: ARCHITECTURE & PATTERNS                                     │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 11: Architecture Styles (N-Tier, Microservices)         │   │
│  │  Chapter 12: Design Patterns (Reliability, Messaging)            │   │
│  │  Chapter 13: Anti-Patterns (Performance issues)                  │   │
│  │  WHY HERE: Apply knowledge to real architectures                 │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                              ↓                                          │
│  WEEK 11-12: AI, OPERATIONS & CERTIFICATION                             │
│  ┌──────────────────────────────────────────────────────────────────┐   │
│  │  Chapter 14-15: AI Platform (Azure OpenAI, ML)                   │   │
│  │  Chapter 16: Observability (Monitor, Log Analytics)              │   │
│  │  Chapter 17: Infrastructure as Code (Bicep, Terraform)           │   │
│  │  Chapter 18: FinOps (Cost optimization)                          │   │
│  │  Chapter 19-20: Case Studies & Certifications                    │   │
│  │  WHY LAST: Operational excellence and exam preparation           │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
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
├── README.md                          # Main navigation & quick start
│
├── FOUNDATION & STRATEGY
│   ├── 00-foundation/                 # This chapter - fundamentals
│   └── 01-cloud-adoption-framework/   # CAF, Landing Zones
│
├── IDENTITY & SECURITY
│   ├── 02-identity/                   # Entra ID, RBAC, Conditional Access
│   ├── 03-security/                   # Zero Trust, Defender, Key Vault
│   └── 04-compliance/                 # Purview, DLP, Information Protection
│
├── NETWORKING
│   ├── 05-networking/                 # VNets, VPN, ExpressRoute, DNS
│   ├── 06-network-security/           # NSG, Azure Firewall, Private Link
│   └── 07-load-balancing-cdn/         # App Gateway, Load Balancer, Front Door
│
├── COMPUTE & DATA
│   ├── 08-containers-aks/             # ACI, AKS, Container Apps
│   ├── 09-databases/                  # SQL, Cosmos DB, PostgreSQL, Redis
│   └── 10-api-integration/            # APIM, Service Bus, Event Grid
│
├── ARCHITECTURE & PATTERNS
│   ├── 11-architecture-styles/        # N-Tier, Microservices, Event-Driven
│   ├── 12-design-patterns/            # Reliability, messaging, deployment
│   └── 13-anti-patterns/              # Busy Database, Chatty I/O, etc.
│
├── AI & MACHINE LEARNING
│   ├── 14-ai-platform/                # Azure OpenAI, Cognitive Services
│   └── 15-ai-ml-comprehensive/        # AI agents, RAG, Responsible AI
│
├── OPERATIONS
│   ├── 16-observability/              # Azure Monitor, Log Analytics
│   ├── 17-infrastructure-as-code/     # Bicep, Terraform, ARM
│   └── 18-finops/                     # Cost management, optimization
│
└── LEARNING & CERTIFICATION
    ├── 19-case-studies/               # E-commerce, data platform, IoT
    └── 20-certifications/             # AZ-305, AZ-104 study guides
```

Each chapter contains:
- `README.md` - Chapter overview and navigation
- `quick-reference.md` - 5-10 minute cheat sheet
- Topic deep-dive files (e.g., `01-topic.md`, `02-topic.md`)
- `case-studies.md` - Real-world scenarios
- `extra-resources.md` - Official docs, videos, labs

---

## Chapter Navigation

### Foundation & Strategy
| Chapter | Topic | Quick Sync | Overview |
|---------|-------|------------|----------|
| 00 | Foundation | [Quick Reference](quick-reference.md) | This chapter |
| 01 | Cloud Adoption Framework | [Quick Reference](../01-cloud-adoption-framework/quick-reference.md) | [Overview](../01-cloud-adoption-framework/README.md) |

### Identity & Security
| Chapter | Topic | Quick Sync | Overview |
|---------|-------|------------|----------|
| 02 | Identity & Access | [Quick Reference](../02-identity/quick-reference.md) | [Overview](../02-identity/README.md) |
| 03 | Security | [Quick Reference](../03-security/quick-reference.md) | [Overview](../03-security/README.md) |
| 04 | Compliance | [Quick Reference](../04-compliance/quick-reference.md) | [Overview](../04-compliance/README.md) |

### Networking
| Chapter | Topic | Quick Sync | Overview |
|---------|-------|------------|----------|
| 05 | Networking | [Quick Reference](../05-networking/quick-reference.md) | [Overview](../05-networking/README.md) |
| 06 | Network Security | [Quick Reference](../06-network-security/quick-reference.md) | [Overview](../06-network-security/README.md) |
| 07 | Load Balancing & CDN | [Quick Reference](../07-load-balancing-cdn/quick-reference.md) | [Overview](../07-load-balancing-cdn/README.md) |

### Compute & Data
| Chapter | Topic | Quick Sync | Overview |
|---------|-------|------------|----------|
| 08 | Containers & AKS | [Quick Reference](../08-containers-aks/quick-reference.md) | [Overview](../08-containers-aks/README.md) |
| 09 | Databases | [Quick Reference](../09-databases/quick-reference.md) | [Overview](../09-databases/README.md) |
| 10 | API & Integration | [Quick Reference](../10-api-integration/quick-reference.md) | [Overview](../10-api-integration/README.md) |

### Architecture & Patterns
| Chapter | Topic | Quick Sync | Overview |
|---------|-------|------------|----------|
| 11 | Architecture Styles | [Quick Reference](../11-architecture-styles/quick-reference.md) | [Overview](../11-architecture-styles/README.md) |
| 12 | Design Patterns | [Quick Reference](../12-design-patterns/quick-reference.md) | [Overview](../12-design-patterns/README.md) |
| 13 | Anti-Patterns | [Quick Reference](../13-anti-patterns/quick-reference.md) | [Overview](../13-anti-patterns/README.md) |

### AI & Operations
| Chapter | Topic | Quick Sync | Overview |
|---------|-------|------------|----------|
| 14 | AI Platform | [Quick Reference](../14-ai-platform/quick-reference.md) | [Overview](../14-ai-platform/README.md) |
| 15 | AI & ML Comprehensive | [Quick Reference](../15-ai-ml-comprehensive/quick-reference.md) | [Overview](../15-ai-ml-comprehensive/README.md) |
| 16 | Observability | [Quick Reference](../16-observability/quick-reference.md) | [Overview](../16-observability/README.md) |
| 17 | Infrastructure as Code | [Quick Reference](../17-infrastructure-as-code/quick-reference.md) | [Overview](../17-infrastructure-as-code/README.md) |
| 18 | FinOps | [Quick Reference](../18-finops/quick-reference.md) | [Overview](../18-finops/README.md) |

### Learning & Certification
| Chapter | Topic | Quick Sync | Overview |
|---------|-------|------------|----------|
| 19 | Case Studies | — | [Overview](../19-case-studies/README.md) |
| 20 | Certifications | [Quick Reference](../20-certifications/quick-reference.md) | [Overview](../20-certifications/README.md) |

---

## Skills Coverage Mapping

Here's how each chapter maps to key Cloud Architect skills:

| Skill Area | Primary Chapters | Supporting Chapters |
|------------|------------------|---------------------|
| Azure reference architecture | 11, 12, 19 | All |
| AI-in-the-enterprise (Foundry/Copilot) | 14, 15 | 02, 03, 04 |
| Identity-first security (Entra/Zero Trust) | 02, 03 | 04, 06 |
| Security + compliance (Defender/Purview) | 03, 04 | 06, 16 |
| Networking (VNets, Firewall, LB) | 05, 06, 07 | 08 |
| Containers & Kubernetes | 08 | 07, 09, 10 |
| IaC (Terraform/Bicep/ARM) | 17 | All |
| Observability (Monitor/Grafana) | 16 | 18 |
| FinOps | 18 | 16, 17 |

---

*Next: [Quick Reference](quick-reference.md)* | *[AWS-Azure Service Mapping](aws-azure-mapping.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
