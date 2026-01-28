# Gateways and Connectivity Deep Dive

## Overview

Azure provides multiple connectivity options for hybrid and multi-cloud architectures. This chapter covers VPN Gateway, ExpressRoute, and VNet Peering - the core building blocks for enterprise network connectivity.

## AWS to Azure Comparison

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| Site-to-Site VPN | VPN Gateway | Similar, multiple SKUs |
| Client VPN | Point-to-Site VPN | Integrated in VPN Gateway |
| Direct Connect | ExpressRoute | Private dedicated connection |
| Transit Gateway | Virtual WAN / Hub VNet | Virtual WAN for large scale |
| VPC Peering | VNet Peering | Global peering available |

## VPN Gateway

### VPN Gateway SKUs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         VPN GATEWAY SKU COMPARISON                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────┬────────────┬────────────┬──────────┬───────────────────┐  │
│  │ SKU          │ Throughput │ S2S Tunnels│ P2S Conn │ Zone Redundant    │  │
│  ├──────────────┼────────────┼────────────┼──────────┼───────────────────┤  │
│  │ Basic        │ 100 Mbps   │ 10         │ 128      │ No                │  │
│  │ VpnGw1       │ 650 Mbps   │ 30         │ 250      │ No                │  │
│  │ VpnGw2       │ 1 Gbps     │ 30         │ 500      │ No                │  │
│  │ VpnGw3       │ 1.25 Gbps  │ 30         │ 1000     │ No                │  │
│  │ VpnGw4       │ 5 Gbps     │ 100        │ 5000     │ No                │  │
│  │ VpnGw5       │ 10 Gbps    │ 100        │ 10000    │ No                │  │
│  │ VpnGw1AZ     │ 650 Mbps   │ 30         │ 250      │ Yes               │  │
│  │ VpnGw2AZ     │ 1 Gbps     │ 30         │ 500      │ Yes               │  │
│  │ VpnGw3AZ     │ 1.25 Gbps  │ 30         │ 1000     │ Yes               │  │
│  │ VpnGw4AZ     │ 5 Gbps     │ 100        │ 5000     │ Yes               │  │
│  │ VpnGw5AZ     │ 10 Gbps    │ 100        │ 10000    │ Yes               │  │
│  └──────────────┴────────────┴────────────┴──────────┴───────────────────┘  │
│                                                                              │
│  RECOMMENDATIONS:                                                            │
│  • Dev/Test: VpnGw1 (cost effective)                                        │
│  • Production: VpnGw2AZ or higher (zone redundancy)                         │
│  • High bandwidth: VpnGw4AZ or VpnGw5AZ                                     │
│  • Basic SKU: Legacy, avoid for new deployments                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Site-to-Site VPN Configuration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      SITE-TO-SITE VPN ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ON-PREMISES                           AZURE                                 │
│  ─────────────                         ─────                                 │
│                                                                              │
│  ┌─────────────────┐                   ┌─────────────────────────────────┐  │
│  │  Corporate      │                   │           Hub VNet              │  │
│  │  Network        │                   │                                 │  │
│  │  10.1.0.0/16    │                   │  ┌───────────────────────────┐ │  │
│  │                 │                   │  │     GatewaySubnet         │ │  │
│  │  ┌───────────┐  │   IPsec Tunnel    │  │                           │ │  │
│  │  │  VPN      │──┼───────────────────┼──│  ┌─────────────────────┐  │ │  │
│  │  │  Device   │  │   (Public IPs)    │  │  │   VPN Gateway       │  │ │  │
│  │  └───────────┘  │                   │  │  │   (Active-Active)   │  │ │  │
│  │                 │                   │  │  └─────────────────────┘  │ │  │
│  └─────────────────┘                   │  └───────────────────────────┘ │  │
│                                        │                                 │  │
│                                        │  10.0.0.0/16                    │  │
│                                        └─────────────────────────────────┘  │
│                                                                              │
│  CONNECTION OPTIONS:                                                         │
│  • Active-Standby: Single tunnel (default)                                  │
│  • Active-Active: Dual tunnels for HA (recommended)                         │
│  • BGP: Dynamic routing between sites                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Creating VPN Gateway

