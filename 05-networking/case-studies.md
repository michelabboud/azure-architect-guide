# Networking Case Studies

## Case Study 1: Enterprise Hub-Spoke Migration

### Scenario

A multinational retail company is migrating from AWS to Azure. They currently have:
- 50+ AWS VPCs across 3 regions
- Direct Connect to 5 data centers
- 10,000+ EC2 instances
- Complex inter-VPC routing via Transit Gateway

### Requirements

- Replicate hub-spoke topology in Azure
- Maintain connectivity to all 5 data centers
- Support 200+ applications with minimal downtime
- Implement zero-trust network segmentation

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE HUB-SPOKE ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              ON-PREMISES                                     │
│                              ───────────                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │ DC 1        │  │ DC 2        │  │ DC 3        │  │ DC 4 & 5    │        │
│  │ (Primary)   │  │ (Secondary) │  │ (Europe)    │  │ (Asia)      │        │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘        │
│         │                │                │                │                 │
│         └────────────────┴────────┬───────┴────────────────┘                │
│                                   │                                          │
│                         ExpressRoute Global Reach                            │
│                                   │                                          │
│  ═══════════════════════════════════════════════════════════════════════════│
│                                   │                                          │
│                    AZURE (Multi-Region Hub-Spoke)                            │
│                    ─────────────────────────────────                         │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                         EAST US REGION                                   ││
│  │                                                                          ││
│  │  ┌─────────────────────────────────────────────────────────────────┐    ││
│  │  │                      HUB VNET (10.0.0.0/16)                      │    ││
│  │  │                                                                  │    ││
│  │  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │    ││
│  │  │  │ ExpressRoute │  │    Azure     │  │   Azure      │           │    ││
│  │  │  │   Gateway    │  │   Firewall   │  │   Bastion    │           │    ││
│  │  │  │   (ErGw3Az)  │  │  (Premium)   │  │              │           │    ││
│  │  │  └──────────────┘  └──────────────┘  └──────────────┘           │    ││
│  │  │                                                                  │    ││
│  │  │  ┌──────────────┐  ┌──────────────┐                             │    ││
│  │  │  │  DNS Private │  │   Shared     │                             │    ││
│  │  │  │   Resolver   │  │  Services    │                             │    ││
│  │  │  └──────────────┘  └──────────────┘                             │    ││
│  │  └─────────────────────────────────────────────────────────────────┘    ││
│  │                              │                                           ││
│  │              ┌───────────────┼───────────────┐                          ││
│  │              │               │               │                          ││
│  │              ▼               ▼               ▼                          ││
│  │  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐           ││
│  │  │ SPOKE 1 (Prod)  │ │ SPOKE 2 (Stage) │ │ SPOKE 3 (Dev)   │           ││
│  │  │ 10.1.0.0/16     │ │ 10.2.0.0/16     │ │ 10.3.0.0/16     │           ││
│  │  │                 │ │                 │ │                 │           ││
│  │  │ • Web tier      │ │ • Pre-prod      │ │ • Developer     │           ││
│  │  │ • App tier      │ │   testing       │ │   workstations  │           ││
│  │  │ • Data tier     │ │                 │ │                 │           ││
│  │  └─────────────────┘ └─────────────────┘ └─────────────────┘           ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                        WEST EUROPE REGION                                ││
│  │                     (Similar architecture)                               ││
│  │                                                                          ││
│  │                     Connected via Global VNet Peering                    ││
│  │                              (Hub to Hub)                                ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Connectivity | ExpressRoute Premium | 5+ DCs, global reach needed |
| Gateway SKU | ErGw3Az | High throughput, zone redundant |
| Firewall | Azure Firewall Premium | TLS inspection for compliance |
| Inter-region | Global VNet Peering | Lower latency than VPN |
| DNS | DNS Private Resolver | Hybrid resolution without VMs |
| Segmentation | NSG + ASG | Micro-segmentation per app tier |

### Implementation Phases

