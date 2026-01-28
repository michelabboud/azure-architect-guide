# AWS to Azure Complete Service Mapping

## Quick Reference Table

> **How to use**: Find your familiar AWS service, understand its Azure equivalent, and note the key differences.

---

## Identity & Access Management

| AWS Service | Azure Equivalent | Key Differences | When to Use Azure |
|-------------|------------------|-----------------|-------------------|
| **IAM Users/Groups** | Entra ID Users/Groups | Entra ID is a full identity platform, not just permissions | All user identity scenarios |
| **IAM Roles** | Managed Identities + RBAC Roles | Azure separates identity (MI) from permissions (RBAC) | Service-to-service auth |
| **IAM Policies** | RBAC Role Definitions + Azure Policy | RBAC = permissions, Policy = compliance/governance | Access control |
| **IAM Identity Center** | Entra ID (workforce) | Entra ID is the single identity plane | Enterprise SSO |
| **Cognito User Pools** | Entra ID B2C | B2C for customer-facing identity | Customer apps |
| **Cognito Identity Pools** | Entra ID + Managed Identity | Different model for federated access | App identity federation |
| **AWS Organizations SCPs** | Management Groups + Azure Policy | Azure uses policy inheritance through MG hierarchy | Multi-subscription governance |
| **AWS Resource Access Manager** | Azure Lighthouse + RBAC | Lighthouse for cross-tenant, RBAC for cross-subscription | Resource sharing |
| **AWS SSO** | Entra ID + Conditional Access | Conditional Access adds context-aware policies | Enterprise authentication |

### Identity Model Comparison

```
AWS Model:                              Azure Model:
─────────                               ───────────
┌─────────────────────┐                ┌─────────────────────────────────┐
│   AWS Account       │                │   Entra ID Tenant               │
│   ┌───────────────┐ │                │   ┌───────────────────────────┐ │
│   │  IAM Users    │ │                │   │  Users                    │ │
│   │  IAM Groups   │ │                │   │  Groups                   │ │
│   │  IAM Roles    │ │                │   │  Service Principals       │ │
│   │  IAM Policies │ │                │   │  Managed Identities       │ │
│   └───────────────┘ │                │   │  App Registrations        │ │
│          ↓          │                │   └───────────────────────────┘ │
│   Resources get     │                │              ↓                   │
│   permissions via   │                │   Conditional Access decides    │
│   attached policies │                │   IF you can authenticate       │
└─────────────────────┘                │              ↓                   │
                                       │   RBAC decides WHAT you         │
                                       │   can do on resources           │
                                       │              ↓                   │
                                       │   Azure Policy enforces         │
                                       │   HOW resources behave          │
                                       └─────────────────────────────────┘
```

---

## Security Services

| AWS Service | Azure Equivalent | Key Differences | Deep Dive Link |
|-------------|------------------|-----------------|----------------|
| **Security Hub** | Microsoft Defender for Cloud | Defender is CNAPP + CSPM + CWPP combined | [Chapter 03](../03-security/README.md) |
| **GuardDuty** | Defender for Cloud (threat detection) | Part of unified Defender platform | [Chapter 03](../03-security/README.md) |
| **Inspector** | Defender for Cloud (vulnerability) | Integrated vulnerability scanning | [Chapter 03](../03-security/README.md) |
| **AWS WAF** | Azure WAF + Front Door | WAF integrated with Front Door/App Gateway | [Chapter 03](../03-security/README.md) |
| **AWS Shield** | Azure DDoS Protection | Standard vs Advanced tiers similar | [Chapter 03](../03-security/README.md) |
| **KMS** | Azure Key Vault | Key Vault includes secrets + certificates | [Chapter 03](../03-security/README.md) |
| **Secrets Manager** | Azure Key Vault (secrets) | Combined in Key Vault | [Chapter 03](../03-security/README.md) |
| **Certificate Manager** | Azure Key Vault (certificates) | Combined in Key Vault | [Chapter 03](../03-security/README.md) |
| **AWS Config** | Azure Policy + Resource Graph | Policy for compliance, Graph for queries | [Chapter 04](../04-compliance/README.md) |
| **CloudTrail** | Azure Activity Log + Entra Audit Logs | Split between resource and identity logs | [Chapter 16](../16-observability/README.md) |
| **Macie** | Microsoft Purview (data discovery) | Purview is broader data governance | [Chapter 04](../04-compliance/README.md) |
| **Verified Access** | Entra ID Conditional Access + App Proxy | Conditional Access is more comprehensive | [Chapter 02](../02-identity/README.md) |

### Security Stack Comparison

