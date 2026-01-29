# Quick Reference: Cloud Adoption Framework

## Management Group CLI Commands

```bash
# Create management group hierarchy
az account management-group create --name "Contoso"
az account management-group create --name "Platform" --parent "Contoso"
az account management-group create --name "LandingZones" --parent "Contoso"
az account management-group create --name "Corp" --parent "LandingZones"
az account management-group create --name "Online" --parent "LandingZones"

# Move subscription to management group
az account management-group subscription add \
    --name "Corp" \
    --subscription "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# List management groups
az account management-group list --output table

# Get management group details
az account management-group show --name "Contoso" --expand
```

## Azure Policy Commands

```bash
# Assign built-in policy
az policy assignment create \
    --name "require-tag-environment" \
    --display-name "Require Environment tag" \
    --policy "/providers/Microsoft.Authorization/policyDefinitions/871b6d14-10aa-478d-b590-94f262ecfa99" \
    --scope "/providers/Microsoft.Management/managementGroups/LandingZones" \
    --params '{"tagName": {"value": "Environment"}}'

# Create custom policy definition
az policy definition create \
    --name "custom-allowed-locations" \
    --display-name "Custom Allowed Locations" \
    --description "Restrict resource locations" \
    --mode "Indexed" \
    --management-group "Contoso" \
    --rules @policy-rules.json \
    --params @policy-params.json

# List policy assignments
az policy assignment list \
    --scope "/providers/Microsoft.Management/managementGroups/LandingZones"

# Get compliance state
az policy state list \
    --management-group "LandingZones" \
    --filter "complianceState eq 'NonCompliant'"
```

## Azure Migrate Commands

```bash
# Create migrate project
az migrate project create \
    --name "contoso-migration" \
    --resource-group "rg-migrate" \
    --location "eastus"

# Create assessment
az migrate assessment create \
    --project-name "contoso-migration" \
    --resource-group "rg-migrate" \
    --name "assessment-1" \
    --group-name "group-1"

# List discovered machines
az migrate machine list \
    --project-name "contoso-migration" \
    --resource-group "rg-migrate"
```

## Hub-Spoke Network Bicep Template

```bicep
// main.bicep - Hub-Spoke Landing Zone Network
targetScope = 'subscription'

param location string = 'eastus'
param hubAddressSpace string = '10.0.0.0/16'
param spoke1AddressSpace string = '10.1.0.0/16'
param spoke2AddressSpace string = '10.2.0.0/16'

// Resource Groups
resource rgHub 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: 'rg-network-hub'
  location: location
}

resource rgSpoke1 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: 'rg-network-spoke1'
  location: location
}

// Hub Network Module
module hubNetwork 'modules/hub-network.bicep' = {
  scope: rgHub
  name: 'hub-network'
  params: {
    location: location
    addressSpace: hubAddressSpace
  }
}

// Spoke Network Module
module spoke1Network 'modules/spoke-network.bicep' = {
  scope: rgSpoke1
  name: 'spoke1-network'
  params: {
    location: location
    addressSpace: spoke1AddressSpace
    hubVnetId: hubNetwork.outputs.vnetId
  }
}

// modules/hub-network.bicep
param location string
param addressSpace string

resource hubVnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'vnet-hub-${location}'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [addressSpace]
    }
    subnets: [
      {
        name: 'GatewaySubnet'
        properties: {
          addressPrefix: cidrSubnet(addressSpace, 24, 0)
        }
      }
      {
        name: 'AzureFirewallSubnet'
        properties: {
          addressPrefix: cidrSubnet(addressSpace, 24, 1)
        }
      }
      {
        name: 'AzureBastionSubnet'
        properties: {
          addressPrefix: cidrSubnet(addressSpace, 24, 2)
        }
      }
      {
        name: 'snet-shared'
        properties: {
          addressPrefix: cidrSubnet(addressSpace, 24, 3)
        }
      }
    ]
  }
}

resource firewall 'Microsoft.Network/azureFirewalls@2023-05-01' = {
  name: 'afw-hub-${location}'
  location: location
  properties: {
    sku: {
      name: 'AZFW_VNet'
      tier: 'Standard'
    }
    firewallPolicy: {
      id: firewallPolicy.id
    }
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: {
            id: '${hubVnet.id}/subnets/AzureFirewallSubnet'
          }
          publicIPAddress: {
            id: publicIp.id
          }
        }
      }
    ]
  }
}

output vnetId string = hubVnet.id
output firewallPrivateIp string = firewall.properties.ipConfigurations[0].properties.privateIPAddress
```

