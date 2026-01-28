# Routing Deep Dive

## Overview

Azure routing controls how network traffic flows within and between VNets. Understanding system routes, user-defined routes (UDRs), and BGP is essential for designing secure and efficient network architectures.

## AWS to Azure Comparison

| Concept | AWS | Azure | Key Difference |
|---------|-----|-------|----------------|
| Route Table | Route Table | Route Table | Similar concept |
| Default Routes | Explicit local route | System routes (implicit) | Azure has automatic routes |
| Custom Routes | Route entries | User-Defined Routes (UDR) | Same concept |
| Route Propagation | Gateway route propagation | Virtual network gateway route propagation | Similar |
| BGP | Supported over VPN/DX | Supported over VPN/ER | Same |

## System Routes

Azure automatically creates system routes for common scenarios.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AZURE SYSTEM ROUTES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AUTOMATICALLY CREATED ROUTES                                                │
│  ─────────────────────────────                                               │
│                                                                              │
│  ┌────────────────────┬───────────────────┬─────────────────────────────┐   │
│  │ Address Prefix     │ Next Hop Type     │ When Created                │   │
│  ├────────────────────┼───────────────────┼─────────────────────────────┤   │
│  │ VNet address space │ Virtual network   │ Always (for each VNet CIDR) │   │
│  │ 0.0.0.0/0          │ Internet          │ Always                      │   │
│  │ 10.0.0.0/8         │ None              │ Always (drops RFC1918)      │   │
│  │ 172.16.0.0/12      │ None              │ Always (drops RFC1918)      │   │
│  │ 192.168.0.0/16     │ None              │ Always (drops RFC1918)      │   │
│  │ 100.64.0.0/10      │ None              │ Always (CGNAT)              │   │
│  │ Peered VNet CIDR   │ VNet peering      │ When peering established    │   │
│  │ On-prem prefixes   │ Virtual network   │ When gateway propagation on │   │
│  │                    │ gateway           │                             │   │
│  └────────────────────┴───────────────────┴─────────────────────────────┘   │
│                                                                              │
│  ROUTE PRIORITY (Highest to Lowest):                                        │
│  1. User-Defined Routes (most specific prefix wins)                         │
│  2. BGP routes                                                              │
│  3. System routes                                                           │
│                                                                              │
│  NOTE: More specific prefixes always win regardless of route type           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## User-Defined Routes (UDR)

UDRs override system routes to control traffic flow.

### Common Next Hop Types

| Next Hop Type | Description | Use Case |
|---------------|-------------|----------|
| Virtual appliance | NVA or Firewall IP | Force traffic through firewall |
| Virtual network gateway | VPN/ExpressRoute gateway | Route to on-premises |
| Virtual network | Within VNet | Override 0.0.0.0/0 for specific prefixes |
| Internet | Public internet | Direct internet access for specific IPs |
| None | Drop traffic (blackhole) | Block specific destinations |

### Creating Route Tables

```bash
# Create route table
az network route-table create \
  --resource-group myRG \
  --name rt-spoke-to-firewall \
  --location eastus \
  --disable-bgp-route-propagation false

# Add route to force traffic through firewall
az network route-table route create \
  --resource-group myRG \
  --route-table-name rt-spoke-to-firewall \
  --name route-to-internet \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4  # Azure Firewall private IP

# Add route for east-west traffic through firewall
az network route-table route create \
  --resource-group myRG \
  --route-table-name rt-spoke-to-firewall \
  --name route-to-spoke2 \
  --address-prefix 10.2.0.0/16 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4

# Associate route table with subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name spoke1-vnet \
  --name workload-subnet \
  --route-table rt-spoke-to-firewall
```

### Bicep Template

```bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Azure Firewall private IP')
param firewallPrivateIp string

@description('VNet name to associate')
param vnetName string

@description('Subnet name to associate')
param subnetName string

// Route Table
resource routeTable 'Microsoft.Network/routeTables@2023-05-01' = {
  name: 'rt-spoke-to-firewall'
  location: location
  properties: {
    disableBgpRoutePropagation: false
    routes: [
      {
        name: 'route-to-internet'
        properties: {
          addressPrefix: '0.0.0.0/0'
          nextHopType: 'VirtualAppliance'
          nextHopIpAddress: firewallPrivateIp
        }
      }
      {
        name: 'route-to-onprem'
        properties: {
          addressPrefix: '10.100.0.0/16'
          nextHopType: 'VirtualAppliance'
          nextHopIpAddress: firewallPrivateIp
        }
      }
    ]
  }
}

// Associate with subnet
resource subnet 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' existing = {
  name: '${vnetName}/${subnetName}'
}

resource subnetRouteTableAssociation 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = {
  name: '${vnetName}/${subnetName}'
  properties: {
    addressPrefix: subnet.properties.addressPrefix
    routeTable: {
      id: routeTable.id
    }
  }
}

output routeTableId string = routeTable.id
```