```
AWS Security Stack:                    Azure Security Stack:
──────────────────                     ────────────────────

┌──────────────────┐                   ┌──────────────────────────────────┐
│ Perimeter        │                   │ Identity Perimeter               │
│ - Shield         │                   │ - Entra ID                       │
│ - WAF            │                   │ - Conditional Access             │
│ - CloudFront     │                   │ - App Proxy                      │
└────────┬─────────┘                   └───────────────┬──────────────────┘
         ↓                                             ↓
┌──────────────────┐                   ┌──────────────────────────────────┐
│ Detection        │                   │ Microsoft Defender Platform      │
│ - GuardDuty      │                   │ - Defender for Cloud             │
│ - Inspector      │                   │ - Defender for Endpoint          │
│ - Security Hub   │                   │ - Defender for Identity          │
└────────┬─────────┘                   │ - Defender for Cloud Apps        │
         ↓                             └───────────────┬──────────────────┘
┌──────────────────┐                                   ↓
│ Data Protection  │                   ┌──────────────────────────────────┐
│ - KMS            │                   │ Data Protection                  │
│ - Macie          │                   │ - Key Vault                      │
│ - Secrets Mgr    │                   │ - Purview (DLP + Classification) │
└──────────────────┘                   │ - Information Protection         │
                                       └──────────────────────────────────┘
```

---

## Compliance & Governance

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **AWS Config Rules** | Azure Policy | Policy is more powerful with deny/audit/modify effects |
| **AWS Audit Manager** | Microsoft Purview Compliance Manager | Purview is broader scope |
| **AWS Artifact** | Azure Compliance Documentation | Similar compliance reports |
| **Control Tower** | Azure Landing Zones | Landing Zones + Blueprints for governance |
| **Service Control Policies** | Management Groups + Policy | Inheritance through management group hierarchy |
| **AWS Organizations** | Management Groups + Subscriptions | Different hierarchy model |

### Governance Hierarchy Comparison

```
AWS:                                   Azure:
────                                   ─────

Organization                           Tenant (Entra ID)
    │                                      │
    ├── Root OU                        Management Groups
    │   └── SCPs                           │ └── Azure Policy
    │                                      │
    ├── OU (Prod)                      ├── MG (Platform)
    │   ├── Account 1                  │   ├── Subscription (Identity)
    │   └── Account 2                  │   ├── Subscription (Management)
    │                                  │   └── Subscription (Connectivity)
    └── OU (Dev)                       │
        ├── Account 3                  └── MG (Landing Zones)
        └── Account 4                      ├── MG (Corp)
                                           │   └── Subscription (App1)
                                           └── MG (Online)
                                               └── Subscription (App2)
```

---

## Compute Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **EC2** | Virtual Machines | Similar IaaS; Azure uses Availability Sets/Zones |
| **Lambda** | Azure Functions | Functions has Durable Functions for stateful |
| **ECS** | Azure Container Instances | ACI for simple containers |
| **EKS** | Azure Kubernetes Service (AKS) | AKS is fully managed, integrated with Entra ID |
| **Fargate** | ACI + AKS Virtual Nodes | Serverless containers |
| **Elastic Beanstalk** | Azure App Service | App Service more feature-rich |
| **Batch** | Azure Batch | Similar HPC batch |
| **Lightsail** | Azure VMs (B-series) | B-series for burstable |
| **Outposts** | Azure Stack / Arc | Arc for hybrid management |
| **Wavelength** | Azure Edge Zones | Edge computing |

---

## Storage Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **S3** | Blob Storage | Azure: Storage Account → Container → Blob |
| **S3 Glacier** | Blob Archive Tier | Integrated tiers in same account |
| **EBS** | Managed Disks | Similar block storage |
| **EFS** | Azure Files | SMB and NFS support |
| **FSx** | Azure NetApp Files | Enterprise file storage |
| **Storage Gateway** | Azure File Sync / StorSimple | Hybrid storage |
| **S3 Transfer Acceleration** | Azure CDN / Front Door | Different approach to acceleration |

### Storage Tiers Comparison

```
AWS S3:                          Azure Blob:
───────                          ──────────
┌─────────────────┐              ┌─────────────────┐
│ S3 Standard     │ ←──────────→ │ Hot             │
├─────────────────┤              ├─────────────────┤
│ S3 Standard-IA  │ ←──────────→ │ Cool            │
├─────────────────┤              ├─────────────────┤
│ S3 One Zone-IA  │ ←─(similar)─→│ Cool (LRS)      │
├─────────────────┤              ├─────────────────┤
│ S3 Glacier IR   │ ←──────────→ │ Cold            │
├─────────────────┤              ├─────────────────┤
│ S3 Glacier Flex │ ←──────────→ │ Archive         │
├─────────────────┤              │                 │
│ S3 Glacier Deep │ ←──────────→ │ Archive         │
└─────────────────┘              └─────────────────┘
```

---

