# Virtual Networks Deep Dive

## Overview

Azure Virtual Networks (VNets) are the fundamental building block for private networks in Azure. If you're coming from AWS, VNets are conceptually similar to VPCs, but with key architectural differences that affect how you design your network topology.

## AWS to Azure Comparison

| Concept | AWS | Azure | Key Difference |
|---------|-----|-------|----------------|
| Virtual Network | VPC | VNet | VNet subnets span all AZs |
| Subnet | Subnet (AZ-specific) | Subnet (region-wide) | Azure subnets not tied to AZ |
| CIDR Blocks | Primary + Secondary | Multiple address spaces | Azure allows non-contiguous |
| Default Behavior | Isolated by default | Isolated by default | Same concept |
| Internet Gateway | Explicit resource | Implicit | No separate IGW needed |
| Reserved IPs | 5 per VPC | 5 per subnet | Azure reserves per subnet |

## VNet Fundamentals

### Address Space Planning

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ADDRESS SPACE PLANNING                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ENTERPRISE ALLOCATION STRATEGY                                              │
│  ─────────────────────────────────                                           │
│                                                                              │
│  Corporate Network: 10.0.0.0/8                                               │
│  │                                                                           │
│  ├── Azure Production:    10.0.0.0/12  (10.0.0.0 - 10.15.255.255)          │
│  │   ├── East US:         10.0.0.0/14                                       │
│  │   ├── West US:         10.4.0.0/14                                       │
│  │   ├── West Europe:     10.8.0.0/14                                       │
│  │   └── Southeast Asia:  10.12.0.0/14                                      │
│  │                                                                           │
│  ├── Azure Non-Prod:      10.16.0.0/12 (10.16.0.0 - 10.31.255.255)         │
│  │   ├── Development:     10.16.0.0/14                                      │
│  │   ├── Staging:         10.20.0.0/14                                      │
│  │   └── Testing:         10.24.0.0/14                                      │
│  │                                                                           │
│  ├── On-Premises:         10.100.0.0/12                                     │
│  │                                                                           │
│  └── Reserved:            10.200.0.0/12 (Future expansion)                  │
│                                                                              │
│  AVOID OVERLAPPING WITH:                                                     │
│  • On-premises networks (critical for hybrid)                               │
│  • Other cloud providers (multi-cloud scenarios)                            │
│  • Azure reserved ranges (see below)                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Azure Reserved IP Addresses

Every subnet in Azure has 5 reserved addresses:

| Address | Purpose | Example (10.0.1.0/24) |
|---------|---------|----------------------|
| Network address | Identifies the subnet | 10.0.1.0 |
| Default gateway | Azure internal routing | 10.0.1.1 |
| DNS mapping | Azure DNS servers | 10.0.1.2, 10.0.1.3 |
| Broadcast | Broadcast address | 10.0.1.255 |

**Usable IPs per subnet:**
- /24 = 251 usable (256 - 5)
- /25 = 123 usable (128 - 5)
- /26 = 59 usable (64 - 5)
- /27 = 27 usable (32 - 5)
- /28 = 11 usable (16 - 5)
- /29 = 3 usable (minimum subnet size)

## Creating Virtual Networks

### Azure CLI

```bash
# Create a VNet with initial subnet
az network vnet create \
  --resource-group myResourceGroup \
  --name myVNet \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name web-subnet \
  --subnet-prefixes 10.0.1.0/24 \
  --location eastus

# Add additional address space (non-contiguous supported)
az network vnet update \
  --resource-group myResourceGroup \
  --name myVNet \
  --address-prefixes 10.0.0.0/16 10.1.0.0/16

# Add more subnets
az network vnet subnet create \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name app-subnet \
  --address-prefixes 10.0.2.0/24

az network vnet subnet create \
  --resource-group myResourceGroup \
  --vnet-name myVNet \
  --name data-subnet \
  --address-prefixes 10.0.3.0/24

# List VNet details
az network vnet show \
  --resource-group myResourceGroup \
  --name myVNet \
  --output table
```

### Bicep Template

```bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Environment name')
@allowed(['dev', 'staging', 'prod'])
param environment string = 'dev'

// VNet configuration
var vnetConfig = {
  name: 'vnet-${environment}-${location}'
  addressPrefixes: ['10.0.0.0/16']
  subnets: [
    {
      name: 'snet-web'
      addressPrefix: '10.0.1.0/24'
      serviceEndpoints: []
      delegations: []
    }
    {
      name: 'snet-app'
      addressPrefix: '10.0.2.0/24'
      serviceEndpoints: [
        { service: 'Microsoft.Sql' }
        { service: 'Microsoft.Storage' }
      ]
      delegations: []
    }
    {
      name: 'snet-data'
      addressPrefix: '10.0.3.0/24'
      serviceEndpoints: []
      delegations: []
    }
    {
      name: 'AzureBastionSubnet'  // Must use this exact name
      addressPrefix: '10.0.255.0/27'
      serviceEndpoints: []
      delegations: []
    }
  ]
}

resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: vnetConfig.name
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: vnetConfig.addressPrefixes
    }
    subnets: [for subnet in vnetConfig.subnets: {
      name: subnet.name
      properties: {
        addressPrefix: subnet.addressPrefix
        serviceEndpoints: subnet.serviceEndpoints
        delegations: subnet.delegations
      }
    }]
  }
}

output vnetId string = vnet.id
output vnetName string = vnet.name
output subnetIds array = [for (subnet, i) in vnetConfig.subnets: vnet.properties.subnets[i].id]
```

