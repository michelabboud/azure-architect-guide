# Azure Firewall

## Overview

Azure Firewall is a managed, cloud-native network security service that provides threat protection for cloud workloads. Unlike NSGs which operate at layers 3-4, Azure Firewall provides layers 3-7 filtering with advanced features like FQDN filtering and threat intelligence.

## AWS Comparison

| Feature | AWS Network Firewall | Azure Firewall |
|---------|---------------------|----------------|
| Deployment | Per VPC | Per VNet (hub) |
| Managed Rules | Yes | Yes (Premium) |
| FQDN Filtering | Yes | Yes |
| TLS Inspection | Yes | Premium only |
| Threat Intelligence | Yes | Yes |
| IDPS | Yes | Premium only |
| Pricing Model | Per hour + data | Per hour + data |

## Firewall Tiers

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| FQDN Filtering | Yes | Yes | Yes |
| Network Rules | Yes | Yes | Yes |
| Threat Intel | No | Alert/Deny | Alert/Deny |
| IDPS | No | No | Yes |
| TLS Inspection | No | No | Yes |
| Web Categories | No | No | Yes |
| Cost (approx) | $0.25/hr | $1.25/hr | $1.75/hr |
| Use Case | Dev/Test | Production | Regulated |

## Architecture Patterns

### Hub-Spoke with Azure Firewall

```
                              Internet
                                  │
                                  ▼
                        ┌─────────────────┐
                        │  Public IP      │
                        └────────┬────────┘
                                 │
                        ┌────────▼────────┐
                        │  Azure Firewall │
                        │  (Hub VNet)     │
                        └────────┬────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
     ┌──────▼──────┐      ┌──────▼──────┐      ┌─────▼───────┐
     │  Spoke 1    │      │  Spoke 2    │      │  On-Prem    │
     │  (Web)      │      │  (App)      │      │  (VPN/ER)   │
     │             │      │             │      │             │
     │ UDR: 0/0    │      │ UDR: 0/0    │      │             │
     │ → Firewall  │      │ → Firewall  │      │             │
     └─────────────┘      └─────────────┘      └─────────────┘
```

### Firewall Policy Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                    Firewall Policy Hierarchy                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Base Policy (Parent)                                        │
│  ├── Common rules for all environments                      │
│  ├── Deny rules, threat intel settings                      │
│  └── DNS settings                                            │
│           │                                                  │
│           ├──────────────┬──────────────────┐               │
│           ▼              ▼                  ▼               │
│   Child Policy     Child Policy      Child Policy           │
│   (Production)     (Development)     (Testing)              │
│   ├── Prod rules   ├── Dev rules     ├── Test rules        │
│   └── Strict       └── Relaxed       └── Permissive        │
│                                                              │
│   Inheritance: Child inherits parent rules                   │
│   Override: Child can add but not remove parent rules       │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

## Rule Types

### 1. Application Rules (Layer 7)

```bicep
resource ruleCollectionGroup 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'ApplicationRules'
  properties: {
    priority: 100
    ruleCollections: [
      {
        name: 'AllowMicrosoftServices'
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        priority: 100
        action: { type: 'Allow' }
        rules: [
          {
            name: 'AllowWindowsUpdate'
            ruleType: 'ApplicationRule'
            sourceAddresses: ['10.0.0.0/8']
            protocols: [
              { protocolType: 'Https', port: 443 }
            ]
            targetFqdns: [
              '*.windowsupdate.microsoft.com'
              '*.update.microsoft.com'
              '*.download.windowsupdate.com'
            ]
          }
          {
            name: 'AllowAzureServices'
            ruleType: 'ApplicationRule'
            sourceAddresses: ['10.0.0.0/8']
            protocols: [
              { protocolType: 'Https', port: 443 }
            ]
            fqdnTags: [
              'AzureKubernetesService'
              'WindowsDiagnostics'
            ]
          }
        ]
      }
    ]
  }
}
```

### 2. Network Rules (Layer 4)

```bicep
resource networkRules 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'NetworkRules'
  properties: {
    priority: 200
    ruleCollections: [
      {
        name: 'AllowInternalTraffic'
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        priority: 100
        action: { type: 'Allow' }
        rules: [
          {
            name: 'AllowSpoke1toSpoke2'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.1.0.0/16']  // Spoke 1
            destinationAddresses: ['10.2.0.0/16']  // Spoke 2
            destinationPorts: ['443', '8080']
            ipProtocols: ['TCP']
          }
          {
            name: 'AllowDNS'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.0.0.0/8']
            destinationAddresses: ['168.63.129.16']
            destinationPorts: ['53']
            ipProtocols: ['UDP']
          }
        ]
      }
    ]
  }
}
```

### 3. DNAT Rules (Inbound NAT)