```bash
# Create GatewaySubnet (required name)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name hub-vnet \
  --name GatewaySubnet \
  --address-prefixes 10.0.255.0/27

# Create public IPs for active-active gateway
az network public-ip create \
  --resource-group myRG \
  --name vpn-pip1 \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

az network public-ip create \
  --resource-group myRG \
  --name vpn-pip2 \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

# Create VPN Gateway (takes 30-45 minutes)
az network vnet-gateway create \
  --resource-group myRG \
  --name vpn-gateway \
  --vnet hub-vnet \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2AZ \
  --public-ip-addresses vpn-pip1 vpn-pip2 \
  --active-active true \
  --enable-bgp true \
  --asn 65515

# Create Local Network Gateway (represents on-prem)
az network local-gateway create \
  --resource-group myRG \
  --name onprem-lng \
  --gateway-ip-address <on-prem-vpn-public-ip> \
  --local-address-prefixes 10.1.0.0/16 192.168.0.0/16 \
  --bgp-peering-address 10.1.0.1 \
  --asn 65001

# Create VPN Connection
az network vpn-connection create \
  --resource-group myRG \
  --name azure-to-onprem \
  --vnet-gateway1 vpn-gateway \
  --local-gateway2 onprem-lng \
  --shared-key "YourPreSharedKey123!" \
  --enable-bgp true
```

### Bicep Template for VPN Gateway

```bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('VNet name')
param vnetName string

@description('VPN Gateway SKU')
@allowed(['VpnGw1AZ', 'VpnGw2AZ', 'VpnGw3AZ', 'VpnGw4AZ', 'VpnGw5AZ'])
param gatewaySku string = 'VpnGw2AZ'

// Public IPs for active-active
resource vpnPip1 'Microsoft.Network/publicIPAddresses@2023-05-01' = {
  name: 'pip-vpn-1'
  location: location
  sku: { name: 'Standard' }
  zones: ['1', '2', '3']
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

resource vpnPip2 'Microsoft.Network/publicIPAddresses@2023-05-01' = {
  name: 'pip-vpn-2'
  location: location
  sku: { name: 'Standard' }
  zones: ['1', '2', '3']
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

// VPN Gateway
resource vpnGateway 'Microsoft.Network/virtualNetworkGateways@2023-05-01' = {
  name: 'vpn-gateway'
  location: location
  properties: {
    gatewayType: 'Vpn'
    vpnType: 'RouteBased'
    activeActive: true
    enableBgp: true
    sku: {
      name: gatewaySku
      tier: gatewaySku
    }
    bgpSettings: {
      asn: 65515
    }
    ipConfigurations: [
      {
        name: 'vnetGatewayConfig1'
        properties: {
          publicIPAddress: { id: vpnPip1.id }
          subnet: { id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, 'GatewaySubnet') }
        }
      }
      {
        name: 'vnetGatewayConfig2'
        properties: {
          publicIPAddress: { id: vpnPip2.id }
          subnet: { id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, 'GatewaySubnet') }
        }
      }
    ]
  }
}

output gatewayId string = vpnGateway.id
output gatewayBgpPeeringAddress string = vpnGateway.properties.bgpSettings.bgpPeeringAddress
```

## ExpressRoute

### Overview

ExpressRoute provides dedicated private connectivity to Azure through a connectivity provider.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       EXPRESSROUTE ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ON-PREMISES          PROVIDER EDGE          MICROSOFT EDGE       AZURE     │
│  ─────────────        ─────────────          ──────────────       ─────     │
│                                                                              │
│  ┌──────────────┐    ┌──────────────┐       ┌──────────────┐    ┌────────┐ │
│  │  Corporate   │    │  Provider    │       │  Microsoft   │    │  VNet  │ │
│  │  Router      │────│  PE Router   │───────│  Enterprise  │────│        │ │
│  │              │    │              │       │  Edge (MSEE) │    │        │ │
│  └──────────────┘    └──────────────┘       └──────────────┘    └────────┘ │
│        │                   │                       │                        │
│        │                   │                       │                        │
│  ┌──────────────┐    ┌──────────────┐       ┌──────────────┐    ┌────────┐ │
│  │  Corporate   │    │  Provider    │       │  Microsoft   │    │  VNet  │ │
│  │  Router      │────│  PE Router   │───────│  Enterprise  │────│        │ │
│  │  (Secondary) │    │  (Secondary) │       │  Edge (MSEE) │    │(Other) │ │
│  └──────────────┘    └──────────────┘       └──────────────┘    └────────┘ │
│                                                                              │
│  CIRCUIT COMPONENTS:                                                         │
│  • Primary Connection:   Active path                                        │
│  • Secondary Connection: Redundant path (different PE routers)              │
│  • Both paths must be configured for SLA                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ExpressRoute SKUs