### Terraform

```hcl
# Variables
variable "resource_group_name" {
  type        = string
  description = "Resource group name"
}

variable "location" {
  type        = string
  description = "Azure region"
  default     = "eastus"
}

variable "environment" {
  type        = string
  description = "Environment name"
  default     = "dev"
}

# VNet
resource "azurerm_virtual_network" "main" {
  name                = "vnet-${var.environment}-${var.location}"
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = ["10.0.0.0/16"]

  tags = {
    environment = var.environment
  }
}

# Subnets
resource "azurerm_subnet" "web" {
  name                 = "snet-web"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "app" {
  name                 = "snet-app"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.2.0/24"]
  service_endpoints    = ["Microsoft.Sql", "Microsoft.Storage"]
}

resource "azurerm_subnet" "data" {
  name                 = "snet-data"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = ["10.0.3.0/24"]
}

# Outputs
output "vnet_id" {
  value = azurerm_virtual_network.main.id
}

output "subnet_ids" {
  value = {
    web  = azurerm_subnet.web.id
    app  = azurerm_subnet.app.id
    data = azurerm_subnet.data.id
  }
}
```

## Subnet Design Patterns

### Special-Purpose Subnets

Azure requires specific subnet names for certain services:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SPECIAL-PURPOSE SUBNETS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  REQUIRED SUBNET NAMES (Exact match required)                               │
│  ─────────────────────────────────────────────                              │
│                                                                              │
│  ┌────────────────────────┬──────────────┬──────────────────────────────┐  │
│  │ Service                │ Subnet Name  │ Minimum Size                  │  │
│  ├────────────────────────┼──────────────┼──────────────────────────────┤  │
│  │ VPN Gateway            │ GatewaySubnet│ /27 (recommended /26)        │  │
│  │ ExpressRoute Gateway   │ GatewaySubnet│ /27 (recommended /26)        │  │
│  │ Azure Bastion          │ AzureBastionSubnet │ /26 minimum            │  │
│  │ Azure Firewall         │ AzureFirewallSubnet│ /26 minimum            │  │
│  │ Azure Firewall Mgmt    │ AzureFirewallManagementSubnet │ /26 min    │  │
│  │ Application Gateway    │ (any name)   │ /24 recommended              │  │
│  │ Route Server           │ RouteServerSubnet │ /27 minimum             │  │
│  └────────────────────────┴──────────────┴──────────────────────────────┘  │
│                                                                              │
│  DELEGATED SUBNETS (Service-managed)                                        │
│  ────────────────────────────────────                                       │
│                                                                              │
│  • Azure Container Instances: Microsoft.ContainerInstance/containerGroups   │
│  • App Service VNet Integration: Microsoft.Web/serverFarms                  │
│  • Azure NetApp Files: Microsoft.NetApp/volumes                             │
│  • Azure SQL Managed Instance: Microsoft.Sql/managedInstances               │
│  • Azure Databricks: Microsoft.Databricks/workspaces                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hub-Spoke Subnet Layout

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     HUB VNET SUBNET LAYOUT                                   │
│                     10.0.0.0/16                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ GatewaySubnet              │ 10.0.0.0/26   │ VPN/ExpressRoute         │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ AzureFirewallSubnet        │ 10.0.1.0/26   │ Azure Firewall           │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ AzureFirewallMgmtSubnet    │ 10.0.1.64/26  │ Firewall Management      │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ AzureBastionSubnet         │ 10.0.2.0/26   │ Azure Bastion            │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ snet-management            │ 10.0.3.0/24   │ Jump boxes, tools        │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ snet-shared-services       │ 10.0.4.0/24   │ DNS, AD, monitoring      │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ snet-private-endpoints     │ 10.0.5.0/24   │ Centralized PEs          │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                     SPOKE VNET SUBNET LAYOUT                                 │
│                     10.1.0.0/16 (Production)                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ snet-web                   │ 10.1.1.0/24   │ Web tier VMs/VMSS        │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ snet-app                   │ 10.1.2.0/24   │ Application tier         │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ snet-data                  │ 10.1.3.0/24   │ Database VMs             │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ snet-aks                   │ 10.1.4.0/22   │ AKS nodes (/22 for scale)│  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ snet-appgw                 │ 10.1.8.0/24   │ Application Gateway      │  │
│  ├────────────────────────────┼───────────────┼──────────────────────────┤  │
│  │ snet-private-endpoints     │ 10.1.9.0/24   │ Private Endpoints        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## VNet Peering

VNet Peering allows you to connect VNets seamlessly through the Azure backbone.

