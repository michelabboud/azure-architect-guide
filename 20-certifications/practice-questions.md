# Azure Certification Practice Questions

> *"The expert in anything was once a beginner."* — Helen Hayes
>
> *Also, the expert probably failed a few practice tests first. That's normal.*

---

## How to Use This Guide

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       PRACTICE QUESTION STRATEGY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. ATTEMPT FIRST                                                           │
│     Don't peek at answers! Write down your reasoning.                       │
│                                                                              │
│  2. CHECK YOUR ANSWER                                                       │
│     Compare with the explanation provided.                                  │
│                                                                              │
│  3. UNDERSTAND THE WHY                                                      │
│     Even if correct, read the explanation.                                  │
│                                                                              │
│  4. NOTE KNOWLEDGE GAPS                                                     │
│     Track topics you struggle with.                                         │
│                                                                              │
│  5. REVISIT WEAK AREAS                                                      │
│     Review relevant chapters before retrying.                               │
│                                                                              │
│  TARGET SCORE: 80%+ before exam                                             │
│  PASSING SCORE: 700/1000 (approximately 70%)                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Section 1: Identity & Access (AZ-305 Focus)

### Question 1.1

**Your company is migrating from AWS to Azure. In AWS, you used IAM roles attached to EC2 instances to access S3 buckets. What is the equivalent pattern in Azure?**

A) Service principals with client secrets stored in Key Vault
B) Managed identities assigned to VMs accessing Blob Storage
C) User accounts with RBAC assignments
D) Shared access signatures (SAS) tokens

<details>
<summary>Click for Answer</summary>

**Answer: B** — Managed identities assigned to VMs accessing Blob Storage

**Explanation:**
- AWS EC2 Instance Roles = Azure Managed Identities
- Both provide automatic credential rotation
- Both allow code to access resources without stored credentials
- Managed identities are the Azure best practice for service-to-service auth

**Why not the others:**
- A) Service principals work but require secret management
- C) User accounts should never be used for service auth
- D) SAS tokens are for temporary access, not ongoing service auth

**AWS → Azure Mapping:**
```
AWS IAM Role + EC2 Instance Profile → Azure Managed Identity + VM
AWS IAM Policy                     → Azure RBAC Role Assignment
AWS S3 Bucket                      → Azure Blob Storage Container
```

</details>

---

### Question 1.2

**A healthcare company requires that all Azure portal access must come from the corporate network OR from devices enrolled in their MDM solution. Which technology should you implement?**

A) Azure AD Multi-Factor Authentication only
B) Network Security Groups on all resources
C) Conditional Access policies with named locations and device compliance
D) Azure Firewall with application rules

<details>
<summary>Click for Answer</summary>

**Answer: C** — Conditional Access policies with named locations and device compliance

**Explanation:**
Conditional Access is the "if-then" security policy engine:
- IF user tries to access Azure portal
- AND location is NOT corporate network
- AND device is NOT compliant
- THEN block access (or require additional verification)

**Configuration Overview:**
```
Conditional Access Policy:
├── Assignments
│   ├── Users: All users
│   ├── Cloud apps: Microsoft Azure Management
│   └── Conditions:
│       ├── Locations: Exclude corporate IPs
│       └── Device state: Require compliant device
└── Access controls
    └── Grant: Require one of the selected controls
        ├── Compliant device
        └── Named location
```

**Why not the others:**
- A) MFA helps but doesn't enforce location/device requirements
- B) NSGs work at network level, not identity level
- D) Azure Firewall doesn't control portal access

</details>

---

### Question 1.3

**Your company has 50 Azure subscriptions across 3 departments. You need to ensure that all subscriptions follow a consistent RBAC model where department admins can manage their resources but cannot affect other departments. How should you structure this?**

A) Create 50 separate Entra ID tenants
B) Use management groups with inherited RBAC assignments
C) Assign Owner role to department admins on each subscription manually
D) Create custom roles that deny cross-department access

<details>
<summary>Click for Answer</summary>

**Answer: B** — Use management groups with inherited RBAC assignments