**Phase 1: Foundation (Weeks 1-4)**
- Deploy hub VNets in primary regions
- Establish ExpressRoute circuits
- Configure Azure Firewall with base policies

**Phase 2: Network Services (Weeks 5-8)**
- Deploy DNS Private Resolver
- Configure Private DNS zones
- Set up Azure Bastion for management

**Phase 3: Spoke Deployment (Weeks 9-16)**
- Deploy spoke VNets by environment
- Configure peering with hub
- Implement UDRs for forced tunneling

**Phase 4: Application Migration (Weeks 17+)**
- Migrate applications spoke by spoke
- Validate connectivity and performance
- Decommission AWS VPCs

### Cost Optimization

| Component | Monthly Cost (Est.) | Optimization |
|-----------|---------------------|--------------|
| ExpressRoute (2x 1Gbps) | $880 | Use Local SKU where possible |
| ER Gateways (2x ErGw3Az) | $1,100 | Right-size based on throughput |
| Azure Firewall Premium | $1,850 | Use Standard for non-prod |
| VNet Peering | ~$500 | Based on 5TB egress |
| **Total** | **~$4,330/month** | |

---

## Case Study 2: Secure PaaS Workload

### Scenario

A healthcare company needs to deploy a HIPAA-compliant application using Azure PaaS services with no public internet exposure.

### Requirements

- All traffic must stay private (no public endpoints)
- Data encryption in transit and at rest
- Audit logging for all network flows
- Integration with on-premises identity (Active Directory)

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SECURE PAAS ARCHITECTURE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         USERS (Internet)                                     │
│                               │                                              │
│                               ▼                                              │
│                    ┌─────────────────────┐                                  │
│                    │   Azure Front Door  │                                  │
│                    │   (WAF + Private    │                                  │
│                    │    Link Origin)     │                                  │
│                    └──────────┬──────────┘                                  │
│                               │ (Private Link)                               │
│  ════════════════════════════════════════════════════════════════════════   │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                        APPLICATION VNET                                │  │
│  │                        10.1.0.0/16                                     │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  APP GATEWAY SUBNET (10.1.1.0/24)                               │  │  │
│  │  │  ┌─────────────────────────────────────────────────────────┐    │  │  │
│  │  │  │  Application Gateway (WAF v2)                            │    │  │  │
│  │  │  │  • Private IP only                                       │    │  │  │
│  │  │  │  • Backend: App Service Private Endpoint                 │    │  │  │
│  │  │  └─────────────────────────────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  APP SERVICE SUBNET (10.1.2.0/24) - Delegated                   │  │  │
│  │  │  ┌─────────────────────────────────────────────────────────┐    │  │  │
│  │  │  │  App Service (VNet Integration)                          │    │  │  │
│  │  │  │  • Outbound through VNet                                 │    │  │  │
│  │  │  │  • Access PaaS via Private Endpoints                     │    │  │  │
│  │  │  └─────────────────────────────────────────────────────────┘    │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  PRIVATE ENDPOINT SUBNET (10.1.3.0/24)                          │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │  │  │
│  │  │  │ SQL DB   │  │ Storage  │  │ Key Vault│  │ Service  │        │  │  │
│  │  │  │   PE     │  │   PE     │  │   PE     │  │ Bus PE   │        │  │  │
│  │  │  │10.1.3.4  │  │10.1.3.5  │  │10.1.3.6  │  │10.1.3.7  │        │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  NSG Rules:                                                            │  │
│  │  • Deny all internet inbound                                          │  │
│  │  • Allow AppGW to App Service subnet                                  │  │
│  │  • Allow App Service to PE subnet                                     │  │
│  │  • Deny all other                                                     │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  PRIVATE DNS ZONES (Linked):                                                 │
│  • privatelink.database.windows.net                                         │
│  • privatelink.blob.core.windows.net                                        │
│  • privatelink.vaultcore.azure.net                                          │
│  • privatelink.servicebus.windows.net                                       │
│  • privatelink.azurewebsites.net                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Security Controls