| SKU | Description | Use Case |
|-----|-------------|----------|
| Local | Access to Azure region near peering location | Cost-optimized local workloads |
| Standard | Access to all regions within a geopolitical region | Regional deployments |
| Premium | Global access + more routes | Multi-region, global deployments |

### ExpressRoute Bandwidths

| Bandwidth | Monthly Cost (approx.) | Use Case |
|-----------|------------------------|----------|
| 50 Mbps | ~$55 | Dev/test, small workloads |
| 100 Mbps | ~$110 | Small production |
| 200 Mbps | ~$180 | Medium workloads |
| 500 Mbps | ~$360 | Large workloads |
| 1 Gbps | ~$440 | Enterprise production |
| 2 Gbps | ~$880 | High-bandwidth |
| 5 Gbps | ~$2,200 | Data-intensive |
| 10 Gbps | ~$4,400 | Maximum bandwidth |

### Creating ExpressRoute

```bash
# Create ExpressRoute Circuit
az network express-route create \
  --resource-group myRG \
  --name er-circuit-east \
  --peering-location "Washington DC" \
  --provider "Equinix" \
  --bandwidth-in-mbps 1000 \
  --sku-tier Premium \
  --sku-family MeteredData

# Get service key (provide to connectivity provider)
az network express-route show \
  --resource-group myRG \
  --name er-circuit-east \
  --query serviceKey

# Create ExpressRoute Gateway (after provider provisions circuit)
az network public-ip create \
  --resource-group myRG \
  --name er-pip \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

az network vnet-gateway create \
  --resource-group myRG \
  --name er-gateway \
  --vnet hub-vnet \
  --gateway-type ExpressRoute \
  --sku ErGw2AZ \
  --public-ip-address er-pip

# Create connection to circuit
az network vpn-connection create \
  --resource-group myRG \
  --name hub-to-er \
  --vnet-gateway1 er-gateway \
  --express-route-circuit2 /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/expressRouteCircuits/er-circuit-east
```