**Explanation:**
```
Root Management Group
├── Finance Department (MG)
│   ├── RBAC: Finance-Admins → Contributor
│   ├── finance-prod-001 (subscription)
│   └── finance-dev-001 (subscription)
│
├── Engineering Department (MG)
│   ├── RBAC: Eng-Admins → Contributor
│   ├── eng-prod-001 (subscription)
│   └── eng-dev-001 (subscription)
│
└── Marketing Department (MG)
    ├── RBAC: Marketing-Admins → Contributor
    └── marketing-001 (subscription)
```

RBAC assigned at management group level automatically inherits down to all subscriptions.

**Why not the others:**
- A) Multiple tenants is overkill and creates identity silos
- C) Manual assignment doesn't scale and is error-prone
- D) Custom deny roles are complex; just don't grant access

</details>

---

## Section 2: Networking (AZ-305 & AZ-104)

### Question 2.1

**You're designing a hub-spoke network for an enterprise. The hub VNet needs to allow spoke VNets to communicate with each other through a central Azure Firewall. Which of the following is required?**

(Select all that apply)

A) User-defined routes (UDRs) on spoke subnets pointing to the firewall
B) VNet peering between all spoke VNets
C) Enable "Allow gateway transit" on hub peering
D) UDRs on the GatewaySubnet in the hub
E) Azure Firewall network rules allowing spoke-to-spoke traffic

<details>
<summary>Click for Answer</summary>

**Answers: A, D, E**

**Explanation:**

**A) UDRs on spoke subnets** ✅
```
Route Table: spoke1-rt
├── 10.2.0.0/16 (Spoke2) → 10.0.1.4 (Firewall)
├── 10.3.0.0/16 (Spoke3) → 10.0.1.4 (Firewall)
└── 0.0.0.0/0           → 10.0.1.4 (Firewall)
```

**B) VNet peering between spokes** ❌
- Not needed! Traffic goes Spoke → Hub (Firewall) → Spoke
- Direct spoke-to-spoke peering bypasses the firewall

**C) Allow gateway transit** ❌
- Only needed for VPN/ExpressRoute gateway sharing
- Not related to spoke-to-spoke routing

**D) UDRs on GatewaySubnet** ✅
- Needed for asymmetric routing prevention
- Ensures return traffic from on-premises goes through firewall

**E) Firewall rules** ✅
- Firewall must explicitly allow the traffic
- "Route to firewall" ≠ "Firewall allows it"

**The Traffic Flow:**
```
Spoke1 VM (10.1.1.10)
    │
    │ UDR: 10.2.0.0/16 → 10.0.1.4
    ▼
Hub VNet Peering
    │
    ▼
Azure Firewall (10.0.1.4)
    │
    │ Network Rule: Allow 10.1.0.0/16 → 10.2.0.0/16
    ▼
Hub VNet Peering
    │
    ▼
Spoke2 VM (10.2.1.10)
```

</details>

---

### Question 2.2

**A company needs private connectivity to Azure SQL Database from their VNet. They also require that on-premises servers can reach the database over ExpressRoute. Which solution should you implement?**

A) Service Endpoints for Azure SQL
B) Private Endpoint for Azure SQL with Private DNS Zone
C) VNet injection for Azure SQL
D) Azure SQL Managed Instance

<details>
<summary>Click for Answer</summary>

**Answer: B** — Private Endpoint for Azure SQL with Private DNS Zone

**Explanation:**

| Requirement | Service Endpoint | Private Endpoint |
|-------------|------------------|------------------|
| Private connectivity from VNet | ✅ | ✅ |
| Private connectivity from on-prem | ❌ | ✅ |
| Private IP in your VNet | ❌ | ✅ |
| Eliminates public endpoint | ❌ | ✅ |

