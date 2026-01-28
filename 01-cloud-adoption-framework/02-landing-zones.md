# Azure Landing Zones

## Overview

Azure Landing Zones provide a scalable, secure foundation for your cloud environment. They represent the target architecture for your cloud adoption efforts and implement best practices across security, governance, and operations.

## Landing Zone Conceptual Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  AZURE LANDING ZONE ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                        ┌─────────────────────────────────────┐              │
│                        │        AZURE AD TENANT             │              │
│                        │   • Entra ID (Identity)            │              │
│                        │   • Conditional Access             │              │
│                        │   • PIM for Just-in-Time           │              │
│                        └─────────────────────────────────────┘              │
│                                       │                                      │
│                        ┌──────────────┴──────────────┐                      │
│                        │    Management Groups         │                      │
│                        │  (Policy Inheritance)        │                      │
│                        └──────────────┬──────────────┘                      │
│                                       │                                      │
│          ┌────────────────────────────┼────────────────────────────┐        │
│          │                            │                            │        │
│          ▼                            ▼                            ▼        │
│   ┌─────────────────┐       ┌─────────────────┐       ┌─────────────────┐  │
│   │    PLATFORM     │       │  LANDING ZONES  │       │    SANDBOX      │  │
│   │                 │       │                 │       │                 │  │
│   │  ┌───────────┐  │       │  ┌───────────┐  │       │  Dev/Test       │  │
│   │  │Management │  │       │  │   Corp    │  │       │  Experiments    │  │
│   │  │• Monitor  │  │       │  │ Internal  │  │       │  POCs           │  │
│   │  │• Backup   │  │       │  │ workloads │  │       │                 │  │
│   │  │• Security │  │       │  └───────────┘  │       │  No prod data   │  │
│   │  └───────────┘  │       │                 │       │  Limited access │  │
│   │                 │       │  ┌───────────┐  │       │                 │  │
│   │  ┌───────────┐  │       │  │  Online   │  │       └─────────────────┘  │
│   │  │Connectiv. │  │       │  │ Internet- │  │                            │
│   │  │• Hub VNet │  │       │  │ facing    │  │                            │
│   │  │• Firewall │  │       │  └───────────┘  │                            │
│   │  │• ExpressR │  │       │                 │                            │
│   │  └───────────┘  │       └─────────────────┘                            │
│   │                 │                                                       │
│   │  ┌───────────┐  │                                                       │
│   │  │ Identity  │  │                                                       │
│   │  │• AD DS    │  │                                                       │
│   │  │• DNS      │  │                                                       │
│   │  └───────────┘  │                                                       │
│   │                 │                                                       │
│   └─────────────────┘                                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Eight Design Areas

### 1. Azure Billing and Entra Tenant

**Design Decisions:**

| Decision | Options | Recommendation |
|----------|---------|----------------|
| Billing structure | Enterprise Agreement, CSP | EA for large enterprises |
| Tenant model | Single vs Multi-tenant | Single for most scenarios |
| Department structure | Cost centers, business units | Align with organizational structure |

```bash
# View enrollment accounts
az billing account list

# View departments
az billing department list --account-name <ea-account>
```

### 2. Identity and Access Management

**Identity Architecture:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       IDENTITY ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        ENTRA ID                                      │   │
│   │                                                                      │   │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │   │
│   │   │   Users     │  │   Groups    │  │    Apps     │                │   │
│   │   │             │  │             │  │             │                │   │
│   │   │ • Employees │  │ • Security  │  │ • SaaS      │                │   │
│   │   │ • Guests    │  │ • M365      │  │ • Custom    │                │   │
│   │   │ • Service   │  │ • Dynamic   │  │ • Legacy    │                │   │
│   │   └─────────────┘  └─────────────┘  └─────────────┘                │   │
│   │                                                                      │   │
│   │   ┌───────────────────────────────────────────────────────────────┐ │   │
│   │   │                  CONDITIONAL ACCESS                            │ │   │
│   │   │  • MFA Requirements                                           │ │   │
│   │   │  • Device Compliance                                          │ │   │
│   │   │  • Location-based Access                                      │ │   │
│   │   │  • Risk-based Access                                          │ │   │
│   │   └───────────────────────────────────────────────────────────────┘ │   │
│   │                                                                      │   │
│   └──────────────────────────────┬──────────────────────────────────────┘   │
│                                  │                                          │
│              ┌───────────────────┴───────────────────┐                     │
│              │                                        │                     │
│              ▼                                        ▼                     │
│   ┌─────────────────────────┐           ┌─────────────────────────┐        │
│   │   AZURE RBAC            │           │   ON-PREMISES AD        │        │
│   │                         │           │                         │        │
│   │ • Management Groups     │  Sync via │ • Domain Services       │        │
│   │ • Subscriptions         │  ◄─────►  │ • Legacy Apps           │        │
│   │ • Resource Groups       │ AD Connect│ • File Servers          │        │
│   │ • Resources             │           │                         │        │
│   └─────────────────────────┘           └─────────────────────────┘        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**RBAC Best Practices:**

