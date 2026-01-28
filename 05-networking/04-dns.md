# DNS Deep Dive

## Overview

Azure DNS provides name resolution services for Azure resources and hybrid environments. Understanding Azure DNS, Private DNS zones, and DNS forwarding is essential for designing robust network architectures.

## AWS to Azure Comparison

| AWS Service | Azure Equivalent | Key Difference |
|-------------|------------------|----------------|
| Route 53 Public Hosted Zone | Azure DNS Zone | Similar functionality |
| Route 53 Private Hosted Zone | Azure Private DNS Zone | Requires VNet linking |
| Route 53 Resolver | Azure DNS Private Resolver | Newer service, more flexible |
| Route 53 Health Checks | Traffic Manager / Front Door | Separate services in Azure |

## Azure DNS Services Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AZURE DNS SERVICES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  AZURE DNS (Public)                                                  │    │
│  │  • Host public DNS zones                                             │    │
│  │  • Global anycast network                                            │    │
│  │  • Alias records for Azure resources                                 │    │
│  │  • DNSSEC support                                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  AZURE PRIVATE DNS                                                   │    │
│  │  • Private name resolution within VNets                              │    │
│  │  • Auto-registration of VM hostnames                                 │    │
│  │  • Required for Private Endpoints                                    │    │
│  │  • Cross-VNet resolution via linking                                 │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  AZURE DNS PRIVATE RESOLVER                                          │    │
│  │  • Conditional forwarding to on-premises                             │    │
│  │  • On-premises to Azure DNS resolution                               │    │
│  │  • Replaces need for DNS VMs                                         │    │
│  │  • Inbound and outbound endpoints                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Azure-Provided DNS

Every VNet has Azure-provided DNS by default.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AZURE-PROVIDED DNS                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DEFAULT DNS SERVER: 168.63.129.16 (Azure internal)                         │
│                                                                              │
│  RESOLUTION CAPABILITIES:                                                    │
│  • Public internet names ✓                                                  │
│  • Azure service endpoints ✓                                                │
│  • VM names within same VNet ✓ (internal.cloudapp.net)                     │
│  • Private DNS zones (if linked) ✓                                         │
│  • On-premises names ✗ (requires custom DNS)                                │
│  • Cross-VNet VM names ✗ (requires Private DNS)                            │
│                                                                              │
│  VM DEFAULT NAMING:                                                          │
│  <vm-name>.<region>.cloudapp.azure.com (public, if public IP)              │
│  <vm-name>.internal.cloudapp.net (internal, same VNet only)                │
│                                                                              │
│  LIMITATIONS:                                                                │
│  • Cannot customize internal suffix                                         │
│  • No cross-VNet resolution                                                 │
│  • No on-premises integration                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Private DNS Zones

### Creating Private DNS Zones

```bash
# Create Private DNS Zone
az network private-dns zone create \
  --resource-group myRG \
  --name contoso.internal

# Link to VNet (with auto-registration)
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name contoso.internal \
  --name hub-vnet-link \
  --virtual-network hub-vnet \
  --registration-enabled true

# Link to spoke VNet (no auto-registration - resolution only)
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name contoso.internal \
  --name spoke1-vnet-link \
  --virtual-network spoke1-vnet \
  --registration-enabled false

# Add A record manually
az network private-dns record-set a add-record \
  --resource-group myRG \
  --zone-name contoso.internal \
  --record-set-name app1 \
  --ipv4-address 10.1.1.10

# Add CNAME record
az network private-dns record-set cname set-record \
  --resource-group myRG \
  --zone-name contoso.internal \
  --record-set-name www \
  --cname app1.contoso.internal
```

### Bicep Template

```bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Private DNS zone name')
param zoneName string = 'contoso.internal'

@description('VNets to link')
param vnetsToLink array = [
  {
    name: 'hub-vnet'
    id: '/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/hub-vnet'
    registrationEnabled: true
  }
  {
    name: 'spoke1-vnet'
    id: '/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/spoke1-vnet'
    registrationEnabled: false
  }
]

// Private DNS Zone
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: zoneName
  location: 'global'
  properties: {}
}

// VNet Links
resource vnetLinks 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = [for vnet in vnetsToLink: {
  parent: privateDnsZone
  name: '${vnet.name}-link'
  location: 'global'
  properties: {
    virtualNetwork: {
      id: vnet.id
    }
    registrationEnabled: vnet.registrationEnabled
  }
}]

// A Record
resource aRecord 'Microsoft.Network/privateDnsZones/A@2020-06-01' = {
  parent: privateDnsZone
  name: 'app1'
  properties: {
    ttl: 300
    aRecords: [
      {
        ipv4Address: '10.1.1.10'
      }
    ]
  }
}

output zoneId string = privateDnsZone.id
```