## Common Routing Patterns

### Pattern 1: Hub-Spoke with Forced Tunneling

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HUB-SPOKE FORCED TUNNELING                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  ▲                                           │
│                                  │                                           │
│                    ┌─────────────┴─────────────┐                            │
│                    │      Azure Firewall       │                            │
│                    │      10.0.1.4             │                            │
│                    └─────────────┬─────────────┘                            │
│                                  │                                           │
│  ┌───────────────────────────────┴───────────────────────────────────────┐  │
│  │                         HUB VNET 10.0.0.0/16                           │  │
│  │                                                                        │  │
│  │   Route Table: rt-gateway-subnet                                       │  │
│  │   ┌────────────────────────────────────────────────────────────────┐  │  │
│  │   │ 10.1.0.0/16 → 10.0.1.4 (Spoke1 via FW)                        │  │  │
│  │   │ 10.2.0.0/16 → 10.0.1.4 (Spoke2 via FW)                        │  │  │
│  │   └────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │ (VNet Peering)                            │
│            ┌─────────────────────┼─────────────────────┐                    │
│            │                     │                     │                    │
│            ▼                     ▼                     ▼                    │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│  │  SPOKE 1        │   │  SPOKE 2        │   │  SPOKE 3        │          │
│  │  10.1.0.0/16    │   │  10.2.0.0/16    │   │  10.3.0.0/16    │          │
│  │                 │   │                 │   │                 │          │
│  │  Route Table:   │   │  Route Table:   │   │  Route Table:   │          │
│  │  ┌───────────┐  │   │  ┌───────────┐  │   │  ┌───────────┐  │          │
│  │  │0.0.0.0/0  │  │   │  │0.0.0.0/0  │  │   │  │0.0.0.0/0  │  │          │
│  │  │→ 10.0.1.4 │  │   │  │→ 10.0.1.4 │  │   │  │→ 10.0.1.4 │  │          │
│  │  │           │  │   │  │           │  │   │  │           │  │          │
│  │  │10.2.0.0/16│  │   │  │10.1.0.0/16│  │   │  │10.1.0.0/16│  │          │
│  │  │→ 10.0.1.4 │  │   │  │→ 10.0.1.4 │  │   │  │→ 10.0.1.4 │  │          │
│  │  └───────────┘  │   │  └───────────┘  │   │  └───────────┘  │          │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘          │
│                                                                              │
│  TRAFFIC FLOW:                                                              │
│  Spoke1 → Internet: Spoke1 → FW → Internet                                  │
│  Spoke1 → Spoke2:   Spoke1 → FW → Spoke2                                    │
│  Spoke1 → On-Prem:  Spoke1 → FW → Gateway → On-Prem                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Pattern 2: Asymmetric Routing Prevention

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  ASYMMETRIC ROUTING PROBLEM AND SOLUTION                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PROBLEM: Traffic from on-prem bypasses firewall on return                  │
│  ─────────────────────────────────────────────────────────                  │
│                                                                              │
│  On-Prem → Gateway → Spoke (direct via peering)                             │
│  Spoke → Firewall → Gateway → On-Prem                                       │
│                        ↑                                                     │
│                    ASYMMETRIC! Firewall sees only one direction             │
│                                                                              │
│  SOLUTION: Route table on GatewaySubnet                                     │
│  ───────────────────────────────────────                                    │
│                                                                              │
│  GatewaySubnet Route Table:                                                 │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Destination      │ Next Hop Type     │ Next Hop IP               │    │
│  ├──────────────────┼───────────────────┼───────────────────────────┤    │
│  │ 10.1.0.0/16      │ Virtual Appliance │ 10.0.1.4 (Firewall)       │    │
│  │ 10.2.0.0/16      │ Virtual Appliance │ 10.0.1.4 (Firewall)       │    │
│  │ 10.3.0.0/16      │ Virtual Appliance │ 10.0.1.4 (Firewall)       │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  NOW: On-Prem → Gateway → Firewall → Spoke (symmetric)                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Pattern 3: Service Chaining

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SERVICE CHAINING PATTERN                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Traffic flows through multiple NVAs in sequence                            │
│                                                                              │
│            ┌──────────────────────────────────────────────────────────┐     │
│            │                      HUB VNET                             │     │
│            │                                                           │     │
│            │   ┌─────────┐     ┌─────────┐     ┌─────────┐            │     │
│  Internet ─┼──►│   WAF   │────►│   IDS   │────►│ Firewall│────┐       │     │
│            │   │ 10.0.1.4│     │10.0.1.20│     │10.0.1.36│    │       │     │
│            │   └─────────┘     └─────────┘     └─────────┘    │       │     │
│            │                                                   │       │     │
│            └───────────────────────────────────────────────────┼───────┘     │
│                                                                │             │
│                                                                ▼             │
│            ┌───────────────────────────────────────────────────────────┐    │
│            │                    SPOKE VNET                              │    │
│            │                                                            │    │
│            │   Route Table:                                             │    │
│            │   0.0.0.0/0 → 10.0.1.4 (WAF - first hop)                  │    │
│            │                                                            │    │
│            │   WAF Route Table:                                         │    │
│            │   10.1.0.0/16 → 10.0.1.20 (IDS - second hop)              │    │
│            │                                                            │    │
│            │   IDS Route Table:                                         │    │
│            │   10.1.0.0/16 → 10.0.1.36 (Firewall - third hop)          │    │
│            │                                                            │    │
│            │   Firewall Route Table:                                    │    │
│            │   10.1.0.0/16 → VNet Peering                              │    │
│            │                                                            │    │
│            └────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## BGP Routing

