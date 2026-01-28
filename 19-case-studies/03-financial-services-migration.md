# Case Study 3: Financial Services Migration

## Executive Summary

**Company**: SecureWealth Asset Management
**Industry**: Financial Services (Regulated)
**Challenge**: Migrate from on-premises to Azure while maintaining PCI-DSS, SOX compliance
**Outcome**: Zero-downtime migration, passed all compliance audits, 35% infrastructure cost reduction

### The Business Context

SecureWealth manages $50B in assets with strict regulatory requirements. Their aging on-premises infrastructure couldn't scale for digital transformation initiatives. The migration had to:

1. Maintain 99.99% availability for trading systems
2. Pass PCI-DSS Level 1 and SOX audits
3. Enable new digital channels (mobile, API banking)
4. Implement zero-trust security architecture

## Architecture Overview

```
                           ┌────────────────────────────────────────────────────────────┐
                           │                    Azure Landing Zone                       │
                           │                  (Enterprise-Scale)                         │
                           └────────────────────────────────────────────────────────────┘
                                                      │
            ┌─────────────────────────────────────────┼─────────────────────────────────────────┐
            │                                         │                                         │
   ┌────────▼────────┐                      ┌────────▼────────┐                      ┌────────▼────────┐
   │   Platform      │                      │   Connectivity  │                      │   Identity      │
   │   Management    │                      │   Subscription  │                      │   Subscription  │
   └────────┬────────┘                      └────────┬────────┘                      └────────┬────────┘
            │                                        │                                        │
   ┌────────▼────────┐                      ┌────────▼────────┐                      ┌────────▼────────┐
   │ • Azure Policy  │                      │ • Hub VNet      │                      │ • Azure AD      │
   │ • Blueprints    │                      │ • ExpressRoute  │                      │ • Key Vault     │
   │ • Cost Mgmt     │                      │ • Azure FW      │                      │ • PIM           │
   │ • Log Analytics │                      │ • Bastion       │                      │ • MFA           │
   └─────────────────┘                      └────────┬────────┘                      └─────────────────┘
                                                     │
        ┌────────────────────────────────────────────┼────────────────────────────────────────────┐
        │                                            │                                            │
   ┌────▼────┐                               ┌──────▼──────┐                               ┌─────▼─────┐
   │ Corp    │                               │  Landing    │                               │  Online   │
   │ Spoke   │◄──────── VNet Peering ───────►│  Zone Hub   │◄──────── VNet Peering ───────►│  Spoke    │
   └────┬────┘                               └──────┬──────┘                               └─────┬─────┘
        │                                           │                                           │
   ┌────▼────────────────┐               ┌─────────▼─────────┐               ┌─────────────────▼────┐
   │  Corporate Apps     │               │   Azure Firewall  │               │   Public-Facing Apps │
   │  ├── Trading        │               │   ├── IDPS        │               │   ├── API Gateway    │
   │  ├── Risk Engine    │               │   ├── TLS Inspect │               │   ├── Web App        │
   │  └── Reporting      │               │   └── DNS Proxy   │               │   └── CDN            │
   └─────────────────────┘               └───────────────────┘               └──────────────────────┘
```

## AWS to Azure Service Mapping (Regulated Workloads)

| Component | AWS Equivalent | Azure Implementation | Compliance Feature |
|-----------|---------------|---------------------|-------------------|
| Network Isolation | VPC + TGW | Virtual WAN + Hub-Spoke | Isolated spoke VNets |
| Firewall | Network Firewall | Azure Firewall Premium | IDPS, TLS inspection |
| DDoS Protection | Shield Advanced | DDoS Protection Standard | SLA-backed protection |
| Secret Management | Secrets Manager | Key Vault (HSM-backed) | FIPS 140-2 Level 3 |
| Identity | IAM + Organizations | Azure AD + PIM | Just-in-time access |
| Logging | CloudTrail + Config | Activity Log + Policy | Immutable audit trail |
| Encryption | KMS | Key Vault + CMK | BYOK support |
| Compliance | Artifact | Compliance Manager | Built-in assessments |

## Key Architectural Decisions

### Decision 1: Landing Zone Pattern Selection

**The Debate:**

