# AWS to Azure Service Mapping Guide

A comprehensive mapping of AWS services to their Azure equivalents for Cloud Architects transitioning between platforms.

## Compute Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| EC2 | Virtual Machines | Similar IaaS offerings; Azure uses Availability Sets/Zones |
| Lambda | Azure Functions | Azure has Durable Functions for stateful workflows |
| ECS/EKS | Azure Kubernetes Service (AKS) | AKS is fully managed; Azure also has Container Instances |
| Elastic Beanstalk | Azure App Service | App Service supports more languages natively |
| AWS Batch | Azure Batch | Similar HPC batch processing capabilities |
| Lightsail | Azure Virtual Machines (B-series) | Azure uses burstable VM sizes instead |
| AWS Outposts | Azure Stack | Both provide on-premises cloud extensions |

### Compute Deep Dive

**EC2 vs Azure VMs:**
- AWS: Instance families (t3, m5, c5, r5, etc.)
- Azure: VM series (B, D, E, F, M, N, etc.)
- Both support spot/preemptible instances (AWS Spot, Azure Spot VMs)
- Both support reserved instances for cost savings

**Serverless Comparison:**
```
AWS Lambda                    Azure Functions
├── 15 min max timeout       ├── 10 min (Consumption), unlimited (Premium)
├── 10GB memory max          ├── 14GB memory max (Premium)
├── Layers for dependencies  ├── Deployment slots
└── Step Functions           └── Durable Functions (built-in)
```

## Storage Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| S3 | Blob Storage | Azure has hot/cool/archive tiers built-in |
| EBS | Managed Disks | Similar block storage for VMs |
| EFS | Azure Files | Azure Files supports SMB and NFS |
| S3 Glacier | Azure Archive Storage | Both for long-term cold storage |
| Storage Gateway | Azure StorSimple | Hybrid storage solutions |
| FSx | Azure NetApp Files | Managed file systems |

### Storage Tiers Comparison

```
AWS S3 Tiers              Azure Blob Tiers
├── S3 Standard          ├── Hot
├── S3 Standard-IA       ├── Cool
├── S3 One Zone-IA       ├── Cool (LRS)
├── S3 Glacier Instant   ├── Cool
├── S3 Glacier Flexible  ├── Archive
└── S3 Glacier Deep      └── Archive
```

## Database Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| RDS | Azure SQL Database / Azure Database for PostgreSQL/MySQL | Azure SQL has built-in intelligence |
| DynamoDB | Cosmos DB | Cosmos DB offers multiple APIs (SQL, MongoDB, Cassandra) |
| Aurora | Azure SQL Hyperscale | Both are cloud-native relational DBs |
| ElastiCache | Azure Cache for Redis | Similar managed Redis/Memcached |
| Redshift | Azure Synapse Analytics | Synapse integrates with Spark |
| DocumentDB | Cosmos DB (MongoDB API) | Cosmos DB is more flexible |
| Neptune | Cosmos DB (Gremlin API) | Graph database capabilities |
| Timestream | Azure Time Series Insights | IoT time-series data |

### Cosmos DB Multi-API Advantage

```
AWS Equivalents          →  Cosmos DB APIs
├── DynamoDB            →  Table API
├── DocumentDB/MongoDB  →  MongoDB API
├── Neptune             →  Gremlin API
├── Cassandra           →  Cassandra API
└── Custom              →  SQL API (native)
```

## Networking Services

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| VPC | Virtual Network (VNet) | Similar concepts, different CIDR defaults |
| Route 53 | Azure DNS + Traffic Manager | Traffic Manager for routing policies |
| CloudFront | Azure CDN / Front Door | Front Door adds WAF and load balancing |
| Direct Connect | ExpressRoute | Both provide dedicated connections |
| ELB/ALB/NLB | Azure Load Balancer / App Gateway | App Gateway includes WAF |
| API Gateway | Azure API Management | APIM has more built-in policies |
| Transit Gateway | Azure Virtual WAN | Hub-spoke network architectures |
| PrivateLink | Private Link | Similar private endpoint services |

### Load Balancer Comparison

```
AWS                          Azure
├── ALB (Layer 7)           ├── Application Gateway (Layer 7)
├── NLB (Layer 4)           ├── Azure Load Balancer (Layer 4)
├── CLB (Classic)           ├── (Deprecated equivalent)
└── GLB (Gateway LB)        └── Gateway Load Balancer
```

## Security & Identity

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| IAM | Azure Active Directory (Entra ID) + RBAC | Azure AD is a full identity platform |
| Cognito | Azure AD B2C | B2C for customer identity |
| KMS | Azure Key Vault | Key Vault includes secrets management |
| Secrets Manager | Azure Key Vault | Combined in Key Vault |
| WAF | Azure WAF | Similar web application firewall |
| Shield | Azure DDoS Protection | DDoS mitigation services |
| GuardDuty | Microsoft Defender for Cloud | Threat detection |
| Security Hub | Microsoft Defender for Cloud | Security posture management |
| Inspector | Microsoft Defender for Cloud | Vulnerability assessment |

