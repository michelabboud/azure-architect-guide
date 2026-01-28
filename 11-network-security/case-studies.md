# Network Security Case Studies

## Case Study 1: Enterprise Zero-Trust Implementation

### Scenario

A financial services company is migrating from AWS to Azure and requires a zero-trust network architecture that meets SOX and PCI-DSS compliance requirements.

### Requirements

- No direct internet access for workloads
- All traffic inspected (including TLS)
- Micro-segmentation between application tiers
- Secure management access (no public IPs)
- Full audit trail of network traffic

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Zero-Trust Network Architecture                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                           ┌─────────────────────┐                           │
│                           │   Azure Front Door  │                           │
│                           │   (WAF + CDN)       │                           │
│                           └──────────┬──────────┘                           │
│                                      │                                       │
│  ┌───────────────────────────────────┼───────────────────────────────────┐  │
│  │                        Hub VNet   │                                    │  │
│  │                                   │                                    │  │
│  │  ┌──────────────┐    ┌───────────▼───────────┐    ┌──────────────┐   │  │
│  │  │   Bastion    │    │   Azure Firewall      │    │   VPN/ER     │   │  │
│  │  │   (Mgmt)     │    │   Premium             │    │   Gateway    │   │  │
│  │  │              │    │   • TLS Inspection    │    │              │   │  │
│  │  │              │    │   • IDPS              │    │   On-Prem    │   │  │
│  │  │              │    │   • Threat Intel      │    │   Connect    │   │  │
│  │  └──────────────┘    └───────────┬───────────┘    └──────────────┘   │  │
│  │                                   │                                    │  │
│  └───────────────────────────────────┼────────────────────────────────────┘  │
│                                      │                                       │
│         ┌────────────────────────────┼────────────────────────────┐         │
│         │                            │                            │         │
│  ┌──────▼──────┐              ┌──────▼──────┐              ┌──────▼──────┐  │
│  │  Web Spoke  │              │  App Spoke  │              │  Data Spoke │  │
│  │             │              │             │              │             │  │
│  │ ┌─────────┐ │              │ ┌─────────┐ │              │ ┌─────────┐ │  │
│  │ │  NSG    │ │              │ │  NSG    │ │              │ │  NSG    │ │  │
│  │ │ (Web)   │ │              │ │ (App)   │ │              │ │ (Data)  │ │  │
│  │ └─────────┘ │              │ └─────────┘ │              │ └─────────┘ │  │
│  │ ┌─────────┐ │              │ ┌─────────┐ │              │ ┌─────────┐ │  │
│  │ │App GW   │ │─────NSG─────▶│ │  AKS    │ │─────NSG─────▶│ │Priv EP  │ │  │
│  │ │(WAF v2) │ │   Rules      │ │         │ │   Rules      │ │(SQL,KV) │ │  │
│  │ └─────────┘ │              │ └─────────┘ │              │ └─────────┘ │  │
│  │             │              │             │              │             │  │
│  │ UDR→FW     │              │ UDR→FW     │              │ UDR→FW     │  │
│  └─────────────┘              └─────────────┘              └─────────────┘  │
│                                                                              │
│  All outbound: 0.0.0.0/0 → Azure Firewall                                   │
│  All inbound: Through Application Gateway + WAF                              │
│  Management: Azure Bastion only (no public IPs on VMs)                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Edge protection | Front Door + WAF | Global anycast, DDoS, bot protection |
| Internal firewall | Azure Firewall Premium | TLS inspection for compliance |
| Segmentation | NSG + ASG | Micro-segmentation without NVAs |
| Data access | Private Endpoints | Zero public exposure |
| Management | Azure Bastion | No public IPs, audited access |

### Implementation Highlights

**Azure Firewall Policy for TLS Inspection:**

```bicep
resource premiumPolicy 'Microsoft.Network/firewallPolicies@2023-05-01' = {
  name: 'zerotrust-policy'
  location: location
  properties: {
    sku: { tier: 'Premium' }
    transportSecurity: {
      certificateAuthority: {
        name: 'intermediate-ca'
        keyVaultSecretId: keyVault.properties.vaultUri
      }
    }
    intrusionDetection: {
      mode: 'Deny'
    }
    threatIntelMode: 'Deny'
  }
}
```

### Compliance Mapping

| Requirement | Implementation |
|-------------|----------------|
| PCI-DSS 1.2 (Firewall) | Azure Firewall + NSGs |
| PCI-DSS 1.3 (DMZ) | Hub-spoke with web tier |
| PCI-DSS 4.1 (Encryption) | TLS inspection + in-transit encryption |
| SOX (Audit) | NSG flow logs + Firewall logs |

---

## Case Study 2: Secure Hybrid Network

### Scenario

A manufacturing company needs to connect on-premises factory systems to Azure while maintaining strict network isolation for OT (Operational Technology) systems.

### Requirements