```bash
# Create custom role for Landing Zone operator
az role definition create --role-definition '{
  "Name": "Landing Zone Operator",
  "Description": "Can manage resources in landing zones",
  "Actions": [
    "Microsoft.Compute/*",
    "Microsoft.Network/*",
    "Microsoft.Storage/*",
    "Microsoft.Resources/*"
  ],
  "NotActions": [
    "Microsoft.Authorization/*/Delete",
    "Microsoft.Authorization/elevateAccess/Action"
  ],
  "AssignableScopes": ["/providers/Microsoft.Management/managementGroups/LandingZones"]
}'
```

### 3. Resource Organization

**Subscription Strategy:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SUBSCRIPTION STRATEGIES                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Strategy 1: Application-Centric                                           │
│   ─────────────────────────────────                                         │
│   • One subscription per application/workload                               │
│   • Clear cost attribution                                                  │
│   • Independent RBAC                                                        │
│   • Best for: Large enterprises with many applications                      │
│                                                                              │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                  │
│   │  CRM-Prod     │  │  ERP-Prod     │  │  Web-Prod     │                  │
│   │  Subscription │  │  Subscription │  │  Subscription │                  │
│   └───────────────┘  └───────────────┘  └───────────────┘                  │
│                                                                              │
│   Strategy 2: Environment-Based                                             │
│   ─────────────────────────────────                                         │
│   • Subscriptions by environment (prod, dev, test)                          │
│   • Simpler management                                                      │
│   • Best for: Smaller organizations                                         │
│                                                                              │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                  │
│   │  Production   │  │  Development  │  │     Test      │                  │
│   │  Subscription │  │  Subscription │  │  Subscription │                  │
│   └───────────────┘  └───────────────┘  └───────────────┘                  │
│                                                                              │
│   Strategy 3: Hybrid                                                        │
│   ────────────────────                                                      │
│   • Platform subscriptions (dedicated)                                      │
│   • Workload subscriptions (application + environment)                      │
│   • Best for: Most enterprise scenarios                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Naming Convention:**

```
Resource Type    | Pattern                        | Example
─────────────────────────────────────────────────────────────────────────────
Subscription     | sub-<workload>-<env>          | sub-crm-prod
Resource Group   | rg-<workload>-<env>-<region>  | rg-crm-prod-eastus
Virtual Network  | vnet-<workload>-<env>-<region>| vnet-crm-prod-eastus
Subnet           | snet-<purpose>                 | snet-web, snet-db
VM               | vm-<workload>-<role>-<###>    | vm-crm-web-001
Storage Account  | st<workload><env><unique>     | stcrmprodx7k2
Key Vault        | kv-<workload>-<env>           | kv-crm-prod
```

### 4. Network Topology and Connectivity

**Hub-Spoke Architecture:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       HUB-SPOKE NETWORK TOPOLOGY                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                           ┌─────────────────────────────────┐               │
│                           │         ON-PREMISES             │               │
│                           │    ┌───────────────────┐        │               │
│                           │    │   Datacenter      │        │               │
│                           │    │   10.0.0.0/8      │        │               │
│                           │    └─────────┬─────────┘        │               │
│                           └──────────────┼──────────────────┘               │
│                                          │                                   │
│                              ExpressRoute / S2S VPN                         │
│                                          │                                   │
│   ┌──────────────────────────────────────┼──────────────────────────────┐   │
│   │                                      │           AZURE               │   │
│   │                           ┌──────────┴──────────┐                   │   │
│   │                           │        HUB          │                   │   │
│   │                           │    10.100.0.0/16    │                   │   │
│   │                           │                     │                   │   │
│   │                           │ ┌─────────────────┐ │                   │   │
│   │                           │ │ Azure Firewall  │ │                   │   │
│   │                           │ │ 10.100.1.0/24   │ │                   │   │
│   │                           │ └─────────────────┘ │                   │   │
│   │                           │ ┌─────────────────┐ │                   │   │
│   │                           │ │ Gateway Subnet  │ │                   │   │
│   │                           │ │ 10.100.0.0/24   │ │                   │   │
│   │                           │ └─────────────────┘ │                   │   │
│   │                           │ ┌─────────────────┐ │                   │   │
│   │                           │ │    Bastion      │ │                   │   │
│   │                           │ │ 10.100.2.0/24   │ │                   │   │
│   │                           │ └─────────────────┘ │                   │   │
│   │                           └──────────┬──────────┘                   │   │
│   │                                      │                              │   │
│   │              ┌───────────────────────┼───────────────────────┐      │   │
│   │              │ VNet Peering          │           VNet Peering│      │   │
│   │              │                       │                       │      │   │
│   │    ┌─────────┴─────────┐   ┌────────┴────────┐   ┌─────────┴─────┐ │   │
│   │    │    SPOKE: Corp    │   │  SPOKE: Online  │   │ SPOKE: DevTest│ │   │
│   │    │   10.101.0.0/16   │   │  10.102.0.0/16  │   │ 10.103.0.0/16 │ │   │
│   │    │                   │   │                 │   │               │ │   │
│   │    │ • ERP App         │   │ • Public Web    │   │ • Dev VMs     │ │   │
│   │    │ • HR System       │   │ • API Gateway   │   │ • Test envs   │ │   │
│   │    │ • Internal Apps   │   │ • CDN Origin    │   │               │ │   │
│   │    └───────────────────┘   └─────────────────┘   └───────────────┘ │   │
│   │                                                                     │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Hub Network Bicep:**