BGP enables dynamic route exchange between Azure and on-premises networks.

### BGP Fundamentals

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          BGP IN AZURE                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  RESERVED ASN:                                                               │
│  • Azure uses ASN 65515 by default for VPN Gateways                         │
│  • ExpressRoute Microsoft peering uses 12076                                │
│  • Do not use: 65515, 65517, 65518, 65519, 65520 (Azure reserved)           │
│  • Do not use: 0, 23456, 64496-64511, 65535 (IANA reserved)                │
│                                                                              │
│  BGP PEER ADDRESSES:                                                         │
│  • VPN Gateway creates BGP peer IPs within GatewaySubnet                    │
│  • Active-active has two BGP peer addresses (one per instance)              │
│  • On-prem router must peer with both for full HA                           │
│                                                                              │
│  ROUTE ADVERTISEMENT:                                                        │
│  • VPN Gateway advertises VNet address space                                │
│  • Peered VNets advertised if "Use remote gateway" enabled                  │
│  • On-prem routes learned and propagated to subnets                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### BGP Configuration Example

```bash
# Create VPN Gateway with BGP
az network vnet-gateway create \
  --resource-group myRG \
  --name vpn-gateway \
  --vnet hub-vnet \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2AZ \
  --enable-bgp true \
  --asn 65515

# View BGP settings
az network vnet-gateway show \
  --resource-group myRG \
  --name vpn-gateway \
  --query "bgpSettings"

# Create local gateway with BGP
az network local-gateway create \
  --resource-group myRG \
  --name onprem-lng \
  --gateway-ip-address 203.0.113.1 \
  --asn 65001 \
  --bgp-peering-address 10.1.0.1

# Create connection with BGP enabled
az network vpn-connection create \
  --resource-group myRG \
  --name azure-to-onprem \
  --vnet-gateway1 vpn-gateway \
  --local-gateway2 onprem-lng \
  --shared-key "SecureKey123!" \
  --enable-bgp true
```

### BGP Route Manipulation

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       BGP ROUTE PREFERENCES                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ROUTE SELECTION ORDER (highest priority first):                            │
│  1. Longest prefix match                                                    │
│  2. Locally originated routes                                               │
│  3. Shortest AS path                                                        │
│  4. Lowest origin type (IGP < EGP < Incomplete)                            │
│  5. Lowest MED (Multi-Exit Discriminator)                                   │
│  6. eBGP over iBGP                                                          │
│  7. Lowest router ID                                                        │
│                                                                              │
│  INFLUENCING ROUTE SELECTION:                                               │
│                                                                              │
│  From Azure (advertise preference to on-prem):                              │
│  • Use AS path prepending on on-prem router                                 │
│  • Configure MED values                                                     │
│                                                                              │
│  To Azure (prefer specific path):                                           │
│  • On-prem: Prepend AS path for less-preferred routes                       │
│  • On-prem: Set higher MED for backup paths                                 │
│                                                                              │
│  EXAMPLE - PREFER EXPRESSROUTE OVER VPN:                                    │
│  On-prem advertises to VPN with AS path prepend:                            │
│    10.100.0.0/16 via ExpressRoute: AS 65001                                 │
│    10.100.0.0/16 via VPN: AS 65001 65001 65001 (prepended)                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Route Propagation

### Virtual Network Gateway Route Propagation

```bash
# Disable BGP route propagation (isolate subnet from on-prem routes)
az network route-table update \
  --resource-group myRG \
  --name rt-isolated-subnet \
  --disable-bgp-route-propagation true

# Enable BGP route propagation (default)
az network route-table update \
  --resource-group myRG \
  --name rt-spoke-subnet \
  --disable-bgp-route-propagation false
```

