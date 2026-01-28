# Network Security Groups (NSG)

## Overview

Network Security Groups are Azure's primary mechanism for filtering network traffic. Unlike AWS where you have both Security Groups (stateful) and NACLs (stateless), Azure NSGs are stateful and can be applied at both subnet and NIC levels.

## AWS Comparison

| Feature | AWS Security Groups | AWS NACLs | Azure NSG |
|---------|--------------------|-----------| ----------|
| Stateful | Yes | No | Yes |
| Apply to | Instance/ENI | Subnet | Subnet AND/OR NIC |
| Rules | Allow only | Allow + Deny | Allow + Deny |
| Default | Deny inbound | Allow all | Deny inbound (with exceptions) |
| Evaluation | All rules | First match | Priority-based first match |

## NSG Fundamentals

### Rule Components

| Component | Description | Values |
|-----------|-------------|--------|
| Priority | Order of evaluation | 100-4096 (lower = first) |
| Name | Rule identifier | String |
| Direction | Traffic direction | Inbound / Outbound |
| Action | Allow or Deny | Allow / Deny |
| Source | Traffic origin | IP, CIDR, Service Tag, ASG |
| Source Port | Origin port | Single, range, or * |
| Destination | Traffic target | IP, CIDR, Service Tag, ASG |
| Destination Port | Target port | Single, range, or * |
| Protocol | Network protocol | TCP, UDP, ICMP, or * |

### Rule Evaluation

```
┌────────────────────────────────────────────────────────────────┐
│                    NSG Rule Evaluation                          │
├────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Inbound Traffic                                                │
│  ────────────────                                               │
│                                                                 │
│  1. Subnet NSG Rules (if attached)                             │
│     ├── Evaluate in priority order (100 → 4096)                │
│     ├── First match wins                                        │
│     └── If no match, continue to NIC NSG                       │
│                                                                 │
│  2. NIC NSG Rules (if attached)                                │
│     ├── Evaluate in priority order                              │
│     └── First match wins                                        │
│                                                                 │
│  3. If no rule matches → Default rules apply                   │
│                                                                 │
│  Outbound Traffic                                               │
│  ─────────────────                                              │
│                                                                 │
│  1. NIC NSG Rules first                                        │
│  2. Then Subnet NSG Rules                                      │
│  3. Default rules if no match                                  │
│                                                                 │
└────────────────────────────────────────────────────────────────┘
```

## Application Security Groups (ASG)

ASGs provide logical grouping of VMs, allowing you to create rules based on application tiers rather than IP addresses.

### Use Case Example

```
Without ASG:                          With ASG:
──────────────                        ──────────

Source: 10.0.1.4                      Source: webServers-ASG
        10.0.1.5                      Dest: appServers-ASG
        10.0.1.6                      Port: 8080
Dest:   10.0.2.4                      Action: Allow
        10.0.2.5
Port: 8080
Action: Allow

(Must update when VMs change)         (Automatically includes new VMs)
```

### ASG Configuration

```bicep
// Application Security Groups
resource webASG 'Microsoft.Network/applicationSecurityGroups@2023-05-01' = {
  name: 'web-servers-asg'
  location: location
}

resource appASG 'Microsoft.Network/applicationSecurityGroups@2023-05-01' = {
  name: 'app-servers-asg'
  location: location
}

resource dbASG 'Microsoft.Network/applicationSecurityGroups@2023-05-01' = {
  name: 'db-servers-asg'
  location: location
}

// NSG with ASG-based rules
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'app-nsg'
  location: location
  properties: {
    securityRules: [
      {
        name: 'Allow-Web-to-App'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceApplicationSecurityGroups: [
            { id: webASG.id }
          ]
          sourcePortRange: '*'
          destinationApplicationSecurityGroups: [
            { id: appASG.id }
          ]
          destinationPortRange: '8080'
        }
      }
      {
        name: 'Allow-App-to-DB'
        properties: {
          priority: 110
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceApplicationSecurityGroups: [
            { id: appASG.id }
          ]
          sourcePortRange: '*'
          destinationApplicationSecurityGroups: [
            { id: dbASG.id }
          ]
          destinationPortRange: '1433'
        }
      }
    ]
  }
}
```

## Best Practices

### 1. Rule Organization