| Pattern | Pros | Cons |
|---------|------|------|
| **Start Small** | Quick deployment, iterative | Rework needed as scale grows |
| **Enterprise-Scale** | Production-ready, best practices | Complex initial setup |
| **Hub-Spoke Custom** | Full control | Maintenance burden |
| **Virtual WAN** | Simplified connectivity | Higher cost, less flexibility |

**Decision: Enterprise-Scale Landing Zone with Azure Firewall**

**Rationale:**
1. Regulatory requirements demand consistent policy enforcement
2. Multiple business units need isolated environments
3. Hybrid connectivity to on-premises data centers required
4. Built-in compliance policies accelerate audit preparation

**Bicep Implementation:**
```bicep
// main.bicep - Landing Zone Core
targetScope = 'managementGroup'

@description('Enterprise-Scale Landing Zone Configuration')
param parLandingZonePrefix string = 'secwealth'
param parLocation string = 'eastus'
param parConnectivitySubscriptionId string
param parIdentitySubscriptionId string

// Management Group Structure
module managementGroups 'modules/managementGroups.bicep' = {
  name: 'deploy-management-groups'
  params: {
    parTopLevelMgId: '${parLandingZonePrefix}-root'
    parPlatformMgId: '${parLandingZonePrefix}-platform'
    parLandingZonesMgId: '${parLandingZonePrefix}-landingzones'
    parCorpMgId: '${parLandingZonePrefix}-corp'
    parOnlineMgId: '${parLandingZonePrefix}-online'
  }
}

// Policy Assignments for Compliance
module policyAssignments 'modules/policyAssignments.bicep' = {
  name: 'deploy-policy-assignments'
  dependsOn: [managementGroups]
  params: {
    parTargetMgId: managementGroups.outputs.outRootMgId
    parPolicies: [
      {
        name: 'Enforce-Encryption-CMK'
        policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/0961003e-5a0a-4549-abde-af6a37f2724d'
        enforcementMode: 'Default'
      }
      {
        name: 'Enforce-TLS-1.2'
        policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/f0e6e85b-9b9f-4a4b-b67b-f730d42f1b0b'
        enforcementMode: 'Default'
      }
      {
        name: 'Deny-Public-IP'
        policyDefinitionId: '/providers/Microsoft.Authorization/policyDefinitions/6c112d4e-5bc7-47ae-a041-ea2d9dccd749'
        enforcementMode: 'Default'
        parameters: {
          listOfAllowedLocations: {
            value: ['eastus', 'westus2']
          }
        }
      }
      {
        name: 'Deploy-Diagnostics-LogAnalytics'
        policyDefinitionId: '/providers/Microsoft.Authorization/policySetDefinitions/7f89b1eb-583c-429a-8828-af049802c1d9'
        enforcementMode: 'Default'
      }
    ]
  }
}

// Hub Network with Azure Firewall
module hubNetwork 'modules/hubNetwork.bicep' = {
  name: 'deploy-hub-network'
  scope: subscription(parConnectivitySubscriptionId)
  params: {
    parHubNetworkName: '${parLandingZonePrefix}-hub-vnet'
    parLocation: parLocation
    parHubAddressPrefix: '10.0.0.0/16'
    parAzFirewallSubnetPrefix: '10.0.1.0/24'
    parGatewaySubnetPrefix: '10.0.2.0/24'
    parBastionSubnetPrefix: '10.0.3.0/24'
    parAzFirewallEnabled: true
    parAzFirewallTier: 'Premium'  // Required for IDPS
    parBastionEnabled: true
  }
}
```

### Decision 2: Zero-Trust Network Architecture

**The Debate:**

| Approach | Security Level | Complexity | Cost |
|----------|---------------|------------|------|
| **Traditional Perimeter** | Medium | Low | $ |
| **Zero-Trust with NSG Only** | Medium-High | Medium | $ |
| **Zero-Trust with Firewall** | High | High | $$ |
| **Full Microsegmentation** | Very High | Very High | $$$ |

**Decision: Zero-Trust with Azure Firewall Premium + Private Endpoints**

**Rationale:**
1. PCI-DSS requires network segmentation and traffic inspection
2. TLS inspection needed for encrypted traffic analysis
3. Private endpoints eliminate public exposure of data services
4. IDPS provides threat detection at network layer

