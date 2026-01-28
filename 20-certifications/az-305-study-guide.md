# AZ-305: Azure Solutions Architect Expert - Study Guide

## Exam Overview

| Attribute | Details |
|-----------|---------|
| Exam Code | AZ-305 |
| Duration | 120 minutes |
| Questions | 40-60 (case studies, scenarios, multiple choice) |
| Passing Score | 700/1000 |
| Prerequisite | AZ-104 recommended (not required) |
| Cost | $165 USD |

## Exam Domains

### Domain 1: Design Identity, Governance, and Monitoring (25-30%)

#### Key Topics

**Identity Design**
- Azure AD tenant design
- Hybrid identity (Azure AD Connect)
- B2B and B2C scenarios
- Conditional Access policies
- Privileged Identity Management (PIM)

**Governance**
- Management group hierarchy
- Azure Policy (built-in vs custom)
- Azure Blueprints
- Cost management strategies
- Resource tagging strategies

**Monitoring**
- Azure Monitor architecture
- Log Analytics workspaces
- Application Insights
- Alert rules and action groups
- Workbooks and dashboards

#### AWS to Azure Mapping

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| IAM Users/Roles | Azure AD Users/Groups | Azure AD is a full IdP |
| Organizations | Management Groups | Hierarchical with inheritance |
| Config Rules | Azure Policy | Built-in policy library |
| CloudWatch | Azure Monitor | Integrated metrics and logs |
| X-Ray | Application Insights | Deeper APM integration |

#### Practice Scenarios

**Scenario 1**: Design identity for a company with:
- 10,000 employees with on-premises AD
- 500 external contractors
- Mobile workforce requiring SSO

**Solution considerations**:
- Azure AD Connect for hybrid identity
- Azure AD B2B for contractors
- Conditional Access for mobile devices
- MFA enforcement policies

### Domain 2: Design Data Storage Solutions (15-20%)

#### Key Topics

**Relational Data**
- Azure SQL deployment options (Single, Elastic Pool, MI)
- High availability configurations
- Geo-replication strategies
- Performance tuning

**Non-Relational Data**
- Cosmos DB APIs and consistency levels
- Partitioning strategies
- Azure Table Storage vs Cosmos DB

**Data Integration**
- Azure Data Factory
- Synapse Analytics
- Data Lake Storage Gen2
- Event Hubs for streaming

#### AWS to Azure Mapping

| AWS Service | Azure Equivalent | Exam Focus |
|-------------|------------------|------------|
| RDS | Azure SQL | Deployment options |
| Aurora | Azure SQL Hyperscale | Scaling patterns |
| DynamoDB | Cosmos DB | Consistency levels |
| Redshift | Synapse Analytics | Data warehouse design |
| S3 | Blob Storage / ADLS | Tiering, lifecycle |

#### Key Decision Framework