| Requirement | Implementation |
|-------------|----------------|
| No public endpoints | Private Endpoints for all PaaS |
| Encryption in transit | TLS 1.2+ enforced everywhere |
| Encryption at rest | Service-managed keys (or CMK) |
| Network logging | NSG flow logs, Firewall logs |
| Access control | Managed Identity, no credentials |
| WAF | Front Door WAF + App Gateway WAF |

### Bicep Implementation (Excerpt)

```bicep
// Private Endpoint for SQL Database
resource sqlPrivateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'pe-sql-${environment}'
  location: location
  properties: {
    subnet: {
      id: privateEndpointSubnet.id
    }
    privateLinkServiceConnections: [
      {
        name: 'sql-connection'
        properties: {
          privateLinkServiceId: sqlServer.id
          groupIds: ['sqlServer']
        }
      }
    ]
  }
}

// DNS Zone Group for automatic registration
resource sqlDnsZoneGroup 'Microsoft.Network/privateEndpoints/privateDnsZoneGroups@2023-05-01' = {
  parent: sqlPrivateEndpoint
  name: 'default'
  properties: {
    privateDnsZoneConfigs: [
      {
        name: 'privatelink-database-windows-net'
        properties: {
          privateDnsZoneId: sqlPrivateDnsZone.id
        }
      }
    ]
  }
}

// App Service with VNet Integration
resource appService 'Microsoft.Web/sites@2023-01-01' = {
  name: 'app-${environment}-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    virtualNetworkSubnetId: appServiceSubnet.id
    httpsOnly: true
    siteConfig: {
      vnetRouteAllEnabled: true  // Route all outbound through VNet
      ftpsState: 'Disabled'
      minTlsVersion: '1.2'
    }
  }
}
```

---

## Case Study 3: Multi-Region Active-Active

### Scenario

A global e-commerce platform needs active-active deployment across 3 regions with automatic failover and data replication.

### Requirements

- < 100ms latency for users globally
- 99.99% availability SLA
- Automatic failover without data loss
- Consistent user experience across regions

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   MULTI-REGION ACTIVE-ACTIVE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              GLOBAL USERS                                    │
│                                   │                                          │
│                                   ▼                                          │
│                    ┌─────────────────────────────┐                          │
│                    │      Azure Front Door       │                          │
│                    │      (Global Load Balancer) │                          │
│                    │                             │                          │
│                    │  • Anycast routing          │                          │
│                    │  • Health probes            │                          │
│                    │  • WAF policies             │                          │
│                    │  • Session affinity         │                          │
│                    └──────────┬──────────────────┘                          │
│                               │                                              │
│         ┌─────────────────────┼─────────────────────┐                       │
│         │                     │                     │                       │
│         ▼                     ▼                     ▼                       │
│  ┌─────────────┐       ┌─────────────┐       ┌─────────────┐               │
│  │  EAST US    │       │ WEST EUROPE │       │ SOUTHEAST   │               │
│  │             │       │             │       │    ASIA     │               │
│  │  ┌───────┐  │       │  ┌───────┐  │       │  ┌───────┐  │               │
│  │  │App GW │  │       │  │App GW │  │       │  │App GW │  │               │
│  │  └───┬───┘  │       │  └───┬───┘  │       │  └───┬───┘  │               │
│  │      │      │       │      │      │       │      │      │               │
│  │  ┌───▼───┐  │       │  ┌───▼───┐  │       │  ┌───▼───┐  │               │
│  │  │  AKS  │  │       │  │  AKS  │  │       │  │  AKS  │  │               │
│  │  │Cluster│  │       │  │Cluster│  │       │  │Cluster│  │               │
│  │  └───┬───┘  │       │  └───┬───┘  │       │  └───┬───┘  │               │
│  │      │      │       │      │      │       │      │      │               │
│  │  ┌───▼───┐  │       │  ┌───▼───┐  │       │  ┌───▼───┐  │               │
│  │  │Cosmos │◄─┼───────┼──│Cosmos │──┼───────┼─►│Cosmos │  │               │
│  │  │  DB   │  │ Multi │  │  DB   │  │ Write │  │  DB   │  │               │
│  │  │(Read) │  │ Region│  │(Read) │  │ Sync  │  │(Read) │  │               │
│  │  └───────┘  │ Writes│  └───────┘  │       │  └───────┘  │               │
│  │             │       │             │       │             │               │
│  │  ┌───────┐  │       │  ┌───────┐  │       │  ┌───────┐  │               │
│  │  │ Redis │◄─┼───────┼──│ Redis │──┼───────┼─►│ Redis │  │               │
│  │  │ Cache │  │  Geo  │  │ Cache │  │  Geo  │  │ Cache │  │               │
│  │  │       │  │ Repli │  │       │  │ Repli │  │       │  │               │
│  │  └───────┘  │ cation│  └───────┘  │ cation│  └───────┘  │               │
│  └─────────────┘       └─────────────┘       └─────────────┘               │
│                                                                              │
│  TRAFFIC FLOW:                                                              │
│  1. User → Front Door (nearest POP)                                         │
│  2. Front Door → Nearest healthy App Gateway                                │
│  3. App Gateway → AKS (within region)                                       │
│  4. AKS → Cosmos DB (local replica, multi-master writes)                    │
│                                                                              │
│  FAILOVER:                                                                  │
│  • Front Door detects unhealthy region (health probes)                      │
│  • Traffic automatically routed to next nearest region                      │
│  • Cosmos DB continues with remaining replicas                              │
│  • Session state preserved in Redis geo-replica                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Network Design Details