- Secure connectivity between on-prem and Azure
- Isolated network for OT systems
- Central security management
- Redundant connectivity

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Secure Hybrid Network Architecture                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  On-Premises                                                                 │
│  ────────────                                                                │
│  ┌───────────────────┐    ┌───────────────────┐                             │
│  │  Corporate        │    │  Factory (OT)     │                             │
│  │  Network          │    │  Network          │                             │
│  │  10.1.0.0/16      │    │  10.2.0.0/16      │                             │
│  └────────┬──────────┘    └────────┬──────────┘                             │
│           │                         │                                        │
│           └─────────┬───────────────┘                                        │
│                     │                                                        │
│           ┌─────────▼─────────┐                                             │
│           │   On-Prem         │                                             │
│           │   Firewall        │                                             │
│           └─────────┬─────────┘                                             │
│                     │                                                        │
│  ════════════════════╪═══════════════════════════════════════════════════   │
│                     │                                                        │
│  Azure              │ ExpressRoute (Primary)                                 │
│  ─────              │ VPN Gateway (Backup)                                   │
│                     │                                                        │
│           ┌─────────▼─────────────────────────────────────────┐             │
│           │              Connectivity Hub VNet                 │             │
│           │                                                    │             │
│           │  ┌────────────────┐    ┌────────────────┐         │             │
│           │  │  ExpressRoute  │    │  VPN Gateway   │         │             │
│           │  │  Gateway       │    │  (Backup)      │         │             │
│           │  └───────┬────────┘    └───────┬────────┘         │             │
│           │          │                     │                   │             │
│           │          └──────────┬──────────┘                   │             │
│           │                     │                              │             │
│           │          ┌──────────▼──────────┐                   │             │
│           │          │   Azure Firewall    │                   │             │
│           │          │   (Central)         │                   │             │
│           │          └──────────┬──────────┘                   │             │
│           │                     │                              │             │
│           └─────────────────────┼──────────────────────────────┘             │
│                                 │                                            │
│       ┌─────────────────────────┼─────────────────────────────┐             │
│       │                         │                             │             │
│  ┌────▼─────────┐        ┌──────▼──────┐        ┌─────────────▼──┐          │
│  │  Corporate   │        │   OT        │        │    Shared      │          │
│  │  Spoke       │        │   Spoke     │        │    Services    │          │
│  │              │        │  (Isolated) │        │    Spoke       │          │
│  │  Apps/Data   │        │             │        │                │          │
│  │              │        │  No Internet│        │  DNS, AD,      │          │
│  │              │        │  No Peering │        │  Monitoring    │          │
│  └──────────────┘        │  FW Only    │        └────────────────┘          │
│                          └─────────────┘                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### OT Network Isolation Rules

```bicep
// OT Spoke - Maximum Isolation
resource otFirewallRules 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'OT-Isolation'
  properties: {
    priority: 100
    ruleCollections: [
      {
        name: 'Allow-OT-to-OnPrem'
        priority: 100
        action: { type: 'Allow' }
        rules: [
          {
            name: 'OT-to-Factory'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.100.0.0/16']  // OT Spoke
            destinationAddresses: ['10.2.0.0/16']  // On-prem Factory
            destinationPorts: ['443', '502']  // HTTPS, Modbus
            ipProtocols: ['TCP']
          }
        ]
      }
      {
        name: 'Deny-OT-to-Internet'
        priority: 200
        action: { type: 'Deny' }
        rules: [
          {
            name: 'Deny-All-Internet'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.100.0.0/16']
            destinationAddresses: ['Internet']
            destinationPorts: ['*']
            ipProtocols: ['Any']
          }
        ]
      }
      {
        name: 'Deny-OT-to-Corporate'
        priority: 300
        action: { type: 'Deny' }
        rules: [
          {
            name: 'Deny-Corporate'
            ruleType: 'NetworkRule'
            sourceAddresses: ['10.100.0.0/16']
            destinationAddresses: ['10.50.0.0/16']  // Corporate spoke
            destinationPorts: ['*']
            ipProtocols: ['Any']
          }
        ]
      }
    ]
  }
}
```

---

## Case Study 3: Multi-Region Security Architecture

### Scenario

A global e-commerce company requires consistent security policies across multiple Azure regions with centralized management.

### Solution: Azure Firewall Manager

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Multi-Region Security Architecture                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌────────────────────────────────┐                       │
│                    │   Azure Firewall Manager       │                       │
│                    │                                │                       │
│                    │   ┌────────────────────────┐  │                       │
│                    │   │   Global Base Policy    │  │                       │
│                    │   │   • Deny rules          │  │                       │
│                    │   │   • Threat Intel        │  │                       │
│                    │   │   • Common FQDN tags    │  │                       │
│                    │   └───────────┬────────────┘  │                       │
│                    │               │               │                       │
│                    │       ┌───────┴───────┐       │                       │
│                    │       │               │       │                       │
│                    │   ┌───▼────┐     ┌────▼───┐   │                       │
│                    │   │ US     │     │ EU     │   │                       │
│                    │   │ Policy │     │ Policy │   │                       │
│                    │   │        │     │        │   │                       │
│                    │   │Regional│     │Regional│   │                       │
│                    │   │ Rules  │     │ Rules  │   │                       │
│                    │   └───┬────┘     └────┬───┘   │                       │
│                    └───────┼───────────────┼───────┘                       │
│                            │               │                                │
│         ┌──────────────────┼───────────────┼──────────────────┐            │
│         │                  │               │                  │            │
│         ▼                  ▼               ▼                  ▼            │
│  ┌─────────────┐    ┌─────────────┐ ┌─────────────┐    ┌─────────────┐    │
│  │  East US    │    │  West US    │ │  West EU    │    │  North EU   │    │
│  │  Hub        │    │  Hub        │ │  Hub        │    │  Hub        │    │
│  │             │    │             │ │             │    │             │    │
│  │ ┌─────────┐ │    │ ┌─────────┐ │ │ ┌─────────┐ │    │ ┌─────────┐ │    │
│  │ │Firewall │ │    │ │Firewall │ │ │ │Firewall │ │    │ │Firewall │ │    │
│  │ │ US Pol. │ │    │ │ US Pol. │ │ │ │ EU Pol. │ │    │ │ EU Pol. │ │    │
│  │ └─────────┘ │    │ └─────────┘ │ │ └─────────┘ │    │ └─────────┘ │    │
│  └─────────────┘    └─────────────┘ └─────────────┘    └─────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Back to [Network Security Overview](README.md)*
