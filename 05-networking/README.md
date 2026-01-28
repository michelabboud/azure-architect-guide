# Chapter 10: Azure Networking Fundamentals

## From AWS VPC to Azure VNet

This chapter covers the core networking concepts in Azure, mapped from your AWS VPC knowledge.

## Chapter Contents

| Document | Purpose | Time |
|----------|---------|------|
| [Quick Reference](quick-reference.md) | Networking cheat sheet | 5 min |
| [01 - Virtual Networks](01-virtual-networks.md) | VNets, subnets, addressing | 30 min |
| [02 - Gateways & Connectivity](02-gateways-connectivity.md) | VPN, ExpressRoute, peering | 30 min |
| [03 - Routing](03-routing.md) | Route tables, UDRs, BGP | 20 min |
| [04 - DNS](04-dns.md) | Azure DNS, Private DNS zones | 20 min |
| [Case Studies](case-studies.md) | Network architecture scenarios | 20 min |

---

## AWS to Azure Networking Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AWS VPC vs AZURE VNET ARCHITECTURE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS:                                  Azure:                               │
│  ────                                  ─────                                │
│                                                                              │
│  ┌─────────────────────┐              ┌─────────────────────┐              │
│  │       Region        │              │       Region        │              │
│  │  ┌───────────────┐  │              │  ┌───────────────┐  │              │
│  │  │      VPC      │  │              │  │     VNet      │  │              │
│  │  │  10.0.0.0/16  │  │              │  │  10.0.0.0/16  │  │              │
│  │  │               │  │              │  │               │  │              │
│  │  │ ┌───────────┐ │  │              │  │ ┌───────────┐ │  │              │
│  │  │ │Subnet AZ-a│ │  │              │  │ │  Subnet   │ │  │              │
│  │  │ │10.0.1.0/24│ │  │              │  │ │10.0.1.0/24│ │  │              │
│  │  │ └───────────┘ │  │              │  │ │(spans AZs)│ │  │              │
│  │  │ ┌───────────┐ │  │              │  │ └───────────┘ │  │              │
│  │  │ │Subnet AZ-b│ │  │              │  │ ┌───────────┐ │  │              │
│  │  │ │10.0.2.0/24│ │  │              │  │ │  Subnet   │ │  │              │
│  │  │ └───────────┘ │  │              │  │ │10.0.2.0/24│ │  │              │
│  │  └───────────────┘  │              │  │ │(spans AZs)│ │  │              │
│  │                     │              │  │ └───────────┘ │  │              │
│  │  [Internet Gateway] │              │  └───────────────┘  │              │
│  │  [NAT Gateway]      │              │                     │              │
│  └─────────────────────┘              │  [Implicit Internet]│              │
│                                       │  [NAT Gateway]      │              │
│                                       └─────────────────────┘              │
│                                                                              │
│  KEY DIFFERENCE: Azure subnets span ALL Availability Zones in a region     │
│                  AWS subnets are tied to a single AZ                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Service Mapping

| AWS Concept | Azure Equivalent | Key Difference |
|-------------|------------------|----------------|
| VPC | Virtual Network (VNet) | VNet subnets span all AZs |
| Subnet | Subnet | Not AZ-specific |
| Internet Gateway | (Implicit) | No separate resource needed |
| NAT Gateway | NAT Gateway | Similar functionality |
| Elastic IP | Public IP | Standard vs Basic SKU |
| Route Table | Route Table | Similar with system routes |
| NACL | (NSG at subnet level) | NSGs replace NACLs |
| Security Group | Network Security Group (NSG) | Can attach to subnet or NIC |
| VPC Peering | VNet Peering | Global VNet Peering available |
| Transit Gateway | Azure Virtual WAN | Virtual WAN for hub-spoke |
| Direct Connect | ExpressRoute | Dedicated private connection |
| VPN Gateway | VPN Gateway | Site-to-site and P2S |
| PrivateLink | Private Link / Private Endpoint | Very similar |
| VPC Endpoints | Service Endpoints / Private Endpoints | Two options |
| Route 53 (Private) | Azure Private DNS | Private DNS zones |
| Route 53 (Public) | Azure DNS | Public DNS hosting |

---

## Architecture Patterns

