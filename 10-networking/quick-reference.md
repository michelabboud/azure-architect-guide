# Networking Quick Reference

## VNet Essentials

### Address Space Planning

```
RECOMMENDED CIDR ALLOCATION:
────────────────────────────

Enterprise Network: 10.0.0.0/8 (or your corporate allocation)
│
├── Hub VNet:     10.0.0.0/16
│   ├── GatewaySubnet:           10.0.0.0/27   (VPN/ExpressRoute)
│   ├── AzureFirewallSubnet:     10.0.1.0/26   (Firewall - min /26)
│   ├── AzureBastionSubnet:      10.0.2.0/27   (Bastion - min /27)
│   └── ManagementSubnet:        10.0.3.0/24   (Jump boxes, etc.)
│
├── Spoke 1 (Prod):  10.1.0.0/16
│   ├── WebTier:     10.1.1.0/24
│   ├── AppTier:     10.1.2.0/24
│   ├── DataTier:    10.1.3.0/24
│   └── PrivateEndpoints: 10.1.4.0/24
│
├── Spoke 2 (Dev):   10.2.0.0/16
│   └── ...
│
└── On-Premises:     192.168.0.0/16

SUBNET SIZING GUIDE:
────────────────────
/24 = 251 usable IPs    (Azure reserves 5 per subnet)
/25 = 123 usable IPs
/26 = 59 usable IPs
/27 = 27 usable IPs
/28 = 11 usable IPs

RESERVED IPs PER SUBNET:
• .0   Network address
• .1   Default gateway
• .2   DNS (Azure provided)
• .3   DNS (Azure provided)
• .255 Broadcast
```

### AWS to Azure Network Commands

| Operation | AWS CLI | Azure CLI |
|-----------|---------|-----------|
| Create VNet/VPC | `aws ec2 create-vpc --cidr-block 10.0.0.0/16` | `az network vnet create --address-prefixes 10.0.0.0/16` |
| Create Subnet | `aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24` | `az network vnet subnet create --vnet-name vnet --address-prefixes 10.0.1.0/24` |
| Create NAT Gateway | `aws ec2 create-nat-gateway --subnet-id subnet-xxx` | `az network nat gateway create --name nat` |
| Create Peering | `aws ec2 create-vpc-peering-connection` | `az network vnet peering create` |
| List VNets | `aws ec2 describe-vpcs` | `az network vnet list` |

---

## Connectivity Options

### Decision Tree

```
                    ┌─────────────────────────────┐
                    │ What are you connecting?    │
                    └─────────────┬───────────────┘
                                  │
           ┌──────────────────────┼──────────────────────┐
           │                      │                      │
           ▼                      ▼                      ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│ VNet to VNet        │ │ VNet to On-Premises │ │ VNet to Internet    │
│ (Azure to Azure)    │ │                     │ │                     │
└──────────┬──────────┘ └──────────┬──────────┘ └──────────┬──────────┘
           │                       │                       │
           ▼                       ▼                       ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│ Same Region?        │ │ Bandwidth needs?    │ │ Outbound or Inbound?│
│                     │ │                     │ │                     │
│ Yes → VNet Peering  │ │ < 1Gbps → VPN      │ │ Out → NAT Gateway   │
│ No  → Global Peering│ │ > 1Gbps →          │ │ In  → Public IP     │
│       or VPN/ER     │ │   ExpressRoute      │ │       or LB         │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
```

### Connectivity Comparison

| Connection Type | Bandwidth | Latency | Use Case | Cost |
|-----------------|-----------|---------|----------|------|
| VNet Peering (same region) | VNet bandwidth | < 1ms | VNet-to-VNet same region | ~$0.01/GB |
| Global VNet Peering | VNet bandwidth | Varies | VNet-to-VNet cross-region | ~$0.035/GB |
| VPN Gateway (Basic) | 100 Mbps | ~50ms | Dev/test, backup | ~$27/month |
| VPN Gateway (VpnGw1) | 650 Mbps | ~50ms | Small production | ~$140/month |
| VPN Gateway (VpnGw3) | 1.25 Gbps | ~50ms | Large production | ~$490/month |
| ExpressRoute (50 Mbps) | 50 Mbps | < 10ms | Hybrid workloads | ~$55/month |
| ExpressRoute (1 Gbps) | 1 Gbps | < 10ms | Enterprise hybrid | ~$440/month |
| ExpressRoute (10 Gbps) | 10 Gbps | < 10ms | Data-intensive | ~$4,400/month |