| Component | Configuration | Purpose |
|-----------|---------------|---------|
| Front Door | Premium SKU, Private Link | Global LB, WAF, CDN |
| App Gateway | v2, zone-redundant | Regional LB, additional WAF |
| AKS | Azure CNI, multiple node pools | Container orchestration |
| VNet Peering | Global peering between regions | Cross-region communication |
| Cosmos DB | Multi-region writes | Active-active data layer |
| Redis | Geo-replication | Session state, caching |

### Latency Optimization

```
USER LOCATION → AZURE POP → REGION

North America:
  New York    → Ashburn POP  → East US      (~10ms)
  Los Angeles → San Jose POP → West US      (~15ms)

Europe:
  London      → London POP   → West Europe  (~5ms)
  Frankfurt   → Frankfurt POP→ West Europe  (~10ms)

Asia Pacific:
  Singapore   → Singapore POP→ Southeast Asia (~5ms)
  Tokyo       → Tokyo POP    → Japan East   (~10ms)

Front Door Smart Routing:
• Measures latency to each origin
• Routes to fastest healthy origin
• Accounts for origin load
```

---

## Case Study 4: Network Virtual Appliance Integration

### Scenario

A company requires third-party firewall (Palo Alto) integration for advanced threat protection and compliance with existing security policies.

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      NVA INTEGRATION ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         HUB VNET (10.0.0.0/16)                         │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  NVA SUBNET (10.0.1.0/24)                                       │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌─────────────────┐         ┌─────────────────┐                │  │  │
│  │  │  │  Palo Alto VM   │         │  Palo Alto VM   │                │  │  │
│  │  │  │  (Active)       │◄───HA──►│  (Standby)      │                │  │  │
│  │  │  │  10.0.1.4       │         │  10.0.1.5       │                │  │  │
│  │  │  └────────┬────────┘         └─────────────────┘                │  │  │
│  │  │           │                                                      │  │  │
│  │  │  ┌────────▼────────┐                                            │  │  │
│  │  │  │  Internal LB    │  ← All traffic via ILB for HA              │  │  │
│  │  │  │  10.0.1.100     │                                            │  │  │
│  │  │  └─────────────────┘                                            │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  GATEWAY SUBNET (10.0.0.0/27)                                   │  │  │
│  │  │  • VPN Gateway (for on-prem)                                    │  │  │
│  │  │  • Route table: spokes → 10.0.1.100 (NVA ILB)                  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  ROUTE SERVER SUBNET (10.0.10.0/27)                             │  │  │
│  │  │  • Azure Route Server                                           │  │  │
│  │  │  • BGP peering with NVA                                        │  │  │
│  │  │  • Dynamic route propagation                                    │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                         │
│                              VNet Peering                                    │
│                                    │                                         │
│         ┌──────────────────────────┼──────────────────────────┐             │
│         │                          │                          │             │
│         ▼                          ▼                          ▼             │
│  ┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐ │
│  │ SPOKE 1         │        │ SPOKE 2         │        │ SPOKE 3         │ │
│  │ (Production)    │        │ (Development)   │        │ (DMZ)           │ │
│  │                 │        │                 │        │                 │ │
│  │ Route Table:    │        │ Route Table:    │        │ Route Table:    │ │
│  │ 0.0.0.0/0 →     │        │ 0.0.0.0/0 →     │        │ 0.0.0.0/0 →     │ │
│  │   10.0.1.100    │        │   10.0.1.100    │        │   10.0.1.100    │ │
│  │ 10.2.0.0/16 →   │        │ 10.1.0.0/16 →   │        │                 │ │
│  │   10.0.1.100    │        │   10.0.1.100    │        │                 │ │
│  └─────────────────┘        └─────────────────┘        └─────────────────┘ │
│                                                                              │
│  TRAFFIC FLOWS:                                                             │
│  1. Internet Outbound: Spoke → NVA ILB → NVA → Internet                    │
│  2. Internet Inbound:  Internet → NVA → App Gateway → Spoke                │
│  3. East-West:         Spoke1 → NVA ILB → NVA → Spoke2                     │
│  4. On-Premises:       Spoke → NVA ILB → NVA → VPN GW → On-Prem           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### NVA High Availability

