# Private Endpoints & Private Link

## Overview

Azure Private Link enables private connectivity to Azure PaaS services and customer-owned services, eliminating exposure to the public internet. Traffic stays on the Microsoft backbone network.

## AWS Comparison

| Feature | AWS PrivateLink | Azure Private Link |
|---------|----------------|-------------------|
| Private access to services | Yes | Yes |
| DNS resolution | Route 53 Resolver | Azure Private DNS |
| Cross-region | Yes | Yes |
| Cross-account/tenant | Yes | Yes (manual approval) |
| Interface endpoints | ENI in VPC | NIC in subnet |
| Gateway endpoints | S3, DynamoDB | N/A (use service endpoints) |

## Service Endpoints vs Private Endpoints

| Feature | Service Endpoints | Private Endpoints |
|---------|------------------|-------------------|
| Traffic path | Microsoft backbone | Microsoft backbone |
| Source IP | Public IP | Private IP |
| DNS | Public endpoint | Private endpoint |
| Network isolation | Firewall rules | Full isolation |
| On-prem access | Requires NAT | Direct access |
| Cost | Free | Per hour + data |
| Services | Limited set | Most PaaS services |

### When to Use Each

```
Service Endpoints                    Private Endpoints
─────────────────                    ─────────────────
✓ Cost-sensitive                     ✓ On-premises access needed
✓ Azure-only access                  ✓ Full network isolation
✓ Simple setup                       ✓ Cross-region access
✓ Limited service set                ✓ Compliance requirements
                                     ✓ ExpressRoute/VPN integration
```

## Architecture

### Private Endpoint Flow

```
                    ┌───────────────────────────────────────────────────┐
                    │                    Your VNet                       │
                    │                                                    │
                    │  ┌─────────────┐      ┌─────────────────────────┐ │
                    │  │    VM       │      │   Private Endpoint      │ │
                    │  │             │──────│   (NIC: 10.0.1.10)     │ │
                    │  │ 10.0.1.4    │      │                         │ │
                    │  └─────────────┘      │   blob.core.windows.net │ │
                    │                       │   → 10.0.1.10           │ │
                    │                       └────────────┬────────────┘ │
                    │                                    │              │
                    └────────────────────────────────────┼──────────────┘
                                                         │
                                                         │ Private Link
                                                         │ (Azure Backbone)
                                                         │
                    ┌────────────────────────────────────▼──────────────┐
                    │                                                    │
                    │             Azure Storage Account                  │
                    │             (mystorageaccount)                     │
                    │                                                    │
                    │   Public access: Disabled                         │
                    │                                                    │
                    └────────────────────────────────────────────────────┘
```

### DNS Resolution

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    Private Endpoint DNS Resolution                        │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                           │
│  Without Private Endpoint:                                               │
│  ────────────────────────                                                │
│  mystorageaccount.blob.core.windows.net → 52.239.x.x (Public IP)        │
│                                                                           │
│  With Private Endpoint (Azure Private DNS Zone):                         │
│  ──────────────────────────────────────────────                          │
│  1. Query: mystorageaccount.blob.core.windows.net                        │
│  2. CNAME: mystorageaccount.privatelink.blob.core.windows.net            │
│  3. A Record: 10.0.1.10 (Private IP)                                     │
│                                                                           │
│  DNS Zone: privatelink.blob.core.windows.net                             │
│  └── Linked to VNet                                                       │
│                                                                           │
└──────────────────────────────────────────────────────────────────────────┘
```

## Configuration

### Create Private Endpoint

```bicep
// Private Endpoint for Storage Account
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-storage-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    subnet: {
      id: subnet.id
    }
    privateLinkServiceConnections: [
      {
        name: 'storage-connection'
        properties: {
          privateLinkServiceId: storageAccount.id
          groupIds: ['blob']
        }
      }
    ]
  }
}

// Private DNS Zone
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.blob.core.windows.net'
  location: 'global'
}

// Link DNS Zone to VNet
resource privateDnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZone
  name: 'vnet-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}

// DNS Zone Group (auto-create DNS records)
resource dnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-05-01' = {
  parent: privateEndpoint
  name: 'dns-zone-group'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'config1'
        properties: {
          privateDnsZoneId: privateDnsZone.id
        }
      }
    ]
  }
}
```

### CLI Commands

```bash
# Create private endpoint
az network private-endpoint create \
  --resource-group myRG \
  --name myStoragePE \
  --vnet-name myVNet \
  --subnet privateEndpointSubnet \
  --private-connection-resource-id /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --group-id blob \
  --connection-name myConnection

