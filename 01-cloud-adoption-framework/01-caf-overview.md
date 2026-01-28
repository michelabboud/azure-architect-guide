# Cloud Adoption Framework Deep Dive

## Framework Philosophy

The Cloud Adoption Framework provides a structured approach to cloud adoption that balances speed with governance. Unlike ad-hoc cloud adoption, CAF ensures organizations build a sustainable cloud foundation.

## Detailed Phase Guide

### Strategy Phase Deep Dive

#### Motivation Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CLOUD ADOPTION MOTIVATIONS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   CRITICAL BUSINESS EVENTS                                                  │
│   ─────────────────────────                                                 │
│   • Datacenter exit/lease expiration                                        │
│   • Mergers & acquisitions                                                  │
│   • End of support for existing systems                                     │
│   • Regulatory compliance requirements                                      │
│   • Business continuity requirements                                        │
│                                                                              │
│   MIGRATION MOTIVATIONS                                                     │
│   ──────────────────────                                                    │
│   • Cost savings (CAPEX → OPEX)                                            │
│   • Reduction in complexity                                                 │
│   • Operational optimization                                                │
│   • Agility and speed to market                                            │
│   • Global reach                                                            │
│                                                                              │
│   INNOVATION MOTIVATIONS                                                    │
│   ──────────────────────                                                    │
│   • AI/ML capabilities                                                      │
│   • IoT scenarios                                                           │
│   • New application development                                             │
│   • Customer experience improvements                                        │
│   • Product/service differentiation                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### Business Case Template

```markdown
## Cloud Business Case

### Executive Summary
[One paragraph summary of the cloud adoption initiative]

### Current State
- Infrastructure costs: $X/year
- Operational overhead: Y FTEs
- Time to market: Z weeks
- Technical debt: [High/Medium/Low]

### Proposed Future State
- Target architecture: [Landing Zone + Workloads]
- Migration timeline: [Months]
- Expected benefits: [List]

### Financial Analysis

| Category | Current | Year 1 | Year 2 | Year 3 |
|----------|---------|--------|--------|--------|
| Infrastructure | $X | $Y | $Y | $Y |
| Operations | $X | $Y | $Y | $Y |
| Migration Costs | - | $Z | - | - |
| **Total** | $X | $Y | $Y | $Y |

### Risk Assessment
1. [Risk 1] - Mitigation: [Strategy]
2. [Risk 2] - Mitigation: [Strategy]

### Recommendation
[Proceed/Defer/Alternative]
```

### Plan Phase Deep Dive

#### Digital Estate Assessment

The digital estate assessment involves inventorying all IT assets and classifying them for cloud migration.

**Workload Classification:**

| Category | Criteria | Migration Approach |
|----------|----------|-------------------|
| Tier 1 | Mission-critical, high availability | Careful planning, parallel run |
| Tier 2 | Important business systems | Standard migration |
| Tier 3 | Development/test | Quick migration |
| Tier 4 | End-of-life systems | Retire or replace |

**Assessment Tools:**

```bash
# Azure Migrate for server assessment
az migrate project create \
    --name "assessment-project" \
    --resource-group "rg-migrate" \
    --location "eastus"

# Create assessment
az migrate assessment create \
    --project-name "assessment-project" \
    --resource-group "rg-migrate" \
    --name "myassessment" \
    --group-name "mygroup" \
    --azure-location "eastus" \
    --vm-size "Standard"
```

#### Skills Readiness

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SKILLS ASSESSMENT MATRIX                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Role                  Current Skills         Azure Skills Needed          │
│   ────                  ──────────────         ────────────────────          │
│                                                                              │
│   Infrastructure        VMware, Windows        Azure IaaS, ARM, Bicep        │
│   Admin                 Server, Active         Entra ID, Azure AD DS         │
│                         Directory                                            │
│                                                                              │
│   Database Admin        SQL Server,            Azure SQL, Cosmos DB,         │
│                         Oracle                 Data Migration                │
│                                                                              │
│   Developer             .NET, Java             Azure PaaS, Functions,        │
│                         On-prem                Container Apps                │
│                                                                              │
│   Security              Firewall, VPN,         Defender, Sentinel,           │
│                         On-prem SIEM           Entra ID Security             │
│                                                                              │
│   Network               Cisco, routing         Azure Networking, VPN,        │
│                         On-prem                ExpressRoute, NSG             │
│                                                                              │
│   Training Path: Microsoft Learn → AZ-104 → AZ-305 → Specialty             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Ready Phase Deep Dive