### Basic Single VNet

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BASIC VNET ARCHITECTURE                              │
│                         (Equivalent to simple VPC)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│                    ┌─────────────┴─────────────┐                            │
│                    │      Public IP Addresses   │                            │
│                    │  (Attached to resources)   │                            │
│                    └─────────────┬─────────────┘                            │
│                                  │                                           │
│  ┌───────────────────────────────┴───────────────────────────────────────┐  │
│  │                         VNET: 10.0.0.0/16                              │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  WEB SUBNET: 10.0.1.0/24                                        │  │  │
│  │  │  • NSG attached (allow 443 from internet)                       │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                      │  │  │
│  │  │  │  VM 1    │  │  VM 2    │  │  VM 3    │ (Load balanced)      │  │  │
│  │  │  │  (AZ 1)  │  │  (AZ 2)  │  │  (AZ 3)  │                      │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘                      │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  DATA SUBNET: 10.0.2.0/24                                       │  │  │
│  │  │  • NSG attached (allow SQL from web subnet only)                │  │  │
│  │  │  • Private Endpoint for Azure SQL                               │  │  │
│  │  │  ┌──────────────────────────────┐                              │  │  │
│  │  │  │  Private Endpoint            │                              │  │  │
│  │  │  │  (Azure SQL Database)        │                              │  │  │
│  │  │  └──────────────────────────────┘                              │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hub-Spoke Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      HUB-SPOKE NETWORK TOPOLOGY                              │
│              (Equivalent to AWS Transit Gateway pattern)                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│                    ┌─────────────┴─────────────┐                            │
│                    │     Azure Firewall        │                            │
│                    │     OR NVA                │                            │
│                    └─────────────┬─────────────┘                            │
│                                  │                                           │
│  ┌───────────────────────────────┴───────────────────────────────────────┐  │
│  │                           HUB VNET                                     │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │  Firewall   │  │ VPN Gateway │  │ ExpressRoute│  │   Bastion   │  │  │
│  │  │  Subnet     │  │   Subnet    │  │   Gateway   │  │   Subnet    │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │ (VNet Peering)                            │
│            ┌─────────────────────┼─────────────────────┐                    │
│            │                     │                     │                    │
│            ▼                     ▼                     ▼                    │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│  │  SPOKE 1        │   │  SPOKE 2        │   │  SPOKE 3        │          │
│  │  (Production)   │   │  (Development)  │   │  (Shared Svcs)  │          │
│  │                 │   │                 │   │                 │          │
│  │  10.1.0.0/16    │   │  10.2.0.0/16    │   │  10.3.0.0/16    │          │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘          │
│                                                                              │
│  TRAFFIC FLOW:                                                              │
│  • Spoke → Hub → Internet (via Firewall)                                   │
│  • Spoke → Hub → On-premises (via VPN/ExpressRoute)                        │
│  • Spoke → Hub → Spoke (via Firewall inspection)                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Availability Zones in Azure

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              AVAILABILITY ZONES: AWS vs AZURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS APPROACH:                          AZURE APPROACH:                     │
│  ─────────────                          ───────────────                     │
│                                                                              │
│  Subnet per AZ:                         Subnet spans all AZs:               │
│  ┌─────────────────┐                    ┌─────────────────────────────────┐ │
│  │     VPC         │                    │           VNet                  │ │
│  │ ┌─────────────┐ │                    │ ┌─────────────────────────────┐ │ │
│  │ │ Subnet-1a   │ │                    │ │        Subnet-Web           │ │ │
│  │ │ (us-east-1a)│ │                    │ │                             │ │ │
│  │ └─────────────┘ │                    │ │  ┌───┐   ┌───┐   ┌───┐     │ │ │
│  │ ┌─────────────┐ │                    │ │  │VM1│   │VM2│   │VM3│     │ │ │
│  │ │ Subnet-1b   │ │                    │ │  │AZ1│   │AZ2│   │AZ3│     │ │ │
│  │ │ (us-east-1b)│ │                    │ │  └───┘   └───┘   └───┘     │ │ │
│  │ └─────────────┘ │                    │ │                             │ │ │
│  │ ┌─────────────┐ │                    │ └─────────────────────────────┘ │ │
│  │ │ Subnet-1c   │ │                    │                                 │ │
│  │ │ (us-east-1c)│ │                    │ AZ is a property of the        │ │
│  │ └─────────────┘ │                    │ RESOURCE, not the subnet       │ │
│  └─────────────────┘                    └─────────────────────────────────┘ │
│                                                                              │
│  AWS: Create 3 subnets for HA          Azure: Create 1 subnet, deploy      │
│       (one per AZ)                           resources across zones         │
│                                                                              │
│  DEPLOYMENT EXAMPLE:                                                        │
│                                                                              │
│  # Azure: Deploy VM to specific zone                                        │
│  az vm create --name vm1 --zones 1                                          │
│  az vm create --name vm2 --zones 2                                          │
│  az vm create --name vm3 --zones 3                                          │
│                                                                              │
│  # All VMs in same subnet, different physical datacenters                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Zone-Redundant Services

| Service | Zone-Redundant Option | How It Works |
|---------|----------------------|--------------|
| VPN Gateway | Zone-redundant SKU | Automatic failover |
| Load Balancer | Standard SKU (default) | Spans all zones |
| Application Gateway | v2 SKU | Zone-redundant |
| Azure Firewall | Zone-redundant | Spans all zones |
| Public IP | Standard SKU | Zone-redundant |
| Storage | ZRS (Zone-Redundant Storage) | Data replicated across zones |
| Azure SQL | Zone-redundant | Automatic failover |
| Cosmos DB | Multi-region + zone | High availability |

---

## Quick CLI Reference

```bash
# Create VNet
az network vnet create \
  --resource-group myRG \
  --name myVNet \
  --address-prefixes 10.0.0.0/16 \
  --subnet-name web-subnet \
  --subnet-prefixes 10.0.1.0/24

# Add another subnet
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name data-subnet \
  --address-prefixes 10.0.2.0/24

# Create NAT Gateway
az network public-ip create \
  --resource-group myRG \
  --name nat-pip \
  --sku Standard \
  --zone 1 2 3

az network nat gateway create \
  --resource-group myRG \
  --name myNATGateway \
  --public-ip-addresses nat-pip \
  --idle-timeout 10

# Associate NAT Gateway with subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name web-subnet \
  --nat-gateway myNATGateway

# Create VNet peering
az network vnet peering create \
  --resource-group myRG \
  --name hub-to-spoke1 \
  --vnet-name hub-vnet \
  --remote-vnet /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/spoke1-vnet \
  --allow-vnet-access \
  --allow-forwarded-traffic
```

---

## Learning Objectives

After this chapter:

1. **Create** VNets and subnets with proper addressing
2. **Design** hub-spoke and other network topologies
3. **Configure** NAT Gateways for outbound internet
4. **Implement** VNet peering for network connectivity
5. **Understand** Availability Zones in Azure networking
6. **Compare** AWS VPC patterns to Azure equivalents

---

*Start with: [Quick Reference](quick-reference.md)* | *Next: [Virtual Networks Deep Dive](01-virtual-networks.md)*