# Create private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name "privatelink.blob.core.windows.net"

# Link to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.blob.core.windows.net" \
  --name myVNetLink \
  --virtual-network myVNet \
  --registration-enabled false

# Create DNS record (or use DNS zone group for auto)
az network private-endpoint dns-zone-group create \
  --resource-group myRG \
  --endpoint-name myStoragePE \
  --name myZoneGroup \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name blob
```

## Common Private DNS Zones

| Service | Private DNS Zone |
|---------|-----------------|
| Blob Storage | privatelink.blob.core.windows.net |
| File Storage | privatelink.file.core.windows.net |
| Queue Storage | privatelink.queue.core.windows.net |
| Table Storage | privatelink.table.core.windows.net |
| Azure SQL | privatelink.database.windows.net |
| Cosmos DB | privatelink.documents.azure.com |
| Key Vault | privatelink.vaultcore.azure.net |
| ACR | privatelink.azurecr.io |
| Event Hub | privatelink.servicebus.windows.net |
| Azure ML | privatelink.api.azureml.ms |

## Hybrid Scenarios

### On-Premises DNS Resolution

```
┌───────────────────────────────────────────────────────────────────────┐
│                  On-Prem to Private Endpoint Access                    │
├───────────────────────────────────────────────────────────────────────┤
│                                                                        │
│   On-Premises                    Azure                                 │
│   ────────────                   ─────                                 │
│                                                                        │
│   ┌─────────────┐               ┌──────────────────────────────┐     │
│   │   Client    │               │        Hub VNet              │     │
│   │             │               │                              │     │
│   │ DNS: Azure  │──ExpressRoute─│  ┌──────────────────────┐   │     │
│   │ DNS Private │   /VPN        │  │  Azure DNS Private   │   │     │
│   │ Resolver    │               │  │  Resolver            │   │     │
│   └─────────────┘               │  │  (Inbound Endpoint)  │   │     │
│                                 │  └──────────┬───────────┘   │     │
│   DNS Forwarder:                │             │               │     │
│   *.blob.core.windows.net       │             ▼               │     │
│   → Azure DNS Resolver IP       │  ┌──────────────────────┐   │     │
│                                 │  │ Private DNS Zone     │   │     │
│                                 │  │ privatelink.blob...  │   │     │
│                                 │  └──────────────────────┘   │     │
│                                 └──────────────────────────────┘     │
│                                                                        │
└───────────────────────────────────────────────────────────────────────┘
```

### Azure DNS Private Resolver

```bicep
// DNS Private Resolver
resource dnsResolver 'Microsoft.Network/dnsResolvers@2022-07-01' = {
  name: 'dns-resolver'
  location: location
  properties: {
    virtualNetwork: {
      id: vnet.id
    }
  }
}

// Inbound Endpoint (for on-prem queries)
resource inboundEndpoint 'Microsoft.Network/dnsResolvers/inboundEndpoints@2022-07-01' = {
  parent: dnsResolver
  name: 'inbound'
  location: location
  properties: {
    ipConfigurations: [
      {
        privateIpAllocationMethod: 'Dynamic'
        subnet: {
          id: inboundSubnet.id
        }
      }
    ]
  }
}
```

## Best Practices

### 1. Centralized Private DNS

```
Hub VNet:
├── Private DNS Zones (all zones)
│   ├── privatelink.blob.core.windows.net
│   ├── privatelink.database.windows.net
│   └── privatelink.vaultcore.azure.net
│
├── DNS Private Resolver (for on-prem)
│
└── VNet Links (to all spoke VNets)
```

### 2. Disable Public Access

```bash
# Storage Account
az storage account update \
  --name mystorageaccount \
  --resource-group myRG \
  --public-network-access Disabled

# Azure SQL
az sql server update \
  --name mysqlserver \
  --resource-group myRG \
  --public-network-access Disabled

# Key Vault
az keyvault update \
  --name mykv \
  --resource-group myRG \
  --public-network-access Disabled
```

### 3. Use DNS Zone Groups

Always use DNS zone groups for automatic DNS record management instead of manual record creation.

---

*Continue to [Case Studies](case-studies.md)*

*Back to [Network Security Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