### Auto-Registration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PRIVATE DNS AUTO-REGISTRATION                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  WHEN REGISTRATION-ENABLED = TRUE:                                          │
│                                                                              │
│  1. VM deployed to linked VNet                                              │
│     ↓                                                                        │
│  2. Azure automatically creates A record:                                   │
│     <vm-name>.<private-zone> → VM private IP                               │
│     ↓                                                                        │
│  3. When VM deleted, record automatically removed                           │
│                                                                              │
│  RULES:                                                                      │
│  • Only ONE VNet link can have registration enabled per zone                │
│  • VMs only (not other resources)                                           │
│  • Uses VM name as hostname                                                 │
│  • Creates A record only (no reverse PTR)                                   │
│                                                                              │
│  EXAMPLE:                                                                    │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  Private DNS Zone: contoso.internal                                 │    │
│  │                                                                     │    │
│  │  VNet Link: hub-vnet (registration-enabled: true)                  │    │
│  │                                                                     │    │
│  │  ┌──────────────┬────────┬─────────────────────────────────────┐  │    │
│  │  │ Name         │ Type   │ Value                               │  │    │
│  │  ├──────────────┼────────┼─────────────────────────────────────┤  │    │
│  │  │ web-vm-1     │ A      │ 10.0.1.4 (auto-registered)         │  │    │
│  │  │ web-vm-2     │ A      │ 10.0.1.5 (auto-registered)         │  │    │
│  │  │ app-server   │ A      │ 10.0.2.10 (manual)                 │  │    │
│  │  └──────────────┴────────┴─────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Private Endpoint DNS

Private Endpoints require specific DNS zones for name resolution.

### Required Private DNS Zones

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   PRIVATE ENDPOINT DNS ZONES                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SERVICE                    PRIVATE DNS ZONE                                 │
│  ───────                    ────────────────                                 │
│                                                                              │
│  Storage (Blob)             privatelink.blob.core.windows.net               │
│  Storage (File)             privatelink.file.core.windows.net               │
│  Storage (Table)            privatelink.table.core.windows.net              │
│  Storage (Queue)            privatelink.queue.core.windows.net              │
│  Storage (Web)              privatelink.web.core.windows.net                │
│  Storage (Data Lake)        privatelink.dfs.core.windows.net                │
│                                                                              │
│  SQL Database               privatelink.database.windows.net                │
│  SQL Managed Instance       privatelink.<region>.database.windows.net       │
│  Cosmos DB (SQL)            privatelink.documents.azure.com                 │
│  PostgreSQL                 privatelink.postgres.database.azure.com         │
│  MySQL                      privatelink.mysql.database.azure.com            │
│                                                                              │
│  Key Vault                  privatelink.vaultcore.azure.net                 │
│  App Configuration          privatelink.azconfig.io                         │
│  Event Hub                  privatelink.servicebus.windows.net              │
│  Service Bus                privatelink.servicebus.windows.net              │
│                                                                              │
│  Azure Container Registry   privatelink.azurecr.io                          │
│  App Service                privatelink.azurewebsites.net                   │
│  Azure Functions            privatelink.azurewebsites.net                   │
│  Azure Search               privatelink.search.windows.net                  │
│  Azure Monitor              privatelink.monitor.azure.com                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### DNS Resolution Flow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                PRIVATE ENDPOINT DNS RESOLUTION FLOW                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Query: mystorageaccount.blob.core.windows.net                              │
│                                                                              │
│  STEP 1: Client queries Azure DNS (168.63.129.16)                           │
│          │                                                                   │
│          ▼                                                                   │
│  STEP 2: Public DNS returns CNAME                                           │
│          mystorageaccount.blob.core.windows.net                             │
│          → mystorageaccount.privatelink.blob.core.windows.net               │
│          │                                                                   │
│          ▼                                                                   │
│  STEP 3: Azure DNS checks Private DNS Zones linked to VNet                  │
│          │                                                                   │
│          ├── Zone: privatelink.blob.core.windows.net (linked)              │
│          │   Record: mystorageaccount → 10.0.5.4                            │
│          │                                                                   │
│          ▼                                                                   │
│  STEP 4: Returns Private Endpoint IP: 10.0.5.4                              │
│                                                                              │
│  WITHOUT PRIVATE DNS ZONE:                                                   │
│  Resolution would return public IP, bypassing Private Endpoint              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Creating Private Endpoint with DNS