**Why Private Endpoint wins:**
```
┌─────────────────────────────────────────────────────────────────┐
│                                                                  │
│  On-Premises          ExpressRoute          Azure VNet          │
│  ────────────         ───────────           ──────────          │
│                                                                  │
│  App Server ─────────────────────────────► Private Endpoint     │
│  10.100.1.5                                 10.0.5.4            │
│                                                  │               │
│                                                  ▼               │
│                                            Azure SQL            │
│                                            (No public endpoint) │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**DNS Configuration:**
```
Private DNS Zone: privatelink.database.windows.net
├── A Record: myserver → 10.0.5.4
└── Linked to: All VNets that need access

On-premises DNS:
├── Conditional forwarder: *.database.windows.net
└── Forward to: Azure DNS Private Resolver (10.0.10.4)
```

**Why not the others:**
- A) Service Endpoints don't work over ExpressRoute
- C) VNet injection isn't available for Azure SQL Database
- D) Managed Instance is different service, not relevant here

</details>

---

### Question 2.3

**You need to expose a web application running in AKS to the internet with the following requirements:**
- **DDoS protection**
- **Web Application Firewall**
- **SSL termination**
- **Path-based routing**

**What combination should you use?**

A) Azure Load Balancer Standard + Azure DDoS Protection
B) Application Gateway with WAF + Azure DDoS Protection
C) Azure Front Door Standard
D) Azure Front Door Premium

<details>
<summary>Click for Answer</summary>

**Answer: D** — Azure Front Door Premium

**Explanation:**

| Requirement | Load Balancer | App Gateway WAF | Front Door Standard | Front Door Premium |
|-------------|---------------|-----------------|--------------------|--------------------|
| DDoS protection | Basic only | Standard plan | Built-in | Built-in |
| WAF | ❌ | ✅ | Basic rules | Custom + Managed |
| SSL termination | ❌ | ✅ | ✅ | ✅ |
| Path-based routing | ❌ | ✅ | ✅ | ✅ |
| Global anycast | ❌ | ❌ | ✅ | ✅ |
| Bot protection | ❌ | ❌ | ❌ | ✅ |

**Why Front Door Premium over Standard:**
- Advanced WAF rules including bot protection
- Better protection for production workloads
- Private Link support for backends (if needed)

**Why Front Door over App Gateway:**
- Global presence (anycast) = faster for global users
- Built-in DDoS (no separate DDoS plan needed)
- CDN capabilities included

**Architecture:**
```
Internet Users (Global)
        │
        ▼