## Policy Definitions

### Require Tags Policy

```json
{
  "mode": "Indexed",
  "parameters": {
    "tagName": {
      "type": "String",
      "metadata": {
        "displayName": "Tag Name",
        "description": "Name of the tag to require"
      }
    }
  },
  "policyRule": {
    "if": {
      "field": "[concat('tags[', parameters('tagName'), ']')]",
      "exists": "false"
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

### Allowed Locations Policy

```json
{
  "mode": "All",
  "parameters": {
    "allowedLocations": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed locations",
        "description": "List of allowed locations"
      },
      "defaultValue": ["eastus", "westus2", "northeurope", "westeurope"]
    }
  },
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "location",
          "notIn": "[parameters('allowedLocations')]"
        },
        {
          "field": "location",
          "notEquals": "global"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

## CAF Decision Trees

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MIGRATION STRATEGY DECISION TREE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Is the application still needed?                                          │
│   │                                                                         │
│   ├── No ──────────────────────────────────────────────► RETIRE             │
│   │                                                                         │
│   └── Yes ──► Is there a SaaS alternative?                                  │
│               │                                                             │
│               ├── Yes ──► Does SaaS meet requirements?                      │
│               │           │                                                 │
│               │           ├── Yes ────────────────────► REPLACE (SaaS)      │
│               │           └── No ──► Continue                               │
│               │                                                             │
│               └── No ──► Continue                                           │
│                          │                                                  │
│                          ▼                                                  │
│   Does it need significant modernization?                                   │
│   │                                                                         │
│   ├── Yes ──► Can you afford to rebuild?                                    │
│   │           │                                                             │
│   │           ├── Yes ──► Business case for rebuild?                        │
│   │           │           │                                                 │
│   │           │           ├── Yes ────────────────────► REBUILD             │
│   │           │           └── No ──────────────────────► REARCHITECT        │
│   │           │                                                             │
│   │           └── No ────────────────────────────────► REARCHITECT          │
│   │                                                                         │
│   └── No ──► Can it benefit from PaaS?                                      │
│              │                                                              │
│              ├── Yes ─────────────────────────────────► REFACTOR (PaaS)     │
│              │                                                              │
│              └── No ──────────────────────────────────► REHOST (IaaS)       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SUBSCRIPTION DESIGN DECISION TREE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Does the workload need isolation?                                         │
│   │                                                                         │
│   ├── Yes (Security/Compliance) ─────► Dedicated subscription               │
│   │                                                                         │
│   └── No ──► Is it production?                                              │
│              │                                                              │
│              ├── Yes ──► Is it a large application?                         │
│              │           │                                                  │
│              │           ├── Yes ─────────────► Dedicated subscription      │
│              │           │                                                  │
│              │           └── No ──► Can share with similar workloads?       │
│              │                      │                                       │
│              │                      ├── Yes ──► Shared prod subscription    │
│              │                      └── No ───► Dedicated subscription      │
│              │                                                              │
│              └── No (Dev/Test) ──► Sandbox subscription                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Landing Zone Checklist

### Pre-Deployment

- [ ] Define management group structure
- [ ] Plan subscription strategy
- [ ] Design network topology (hub-spoke vs Virtual WAN)
- [ ] Establish naming convention
- [ ] Define tagging strategy
- [ ] Create RBAC role assignments
- [ ] Plan policy assignments

### Platform Landing Zone

- [ ] Deploy management subscription
  - [ ] Log Analytics workspace
  - [ ] Automation account
  - [ ] Azure Monitor baseline
- [ ] Deploy connectivity subscription
  - [ ] Hub VNet
  - [ ] Azure Firewall
  - [ ] VPN/ExpressRoute gateway
  - [ ] DNS zones
- [ ] Deploy identity subscription (if needed)
  - [ ] Azure AD DS
  - [ ] Domain controllers

### Workload Landing Zone

- [ ] Create subscription
- [ ] Move to correct management group
- [ ] Deploy spoke VNet
- [ ] Configure VNet peering
- [ ] Apply policies
- [ ] Configure diagnostics
- [ ] Set up RBAC

### Governance

- [ ] Assign required policies
  - [ ] Allowed locations
  - [ ] Required tags
  - [ ] Allowed VM SKUs
  - [ ] Defender for Cloud
- [ ] Configure compliance dashboard
- [ ] Set up cost management
  - [ ] Budgets
  - [ ] Alerts
  - [ ] Cost allocation tags

---

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