## Database Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **RDS (MySQL/PostgreSQL)** | Azure Database for MySQL/PostgreSQL | Similar managed databases |
| **RDS (SQL Server)** | Azure SQL Database | Azure SQL has unique features (Hyperscale) |
| **Aurora** | Azure SQL Hyperscale / Cosmos DB | Different architectures |
| **DynamoDB** | Cosmos DB (Table API or NoSQL API) | Cosmos DB supports multiple APIs |
| **DocumentDB** | Cosmos DB (MongoDB API) | Cosmos DB is multi-model |
| **Neptune** | Cosmos DB (Gremlin API) | Graph via Gremlin |
| **ElastiCache** | Azure Cache for Redis | Redis-compatible |
| **Redshift** | Azure Synapse Analytics | Synapse includes Spark integration |
| **Timestream** | Azure Data Explorer | Time-series data |
| **QLDB** | Azure Confidential Ledger | Immutable ledger |

### Cosmos DB Multi-API Advantage

```
┌────────────────────────────────────────────────────────────────────┐
│                         Cosmos DB                                   │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │
│  │  NoSQL API  │  │ MongoDB API │  │ Gremlin API │  │ Table API │ │
│  │  (Native)   │  │             │  │  (Graph)    │  │           │ │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘ │
│         │                │                │               │        │
│         └────────────────┼────────────────┼───────────────┘        │
│                          ↓                                          │
│              ┌───────────────────────┐                             │
│              │  Global Distribution  │                             │
│              │  Multi-Region Writes  │                             │
│              │  Automatic Failover   │                             │
│              └───────────────────────┘                             │
│                                                                     │
│  AWS Equivalents:                                                  │
│  • NoSQL API ≈ DynamoDB                                            │
│  • MongoDB API ≈ DocumentDB                                        │
│  • Gremlin API ≈ Neptune                                           │
│  • Table API ≈ DynamoDB (simple key-value)                         │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

---

## Networking Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **VPC** | Virtual Network (VNet) | Similar concepts, different defaults |
| **Subnets** | Subnets | Azure subnets span availability zones |
| **Internet Gateway** | (Implicit) | VNets have implicit internet access |
| **NAT Gateway** | NAT Gateway | Similar functionality |
| **Route Tables** | Route Tables | Similar functionality |
| **Security Groups** | Network Security Groups (NSG) | NSGs can attach to subnet or NIC |
| **NACLs** | NSG (subnet-level) | Azure uses NSGs for both |
| **VPC Peering** | VNet Peering | Similar, plus Global VNet Peering |
| **Transit Gateway** | Azure Virtual WAN | Virtual WAN for hub-spoke |
| **Direct Connect** | ExpressRoute | Dedicated connections |
| **Route 53** | Azure DNS + Traffic Manager | Split DNS and routing |
| **CloudFront** | Azure CDN / Front Door | Front Door adds WAF + routing |
| **ELB/ALB/NLB** | Load Balancer / App Gateway | App Gateway is L7 with WAF |
| **API Gateway** | Azure API Management | APIM more feature-rich |
| **PrivateLink** | Private Link / Private Endpoints | Similar private connectivity |
| **Global Accelerator** | Azure Front Door | Global load balancing |

### Load Balancer Comparison

```
AWS:                                    Azure:
────                                    ─────

Layer 7:                                Layer 7:
┌─────────────────────┐                 ┌─────────────────────────────┐
│ Application Load    │                 │ Application Gateway         │
│ Balancer (ALB)      │                 │ - Path-based routing        │
│ - Path routing      │ ←─────────────→ │ - WAF integration           │
│ - Host routing      │                 │ - SSL termination           │
└─────────────────────┘                 │ - Autoscaling               │
                                        └─────────────────────────────┘

Layer 4:                                Layer 4:
┌─────────────────────┐                 ┌─────────────────────────────┐
│ Network Load        │                 │ Azure Load Balancer         │
│ Balancer (NLB)      │ ←─────────────→ │ - Standard (zone-redundant) │
│ - TCP/UDP           │                 │ - Basic (single zone)       │
└─────────────────────┘                 └─────────────────────────────┘

Global:                                 Global:
┌─────────────────────┐                 ┌─────────────────────────────┐
│ Global Accelerator  │                 │ Azure Front Door            │
│ - Anycast IPs       │ ←─────────────→ │ - Global HTTP LB            │
│ - TCP/UDP           │                 │ - WAF + CDN integrated      │
└─────────────────────┘                 │ - Anycast                   │
                                        └─────────────────────────────┘