```
┌─────────────────────────────────────────────────────────────────┐
│              Storage Solution Decision Tree                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Relational Data?                                                │
│       │                                                          │
│       ├── Yes ──► SQL Server features needed?                   │
│       │              │                                           │
│       │              ├── Yes ──► SQL MI                         │
│       │              └── No ──► Azure SQL DB                    │
│       │                                                          │
│       └── No ──► Global distribution needed?                    │
│                      │                                           │
│                      ├── Yes ──► Cosmos DB                      │
│                      └── No ──► Document model?                 │
│                                    │                             │
│                                    ├── Yes ──► Cosmos DB        │
│                                    └── No ──► Table Storage     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Domain 3: Design Business Continuity Solutions (15-20%)

#### Key Topics

**High Availability**
- Availability Sets vs Availability Zones
- Zone-redundant services
- Application Gateway with zones
- SQL Always On / Geo-replication

**Disaster Recovery**
- Azure Site Recovery (ASR)
- RPO/RTO requirements
- Failover strategies
- DR testing and runbooks

**Backup**
- Azure Backup architecture
- Backup policies and retention
- Cross-region backup
- Backup Center

#### Critical Concepts

**RPO/RTO Decision Matrix**

| Requirement | Solution | Cost |
|-------------|----------|------|
| RPO < 1 min, RTO < 1 min | Active-Active, Zone-redundant | $$$$$ |
| RPO < 5 min, RTO < 1 hr | Active-Passive, Geo-replication | $$$ |
| RPO < 1 hr, RTO < 4 hr | Azure Site Recovery | $$ |
| RPO < 24 hr, RTO < 24 hr | Azure Backup | $ |

### Domain 4: Design Infrastructure Solutions (30-35%)

#### Key Topics

**Compute**
- VM sizing and series selection
- Virtual Machine Scale Sets
- App Service plans and scaling
- Azure Functions hosting plans
- Container options (ACI, AKS, ACA)

**Networking**
- Hub-spoke vs flat network
- VNet peering vs VPN Gateway
- Private Link / Private Endpoints
- Load balancing options
- DNS design

**Migrations**
- Azure Migrate assessment
- Database migration strategies
- Application modernization paths
- Hybrid connectivity

#### Compute Decision Framework

```
Compute Selection:

1. Full VM control needed?
   └── Yes ──► Virtual Machines / VMSS

2. Just run code (no infra)?
   ├── Event-driven, short duration ──► Azure Functions
   └── Always-on, HTTP ──► App Service

3. Containers?
   ├── Simple, single container ──► ACI
   ├── Microservices, orchestration ──► AKS
   └── Serverless containers ──► Container Apps
