# Azure Cloud Architect Preparation Guide

A comprehensive guide for AWS architects transitioning to Azure, designed for professionals preparing for Cloud Architect roles. This guide provides AWS-to-Azure comparisons throughout, with quick sync and deep dive content for each topic.

## Who This Guide Is For

- **AWS Architects** learning Azure for multi-cloud expertise
- **Cloud Architects** preparing for Azure-focused roles
- **Solution Architects** expanding their cloud platform knowledge
- **Engineers** pursuing Azure certifications (AZ-305, AZ-104)

## Guide Structure

Each chapter includes:
- **Quick Reference** - 5-10 minute sync on essential commands and concepts
- **Deep Dive Content** - 30-60 minute comprehensive coverage
- **AWS Comparisons** - Direct service mappings and terminology differences
- **Case Studies** - Real-world scenarios with architectural decision debates
- **Extra Resources** - Official docs, videos, hands-on labs

---

## Table of Contents

### Foundation & Identity

| Chapter | Topic | Description |
|---------|-------|-------------|
| [00](00-foundation/README.md) | **Azure Foundation** | Subscriptions, resource groups, management hierarchy |
| [01](01-identity/README.md) | **Identity & Access** | Azure AD, RBAC, conditional access, hybrid identity |
| [02](02-security/README.md) | **Security** | Microsoft Defender, Sentinel, Key Vault, encryption |
| [03](03-compliance/README.md) | **Compliance & Governance** | Azure Policy, Blueprints, regulatory compliance |

### Operations & Platform

| Chapter | Topic | Description |
|---------|-------|-------------|
| [04](04-observability/README.md) | **Observability** | Azure Monitor, Log Analytics, Application Insights |
| [05](05-ai-platform/README.md) | **AI Platform** | Azure OpenAI, Cognitive Services, Machine Learning |
| [06](06-infrastructure-as-code/README.md) | **Infrastructure as Code** | Bicep, Terraform, ARM templates |
| [07](07-finops/README.md) | **FinOps & Cost Management** | Cost analysis, budgets, reservations, optimization |

### Architecture & Patterns

| Chapter | Topic | Description |
|---------|-------|-------------|
| [08](08-case-studies/README.md) | **Architecture Case Studies** | E-commerce, data platform, financial services, IoT |
| [09-Design](09-design-patterns/README.md) | **Cloud Design Patterns** | Reliability, messaging, data, deployment patterns |
| [09-Cert](09-certifications/README.md) | **Certifications** | AZ-305, AZ-104 study guides, exam preparation |

### Networking & Security

| Chapter | Topic | Description |
|---------|-------|-------------|
| [10](10-networking/README.md) | **Networking** | VNets, peering, VPN Gateway, ExpressRoute |
| [11](11-network-security/README.md) | **Network Security** | NSGs, Azure Firewall, DDoS, Private Link |
| [12](12-load-balancing-cdn/README.md) | **Load Balancing & CDN** | Load Balancer, App Gateway, Front Door, Traffic Manager |

### Application Services

| Chapter | Topic | Description |
|---------|-------|-------------|
| [13](13-containers-aks/README.md) | **Containers & AKS** | ACI, AKS, Container Apps, service mesh |
| [14](14-api-integration/README.md) | **API & Integration** | APIM, Service Bus, Event Grid, Logic Apps |
| [15](15-databases/README.md) | **Databases** | Azure SQL, Cosmos DB, PostgreSQL, Redis |

### Advanced Architecture

| Chapter | Topic | Description |
|---------|-------|-------------|
| [16](16-architecture-styles/README.md) | **Architecture Styles** | N-Tier, Microservices, Event-Driven, Big Data |
| [17](17-anti-patterns/README.md) | **Anti-Patterns** | Busy Database, Chatty I/O, Retry Storm, more |
| [18](18-cloud-adoption-framework/README.md) | **Cloud Adoption Framework** | CAF phases, Landing Zones, governance |
| [19](19-ai-ml-comprehensive/README.md) | **AI & Machine Learning** | AI agents, RAG, Azure OpenAI, Responsible AI |

---

## Quick Start Paths

### Path 1: Fast Track (2-3 weeks)

For experienced AWS architects needing rapid Azure knowledge:

```
Week 1: Foundation
├── 00-foundation (Quick Reference)
├── 01-identity (Azure AD focus)
├── 10-networking (VNet vs VPC)
└── 12-load-balancing

Week 2: Services
├── 15-databases (SQL, Cosmos DB)
├── 13-containers-aks
├── 14-api-integration
└── 06-infrastructure-as-code (Bicep)

Week 3: Architecture
├── 09-design-patterns
├── 16-architecture-styles
├── 17-anti-patterns
├── 08-case-studies
└── 09-certifications (AZ-305 prep)
```

### Path 2: Comprehensive (6-8 weeks)

For thorough coverage with hands-on practice:

```
Weeks 1-2: Foundation & Governance
├── All foundation chapters (00-03)
└── Labs: Create landing zone, configure policies

Weeks 3-4: Core Services
├── Networking (10-12)
├── Compute & Containers (13)
└── Labs: Hub-spoke network, AKS deployment

Weeks 5-6: Data & Integration
├── Databases (15)
├── API & Integration (14)
└── Labs: Cosmos DB, Event Grid patterns

Weeks 7-8: Architecture & AI
├── Architecture Styles (16)
├── Anti-Patterns (17)
├── Cloud Adoption Framework (18)
├── AI & Machine Learning (19)
└── Case Studies (08)

Weeks 9-10: Certification Prep
├── Design Patterns (09)
├── Certifications (09-cert)
└── Practice exams
```

---

## AWS to Azure Quick Reference

| Category | AWS | Azure |
|----------|-----|-------|
| **Compute** | EC2 | Virtual Machines |
| | Lambda | Azure Functions |
| | ECS/EKS | ACI/AKS |
| **Storage** | S3 | Blob Storage |
| | EBS | Managed Disks |
| | EFS | Azure Files |
| **Database** | RDS | Azure SQL |
| | DynamoDB | Cosmos DB |
| | ElastiCache | Azure Cache for Redis |
| **Networking** | VPC | Virtual Network |
| | Direct Connect | ExpressRoute |
| | Route 53 | Azure DNS |
| **Identity** | IAM | Azure AD + RBAC |
| | Cognito | Azure AD B2C |
| **Security** | GuardDuty | Microsoft Defender |
| | WAF | Azure WAF |
| | KMS | Key Vault |
| **Monitoring** | CloudWatch | Azure Monitor |
| | X-Ray | Application Insights |

---

## Key Learning Resources

### Official Documentation
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)
- [Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
- [AWS to Azure Comparison](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/)

### Certifications
- **AZ-305**: Azure Solutions Architect Expert
- **AZ-104**: Azure Administrator Associate
- **AZ-500**: Azure Security Engineer Associate
- **AZ-700**: Azure Network Engineer Associate

### Hands-On
- [Azure Free Account](https://azure.microsoft.com/free/) - $200 credit
- [Microsoft Learn Sandboxes](https://learn.microsoft.com/training/) - Free environments
- [Azure Quickstart Templates](https://github.com/Azure/azure-quickstart-templates)

---

## Contributing

Contributions welcome! Please submit issues or pull requests for:
- Additional case studies
- AWS comparison clarifications
- New design patterns
- Updated best practices

## License

MIT License - Use freely for learning and career development.

---

*Last Updated: January 2026*