**Network Policy Configuration:**
```bicep
// Azure Firewall Policy for Financial Services
resource firewallPolicy 'Microsoft.Network/firewallPolicies@2023-05-01' = {
  name: 'fw-policy-financial'
  location: location
  properties: {
    sku: {
      tier: 'Premium'
    }
    threatIntelMode: 'Deny'
    intrusionDetection: {
      mode: 'Deny'
      configuration: {
        signatureOverrides: []
        bypassTrafficSettings: []
      }
    }
    transportSecurity: {
      certificateAuthority: {
        name: 'intermediate-ca'
        keyVaultSecretId: keyVault.properties.vaultUri
      }
    }
  }
}

// Application Rules for Trading System
resource tradingRuleCollection 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'trading-rules'
  properties: {
    priority: 100
    ruleCollections: [
      {
        name: 'Allow-Trading-APIs'
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        priority: 100
        action: {
          type: 'Allow'
        }
        rules: [
          {
            name: 'Market-Data-Feeds'
            ruleType: 'ApplicationRule'
            sourceAddresses: ['10.1.0.0/16']  // Trading subnet
            protocols: [
              { protocolType: 'Https', port: 443 }
            ]
            targetFqdns: [
              'feeds.bloomberg.com'
              'api.refinitiv.com'
              '*.nasdaq.com'
            ]
            terminateTLS: true
          }
          {
            name: 'Order-Execution'
            ruleType: 'ApplicationRule'
            sourceAddresses: ['10.1.0.0/16']
            protocols: [
              { protocolType: 'Https', port: 443 }
            ]
            targetFqdns: [
              '*.nyse.com'
              '*.cme.com'
            ]
            terminateTLS: true
          }
        ]
      }
      {
        name: 'Deny-All-Other'
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        priority: 65000
        action: {
          type: 'Deny'
        }
        rules: [
          {
            name: 'Deny-Internet'
            ruleType: 'NetworkRule'
            sourceAddresses: ['*']
            destinationAddresses: ['*']
            destinationPorts: ['*']
            ipProtocols: ['Any']
          }
        ]
      }
    ]
  }
}
```

### Decision 3: Data Protection Strategy

**The Debate:**

| Option | Key Management | Encryption | Compliance |
|--------|---------------|------------|------------|
| **Platform-Managed Keys** | Simple | At-rest only | Basic |
| **Customer-Managed Keys (CMK)** | Medium | At-rest with control | PCI-DSS |
| **CMK + HSM-Backed** | Complex | Hardware-protected | SOX, FIPS |
| **Double Encryption** | Complex | Defense in depth | Maximum |

**Decision: CMK with HSM-backed Key Vault + Double Encryption for cardholder data**

**Rationale:**
1. PCI-DSS requires key rotation and separation of duties
2. HSM backing provides FIPS 140-2 Level 3 compliance
3. Double encryption adds infrastructure-level protection
4. Purview integration for data classification

**Implementation:**
```bicep
// HSM-backed Key Vault for Cardholder Data Encryption
resource keyVault 'Microsoft.KeyVault/vaults@2023-02-01' = {
  name: 'kv-${prefix}-pci'
  location: location
  properties: {
    sku: {
      family: 'A'
      name: 'premium'  // Required for HSM
    }
    tenantId: subscription().tenantId
    enableRbacAuthorization: true
    enableSoftDelete: true
    softDeleteRetentionInDays: 90
    enablePurgeProtection: true
    publicNetworkAccess: 'Disabled'
    networkAcls: {
      defaultAction: 'Deny'
      bypass: 'None'
    }
  }
}

// Customer-Managed Key for SQL Database
resource sqlCmk 'Microsoft.KeyVault/vaults/keys@2023-02-01' = {
  parent: keyVault
  name: 'sql-cmk-${uniqueString(resourceGroup().id)}'
  properties: {
    kty: 'RSA-HSM'
    keySize: 3072
    keyOps: ['wrapKey', 'unwrapKey']
    rotationPolicy: {
      lifetimeActions: [
        {
          trigger: {
            timeBeforeExpiry: 'P30D'
          }
          action: {
            type: 'Notify'
          }
        }
        {
          trigger: {
            timeAfterCreate: 'P335D'
          }
          action: {
            type: 'Rotate'
          }
        }
      ]
      attributes: {
        expiryTime: 'P365D'
      }
    }
  }
}

// Azure SQL with CMK and Always Encrypted
resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: 'sql-${prefix}-pci'
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    publicNetworkAccess: 'Disabled'
    minimalTlsVersion: '1.2'
  }
}

resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'CardholderData'
  location: location
  sku: {
    name: 'BC_Gen5_4'
    tier: 'BusinessCritical'
  }
  properties: {
    isLedgerOn: true  // Immutable ledger for audit trail
  }
}

// Transparent Data Encryption with CMK
resource sqlTde 'Microsoft.Sql/servers/databases/transparentDataEncryption@2023-05-01-preview' = {
  parent: sqlDatabase
  name: 'current'
  properties: {
    state: 'Enabled'
  }
}
```