Azure Front Door Premium
├── WAF Policy (Prevention mode)
├── DDoS Protection (Built-in)
├── SSL Termination
└── Route: /api/* → AKS Ingress
    Route: /static/* → Storage Account
        │
        ▼
AKS Cluster (Backend)
```

</details>

---

## Section 3: Compute & Containers

### Question 3.1

**You're designing a solution for a microservices application with the following requirements:**
- **100 microservices**
- **Auto-scaling based on HTTP requests and queue depth**
- **Developers should not manage Kubernetes**
- **Scale to zero when idle**

**Which service should you recommend?**

A) Azure Kubernetes Service (AKS)
B) Azure Container Apps
C) Azure Container Instances
D) Azure App Service

<details>
<summary>Click for Answer</summary>

**Answer: B** — Azure Container Apps

**Explanation:**

| Requirement | AKS | Container Apps | ACI | App Service |
|-------------|-----|----------------|-----|-------------|
| 100+ services | ✅ | ✅ | ⚠️ Complex | ⚠️ Costly |
| HTTP auto-scale | Manual HPA | ✅ Built-in | ❌ | ✅ |
| Queue-based scale | KEDA manual | ✅ KEDA built-in | ❌ | ❌ |
| No K8s management | ❌ | ✅ | ✅ | ✅ |
| Scale to zero | ❌ | ✅ | ✅ | ❌ |

**Container Apps is the sweet spot:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    CONTAINER APPS ENVIRONMENT                    │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Service A          Service B          Service C         │    │
│  │  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐    │    │
│  │  │ Replicas: 0 │   │ Replicas: 5 │   │ Replicas: 2 │    │    │
│  │  │ (Idle)      │   │ (Busy)      │   │ (Moderate)  │    │    │
│  │  └─────────────┘   └─────────────┘   └─────────────┘    │    │
│  │                                                          │    │
│  │  Scaling Rules:                                          │    │
│  │  • HTTP: concurrent requests > 10 → scale up            │    │
│  │  • Queue: messages > 100 → scale up                     │    │
│  │  • Min replicas: 0 (scale to zero!)                     │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  COST: Pay only for running replicas                            │
│  (Idle services = $0)                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Why not AKS:**
- Requires Kubernetes expertise
- Minimum 1 node always running (no scale to zero for nodes)
- More operational overhead

</details>

---

### Question 3.2

**Your team is migrating from AWS EKS to Azure. They need to keep using Kubernetes manifests and Helm charts without changes. What networking plugin should you choose?**

A) Kubenet (basic)
B) Azure CNI
C) Azure CNI Overlay
D) Bring your own CNI

<details>
<summary>Click for Answer</summary>

**Answer: B** — Azure CNI

**Explanation:**

Azure CNI gives pods "real" Azure VNet IPs, which provides the most AWS-VPC-CNI-like experience:

```
AWS VPC CNI:                          Azure CNI:
────────────                          ──────────
Pods get VPC IPs                      Pods get VNet IPs
10.0.1.50 (Pod A)                     10.0.1.50 (Pod A)
10.0.1.51 (Pod B)                     10.0.1.51 (Pod B)
    │                                     │
    ▼                                     ▼
Native VPC networking                 Native VNet networking
```

| Feature | Kubenet | Azure CNI | CNI Overlay |
|---------|---------|-----------|-------------|
| Pod IPs from VNet | ❌ | ✅ | ❌ (overlay) |
| Direct pod-to-VM communication | ❌ | ✅ | ❌ |
| IP address usage | Low | High | Low |
| Network policies | Limited | Full | Full |
| Windows containers | ❌ | ✅ | ✅ |
| AWS CNI similarity | Low | High | Medium |

**Consideration - IP Planning:**
```
Azure CNI IP calculation:
(Max pods per node) × (Number of nodes) + (Number of nodes)

Example:
30 pods × 10 nodes + 10 nodes = 310 IPs needed
Recommended subnet: /23 (510 IPs) for growth
```

</details>

---

## Section 4: Data & Storage

### Question 4.1

**A retail company needs to store 50TB of product catalog data with the following requirements:**
- **Single-digit millisecond reads globally**
- **Multi-region writes** (US, Europe, Asia)
- **Automatic conflict resolution**
- **Current data model: DynamoDB with partition key + sort key**

**Which Azure service and API should you recommend?**

A) Cosmos DB with SQL API
B) Cosmos DB with Table API
C) Azure SQL with geo-replication
D) Cosmos DB with MongoDB API

<details>
<summary>Click for Answer</summary>

**Answer: A** — Cosmos DB with SQL API

**Explanation:**

**Why Cosmos DB SQL API:**
```
DynamoDB to Cosmos DB Mapping:
──────────────────────────────

DynamoDB                        Cosmos DB (SQL API)
─────────                       ────────────────────
Table                    →      Container
Partition Key            →      Partition Key
Sort Key                 →      Any property (flexible!)
Item                     →      Document
Global Tables            →      Multi-region writes

SQL API Advantages:
• Rich query language (unlike Table API)
• Familiar syntax for developers
• Better tooling and SDK support
• Same partition key concept as DynamoDB
```

**Multi-region write configuration:**
```json
{
  "locations": [
    { "locationName": "East US", "failoverPriority": 0 },
    { "locationName": "West Europe", "failoverPriority": 1 },
    { "locationName": "Southeast Asia", "failoverPriority": 2 }
  ],
  "enableMultipleWriteLocations": true,
  "defaultConsistencyLevel": "Session",
  "conflictResolutionPolicy": {
    "mode": "LastWriterWins",
    "conflictResolutionPath": "/_ts"
  }
}
```

**Why not the others:**
- B) Table API is too simplistic for complex queries
- C) Azure SQL doesn't support multi-region writes
- D) MongoDB API works but SQL API is simpler for DynamoDB migrations

</details>

---

### Question 4.2

**A financial services company stores audit logs that must be retained for 7 years. The logs are frequently accessed for the first 30 days, occasionally accessed for 1 year, and rarely accessed after that. What storage strategy minimizes cost?**

A) All data in Hot tier with lifecycle management
B) Hot tier (30 days) → Cool tier (1 year) → Archive tier (7 years)
C) All data in Cool tier
D) Hot tier (30 days) → Archive tier (7 years)

<details>
<summary>Click for Answer</summary>

**Answer: B** — Hot tier (30 days) → Cool tier (1 year) → Archive tier (7 years)

**Explanation:**

**Cost Analysis (per 100 TB):**
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STORAGE TIER COST COMPARISON                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Option A: All Hot                                                          │
│  ─────────────────                                                          │
│  Storage: $2,000/month × 84 months = $168,000                               │
│  Total: $168,000                                                            │
│                                                                              │
│  Option B: Hot → Cool → Archive (Recommended)                               │
│  ─────────────────────────────────────────────                              │
│  Hot (30 days):     $2,000 × 1 month  = $2,000                             │
│  Cool (11 months):  $1,000 × 11 months = $11,000                           │
│  Archive (6 years): $200 × 72 months  = $14,400                            │
│  Rehydration costs: ~$500/year        = $3,500                             │
│  Total: $30,900                                                             │
│                                                                              │
│  Option C: All Cool                                                         │
│  ─────────────────                                                          │
│  Storage: $1,000/month × 84 months = $84,000                                │
│  Total: $84,000                                                             │
│                                                                              │
│  Option D: Hot → Archive (Skip Cool)                                        │
│  ────────────────────────────────────                                       │
│  Hot (30 days):     $2,000 × 1 month = $2,000                              │
│  Archive (83 months): $200 × 83 months = $16,600                           │
│  Rehydration costs: ~$2,000/year = $14,000 (high! year 1 access)           │
│  Total: $32,600                                                             │
│                                                                              │
│  WINNER: Option B saves ~$137,000 over 7 years! 💰                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Lifecycle Management Policy:**
```json
{
  "rules": [
    {
      "name": "auditLogLifecycle",
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["audit-logs/"] },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 365 },
            "delete": { "daysAfterModificationGreaterThan": 2555 }
          }
        }
      }
    }
  ]
}
```

</details>

---

## Section 5: Scenario-Based Questions

### Question 5.1 (Multi-part)

**SCENARIO:**
Contoso Ltd is migrating a three-tier e-commerce application from AWS to Azure. The application currently uses:
- AWS ALB → EC2 Auto Scaling Group (web tier)
- EC2 → RDS PostgreSQL (app tier)
- S3 for static assets with CloudFront CDN

**Requirements:**
- 99.99% availability
- Disaster recovery to a secondary region
- PCI-DSS compliance for payment processing
- Minimize migration effort

---

**Question 5.1.1:** Which compute option requires the LEAST migration effort for the web and app tiers?

A) Azure Virtual Machines with VM Scale Sets
B) Azure App Service
C) Azure Kubernetes Service
D) Azure Container Apps

<details>
<summary>Click for Answer</summary>

**Answer: A** — Azure Virtual Machines with VM Scale Sets

**Explanation:**
"Minimize migration effort" = lift-and-shift

```
AWS → Azure Direct Mapping:
───────────────────────────

EC2 Auto Scaling Group → VM Scale Sets
     (same OS, same code, same deployment)

App Service → Requires code modifications (PaaS)
AKS → Requires containerization
Container Apps → Requires containerization
```

For a quick migration, VM Scale Sets provide:
- Same OS and runtime environment
- Similar scaling concepts
- Minimal code changes

</details>

---

**Question 5.1.2:** Which database service provides the BEST path for the RDS PostgreSQL migration?

A) Azure SQL Database
B) Azure Database for PostgreSQL - Flexible Server
C) Cosmos DB with PostgreSQL API
D) PostgreSQL on Azure VMs

<details>
<summary>Click for Answer</summary>

**Answer: B** — Azure Database for PostgreSQL - Flexible Server

**Explanation:**

```
RDS PostgreSQL → Azure Database for PostgreSQL (Flexible Server)
─────────────────────────────────────────────────────────────────

✓ Same database engine
✓ Same connection strings (mostly)
✓ Same SQL syntax
✓ PCI-DSS compliant
✓ Built-in HA and DR
✓ Azure Database Migration Service support

Migration Steps:
1. Create Azure PostgreSQL Flexible Server
2. Use Azure DMS for online migration
3. Validate data
4. Cutover (< 5 minutes downtime)
```

**Why not the others:**
- A) Azure SQL = different SQL dialect, significant code changes
- C) Cosmos DB PostgreSQL = different architecture, not compatible
- D) VMs = more management overhead, not recommended

</details>

---

**Question 5.1.3:** What combination provides the BEST solution for static assets with global delivery?

A) Azure Blob Storage + Azure CDN
B) Azure Blob Storage + Azure Front Door
C) Azure Files + Application Gateway
D) Azure NetApp Files + Traffic Manager

<details>
<summary>Click for Answer</summary>

**Answer: B** — Azure Blob Storage + Azure Front Door

**Explanation:**

```
AWS Architecture:                Azure Equivalent:
─────────────────                ─────────────────
S3                        →      Blob Storage
CloudFront               →      Azure Front Door
S3 Website Hosting       →      Static Website in Blob Storage
CloudFront OAI           →      Front Door + Private Endpoint

Why Front Door over CDN:
────────────────────────
• Built-in WAF (PCI-DSS requirement!)
• Better for dynamic + static content
• Private Link to storage (Premium SKU)
• One service for CDN + security
```

**Architecture:**
```
Internet → Azure Front Door (WAF) → Blob Storage (Private Endpoint)
              │                         │
              │                         └── Static assets
              │
              └── Web tier (VM Scale Sets)
```

</details>

---

## Answer Key & Score Tracker

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SCORE TRACKER                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Section 1: Identity & Access                                               │
│  □ Q1.1: _____  □ Q1.2: _____  □ Q1.3: _____                               │
│  Score: ___/3                                                                │
│                                                                              │
│  Section 2: Networking                                                       │
│  □ Q2.1: _____  □ Q2.2: _____  □ Q2.3: _____                               │
│  Score: ___/3                                                                │
│                                                                              │
│  Section 3: Compute & Containers                                            │
│  □ Q3.1: _____  □ Q3.2: _____                                              │
│  Score: ___/2                                                                │
│                                                                              │
│  Section 4: Data & Storage                                                  │
│  □ Q4.1: _____  □ Q4.2: _____                                              │
│  Score: ___/2                                                                │
│                                                                              │
│  Section 5: Scenario-Based                                                  │
│  □ Q5.1.1: _____ □ Q5.1.2: _____ □ Q5.1.3: _____                          │
│  Score: ___/3                                                                │
│                                                                              │
│  ─────────────────────────────────────────────────────────────────────────  │
│  TOTAL: ___/13                                                               │
│                                                                              │
│  INTERPRETATION:                                                             │
│  12-13: Ready for the exam! 🎉                                              │
│  10-11: Almost there, review weak areas                                      │
│  7-9:   Good foundation, need more practice                                  │
│  <7:    Review the chapters before retrying                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Topics to Review Based on Mistakes

| If you missed... | Review these chapters |
|------------------|----------------------|
| Q1.x (Identity) | Chapter 02: Identity |
| Q2.x (Networking) | Chapters 05, 06, 07 |
| Q3.x (Compute) | Chapter 08: Containers |
| Q4.x (Data) | Chapter 09: Databases |
| Q5.x (Scenarios) | Chapter 19: Case Studies |

---

*More practice questions: [AZ-305 Study Guide](az-305-study-guide.md)* | *[AZ-104 Study Guide](az-104-study-guide.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