| Option | Description | Failover Time |
|--------|-------------|---------------|
| Active-Passive with ILB | ILB health probes detect failure | ~15-30 seconds |
| Active-Active with ILB | Both NVAs process traffic | Immediate |
| Palo Alto HA (Native) | Built-in HA with state sync | ~seconds |
| Azure Route Server | BGP-based failover | ~seconds |

### Key Configuration

```bash
# Enable IP forwarding on NVA NICs
az network nic update \
  --resource-group myRG \
  --name paloalto-nic \
  --ip-forwarding true

# Create Internal Load Balancer
az network lb create \
  --resource-group myRG \
  --name nva-ilb \
  --sku Standard \
  --vnet-name hub-vnet \
  --subnet nva-subnet \
  --frontend-ip-name nva-frontend \
  --private-ip-address 10.0.1.100 \
  --backend-pool-name nva-pool

# Add health probe (must match NVA health endpoint)
az network lb probe create \
  --resource-group myRG \
  --lb-name nva-ilb \
  --name nva-health \
  --protocol Tcp \
  --port 443

# Add HA ports rule (forward all traffic)
az network lb rule create \
  --resource-group myRG \
  --lb-name nva-ilb \
  --name ha-ports-rule \
  --protocol All \
  --frontend-port 0 \
  --backend-port 0 \
  --frontend-ip-name nva-frontend \
  --backend-pool-name nva-pool \
  --probe-name nva-health \
  --enable-floating-ip true
```

---

## Summary

| Case Study | Key Pattern | Primary Technology |
|------------|-------------|-------------------|
| Enterprise Migration | Hub-Spoke | ExpressRoute, Azure Firewall |
| Secure PaaS | Private Endpoints | Private Link, Private DNS |
| Multi-Region | Active-Active | Front Door, Cosmos DB |
| NVA Integration | Third-Party Firewall | Route Server, ILB |

---

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