### ExpressRoute Gateway SKUs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EXPRESSROUTE GATEWAY SKU COMPARISON                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌────────────────┬─────────────────┬────────────────┬───────────────────┐  │
│  │ SKU            │ Max Throughput  │ Connections    │ Zone Redundant    │  │
│  ├────────────────┼─────────────────┼────────────────┼───────────────────┤  │
│  │ Standard       │ 1 Gbps          │ 4              │ No                │  │
│  │ HighPerformance│ 2 Gbps          │ 8              │ No                │  │
│  │ UltraPerformance│ 10 Gbps        │ 16             │ No                │  │
│  │ ErGw1Az        │ 1 Gbps          │ 4              │ Yes               │  │
│  │ ErGw2Az        │ 2 Gbps          │ 8              │ Yes               │  │
│  │ ErGw3Az        │ 10 Gbps         │ 16             │ Yes               │  │
│  │ ErGwScale      │ 1-40 Gbps*      │ Scalable       │ Yes               │  │
│  └────────────────┴─────────────────┴────────────────┴───────────────────┘  │
│                                                                              │
│  * ErGwScale: New scalable SKU with per-unit pricing                        │
│                                                                              │
│  RECOMMENDATIONS:                                                            │
│  • Production: ErGw2Az or higher                                            │
│  • High throughput: ErGw3Az or ErGwScale                                    │
│  • Always use zone-redundant SKUs for production                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ExpressRoute + VPN Coexistence

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  EXPRESSROUTE + VPN COEXISTENCE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ON-PREMISES                              AZURE HUB VNET                     │
│  ─────────────                            ──────────────                     │
│                                                                              │
│  ┌─────────────────┐                      ┌─────────────────────────────┐   │
│  │  Corporate      │                      │      GatewaySubnet          │   │
│  │  Data Center    │                      │                             │   │
│  │                 │   ExpressRoute       │  ┌─────────────────────┐    │   │
│  │  ┌───────────┐  │   (Primary)          │  │   ExpressRoute      │    │   │
│  │  │  Router   │──┼──────────────────────┼──│   Gateway           │    │   │
│  │  │           │  │                      │  │                     │    │   │
│  │  │           │  │   VPN (Backup)       │  └─────────────────────┘    │   │
│  │  │           │──┼──────────────────────┼──┐                          │   │
│  │  └───────────┘  │                      │  │ ┌─────────────────────┐  │   │
│  │                 │                      │  └─│   VPN Gateway       │  │   │
│  └─────────────────┘                      │    │   (Standby)         │  │   │
│                                           │    └─────────────────────┘  │   │
│                                           └─────────────────────────────┘   │
│                                                                              │
│  CONFIGURATION:                                                              │
│  • ExpressRoute: Primary path with lower BGP weight                         │
│  • VPN: Backup path, activates on ExpressRoute failure                      │
│  • Both can be active-active for different traffic types                    │
│                                                                              │
│  REQUIREMENTS:                                                               │
│  • GatewaySubnet must be /27 or larger                                      │
│  • Both gateways deployed in same GatewaySubnet                             │
│  • Route-based VPN required                                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Virtual WAN

For large-scale hub-and-spoke deployments, Virtual WAN provides managed hub infrastructure.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       AZURE VIRTUAL WAN ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         ┌─────────────────────────────┐                     │
│                         │      VIRTUAL WAN            │                     │
│                         │   (Global resource)         │                     │
│                         └─────────────┬───────────────┘                     │
│                                       │                                      │
│           ┌───────────────────────────┼───────────────────────────┐         │
│           │                           │                           │         │
│           ▼                           ▼                           ▼         │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐ │
│  │  Virtual Hub    │        │  Virtual Hub    │        │  Virtual Hub    │ │
│  │  (East US)      │◄──────►│  (West Europe)  │◄──────►│  (Southeast     │ │
│  │                 │ Global │                 │  Hub   │   Asia)         │ │
│  │  ┌───────────┐  │ Hub-   │  ┌───────────┐  │  to    │  ┌───────────┐  │ │
│  │  │VPN Gateway│  │ Hub    │  │ER Gateway │  │  Hub   │  │VPN Gateway│  │ │
│  │  │Firewall   │  │        │  │Firewall   │  │        │  │           │  │ │
│  │  └───────────┘  │        │  └───────────┘  │        │  └───────────┘  │ │
│  └────────┬────────┘        └────────┬────────┘        └────────┬────────┘ │
│           │                          │                          │          │
│     ┌─────┴─────┐              ┌─────┴─────┐              ┌─────┴─────┐    │
│     │           │              │           │              │           │    │
│   Spoke      Spoke           Spoke      Spoke           Spoke      Branch │
│   VNets      VNets           VNets      VNets           VNets      (VPN)  │
│                                                                              │
│  KEY FEATURES:                                                               │
│  • Managed hub infrastructure                                               │
│  • Automatic hub-to-hub connectivity                                        │
│  • Integrated VPN, ExpressRoute, Firewall                                   │
│  • Simplified routing                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Virtual WAN vs Custom Hub-Spoke

| Feature | Custom Hub-Spoke | Virtual WAN |
|---------|-----------------|-------------|
| Hub management | You manage | Microsoft managed |
| Hub-to-hub routing | Manual UDRs | Automatic |
| Scalability | 500 peerings/VNet | 2000+ connections |
| Gateway management | You manage | Integrated |
| Cost model | Resource-based | Hub + connection fees |
| Customization | Full control | Limited |

## Point-to-Site VPN

For remote worker access to Azure resources.