### When to Disable Route Propagation

| Scenario | Propagation | Reason |
|----------|-------------|--------|
| DMZ subnet | Disabled | Isolate from internal routes |
| Management subnet | Enabled | Need access to on-prem |
| AKS subnet | Depends | Disable if using kubenet |
| Azure Firewall subnet | Disabled | Firewall manages own routing |
| Application Gateway subnet | Depends | May need for backend routing |

## Azure Route Server

Route Server enables dynamic routing between NVAs and Azure.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AZURE ROUTE SERVER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PURPOSE:                                                                    │
│  • Exchange routes between NVAs and Azure VNets via BGP                     │
│  • No need for static UDRs that require manual updates                      │
│  • Enables SD-WAN and third-party NVA integration                           │
│                                                                              │
│                     ┌─────────────────────────────┐                         │
│                     │     Azure Route Server      │                         │
│                     │     (Managed BGP service)   │                         │
│                     │     ASN: 65515              │                         │
│                     └──────────┬──────────────────┘                         │
│                                │ BGP Peering                                 │
│            ┌───────────────────┼───────────────────┐                        │
│            │                   │                   │                        │
│            ▼                   ▼                   ▼                        │
│     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐                │
│     │  SD-WAN     │     │  Firewall   │     │  VPN        │                │
│     │  NVA        │     │  NVA        │     │  Gateway    │                │
│     └─────────────┘     └─────────────┘     └─────────────┘                │
│                                                                              │
│  REQUIREMENTS:                                                               │
│  • Dedicated subnet: RouteServerSubnet (/27 minimum)                        │
│  • NVAs must support BGP                                                    │
│  • Standard SKU                                                             │
│                                                                              │
│  BENEFITS:                                                                   │
│  • Dynamic route updates (no UDR management)                                │
│  • Automatic failover                                                       │
│  • Multi-NVA support                                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Creating Route Server

```bash
# Create RouteServerSubnet
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name hub-vnet \
  --name RouteServerSubnet \
  --address-prefixes 10.0.10.0/27

# Create Route Server
az network routeserver create \
  --resource-group myRG \
  --name route-server \
  --hosted-subnet /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/virtualNetworks/hub-vnet/subnets/RouteServerSubnet \
  --public-ip-address rs-pip

# Create BGP peering with NVA
az network routeserver peering create \
  --resource-group myRG \
  --routeserver route-server \
  --name nva-peering \
  --peer-asn 65001 \
  --peer-ip 10.0.5.4
```

## Troubleshooting Routing

### Diagnostic Commands

```bash
# View effective routes for a NIC
az network nic show-effective-route-table \
  --resource-group myRG \
  --name myVM-nic \
  --output table

# View effective routes for a VM
az network watcher show-next-hop \
  --resource-group myRG \
  --vm myVM \
  --source-ip 10.1.1.4 \
  --dest-ip 10.2.1.4

# View BGP learned routes
az network vnet-gateway list-learned-routes \
  --resource-group myRG \
  --name vpn-gateway \
  --output table

# View BGP advertised routes
az network vnet-gateway list-advertised-routes \
  --resource-group myRG \
  --name vpn-gateway \
  --peer 10.1.0.1 \
  --output table

# View BGP peer status
az network vnet-gateway list-bgp-peer-status \
  --resource-group myRG \
  --name vpn-gateway \
  --output table
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Traffic not reaching firewall | Missing UDR | Add 0.0.0.0/0 route to firewall |
| Asymmetric routing | Missing GatewaySubnet UDR | Add spoke routes to GatewaySubnet |
| BGP routes not propagating | Propagation disabled | Enable on route table |
| NVA not receiving traffic | IP forwarding disabled | Enable on NIC |
| Routes not taking effect | Wrong subnet association | Verify route table association |

### Enable IP Forwarding on NVA

```bash
# Enable IP forwarding on NIC (required for NVAs)
az network nic update \
  --resource-group myRG \
  --name nva-nic \
  --ip-forwarding true

# Also enable in OS (Linux)
# sudo sysctl -w net.ipv4.ip_forward=1
# Add to /etc/sysctl.conf for persistence
```

## Best Practices

1. **Document all routes** - Maintain a routing matrix showing all subnets and their route tables
2. **Use consistent naming** - `rt-<vnet>-<subnet>-<purpose>`
3. **Test before production** - Validate routes don't break existing connectivity
4. **Monitor route changes** - Enable diagnostics and alerts
5. **Plan for growth** - Consider future subnets and services in route design

---

*Continue to [DNS](04-dns.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