```bicep
resource dnatRules 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'DnatRules'
  properties: {
    priority: 50
    ruleCollections: [
      {
        name: 'InboundNAT'
        ruleCollectionType: 'FirewallPolicyNatRuleCollection'
        priority: 100
        action: { type: 'Dnat' }
        rules: [
          {
            name: 'DNAT-HTTPS'
            ruleType: 'NatRule'
            sourceAddresses: ['*']
            destinationAddresses: ['<firewall-public-ip>']
            destinationPorts: ['443']
            translatedAddress: '10.1.0.4'
            translatedPort: '443'
            ipProtocols: ['TCP']
          }
        ]
      }
    ]
  }
}
```

## Premium Features

### TLS Inspection

```bicep
resource premiumPolicy 'Microsoft.Network/firewallPolicies@2023-05-01' = {
  name: 'premium-policy'
  location: location
  properties: {
    sku: { tier: 'Premium' }
    transportSecurity: {
      certificateAuthority: {
        name: 'intermediate-ca'
        keyVaultSecretId: 'https://mykv.vault.azure.net/secrets/fw-cert'
      }
    }
    intrusionDetection: {
      mode: 'Deny'
      configuration: {
        signatureOverrides: [
          {
            id: '2024897'
            mode: 'Off'  // Disable specific signature
          }
        ]
        bypassTrafficSettings: [
          {
            name: 'BypassHealthCheck'
            protocol: 'TCP'
            sourceAddresses: ['10.0.0.0/8']
            destinationAddresses: ['168.63.129.16']
            destinationPorts: ['80']
          }
        ]
      }
    }
  }
}
```

### IDPS (Intrusion Detection and Prevention)

| Mode | Behavior |
|------|----------|
| Off | No inspection |
| Alert | Log and alert, don't block |
| Deny | Log, alert, and block threats |

### Web Categories

```bicep
{
  name: 'BlockSocialMedia'
  ruleType: 'ApplicationRule'
  sourceAddresses: ['10.0.0.0/8']
  protocols: [{ protocolType: 'Https', port: 443 }]
  webCategories: [
    'SocialNetworking'
    'StreamingMedia'
    'Gambling'
  ]
  action: 'Deny'
}
```

## Routing Configuration

### User-Defined Routes (UDR)

```bicep
// Route table for spokes
resource spokeRouteTable 'Microsoft.Network/routeTables@2023-05-01' = {
  name: 'spoke-rt'
  location: location
  properties: {
    disableBgpRoutePropagation: true
    routes: [
      {
        name: 'ToFirewall'
        properties: {
          addressPrefix: '0.0.0.0/0'
          nextHopType: 'VirtualAppliance'
          nextHopIpAddress: firewall.properties.ipConfigurations[0].properties.privateIPAddress
        }
      }
      {
        name: 'ToOtherSpoke'
        properties: {
          addressPrefix: '10.2.0.0/16'
          nextHopType: 'VirtualAppliance'
          nextHopIpAddress: firewall.properties.ipConfigurations[0].properties.privateIPAddress
        }
      }
    ]
  }
}
```

## Monitoring and Diagnostics

### Diagnostic Settings

```bicep
resource firewallDiagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: 'fw-diagnostics'
  scope: firewall
  properties: {
    workspaceId: logAnalyticsWorkspace.id
    logs: [
      { category: 'AzureFirewallApplicationRule', enabled: true }
      { category: 'AzureFirewallNetworkRule', enabled: true }
      { category: 'AzureFirewallDnsProxy', enabled: true }
      { category: 'AZFWThreatIntel', enabled: true }
      { category: 'AZFWIdpsSignature', enabled: true }
    ]
    metrics: [
      { category: 'AllMetrics', enabled: true }
    ]
  }
}
```

### KQL Queries

```kusto
// Top blocked connections
AzureDiagnostics
| where Category == "AzureFirewallNetworkRule"
| where msg_s contains "Deny"
| parse msg_s with * "from " sourceIP ":" sourcePort " to " destIP ":" destPort *
| summarize count() by sourceIP, destIP, destPort
| order by count_ desc
| take 20

// Threat intelligence hits
AzureDiagnostics
| where Category == "AZFWThreatIntel"
| parse msg_s with * "from " sourceIP ":" * " to " destIP ":" destPort *
| summarize count() by sourceIP, destIP
| order by count_ desc

// IDPS alerts
AzureDiagnostics
| where Category == "AZFWIdpsSignature"
| project TimeGenerated, msg_s, Protocol_s, SourceIP_s, DestinationIP_s
| order by TimeGenerated desc
```

## Cost Optimization

| Strategy | Impact | Recommendation |
|----------|--------|----------------|
| Use Basic tier for dev | 80% savings | Dev/test environments |
| Consolidate to hub | Shared cost | Hub-spoke architecture |
| Right-size | 10-30% savings | Monitor utilization |
| Optimize rules | Better performance | Reduce rule complexity |

---

*Continue to [DDoS Protection](03-ddos-protection.md)*

*Back to [Network Security Overview](README.md)*