## Technical Deep Dive

### Trading System High Availability

```
                    ┌───────────────────────────────────────────────────────────┐
                    │              Trading System HA Architecture                │
                    └───────────────────────────────────────────────────────────┘

                              Primary Region (East US)
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                                                                              │
    │  ┌────────────────┐      ┌────────────────┐      ┌────────────────┐        │
    │  │   Availability │      │   Availability │      │   Availability │        │
    │  │    Zone 1      │      │    Zone 2      │      │    Zone 3      │        │
    │  │  ┌──────────┐  │      │  ┌──────────┐  │      │  ┌──────────┐  │        │
    │  │  │  VM SS   │  │      │  │  VM SS   │  │      │  │  VM SS   │  │        │
    │  │  │ (Trading)│  │      │  │ (Trading)│  │      │  │ (Trading)│  │        │
    │  │  └────┬─────┘  │      │  └────┬─────┘  │      │  └────┬─────┘  │        │
    │  └───────┼────────┘      └───────┼────────┘      └───────┼────────┘        │
    │          │                       │                       │                  │
    │          └───────────────────────┼───────────────────────┘                  │
    │                                  │                                          │
    │                    ┌─────────────▼─────────────┐                            │
    │                    │    Internal Load Balancer │                            │
    │                    │    (Standard, Zone-Red.)  │                            │
    │                    └─────────────┬─────────────┘                            │
    │                                  │                                          │
    │        ┌─────────────────────────┼─────────────────────────┐               │
    │        │                         │                         │               │
    │  ┌─────▼─────┐            ┌──────▼──────┐           ┌──────▼──────┐        │
    │  │  SQL MI   │◄─── Sync ──│   SQL MI    │───Sync ──►│   SQL MI    │        │
    │  │   (AZ1)   │   Group    │   (AZ2)     │   Group   │   (AZ3)     │        │
    │  │  Primary  │            │  Secondary  │           │  Secondary  │        │
    │  └───────────┘            └─────────────┘           └─────────────┘        │
    │                                                                             │
    └─────────────────────────────────────────────────────────────────────────────┘
                                        │
                           Auto-Failover Group
                                        │
                              DR Region (West US 2)
    ┌─────────────────────────────────────────────────────────────────────────────┐
    │                                  │                                          │
    │                    ┌─────────────▼─────────────┐                            │
    │                    │        SQL MI             │                            │
    │                    │     (Geo-Secondary)       │                            │
    │                    └───────────────────────────┘                            │
    │                                                                             │
    │  Passive infrastructure deployed via IaC, activated during DR              │
    └─────────────────────────────────────────────────────────────────────────────┘
```

### Compliance Monitoring with Azure Policy