```bash
# Create Private DNS Zone for Storage
az network private-dns zone create \
  --resource-group myRG \
  --name privatelink.blob.core.windows.net

# Link to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name privatelink.blob.core.windows.net \
  --name hub-vnet-link \
  --virtual-network hub-vnet \
  --registration-enabled false

# Create Private Endpoint
az network private-endpoint create \
  --resource-group myRG \
  --name storage-pe \
  --vnet-name hub-vnet \
  --subnet pe-subnet \
  --private-connection-resource-id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --group-id blob \
  --connection-name storage-pe-connection

# Create DNS Zone Group (auto-registers DNS record)
az network private-endpoint dns-zone-group create \
  --resource-group myRG \
  --endpoint-name storage-pe \
  --name default \
  --private-dns-zone /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/privateDnsZones/privatelink.blob.core.windows.net \
  --zone-name privatelink-blob-core-windows-net
```

## Azure DNS Private Resolver

For hybrid DNS resolution between Azure and on-premises.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DNS PRIVATE RESOLVER ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ON-PREMISES                              AZURE                              │
│  ─────────────                            ─────                              │
│                                                                              │
│  ┌─────────────────┐                      ┌─────────────────────────────┐   │
│  │  On-Prem DNS    │                      │        Hub VNet             │   │
│  │  Server         │                      │                             │   │
│  │  10.100.1.10    │                      │  ┌─────────────────────┐    │   │
│  │                 │  ◄─── VPN/ER ───►    │  │ DNS Private Resolver│    │   │
│  │  Forwards to    │                      │  │                     │    │   │
│  │  Azure for:     │                      │  │ Inbound Endpoint    │    │   │
│  │  *.azure.com    │  ────────────────►   │  │ 10.0.10.4           │    │   │
│  │  *.windows.net  │                      │  │ (on-prem queries)   │    │   │
│  │                 │                      │  │                     │    │   │
│  │  Receives from  │                      │  │ Outbound Endpoint   │    │   │
│  │  Azure for:     │  ◄────────────────   │  │ 10.0.11.4           │    │   │
│  │  *.onprem.local │                      │  │ (Azure queries)     │    │   │
│  │                 │                      │  │                     │    │   │
│  └─────────────────┘                      │  └─────────────────────┘    │   │
│                                           │                             │   │
│                                           │  Private DNS Zones:         │   │
│                                           │  • contoso.internal         │   │
│                                           │  • privatelink.*.net        │   │
│                                           │                             │   │
│                                           └─────────────────────────────┘   │
│                                                                              │
│  RESOLUTION FLOWS:                                                           │
│                                                                              │
│  1. On-Prem → Azure Private Zone:                                           │
│     On-Prem DNS → Inbound Endpoint → Private DNS Zone                       │
│                                                                              │
│  2. Azure → On-Prem:                                                         │
│     VM → Outbound Endpoint → Forwarding Rule → On-Prem DNS                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Creating DNS Private Resolver

```bash
# Create DNS Resolver
az dns-resolver create \
  --resource-group myRG \
  --name dns-resolver \
  --location eastus \
  --id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/hub-vnet

# Create Inbound Endpoint (for on-prem to Azure queries)
az dns-resolver inbound-endpoint create \
  --resource-group myRG \
  --dns-resolver-name dns-resolver \
  --name inbound-endpoint \
  --location eastus \
  --ip-configurations '[{"private-ip-allocation-method":"Dynamic","id":"/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/hub-vnet/subnets/inbound-dns-subnet"}]'

# Create Outbound Endpoint (for Azure to on-prem queries)
az dns-resolver outbound-endpoint create \
  --resource-group myRG \
  --dns-resolver-name dns-resolver \
  --name outbound-endpoint \
  --location eastus \
  --id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/hub-vnet/subnets/outbound-dns-subnet

# Create Forwarding Ruleset
az dns-resolver forwarding-ruleset create \
  --resource-group myRG \
  --name forward-to-onprem \
  --location eastus \
  --outbound-endpoints '[{"id":"/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/dnsResolvers/dns-resolver/outboundEndpoints/outbound-endpoint"}]'

# Create Forwarding Rule
az dns-resolver forwarding-rule create \
  --resource-group myRG \
  --ruleset-name forward-to-onprem \
  --name onprem-local \
  --domain-name "onprem.local." \
  --forwarding-rule-state Enabled \
  --target-dns-servers '[{"ip-address":"10.100.1.10","port":53}]'

# Link Ruleset to VNet
az dns-resolver vnet-link create \
  --resource-group myRG \
  --ruleset-name forward-to-onprem \
  --name hub-vnet-link \
  --id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/hub-vnet
```

## Custom DNS Configuration

### Setting Custom DNS on VNet

```bash
# Set custom DNS servers for VNet
az network vnet update \
  --resource-group myRG \
  --name spoke-vnet \
  --dns-servers 10.0.10.4 10.0.10.5 168.63.129.16

# Reset to Azure-provided DNS
az network vnet update \
  --resource-group myRG \
  --name spoke-vnet \
  --dns-servers ""
```