### IAM Model Differences

```
AWS IAM                         Azure RBAC
├── Users                      ├── Users (in Azure AD)
├── Groups                     ├── Groups (in Azure AD)
├── Roles                      ├── Roles (with scopes)
├── Policies (JSON)            ├── Role Definitions (JSON)
└── Resource-based policies    └── Azure Policy (compliance)
```

## Monitoring & Management

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| CloudWatch | Azure Monitor | Monitor includes Application Insights |
| CloudTrail | Azure Activity Log | Audit logging |
| X-Ray | Application Insights | Distributed tracing |
| Config | Azure Policy | Compliance and governance |
| Systems Manager | Azure Automation | Runbooks and configuration |
| CloudFormation | ARM Templates / Bicep | Bicep is more readable |
| Organizations | Management Groups | Hierarchy for subscriptions |
| Control Tower | Azure Landing Zones | Multi-account governance |

## Container & Orchestration

| AWS Service | Azure Equivalent | Notes |
|-------------|------------------|-------|
| ECR | Azure Container Registry | Container image storage |
| ECS | Azure Container Instances | Simple container deployment |
| EKS | Azure Kubernetes Service | Managed Kubernetes |
| Fargate | ACI / AKS Virtual Nodes | Serverless containers |
| App Mesh | Azure Service Mesh (OSM) | Service mesh capabilities |

## Analytics & Big Data

| AWS Service | Azure Equivalent | Notes |
|-------------|------------------|-------|
| EMR | Azure HDInsight | Managed Hadoop/Spark |
| Kinesis | Event Hubs | Real-time streaming |
| Athena | Azure Synapse (Serverless SQL) | Query data in storage |
| Glue | Azure Data Factory | ETL and data integration |
| QuickSight | Power BI | Business intelligence |
| Lake Formation | Azure Purview | Data governance |
| Data Pipeline | Azure Data Factory | Data workflows |

## AI/ML Services

| AWS Service | Azure Equivalent | Notes |
|-------------|------------------|-------|
| SageMaker | Azure Machine Learning | ML platform |
| Rekognition | Azure Computer Vision | Image analysis |
| Polly | Azure Speech | Text-to-speech |
| Lex | Azure Bot Service | Chatbots |
| Comprehend | Azure Text Analytics | NLP services |
| Bedrock | Azure OpenAI Service | Generative AI |

## Messaging & Integration

| AWS Service | Azure Equivalent | Notes |
|-------------|------------------|-------|
| SQS | Azure Queue Storage / Service Bus | Service Bus for enterprise |
| SNS | Azure Service Bus Topics | Pub/sub messaging |
| EventBridge | Azure Event Grid | Event-driven architecture |
| Step Functions | Azure Logic Apps / Durable Functions | Workflow orchestration |
| MQ | Azure Service Bus | Message broker |

## Key Terminology Differences

| AWS Term | Azure Term |
|----------|------------|
| Region | Region |
| Availability Zone | Availability Zone |
| Account | Subscription |
| Resource Group | Resource Group |
| VPC | Virtual Network (VNet) |
| Security Group | Network Security Group (NSG) |
| NACL | NSG (subnet-level rules) |
| AMI | VM Image |
| Instance | Virtual Machine |
| S3 Bucket | Storage Account + Container |
| IAM Role | Managed Identity |
| Tags | Tags |

## Pricing Model Comparison

| Concept | AWS | Azure |
|---------|-----|-------|
| Free Tier | 12 months + always free | 12 months + always free |
| Reserved | 1 or 3 years | 1 or 3 years |
| Spot/Preemptible | Spot Instances | Spot VMs |
| Savings Plans | Compute Savings Plans | Azure Reservations |
| Support Plans | Basic/Developer/Business/Enterprise | Basic/Developer/Standard/Professional Direct/Premier |

## Quick Reference: Common Tasks

### Creating a Web Application

**AWS:**
1. Launch EC2 or use Elastic Beanstalk
2. Configure ALB for load balancing
3. Use RDS for database
4. S3 for static assets
5. CloudFront for CDN
6. Route 53 for DNS

**Azure:**
1. Create App Service or VM
2. Configure Application Gateway
3. Use Azure SQL Database
4. Blob Storage for static assets
5. Azure CDN or Front Door
6. Azure DNS + Traffic Manager

### Setting Up a Kubernetes Cluster

**AWS (EKS):**
```bash
eksctl create cluster --name my-cluster --region us-west-2
```

**Azure (AKS):**
```bash
az aks create --resource-group myRG --name myAKSCluster --node-count 3
```

---

*Next: See [02-cloud-architect-interview-topics.md](02-cloud-architect-interview-topics.md) for interview preparation*