---

## Gateway Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         GATEWAY COMPARISON                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  VPN GATEWAY                               EXPRESSROUTE GATEWAY             │
│  ───────────                               ─────────────────────             │
│                                                                              │
│  ┌─────────────────────┐                   ┌─────────────────────┐          │
│  │ Encrypted over      │                   │ Private dedicated   │          │
│  │ public internet     │                   │ connection          │          │
│  └─────────────────────┘                   └─────────────────────┘          │
│                                                                              │
│  Bandwidth: Up to 1.25 Gbps                Bandwidth: Up to 10+ Gbps        │
│  Latency: ~50-100ms (internet)             Latency: < 10ms (private)        │
│  Security: IPsec encryption                Security: Private line           │
│  Redundancy: Active-active possible        Redundancy: Dual circuits        │
│                                                                              │
│  Use when:                                 Use when:                         │
│  • Cost-sensitive                          • High bandwidth needed           │
│  • Dev/test workloads                      • Low latency critical            │
│  • Backup connectivity                     • Compliance requires private     │
│  • Remote workers (P2S)                    • Hybrid production workloads    │
│                                                                              │
│  AWS Equivalent:                           AWS Equivalent:                  │
│  Site-to-Site VPN                          Direct Connect                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Private Endpoints vs Service Endpoints

```
SERVICE ENDPOINT (Legacy):
──────────────────────────
• Traffic stays on Azure backbone (not internet)
• Source IP is VNet private IP
• Service still has public endpoint
• Free

             ┌──────────────┐
   VNet      │  Storage     │
┌─────────┐  │  Account     │
│   VM    │──┼──────────────┤
│         │  │  Public IP   │
└─────────┘  │  (filtered)  │
             └──────────────┘
                    ↑
        Traffic via Azure backbone
        but service has public endpoint


PRIVATE ENDPOINT (Modern - Recommended):
────────────────────────────────────────
• Private IP in your VNet
• No public endpoint exposure
• Works across VNet peering
• Works with on-premises
• Cost: ~$7.30/month + data

             ┌──────────────────────────────┐
   VNet      │  VNet                        │
┌─────────┐  │  ┌─────────────┐             │
│   VM    │──┼──│ Private     │──┐          │
│         │  │  │ Endpoint    │  │          │
└─────────┘  │  │ 10.0.1.50   │  │          │
             │  └─────────────┘  │          │
             └───────────────────┼──────────┘
                                 │
                                 ▼
                         ┌──────────────┐
                         │   Storage    │
                         │   Account    │
                         │  (No public  │
                         │   access)    │
                         └──────────────┘
```

### Quick Commands

```bash
# Create Private Endpoint for Storage
az network private-endpoint create \
  --resource-group myRG \
  --name storage-pe \
  --vnet-name myVNet \
  --subnet pe-subnet \
  --private-connection-resource-id /subscriptions/.../storageAccounts/mystorageacct \
  --group-id blob \
  --connection-name storage-connection

# Create Private DNS Zone (required for name resolution)
az network private-dns zone create \
  --resource-group myRG \
  --name privatelink.blob.core.windows.net

# Link to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name privatelink.blob.core.windows.net \
  --name vnet-link \
  --virtual-network myVNet \
  --registration-enabled false
```

---

## Route Tables (UDR)

### System Routes vs User-Defined Routes