```bicep
// Custom Policy Definition for PCI-DSS
resource pciPolicy 'Microsoft.Authorization/policyDefinitions@2023-04-01' = {
  name: 'pci-dss-compliance-initiative'
  properties: {
    displayName: 'PCI-DSS Compliance Initiative'
    policyType: 'Custom'
    mode: 'All'
    metadata: {
      category: 'Regulatory Compliance'
      version: '1.0.0'
    }
    policyRule: {
      if: {
        allOf: [
          {
            field: 'type'
            equals: 'Microsoft.Storage/storageAccounts'
          }
          {
            anyOf: [
              {
                field: 'Microsoft.Storage/storageAccounts/supportsHttpsTrafficOnly'
                notEquals: true
              }
              {
                field: 'Microsoft.Storage/storageAccounts/minimumTlsVersion'
                notEquals: 'TLS1_2'
              }
              {
                field: 'Microsoft.Storage/storageAccounts/allowBlobPublicAccess'
                equals: true
              }
            ]
          }
        ]
      }
      then: {
        effect: 'Deny'
      }
    }
  }
}

// Diagnostic Settings Policy (all resources must log to Log Analytics)
resource diagnosticsPolicy 'Microsoft.Authorization/policyDefinitions@2023-04-01' = {
  name: 'require-diagnostic-settings'
  properties: {
    displayName: 'Require Diagnostic Settings for All Resources'
    policyType: 'Custom'
    mode: 'All'
    parameters: {
      logAnalyticsWorkspaceId: {
        type: 'String'
        metadata: {
          displayName: 'Log Analytics Workspace ID'
          description: 'Central workspace for audit logs'
        }
      }
    }
    policyRule: {
      if: {
        field: 'type'
        in: [
          'Microsoft.Sql/servers'
          'Microsoft.Storage/storageAccounts'
          'Microsoft.KeyVault/vaults'
          'Microsoft.Network/azureFirewalls'
        ]
      }
      then: {
        effect: 'DeployIfNotExists'
        details: {
          type: 'Microsoft.Insights/diagnosticSettings'
          existenceCondition: {
            allOf: [
              {
                field: 'Microsoft.Insights/diagnosticSettings/workspaceId'
                equals: '[parameters(\'logAnalyticsWorkspaceId\')]'
              }
            ]
          }
          roleDefinitionIds: [
            '/providers/Microsoft.Authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c'
          ]
          deployment: {
            properties: {
              mode: 'incremental'
              template: {
                '$schema': 'https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#'
                contentVersion: '1.0.0.0'
                // Template continues...
              }
            }
          }
        }
      }
    }
  }
}
```

## Migration Strategy

### Phase 1: Foundation (Weeks 1-4)
- Deploy Landing Zone infrastructure
- Configure ExpressRoute connectivity
- Implement identity synchronization
- Establish monitoring baseline

### Phase 2: Non-Critical Workloads (Weeks 5-12)
- Migrate reporting and analytics
- Move development environments
- Validate compliance controls

### Phase 3: Critical Workloads (Weeks 13-20)
- Trading system migration (weekend cutover)
- Database migration with near-zero downtime
- Customer-facing applications

### Phase 4: Decommission (Weeks 21-24)
- Validate all systems operational
- Decommission on-premises infrastructure
- Complete compliance audit

## Cost Analysis

### Monthly Cost Breakdown

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| Azure Firewall Premium | 2 instances (HA) | $6,500 |
| ExpressRoute | 1 Gbps, Global Reach | $2,200 |
| SQL Managed Instance | BC Gen5 8-core (3 AZs) | $18,000 |
| Virtual Machines | Trading workloads | $25,000 |
| Key Vault Premium | HSM operations | $1,500 |
| Azure Monitor | Log Analytics, App Insights | $4,000 |
| DDoS Protection | Standard | $3,000 |
| Microsoft Defender | Full suite | $5,000 |
| **Total** | | **$65,200** |

## Lessons Learned

### What Worked Well

1. **Enterprise-Scale Landing Zone**: Pre-built policies accelerated compliance
2. **Azure Firewall Premium**: TLS inspection satisfied PCI requirement 4.1
3. **Auto-Failover Groups**: Simplified DR with RPO < 5 seconds

### Challenges Encountered

1. **ExpressRoute Provisioning**: 6-week lead time with provider
2. **Key Vault Access Policies**: Initial RBAC migration caused access issues
3. **Policy Conflicts**: Some policies blocked legitimate deployments

### Recommendations

1. **Start ExpressRoute early**: Long provisioning time
2. **Use Policy exemptions sparingly**: Document all exemptions
3. **Test disaster recovery regularly**: Monthly DR drills

---

*Continue to [Case Study 4: Healthcare IoT Platform](04-healthcare-iot.md)*

*Back to [Chapter 08: Case Studies](README.md)*