```

## Study Resources

### Official Microsoft Resources (Free)

| Resource | Link | Best For |
|----------|------|----------|
| Microsoft Learn Path | [AZ-305 Learning Path](https://learn.microsoft.com/en-us/training/paths/microsoft-azure-architect-design-prerequisites/) | Structured learning |
| Exam Skills Outline | [AZ-305 Skills](https://learn.microsoft.com/en-us/certifications/exams/az-305) | Exam objectives |
| Azure Documentation | [Azure Docs](https://learn.microsoft.com/en-us/azure/) | Deep dives |
| Azure Architecture Center | [Architecture Center](https://learn.microsoft.com/en-us/azure/architecture/) | Reference architectures |

### Paid Resources (Optional)

| Resource | Cost | Value |
|----------|------|-------|
| Pluralsight | $29/month | Video courses |
| A Cloud Guru | $35/month | Labs + courses |
| Whizlabs Practice Tests | $20 | Exam simulation |
| MeasureUp Practice Tests | $99 | Official partner |

### Hands-on Labs

1. **Microsoft Learn Sandboxes**: Free, temporary Azure environments
2. **Azure Free Account**: $200 credit for 30 days
3. **GitHub Codespaces**: For IaC labs

## Practice Questions

### Sample Question 1 (Identity)

**Scenario**: A company needs to allow 200 partner users from 5 different organizations to access an internal application. Partners should use their existing corporate credentials.

**What should you recommend?**

A. Create Azure AD accounts for each partner user
B. Implement Azure AD B2B collaboration
C. Deploy Azure AD B2C
D. Configure Azure AD Domain Services

**Answer**: B

**Explanation**: Azure AD B2B allows external users to authenticate with their home organization credentials. B2C is for consumer scenarios, not B2B. Creating accounts defeats the SSO purpose.

### Sample Question 2 (Data Storage)

**Scenario**: An application needs:
- Global distribution (US, Europe, Asia)
- Strong consistency for financial transactions
- Low latency reads (< 10ms)

**What database configuration do you recommend?**

A. Azure SQL with geo-replication
B. Cosmos DB with Session consistency
C. Cosmos DB with Strong consistency
D. Azure SQL Hyperscale

**Answer**: C

**Explanation**: Cosmos DB with Strong consistency provides global distribution with guaranteed consistency. Session consistency doesn't meet the strong consistency requirement for financial data.

### Sample Question 3 (Business Continuity)

**Scenario**: A company requires:
- RPO: 15 minutes
- RTO: 4 hours
- Cost-effective solution
- Application runs on VMs

**What should you implement?**

A. Availability Zones only
B. Azure Site Recovery to secondary region
C. Active-Active with Traffic Manager
D. Azure Backup with 4-hour frequency

**Answer**: B

**Explanation**: ASR provides continuous replication (meeting 15-min RPO) with 4-hour RTO capability. Availability Zones don't protect against regional failures. Active-Active is over-engineered. Backup frequency of 4 hours exceeds RPO.

### Sample Question 4 (Infrastructure)

**Scenario**: You need to host a .NET Core API with:
- Auto-scaling (0 to 1000 instances)
- Per-second billing
- Container-based deployment
- No cluster management

**What compute platform should you use?**

A. Azure Kubernetes Service
B. Azure Container Instances
C. Azure Container Apps
D. App Service

**Answer**: C

**Explanation**: Container Apps provides serverless containers with auto-scaling to zero, per-second billing, and no cluster management. AKS requires cluster management. ACI doesn't auto-scale. App Service doesn't scale to zero.

## Exam Day Tips

### Time Management

- **Case Studies**: 20-30 minutes for 2-3 case studies
- **Standalone Questions**: 1-2 minutes per question
- **Review Time**: Save 10-15 minutes for review

### Common Traps

1. **Over-engineering**: Choose simplest solution that meets requirements
2. **Cost**: Always consider cost when options are technically equivalent
3. **"Most secure"**: Often implies trade-offs with usability
4. **"Best practice"**: Microsoft terminology may differ from AWS

### Keywords to Watch

| Keyword | Implication |
|---------|-------------|
| "Minimize cost" | Choose cheaper option |
| "Minimize latency" | Consider proximity, caching |
| "Compliance requirement" | Security/governance focus |
| "Legacy application" | IaaS over PaaS |
| "Cloud-native" | PaaS, serverless |

## Study Checklist

### Week 1-2: Identity & Governance
- [ ] Azure AD concepts and tenant design
- [ ] Hybrid identity (Azure AD Connect)
- [ ] Conditional Access and MFA
- [ ] Management Groups and subscriptions
- [ ] Azure Policy and RBAC
- [ ] Lab: Configure hybrid identity

### Week 3-4: Data Storage
- [ ] Azure SQL deployment options
- [ ] Cosmos DB consistency and partitioning
- [ ] Storage account types and tiers
- [ ] Data Lake Storage Gen2
- [ ] Lab: Deploy multi-region database

### Week 5-6: Business Continuity
- [ ] Availability Sets and Zones
- [ ] Azure Site Recovery
- [ ] Backup and restore strategies
- [ ] DR planning and testing
- [ ] Lab: Configure ASR for VMs

### Week 7-8: Infrastructure
- [ ] Compute options comparison
- [ ] Networking design patterns
- [ ] Migration strategies
- [ ] Cost optimization
- [ ] Lab: Deploy hub-spoke network
- [ ] Full practice exam

## AWS Architect Advantages

As an AWS architect, you already understand:

| Concept | AWS | Azure | Your Advantage |
|---------|-----|-------|----------------|
| Global infrastructure | Regions, AZs | Regions, AZs | Identical concept |
| IaC | CloudFormation | Bicep/ARM | Template structure similar |
| Containers | ECS/EKS | ACI/AKS | Kubernetes is Kubernetes |
| Serverless | Lambda | Functions | Event-driven patterns |
| Networking | VPC | VNet | Subnets, routing, security |

Focus your study time on Azure-specific services and terminology rather than core concepts.

---

*Continue to [AZ-104 Study Guide](az-104-study-guide.md)*

*Back to [Certifications Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