```
SYSTEM ROUTES (Automatic):
──────────────────────────
• VNet address space → VNet
• 0.0.0.0/0 → Internet
• Peered VNets → VNet peering
• VPN/ExpressRoute → Gateway

USER-DEFINED ROUTES (Override system):
──────────────────────────────────────
• Force traffic through firewall
• Route to NVA
• Block internet access

EXAMPLE: Force all traffic through firewall

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  Route Table: spoke-rt                                                       │
│  ┌────────────────────────────────────────────────────────────────────────┐ │
│  │ Address Prefix     │ Next Hop Type    │ Next Hop Address               │ │
│  ├────────────────────┼──────────────────┼────────────────────────────────┤ │
│  │ 0.0.0.0/0          │ Virtual Appliance│ 10.0.1.4 (Firewall private IP) │ │
│  │ 10.0.0.0/8         │ Virtual Appliance│ 10.0.1.4 (Force through FW)    │ │
│  └────────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
│  Associate to: spoke1-web-subnet, spoke1-app-subnet                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Create route table
az network route-table create \
  --resource-group myRG \
  --name spoke-rt

# Add route to force through firewall
az network route-table route create \
  --resource-group myRG \
  --route-table-name spoke-rt \
  --name to-internet \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4

# Associate with subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name spoke1-vnet \
  --name web-subnet \
  --route-table spoke-rt
```

---

## DNS Quick Reference

### DNS Resolution Flow

```
1. VM queries Azure DNS (168.63.129.16)
2. Azure DNS checks:
   a. Private DNS zones linked to VNet
   b. VNet DNS settings (custom DNS if configured)
   c. Azure-provided resolution
   d. Public DNS (if not private)

PRIVATE DNS ZONE SETUP:
───────────────────────

┌────────────────────────────────────────────────────────────────────┐
│                    Private DNS Zone                                 │
│                    contoso.internal                                 │
│                                                                     │
│  Records:                                                           │
│  ┌────────────────────────────────────────────────────────────┐   │
│  │ Name        │ Type │ TTL  │ Value                          │   │
│  ├─────────────┼──────┼──────┼────────────────────────────────┤   │
│  │ app1        │ A    │ 300  │ 10.0.1.10                      │   │
│  │ database    │ A    │ 300  │ 10.0.2.20                      │   │
│  │ *.blob      │ CNAME│ 300  │ storage.privatelink.blob...    │   │
│  └────────────────────────────────────────────────────────────┘   │
│                                                                     │
│  Linked VNets: hub-vnet, spoke1-vnet, spoke2-vnet                 │
└────────────────────────────────────────────────────────────────────┘
```

```bash
# Create Private DNS Zone
az network private-dns zone create \
  --resource-group myRG \
  --name contoso.internal

# Link to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name contoso.internal \
  --name hub-link \
  --virtual-network hub-vnet \
  --registration-enabled true  # Auto-register VMs

# Add record
az network private-dns record-set a add-record \
  --resource-group myRG \
  --zone-name contoso.internal \
  --record-set-name app1 \
  --ipv4-address 10.0.1.10
```

---

## Common Network Architectures

### Architecture 1: Simple Web App

```bash
# Single VNet with web and data tiers
az network vnet create --name simple-vnet --address-prefixes 10.0.0.0/16 \
  --subnet-name web --subnet-prefixes 10.0.1.0/24
az network vnet subnet create --vnet-name simple-vnet --name data \
  --address-prefixes 10.0.2.0/24
```

### Architecture 2: Hub-Spoke with Firewall

```bash
# See Chapter 11 for full implementation
# Hub: Firewall, Bastion, Gateways
# Spokes: Workloads peered to hub
```

### Architecture 3: Virtual WAN

```bash
# Enterprise-scale with multiple regions
az network vwan create --name my-vwan --resource-group myRG
az network vhub create --name hub-eastus --vwan my-vwan \
  --address-prefix 10.0.0.0/24 --location eastus
```

---

*Deep Dive: [Virtual Networks](01-virtual-networks.md)* | *Back to [Chapter Overview](README.md)*