#### Management Group Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT GROUP HIERARCHY                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Tenant Root Group                                                         │
│   └── Contoso (Intermediate Root)                                           │
│       │                                                                      │
│       ├── Platform                                                          │
│       │   ├── Management                                                    │
│       │   │   └── Sub: Management-Prod                                     │
│       │   │       ├── Log Analytics workspace                              │
│       │   │       ├── Automation account                                   │
│       │   │       └── Azure Monitor resources                              │
│       │   │                                                                 │
│       │   ├── Connectivity                                                  │
│       │   │   └── Sub: Connectivity-Prod                                   │
│       │   │       ├── Hub VNet                                             │
│       │   │       ├── Azure Firewall                                       │
│       │   │       ├── ExpressRoute/VPN                                     │
│       │   │       └── Azure DNS zones                                      │
│       │   │                                                                 │
│       │   └── Identity                                                      │
│       │       └── Sub: Identity-Prod                                       │
│       │           ├── Azure AD DS (if needed)                              │
│       │           └── Domain controllers                                   │
│       │                                                                     │
│       ├── Landing Zones                                                     │
│       │   ├── Corp                                                          │
│       │   │   ├── Sub: HR-Prod                                             │
│       │   │   ├── Sub: Finance-Prod                                        │
│       │   │   └── Sub: ERP-Prod                                            │
│       │   │                                                                 │
│       │   └── Online                                                        │
│       │       ├── Sub: PublicWeb-Prod                                      │
│       │       └── Sub: CustomerPortal-Prod                                 │
│       │                                                                     │
│       ├── Sandbox                                                           │
│       │   └── Sub: Sandbox-Dev                                             │
│       │                                                                     │
│       └── Decommissioned                                                    │
│           └── (Retired subscriptions)                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Create Management Group Hierarchy:**

```bash
# Create management groups
az account management-group create --name "Contoso"
az account management-group create --name "Platform" --parent "Contoso"
az account management-group create --name "Management" --parent "Platform"
az account management-group create --name "Connectivity" --parent "Platform"
az account management-group create --name "Identity" --parent "Platform"
az account management-group create --name "LandingZones" --parent "Contoso"
az account management-group create --name "Corp" --parent "LandingZones"
az account management-group create --name "Online" --parent "LandingZones"
az account management-group create --name "Sandbox" --parent "Contoso"

# Move subscription to management group
az account management-group subscription add \
    --name "Corp" \
    --subscription "HR-Prod-SubId"
```

### Migrate Phase Deep Dive

#### Migration Methodologies

**Azure Migrate Methodology:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MIGRATION METHODOLOGY                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. DISCOVER                                                               │
│   ───────────                                                               │
│   • Deploy Azure Migrate appliance                                          │
│   • Discover servers, databases, web apps                                   │
│   • Collect performance data (30+ days recommended)                         │
│   • Map dependencies                                                        │
│                                                                              │
│   2. ASSESS                                                                 │
│   ────────                                                                  │
│   • Generate Azure readiness reports                                        │
│   • Right-size recommendations                                              │
│   • Cost estimates                                                          │
│   • Identify migration blockers                                             │
│                                                                              │
│   3. MIGRATE                                                                │
│   ────────                                                                  │
│   • Replicate workloads to Azure                                           │
│   • Test migrations                                                         │
│   • Cutover with minimal downtime                                          │
│   • Validate functionality                                                  │
│                                                                              │
│   4. OPTIMIZE                                                               │
│   ──────────                                                                │
│   • Right-size post-migration                                              │
│   • Implement reserved instances                                           │
│   • Enable monitoring                                                       │
│   • Modernize incrementally                                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Database Migration Paths:**

| Source | Target | Tool | Downtime |
|--------|--------|------|----------|
| SQL Server on-prem | Azure SQL Database | DMS | Minimal |
| SQL Server on-prem | Azure SQL MI | DMS | Minimal |
| Oracle | Azure PostgreSQL | Ora2Pg + DMS | Extended |
| MySQL | Azure MySQL | DMS | Minimal |
| MongoDB | Cosmos DB | Native tools | Minimal |