```bash
# Generate root certificate (on admin machine)
# Using PowerShell:
# $cert = New-SelfSignedCertificate -Type Custom -KeySpec Signature \
#   -Subject "CN=P2SRootCert" -KeyExportPolicy Exportable \
#   -HashAlgorithm sha256 -KeyLength 2048 \
#   -CertStoreLocation "Cert:\CurrentUser\My" -KeyUsageProperty Sign

# Configure P2S on existing VPN Gateway
az network vnet-gateway update \
  --resource-group myRG \
  --name vpn-gateway \
  --address-prefixes 172.16.0.0/24 \
  --client-protocol OpenVPN \
  --vpn-client-root-certificates "[{\"name\":\"P2SRootCert\",\"publicCertData\":\"<base64-encoded-cert>\"}]"

# Download VPN client configuration
az network vnet-gateway vpn-client generate \
  --resource-group myRG \
  --name vpn-gateway
```

### P2S Authentication Options

| Method | Description | Use Case |
|--------|-------------|----------|
| Certificate | Client certs from trusted root | Traditional, offline capable |
| Azure AD | Entra ID authentication | Corporate users, SSO |
| RADIUS | External RADIUS server | Existing identity infrastructure |

## Connectivity Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONNECTIVITY DECISION TREE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  What are you connecting?                                                    │
│  │                                                                           │
│  ├── Azure VNet to Azure VNet                                               │
│  │   │                                                                       │
│  │   ├── Same region? → VNet Peering                                        │
│  │   ├── Different regions? → Global VNet Peering                           │
│  │   └── Many VNets (10+)? → Virtual WAN                                    │
│  │                                                                           │
│  ├── Azure to On-Premises                                                   │
│  │   │                                                                       │
│  │   ├── Bandwidth < 1 Gbps?                                                │
│  │   │   ├── Cost sensitive? → VPN Gateway                                  │
│  │   │   └── Need redundancy? → Active-Active VPN                           │
│  │   │                                                                       │
│  │   ├── Bandwidth >= 1 Gbps?                                               │
│  │   │   ├── Need private connection? → ExpressRoute                        │
│  │   │   └── Need backup? → ExpressRoute + VPN                              │
│  │   │                                                                       │
│  │   └── Many branch offices? → Virtual WAN with VPN                        │
│  │                                                                           │
│  └── Remote Users to Azure                                                  │
│      │                                                                       │
│      ├── Corporate users with Entra ID? → P2S with Azure AD                 │
│      ├── Third-party users? → P2S with Certificates                         │
│      └── Large scale? → Azure Virtual Desktop                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Best Practices

### 1. High Availability Design

- Use Active-Active VPN Gateways
- Deploy zone-redundant gateway SKUs
- Implement ExpressRoute + VPN coexistence for critical workloads

### 2. Security

- Use IPsec with IKEv2
- Implement custom IPsec policies for compliance
- Enable BGP for dynamic routing

### 3. Monitoring

```bash
# Enable gateway diagnostics
az monitor diagnostic-settings create \
  --resource /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworkGateways/vpn-gateway \
  --name gateway-diagnostics \
  --logs '[{"category":"GatewayDiagnosticLog","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --workspace /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/my-workspace
```

### 4. Cost Optimization

- Right-size gateway SKUs
- Use ExpressRoute Local where applicable
- Consider Virtual WAN for large deployments

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| VPN tunnel down | Incorrect PSK | Verify shared key matches |
| Intermittent connectivity | MTU issues | Reduce TCP MSS to 1350 |
| BGP not establishing | ASN conflict | Ensure unique ASN per site |
| ExpressRoute not connected | Provider not provisioned | Check circuit status with provider |

### Diagnostic Commands

```bash
# Check VPN connection status
az network vpn-connection show \
  --resource-group myRG \
  --name azure-to-onprem \
  --query connectionStatus

# View BGP peer status
az network vnet-gateway list-bgp-peer-status \
  --resource-group myRG \
  --name vpn-gateway

# Check learned routes
az network vnet-gateway list-learned-routes \
  --resource-group myRG \
  --name vpn-gateway

# ExpressRoute circuit status
az network express-route show \
  --resource-group myRG \
  --name er-circuit-east \
  --query "serviceProviderProvisioningState"
```

---

*Continue to [Routing](03-routing.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