```
Priority Ranges:
├── 100-200:   Allow rules for application traffic
├── 200-300:   Allow rules for management (SSH, RDP)
├── 300-400:   Allow rules for monitoring/health
├── 400-3000:  Application-specific rules
├── 3000-4000: Explicit deny rules (logging)
└── 4096:      Catch-all deny (default exists)
```

### 2. Use Service Tags

Instead of maintaining IP ranges:

```bicep
// Bad - hardcoded IPs
{
  name: 'Allow-Azure-Services'
  properties: {
    sourceAddressPrefix: '13.64.0.0/11'  // Will become outdated
    // ...
  }
}

// Good - Service Tags
{
  name: 'Allow-Azure-Services'
  properties: {
    sourceAddressPrefix: 'AzureCloud'  // Always current
    // ...
  }
}
```

### 3. Naming Convention

```
nsg-<environment>-<region>-<tier>-<number>

Examples:
- nsg-prod-eastus-web-001
- nsg-dev-westus2-app-001
- nsg-prod-eastus-db-001
```

### 4. Documentation in Rules

Use descriptive names and include ticket/change numbers:

```bicep
{
  name: 'Allow-HTTPS-CHG123456'
  properties: {
    description: 'Allow HTTPS traffic for web application - Change Request CHG123456'
    // ...
  }
}
```

## Common Patterns

### Three-Tier Application

```
┌─────────────────────────────────────────────────────────────────┐
│                    Three-Tier NSG Pattern                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Web Tier NSG                                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Allow: Internet → Web Subnet : 80, 443                    │  │
│  │ Allow: Web ASG → App ASG : 8080                          │  │
│  │ Allow: AzureLoadBalancer → * : *                         │  │
│  │ Deny:  * → * : *                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  App Tier NSG                                                    │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Allow: Web ASG → App ASG : 8080                          │  │
│  │ Allow: App ASG → DB ASG : 1433                           │  │
│  │ Allow: AzureLoadBalancer → * : *                         │  │
│  │ Deny:  Internet → * : *                                   │  │
│  │ Deny:  * → * : *                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  DB Tier NSG                                                     │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ Allow: App ASG → DB ASG : 1433                           │  │
│  │ Deny:  Internet → * : *                                   │  │
│  │ Deny:  Web ASG → * : *                                    │  │
│  │ Deny:  * → * : *                                          │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Management Access Pattern

```bicep
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'management-nsg'
  location: location
  properties: {
    securityRules: [
      {
        name: 'Allow-Bastion-SSH'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'AzureBastionSubnet'
          sourcePortRange: '*'
          destinationAddressPrefix: 'VirtualNetwork'
          destinationPortRange: '22'
        }
      }
      {
        name: 'Allow-Bastion-RDP'
        properties: {
          priority: 110
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: 'AzureBastionSubnet'
          sourcePortRange: '*'
          destinationAddressPrefix: 'VirtualNetwork'
          destinationPortRange: '3389'
        }
      }
      {
        name: 'Deny-Direct-SSH'
        properties: {
          priority: 200
          direction: 'Inbound'
          access: 'Deny'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '22'
        }
      }
      {
        name: 'Deny-Direct-RDP'
        properties: {
          priority: 210
          direction: 'Inbound'
          access: 'Deny'
          protocol: 'Tcp'
          sourceAddressPrefix: 'Internet'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '3389'
        }
      }
    ]
  }
}
```

## Troubleshooting

### Common Issues

| Symptom | Possible Cause | Solution |
|---------|---------------|----------|
| Traffic blocked unexpectedly | Higher priority deny rule | Check effective rules |
| Health probes failing | Missing AzureLoadBalancer allow | Add service tag rule |
| Cannot reach Azure services | Service endpoints not configured | Enable service endpoints |
| ASG rule not working | NIC not added to ASG | Associate NIC with ASG |

### Diagnostic Commands

```bash
# View effective rules for a NIC
az network nic show-effective-nsg \
  --resource-group myRG \
  --name myNIC \
  --output table

# IP flow verify
az network watcher test-ip-flow \
  --resource-group myRG \
  --vm myVM \
  --direction Inbound \
  --local 10.0.0.4:80 \
  --remote 203.0.113.5:12345 \
  --protocol Tcp
```

---

*Continue to [Azure Firewall](02-azure-firewall.md)*

*Back to [Network Security Overview](README.md)*