### DNS Configuration Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      DNS CONFIGURATION PATTERNS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PATTERN 1: Azure-Only (Default)                                            │
│  ────────────────────────────────                                           │
│  • Use Azure-provided DNS (168.63.129.16)                                   │
│  • Link Private DNS zones to VNets                                          │
│  • Simple, no maintenance                                                    │
│  • No on-prem integration                                                    │
│                                                                              │
│  PATTERN 2: Hybrid with DNS Private Resolver                                │
│  ───────────────────────────────────────────                                │
│  • DNS Private Resolver in hub VNet                                         │
│  • Inbound endpoint for on-prem to Azure                                    │
│  • Outbound endpoint with forwarding rules                                  │
│  • VNets use Azure DNS (168.63.129.16)                                      │
│  • Recommended for new deployments                                          │
│                                                                              │
│  PATTERN 3: Hybrid with DNS VMs (Legacy)                                    │
│  ────────────────────────────────────────                                   │
│  • Deploy DNS VMs (Windows DNS or BIND)                                     │
│  • Configure conditional forwarding                                         │
│  • VNets point to DNS VMs                                                   │
│  • Higher maintenance, more control                                         │
│                                                                              │
│  PATTERN 4: On-Prem DNS Integration                                         │
│  ──────────────────────────────────                                         │
│  • VNets point to on-prem DNS                                               │
│  • On-prem DNS forwards Azure zones to Azure                                │
│  • Requires reliable connectivity                                           │
│  • Single point of failure                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Public DNS (Azure DNS)

### Creating Public DNS Zone

```bash
# Create public DNS zone
az network dns zone create \
  --resource-group myRG \
  --name contoso.com

# Add A record
az network dns record-set a add-record \
  --resource-group myRG \
  --zone-name contoso.com \
  --record-set-name www \
  --ipv4-address 203.0.113.10

# Add CNAME record
az network dns record-set cname set-record \
  --resource-group myRG \
  --zone-name contoso.com \
  --record-set-name blog \
  --cname contoso.azurewebsites.net

# Add Alias record (points to Azure resource)
az network dns record-set a create \
  --resource-group myRG \
  --zone-name contoso.com \
  --name apex \
  --target-resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/publicIPAddresses/myPublicIP

# Get name servers (update at registrar)
az network dns zone show \
  --resource-group myRG \
  --name contoso.com \
  --query nameServers
```

## Troubleshooting DNS

### Diagnostic Commands

```bash
# Test DNS resolution from VM (Azure CLI)
az vm run-command invoke \
  --resource-group myRG \
  --name myVM \
  --command-id RunShellScript \
  --scripts "nslookup mystorageaccount.blob.core.windows.net"

# Check Private DNS zone records
az network private-dns record-set list \
  --resource-group myRG \
  --zone-name contoso.internal \
  --output table

# Check VNet links
az network private-dns link vnet list \
  --resource-group myRG \
  --zone-name contoso.internal \
  --output table

# Verify Private Endpoint DNS
az network private-endpoint dns-zone-group list \
  --resource-group myRG \
  --endpoint-name storage-pe

# Check custom DNS servers on VNet
az network vnet show \
  --resource-group myRG \
  --name hub-vnet \
  --query "dhcpOptions.dnsServers"
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Private Endpoint not resolving | Missing Private DNS zone | Create and link zone |
| Cross-VNet resolution fails | Zone not linked | Add VNet link |
| On-prem can't resolve Azure | No inbound endpoint | Deploy DNS Private Resolver |
| Stale DNS records | Auto-registration lag | Wait or manually update |
| Public IP returned for PE | Zone not linked | Verify Private DNS zone link |

### DNS Resolution Testing

```bash
# From Azure VM (Linux)
nslookup mystorageaccount.blob.core.windows.net
dig mystorageaccount.blob.core.windows.net

# From Azure VM (Windows)
Resolve-DnsName mystorageaccount.blob.core.windows.net
nslookup mystorageaccount.blob.core.windows.net

# Check what DNS server is being used
cat /etc/resolv.conf  # Linux
Get-DnsClientServerAddress  # Windows PowerShell
```

## Best Practices

1. **Centralize Private DNS zones** in hub VNet/subscription
2. **Use DNS Zone Groups** for automatic Private Endpoint DNS registration
3. **Plan zone naming** to avoid conflicts with public domains
4. **Enable auto-registration** only for one VNet per zone
5. **Use DNS Private Resolver** instead of DNS VMs for new deployments
6. **Monitor DNS queries** using diagnostic logs
7. **Document DNS architecture** including all forwarding rules

---

*Continue to [Case Studies](case-studies.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