### Peering Types

| Type | Description | Latency | Cost |
|------|-------------|---------|------|
| Regional Peering | VNets in same region | Sub-millisecond | ~$0.01/GB |
| Global Peering | VNets in different regions | Varies by distance | ~$0.035/GB |

### Peering Configuration

```bash
# Create bidirectional peering between hub and spoke

# Hub to Spoke
az network vnet peering create \
  --resource-group hub-rg \
  --name hub-to-spoke1 \
  --vnet-name hub-vnet \
  --remote-vnet /subscriptions/{sub-id}/resourceGroups/spoke1-rg/providers/Microsoft.Network/virtualNetworks/spoke1-vnet \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --allow-gateway-transit  # Hub has the gateway

# Spoke to Hub
az network vnet peering create \
  --resource-group spoke1-rg \
  --name spoke1-to-hub \
  --vnet-name spoke1-vnet \
  --remote-vnet /subscriptions/{sub-id}/resourceGroups/hub-rg/providers/Microsoft.Network/virtualNetworks/hub-vnet \
  --allow-vnet-access \
  --allow-forwarded-traffic \
  --use-remote-gateways  # Use hub's gateway
```

### Peering Limitations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       VNET PEERING CONSIDERATIONS                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  NOT TRANSITIVE:                                                             │
│  ─────────────────                                                           │
│                                                                              │
│  VNet A ←→ VNet B ←→ VNet C                                                 │
│                                                                              │
│  VNet A CANNOT reach VNet C through VNet B automatically.                   │
│  Solution: Use hub-spoke with Azure Firewall/NVA for routing                │
│                                                                              │
│  OVERLAPPING ADDRESS SPACES:                                                 │
│  ────────────────────────────                                                │
│                                                                              │
│  VNet A: 10.0.0.0/16 ←→ VNet B: 10.0.0.0/16   ❌ NOT ALLOWED               │
│                                                                              │
│  ADDRESS SPACE CHANGES:                                                      │
│  ──────────────────────                                                      │
│                                                                              │
│  • Can add/remove address spaces while peering exists (recent feature)      │
│  • Must resync peering after changes                                        │
│  • Cannot modify to create overlap                                          │
│                                                                              │
│  MAXIMUM PEERINGS: 500 per VNet                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Network Integration Options

### Service Endpoints

Service Endpoints extend your VNet identity to Azure services:

```bash
# Enable service endpoint on subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name app-subnet \
  --service-endpoints Microsoft.Storage Microsoft.Sql Microsoft.KeyVault
```

### Private Endpoints

Private Endpoints bring Azure services into your VNet:

```bash
# Create private endpoint for storage
az network private-endpoint create \
  --resource-group myRG \
  --name storage-pe \
  --vnet-name myVNet \
  --subnet pe-subnet \
  --private-connection-resource-id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{storage} \
  --group-id blob \
  --connection-name storage-pe-connection
```

### Comparison

| Feature | Service Endpoints | Private Endpoints |
|---------|-------------------|-------------------|
| Traffic path | Azure backbone | Your VNet |
| IP address | Service public IP | Private IP in VNet |
| On-prem access | No | Yes (via VPN/ER) |
| Cross-region | No | Yes |
| Cost | Free | ~$7.30/month + data |
| DNS | No change needed | Private DNS zone |

## Best Practices

### 1. Plan Address Space Carefully

- Avoid RFC 1918 overlap with on-premises
- Reserve space for future growth (use /16 per VNet minimum)
- Document all allocations centrally

### 2. Use Consistent Naming

```
vnet-<environment>-<region>-<workload>
snet-<tier/purpose>

Examples:
vnet-prod-eastus-hub
vnet-prod-eastus-spoke1
snet-web
snet-app
snet-data
```

### 3. Implement Network Segmentation

- Separate tiers into different subnets
- Use NSGs on every subnet
- Consider Application Security Groups for dynamic workloads

### 4. Enable DDoS Protection

```bash
# Create DDoS protection plan
az network ddos-protection create \
  --resource-group myRG \
  --name myDdosPlan

# Associate with VNet
az network vnet update \
  --resource-group myRG \
  --name myVNet \
  --ddos-protection-plan myDdosPlan
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Cannot create subnet | Address overlap | Check existing subnets |
| Peering stuck in "Initiated" | Missing reverse peering | Create peering from both sides |
| Cannot reach peered VNet | Peering not connected | Verify both peerings show "Connected" |
| Gateway transit not working | Wrong peering settings | Check allow-gateway-transit and use-remote-gateways |

### Diagnostic Commands

```bash
# Show VNet details
az network vnet show --resource-group myRG --name myVNet

# List all subnets
az network vnet subnet list --resource-group myRG --vnet-name myVNet -o table

# Check peering status
az network vnet peering list --resource-group myRG --vnet-name myVNet -o table

# Verify effective routes
az network nic show-effective-route-table --resource-group myRG --name myNIC
```

---

*Continue to [Gateways & Connectivity](02-gateways-connectivity.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