### Innovate Phase Deep Dive

#### Cloud-Native Development

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      CLOUD-NATIVE PATTERNS                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Pattern: Microservices                                                    │
│   ─────────────────────                                                     │
│   Services: Azure Kubernetes Service (AKS)                                  │
│            Azure Container Apps                                             │
│            API Management                                                   │
│   Benefits: Independent scaling, deployment, technology choices             │
│                                                                              │
│   Pattern: Event-Driven Architecture                                        │
│   ──────────────────────────────────                                        │
│   Services: Azure Event Grid                                                │
│            Azure Event Hubs                                                 │
│            Azure Functions                                                  │
│   Benefits: Loose coupling, scalability, real-time processing              │
│                                                                              │
│   Pattern: Serverless                                                       │
│   ───────────────────                                                       │
│   Services: Azure Functions                                                 │
│            Logic Apps                                                       │
│            Durable Functions                                                │
│   Benefits: No infrastructure management, pay-per-execution                │
│                                                                              │
│   Pattern: Data Lakehouse                                                   │
│   ────────────────────────                                                  │
│   Services: Azure Synapse Analytics                                         │
│            Azure Databricks                                                 │
│            Data Lake Storage                                                │
│   Benefits: Unified analytics, AI/ML integration                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Govern Phase Deep Dive

#### Azure Policy at Scale

**Policy Assignment Hierarchy:**

```bicep
// Deny public blob access at management group level
targetScope = 'managementGroup'

resource policyAssignment 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'deny-public-blob'
  properties: {
    displayName: 'Deny public blob access'
    policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/4fa4b6c0-31ca-4c0d-b10d-24b96f62a751'
    enforcementMode: 'Default'
    parameters: {}
  }
}
```

**Custom Policy for Tagging:**

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "anyOf": [
        {
          "field": "tags['Environment']",
          "exists": "false"
        },
        {
          "field": "tags['CostCenter']",
          "exists": "false"
        },
        {
          "field": "tags['Owner']",
          "exists": "false"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

**Governance Maturity Model:**

| Level | Characteristics | Policies |
|-------|----------------|----------|
| Initial | Ad-hoc | Minimal or none |
| Developing | Basic controls | Cost alerts, required tags |
| Defined | Standardized | Network controls, security baseline |
| Managed | Measured | Compliance reporting, automation |
| Optimizing | Continuous improvement | AI-powered recommendations |

### Manage Phase Deep Dive

#### Operations Baseline

```bicep
// Deploy monitoring baseline
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'law-${environment}-${location}'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 90
  }
}

resource actionGroup 'Microsoft.Insights/actionGroups@2023-01-01' = {
  name: 'ag-critical-alerts'
  location: 'global'
  properties: {
    groupShortName: 'Critical'
    enabled: true
    emailReceivers: [
      {
        name: 'OpsTeam'
        emailAddress: 'ops@company.com'
        useCommonAlertSchema: true
      }
    ]
  }
}

// VM insights solution
resource vmInsights 'Microsoft.OperationsManagement/solutions@2015-11-01-preview' = {
  name: 'VMInsights(${logAnalytics.name})'
  location: location
  properties: {
    workspaceResourceId: logAnalytics.id
  }
  plan: {
    name: 'VMInsights(${logAnalytics.name})'
    publisher: 'Microsoft'
    product: 'OMSGallery/VMInsights'
    promotionCode: ''
  }
}
```

## CAF vs AWS Comparison

| CAF Phase | AWS Equivalent | Key Differences |
|-----------|---------------|-----------------|
| Strategy | Cloud Adoption Readiness | Azure has more detailed business case templates |
| Plan | Migration Readiness Assessment | Azure Migrate more integrated |
| Ready | Landing Zone | Azure Landing Zone more prescriptive |
| Migrate | Application Discovery Service | Similar tooling, different services |
| Innovate | Well-Architected Framework | Azure CAF includes innovation guidance |
| Secure | Security Pillar | Azure integrates Zero Trust more deeply |
| Govern | Control Tower | Azure Policy more flexible |
| Manage | Systems Manager | Azure Arc extends hybrid management |

---

*Continue to [Landing Zones](02-landing-zones.md)*

*Back to [Chapter Overview](README.md)*
