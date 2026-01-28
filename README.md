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

### Foundation & Strategy

| Chapter | Topic | Description |
|---------|-------|-------------|
| [00](00-foundation/README.md) | **Azure Foundation** | Subscriptions, resource groups, management hierarchy |
| [01](01-cloud-adoption-framework/README.md) | **Cloud Adoption Framework** | CAF phases, Landing Zones, governance strategy |

### Identity & Security

| Chapter | Topic | Description |
|---------|-------|-------------|
| [02](02-identity/README.md) | **Identity & Access** | Entra ID, RBAC, conditional access, hybrid identity |
| [03](03-security/README.md) | **Security** | Microsoft Defender, Sentinel, Key Vault, encryption |
| [04](04-compliance/README.md) | **Compliance & Governance** | Azure Policy, Blueprints, regulatory compliance |

### Networking

| Chapter | Topic | Description |
|---------|-------|-------------|
| [05](05-networking/README.md) | **Networking** | VNets, peering, VPN Gateway, ExpressRoute |
| [06](06-network-security/README.md) | **Network Security** | NSGs, Azure Firewall, DDoS, Private Link |
| [07](07-load-balancing-cdn/README.md) | **Load Balancing & CDN** | Load Balancer, App Gateway, Front Door, Traffic Manager |

### Compute & Containers

| Chapter | Topic | Description |
|---------|-------|-------------|
| [08](08-containers-aks/README.md) | **Containers & AKS** | ACI, AKS, Container Apps, service mesh |

### Data & Integration

| Chapter | Topic | Description |
|---------|-------|-------------|
| [09](09-databases/README.md) | **Databases** | Azure SQL, Cosmos DB, PostgreSQL, Redis |
| [10](10-api-integration/README.md) | **API & Integration** | APIM, Service Bus, Event Grid, Logic Apps |

### Architecture & Patterns

| Chapter | Topic | Description |
|---------|-------|-------------|
| [11](11-architecture-styles/README.md) | **Architecture Styles** | N-Tier, Microservices, Event-Driven, Big Data |
| [12](12-design-patterns/README.md) | **Cloud Design Patterns** | Reliability, messaging, data, deployment patterns |
| [13](13-anti-patterns/README.md) | **Anti-Patterns** | Busy Database, Chatty I/O, Retry Storm, and more |

### AI & Machine Learning

| Chapter | Topic | Description |
|---------|-------|-------------|
| [14](14-ai-platform/README.md) | **AI Platform** | Azure OpenAI, Cognitive Services, Machine Learning |
| [15](15-ai-ml-comprehensive/README.md) | **AI & ML Comprehensive** | AI agents, RAG, orchestration, Responsible AI |

### Operations

| Chapter | Topic | Description |
|---------|-------|-------------|
| [16](16-observability/README.md) | **Observability** | Azure Monitor, Log Analytics, Application Insights |
| [17](17-infrastructure-as-code/README.md) | **Infrastructure as Code** | Bicep, Terraform, ARM templates |
| [18](18-finops/README.md) | **FinOps & Cost Management** | Cost analysis, budgets, reservations, optimization |

### Learning & Certification

| Chapter | Topic | Description |
|---------|-------|-------------|
| [19](19-case-studies/README.md) | **Architecture Case Studies** | E-commerce, data platform, financial services, IoT |
| [20](20-certifications/README.md) | **Certifications** | AZ-305, AZ-104 study guides, exam preparation |

---

## Quick Start Paths

### Path 1: Fast Track (2-3 weeks)

For experienced AWS architects needing rapid Azure knowledge:

```
Week 1: Foundation & Networking
├── 00-foundation (Quick Reference)
├── 01-cloud-adoption-framework (Landing Zones)
├── 02-identity (Entra ID focus)
├── 05-networking (VNet vs VPC)
└── 07-load-balancing-cdn

Week 2: Services & Data
├── 08-containers-aks
├── 09-databases (SQL, Cosmos DB)
├── 10-api-integration
└── 17-infrastructure-as-code (Bicep)

Week 3: Architecture & Certification
├── 11-architecture-styles
├── 12-design-patterns
├── 13-anti-patterns
├── 19-case-studies
└── 20-certifications (AZ-305 prep)
```

### Path 2: Comprehensive (8-10 weeks)

For thorough coverage with hands-on practice:

```
Weeks 1-2: Foundation & Strategy
├── Foundation (00)
├── Cloud Adoption Framework (01)
├── Identity & Security (02-04)
└── Labs: Create landing zone, configure policies

Weeks 3-4: Networking
├── Networking fundamentals (05)
├── Network Security (06)
├── Load Balancing & CDN (07)
└── Labs: Hub-spoke network, Azure Firewall

Weeks 5-6: Compute, Data & Integration
├── Containers & AKS (08)
├── Databases (09)
├── API & Integration (10)
└── Labs: AKS deployment, Cosmos DB, Event Grid

Weeks 7-8: Architecture & Patterns
├── Architecture Styles (11)
├── Design Patterns (12)
├── Anti-Patterns (13)
└── Labs: Implement patterns in sample apps

Weeks 9-10: AI, Operations & Certification
├── AI Platform (14-15)
├── Operations (16-18)
├── Case Studies (19)
├── Certifications (20)
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
| **Identity** | IAM | Entra ID + RBAC |
| | Cognito | Entra External ID |
| **Security** | GuardDuty | Microsoft Defender |
| | WAF | Azure WAF |
| | KMS | Key Vault |
| **Monitoring** | CloudWatch | Azure Monitor |
| | X-Ray | Application Insights |
| **AI/ML** | Bedrock | Azure OpenAI |
| | SageMaker | Azure ML |

---

## Key Learning Resources

### Official Documentation
- [Azure Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/)
- [Well-Architected Framework](https://learn.microsoft.com/en-us/azure/well-architected/)
- [AWS to Azure Comparison](https://learn.microsoft.com/en-us/azure/architecture/aws-professional/)
- [Cloud Adoption Framework](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/)

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

---

## Author & Credits

**Author:** Michel Abboud

**AI Assistance:** This guide was created with the assistance of Claude (Anthropic's AI assistant). In the spirit of full transparency, AI was used to help research, structure, and write the content throughout this repository.

### Transparency Note

This project demonstrates human-AI collaboration in creating educational content. The author provided direction, expertise, and review while AI assisted with research, writing, and organization. All content has been curated and verified for accuracy.

---

## License

This project is licensed under the **Apache License 2.0** - see the [LICENSE](LICENSE) file for details.

```
Copyright 2026 Michel Abboud

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0
```

---

*Last Updated: January 2026*