```

---

## AI/ML Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **Bedrock** | Azure OpenAI Service | Azure has exclusive OpenAI models |
| **SageMaker** | Azure Machine Learning | Similar ML platforms |
| **Rekognition** | Azure AI Vision | Computer vision |
| **Polly** | Azure AI Speech | Text-to-speech |
| **Transcribe** | Azure AI Speech | Speech-to-text |
| **Lex** | Copilot Studio / Bot Service | Copilot Studio for enterprise bots |
| **Comprehend** | Azure AI Language | NLP services |
| **Q Business** | Microsoft 365 Copilot | Enterprise AI assistant |
| **Kendra** | Azure AI Search | Enterprise search |
| **Personalize** | Azure Personalizer | Recommendations |

### AI Platform Comparison

```
AWS AI Stack:                          Azure AI Stack:
─────────────                          ──────────────

┌─────────────────────┐                ┌─────────────────────────────┐
│ Amazon Bedrock      │                │ Azure OpenAI Service        │
│ - Claude            │                │ - GPT-4, GPT-4o             │
│ - Llama             │                │ - Claude (coming)           │
│ - Titan             │ ←────────────→ │ - Llama                     │
│ - Mistral           │                │ - Mistral                   │
└─────────────────────┘                │ - Exclusive OpenAI access   │
                                       └─────────────────────────────┘

┌─────────────────────┐                ┌─────────────────────────────┐
│ Amazon Q Business   │                │ Microsoft 365 Copilot       │
│ - Enterprise RAG    │ ←────────────→ │ - Enterprise AI assistant   │
│ - Doc understanding │                │ - M365 integration          │
└─────────────────────┘                │ - Graph API access          │
                                       └─────────────────────────────┘

┌─────────────────────┐                ┌─────────────────────────────┐
│ Amazon Lex          │                │ Copilot Studio              │
│ - Chatbots          │ ←────────────→ │ - Low-code bot builder      │
│ - Voice/text        │                │ - Enterprise connectors     │
└─────────────────────┘                │ - Power Platform integration│
                                       └─────────────────────────────┘
```

---

## Monitoring & Observability

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **CloudWatch Metrics** | Azure Monitor Metrics | Similar metrics platform |
| **CloudWatch Logs** | Log Analytics | KQL query language |
| **CloudWatch Alarms** | Azure Monitor Alerts | Similar alerting |
| **X-Ray** | Application Insights | App Insights more comprehensive |
| **CloudTrail** | Activity Log + Entra Audit | Split between resource and identity |
| **EventBridge** | Event Grid | Similar event routing |
| **Managed Grafana** | Azure Managed Grafana | Same Grafana, Azure integration |
| **AWS Distro for OpenTelemetry** | Azure Monitor OpenTelemetry | OTel support |

---

## Messaging & Integration

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **SQS** | Azure Queue Storage / Service Bus | Service Bus for enterprise |
| **SNS** | Service Bus Topics / Event Grid | Event Grid for events, Service Bus for messages |
| **EventBridge** | Event Grid | Similar event-driven patterns |
| **Step Functions** | Logic Apps / Durable Functions | Logic Apps low-code, Durable Functions code-first |
| **AppSync** | (No direct equivalent) | Use API Management + Functions |
| **MQ** | Service Bus | Enterprise messaging |

---

## DevOps & IaC

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **CloudFormation** | ARM Templates / Bicep | Bicep is more readable than ARM JSON |
| **CDK** | Bicep / Pulumi | Bicep is Azure-native |
| **CodePipeline** | Azure Pipelines | Part of Azure DevOps |
| **CodeBuild** | Azure Pipelines | Integrated in same service |
| **CodeDeploy** | Azure Pipelines / App Service Deployment | Multiple deployment options |
| **CodeCommit** | Azure Repos | Git repositories |
| **ECR** | Azure Container Registry | Container images |
| **Systems Manager** | Azure Automation | Configuration management |

---

## Cost Management

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| **Cost Explorer** | Azure Cost Management | Similar cost analysis |
| **Budgets** | Azure Budgets | Similar budget alerts |
| **Savings Plans** | Azure Reservations | Similar commitment discounts |
| **Reserved Instances** | Azure Reserved VM Instances | Similar RI model |
| **Spot Instances** | Azure Spot VMs | Similar spot pricing |
| **Compute Optimizer** | Azure Advisor | Advisor covers more than compute |

---

## Quick Reference: Terminology

| AWS Term | Azure Term |
|----------|------------|
| Account | Subscription |
| Organization | Tenant + Management Groups |
| Region | Region |
| Availability Zone | Availability Zone |
| VPC | Virtual Network (VNet) |
| Security Group | Network Security Group (NSG) |
| IAM Role | Managed Identity |
| IAM Policy | RBAC Role Assignment |
| S3 Bucket | Storage Account + Container |
| EC2 Instance | Virtual Machine |
| Lambda Function | Azure Function |
| RDS Instance | Azure SQL Database / Azure Database for X |
| CloudFormation Stack | Resource Group + Deployment |
| Tags | Tags |
| ARN | Resource ID |

---

*Next: [Azure Fundamentals](azure-fundamentals.md)* | *Back to [Foundation Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