```bicep
// Hub VNet with Firewall
param location string = resourceGroup().location
param hubAddressPrefix string = '10.100.0.0/16'

resource hubVnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'vnet-hub-${location}'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [hubAddressPrefix]
    }
    subnets: [
      {
        name: 'GatewaySubnet'
        properties: {
          addressPrefix: '10.100.0.0/24'
        }
      }
      {
        name: 'AzureFirewallSubnet'
        properties: {
          addressPrefix: '10.100.1.0/24'
        }
      }
      {
        name: 'AzureBastionSubnet'
        properties: {
          addressPrefix: '10.100.2.0/24'
        }
      }
    ]
  }
}

resource firewall 'Microsoft.Network/azureFirewalls@2023-05-01' = {
  name: 'fw-hub-${location}'
  location: location
  properties: {
    sku: {
      name: 'AZFW_VNet'
      tier: 'Standard'
    }
    ipConfigurations: [
      {
        name: 'fw-ipconfig'
        properties: {
          subnet: {
            id: hubVnet.properties.subnets[1].id
          }
          publicIPAddress: {
            id: firewallPublicIp.id
          }
        }
      }
    ]
  }
}
```

### 5. Security

**Security Baseline:**

| Control | Azure Service | Policy |
|---------|--------------|--------|
| Endpoint protection | Defender for Cloud | Deploy to all VMs |
| Network security | NSG, Firewall | Default deny |
| Data encryption | Storage encryption | Always enabled |
| Identity protection | Entra ID Protection | Risk-based access |
| Secret management | Key Vault | No plain-text secrets |
| Logging | Log Analytics | Centralized logging |

### 6. Management

**Monitoring Architecture:**

```bicep
// Central monitoring setup
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'law-platform-${location}'
  location: location
  properties: {
    sku: { name: 'PerGB2018' }
    retentionInDays: 365
    features: {
      enableLogAccessUsingOnlyResourcePermissions: true
    }
  }
}

// Diagnostic settings for all resources
resource diagnosticSetting 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'send-to-law'
  scope: hubVnet
  properties: {
    workspaceId: logAnalytics.id
    logs: [
      {
        categoryGroup: 'allLogs'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
      }
    ]
  }
}
```

### 7. Governance

**Policy Initiative for Landing Zones:**

```json
{
  "name": "landing-zone-governance",
  "displayName": "Landing Zone Governance Initiative",
  "description": "Policies for landing zone compliance",
  "policyDefinitions": [
    {
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/required-tags",
      "parameters": {}
    },
    {
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/allowed-locations",
      "parameters": {
        "listOfAllowedLocations": ["eastus", "westus2"]
      }
    },
    {
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/allowed-vm-sizes",
      "parameters": {}
    }
  ]
}
```

### 8. Platform Automation and DevOps

**GitOps for Landing Zones:**

```yaml
# Azure DevOps pipeline for landing zone deployment
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - infrastructure/**

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Validate
    jobs:
      - job: ValidateBicep
        steps:
          - task: AzureCLI@2
            inputs:
              azureSubscription: 'platform-connection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file infrastructure/main.bicep

  - stage: Deploy
    dependsOn: Validate
    jobs:
      - deployment: DeployLandingZone
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: AzureCLI@2
                  inputs:
                    azureSubscription: 'platform-connection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment mg create \
                        --management-group-id Contoso \
                        --location eastus \
                        --template-file infrastructure/main.bicep
```

## Landing Zone Accelerator

The Azure Landing Zone Accelerator provides automated deployment of enterprise-scale landing zones.

**Deployment Options:**

| Option | Method | Best For |
|--------|--------|----------|
| Portal | Azure Portal wizard | Quick start, evaluation |
| Bicep | Infrastructure as Code | Enterprise deployment |
| Terraform | HashiCorp Terraform | Multi-cloud consistency |
| ARM | ARM Templates | Existing ARM investments |

**Deploy with Bicep:**

```bash
# Clone the ALZ Bicep repository
git clone https://github.com/Azure/ALZ-Bicep.git

# Deploy management groups
az deployment tenant create \
  --location eastus \
  --template-file infra-as-code/bicep/modules/managementGroups/managementGroups.bicep \
  --parameters @infra-as-code/bicep/modules/managementGroups/parameters/managementGroups.parameters.json

# Deploy policies
az deployment mg create \
  --management-group-id Contoso \
  --location eastus \
  --template-file infra-as-code/bicep/modules/policy/definitions/customPolicyDefinitions.bicep
```

---

*Continue to [Governance](03-governance.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
