# Load Balancing Case Studies

## Case Study 1: Global E-Commerce Platform

### Scenario

A retail company migrating from AWS needs a globally distributed e-commerce platform with:
- Sub-100ms page load times worldwide
- 99.99% availability during peak sales events
- PCI-DSS compliance for payment processing
- Protection against DDoS and bot attacks

### AWS Architecture (Before)

```
CloudFront → ALB → EC2 (ASG) → RDS
                              ↓
                          ElastiCache
```

### Azure Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GLOBAL E-COMMERCE ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                            GLOBAL USERS                                      │
│                                 │                                            │
│                                 ▼                                            │
│                    ┌────────────────────────┐                               │
│                    │    AZURE FRONT DOOR    │                               │
│                    │    (Premium SKU)       │                               │
│                    │                        │                               │
│                    │  • WAF + Bot protection│                               │
│                    │  • CDN caching         │                               │
│                    │  • SSL termination     │                               │
│                    │  • Geo-filtering       │                               │
│                    └───────────┬────────────┘                               │
│                                │                                             │
│         ┌──────────────────────┼──────────────────────┐                     │
│         │                      │                      │                     │
│         ▼                      ▼                      ▼                     │
│  ┌─────────────┐        ┌─────────────┐        ┌─────────────┐             │
│  │  EAST US    │        │ WEST EUROPE │        │ EAST ASIA   │             │
│  │             │        │             │        │             │             │
│  │ ┌─────────┐ │        │ ┌─────────┐ │        │ ┌─────────┐ │             │
│  │ │ App GW  │ │        │ │ App GW  │ │        │ │ App GW  │ │             │
│  │ │ (WAF v2)│ │        │ │ (WAF v2)│ │        │ │ (WAF v2)│ │             │
│  │ └────┬────┘ │        │ └────┬────┘ │        │ └────┬────┘ │             │
│  │      │      │        │      │      │        │      │      │             │
│  │ ┌────▼────┐ │        │ ┌────▼────┐ │        │ ┌────▼────┐ │             │
│  │ │   AKS   │ │        │ │   AKS   │ │        │ │   AKS   │ │             │
│  │ │ Cluster │ │        │ │ Cluster │ │        │ │ Cluster │ │             │
│  │ └────┬────┘ │        │ └────┬────┘ │        │ └────┬────┘ │             │
│  │      │      │        │      │      │        │      │      │             │
│  │      ▼      │        │      ▼      │        │      ▼      │             │
│  │ ┌─────────┐ │        │ ┌─────────┐ │        │ ┌─────────┐ │             │
│  │ │Internal │ │        │ │Internal │ │        │ │Internal │ │             │
│  │ │   LB    │ │        │ │   LB    │ │        │ │   LB    │ │             │
│  │ └────┬────┘ │        │ └────┬────┘ │        │ └────┬────┘ │             │
│  │      ▼      │        │      ▼      │        │      ▼      │             │
│  │ ┌─────────┐ │        │ ┌─────────┐ │        │ ┌─────────┐ │             │
│  │ │ Azure   │ │◄──────►│ │ Azure   │ │◄──────►│ │ Azure   │ │             │
│  │ │ Redis   │ │  Geo-  │ │ Redis   │ │  Geo-  │ │ Redis   │ │             │
│  │ │ Cache   │ │  Repl  │ │ Cache   │ │  Repl  │ │ Cache   │ │             │
│  │ └─────────┘ │        │ └─────────┘ │        │ └─────────┘ │             │
│  │             │        │             │        │             │             │
│  └─────────────┘        └─────────────┘        └─────────────┘             │
│         │                      │                      │                     │
│         └──────────────────────┼──────────────────────┘                     │
│                                ▼                                             │
│                    ┌────────────────────────┐                               │
│                    │      COSMOS DB         │                               │
│                    │   (Multi-region write) │                               │
│                    └────────────────────────┘                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Global LB | Front Door Premium | CDN + WAF + Private Link |
| Regional LB | App Gateway WAF v2 | Path routing + additional WAF layer |
| Internal LB | Azure LB Standard | AKS services, zone-redundant |
| Caching | Azure Redis (Premium) | Geo-replication, low latency |
| Database | Cosmos DB | Multi-region writes, <10ms reads |

### Performance Results

| Metric | Before (AWS) | After (Azure) |
|--------|--------------|---------------|
| P50 Latency (US) | 85ms | 45ms |
| P50 Latency (EU) | 180ms | 52ms |
| P50 Latency (Asia) | 320ms | 58ms |
| Availability | 99.95% | 99.99% |
| DDoS blocked | N/A | 2.3M requests/day |

---

## Case Study 2: NVA High Availability

### Scenario

A financial services company requires third-party firewall (Palo Alto) inspection for all traffic with:
- Zero downtime during failover
- All traffic (TCP/UDP) inspection
- Session persistence during failover
- Compliance with audit requirements

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NVA HA WITH LOAD BALANCER                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SPOKE VNETS                                  INTERNET                       │
│  (Workloads)                                     │                           │
│       │                                          ▼                           │
│       │ UDR: 0.0.0.0/0 → ILB          ┌─────────────────┐                  │
│       │                                │   Public LB     │                  │
│       ▼                                │   (Inbound)     │                  │
│  ┌─────────────────────────────────────┴─────────────────┐                  │
│  │                      HUB VNET                          │                  │
│  │                                                        │                  │
│  │  ┌─────────────────────────────────────────────────┐  │                  │
│  │  │              NVA SUBNET                          │  │                  │
│  │  │                                                  │  │                  │
│  │  │  ┌──────────────────────────────────────────┐   │  │                  │
│  │  │  │        INTERNAL LOAD BALANCER            │   │  │                  │
│  │  │  │        (HA Ports Rule)                   │   │  │                  │
│  │  │  │                                          │   │  │                  │
│  │  │  │  Frontend IP: 10.0.1.100                 │   │  │                  │
│  │  │  │  Protocol: All                           │   │  │                  │
│  │  │  │  Ports: 0 (all)                          │   │  │                  │
│  │  │  │  Floating IP: Enabled                    │   │  │                  │
│  │  │  └────────────────┬─────────────────────────┘   │  │                  │
│  │  │                   │                              │  │                  │
│  │  │         ┌─────────┼─────────┐                   │  │                  │
│  │  │         │         │         │                   │  │                  │
│  │  │         ▼         ▼         ▼                   │  │                  │
│  │  │  ┌───────────┐┌───────────┐┌───────────┐       │  │                  │
│  │  │  │ Palo Alto ││ Palo Alto ││ Palo Alto │       │  │                  │
│  │  │  │   VM 1    ││   VM 2    ││   VM 3    │       │  │                  │
│  │  │  │  (AZ 1)   ││  (AZ 2)   ││  (AZ 3)   │       │  │                  │
│  │  │  │           ││           ││           │       │  │                  │
│  │  │  │ eth0: Mgmt││ eth0: Mgmt││ eth0: Mgmt│       │  │                  │
│  │  │  │ eth1: Data││ eth1: Data││ eth1: Data│       │  │                  │
│  │  │  └───────────┘└───────────┘└───────────┘       │  │                  │
│  │  │                                                  │  │                  │
│  │  │  Health Probe: TCP 443 (management interface)   │  │                  │
│  │  └──────────────────────────────────────────────────┘  │                  │
│  │                                                        │                  │
│  │  ┌──────────────────────────────────────────────────┐  │                  │
│  │  │  GATEWAY SUBNET                                   │  │                  │
│  │  │  VPN Gateway → On-Premises                       │  │                  │
│  │  │  Route Table: Spokes → 10.0.1.100 (ILB)         │  │                  │
│  │  └──────────────────────────────────────────────────┘  │                  │
│  │                                                        │                  │
│  └────────────────────────────────────────────────────────┘                  │
│                                                                              │
│  TRAFFIC FLOWS:                                                             │
│  1. Outbound: Spoke → ILB → Active NVA → Internet                          │
│  2. Inbound:  Internet → Public LB → NVA → Spoke                           │
│  3. On-Prem:  Spoke → ILB → NVA → VPN GW → On-Prem                        │
│  4. East-West: Spoke1 → ILB → NVA → ILB → Spoke2                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Configuration Details

```bash
# Internal LB with HA Ports
az network lb create \
  --resource-group myRG \
  --name nva-ilb \
  --sku Standard \
  --vnet-name hub-vnet \
  --subnet nva-subnet \
  --frontend-ip-name frontend \
  --private-ip-address 10.0.1.100 \
  --backend-pool-name nva-pool

az network lb probe create \
  --resource-group myRG \
  --lb-name nva-ilb \
  --name health-probe \
  --protocol Tcp \
  --port 443

az network lb rule create \
  --resource-group myRG \
  --lb-name nva-ilb \
  --name ha-ports \
  --protocol All \
  --frontend-port 0 \
  --backend-port 0 \
  --frontend-ip-name frontend \
  --backend-pool-name nva-pool \
  --probe-name health-probe \
  --enable-floating-ip true
```

### Failover Testing Results

| Scenario | Failover Time | Session Loss |
|----------|---------------|--------------|
| Single NVA failure | <15 seconds | None (stateful) |
| Availability Zone failure | <15 seconds | None |
| NVA software upgrade | Zero downtime | None |

---

## Case Study 3: Multi-Tier Application

### Scenario

A SaaS company needs to deploy a three-tier application with:
- Public API endpoint with rate limiting
- Internal microservices communication
- Database tier with read replicas

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MULTI-TIER WITH MULTIPLE LBs                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                            INTERNET                                          │
│                                │                                             │
│                                ▼                                             │
│                    ┌────────────────────────┐                               │
│                    │    FRONT DOOR          │                               │
│                    │    (Rate Limiting)     │                               │
│                    └───────────┬────────────┘                               │
│                                │                                             │
│  ┌─────────────────────────────┴─────────────────────────────────────────┐  │
│  │                         APPLICATION VNET                               │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  API TIER                                                        │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌────────────────────────────────────────────────────────────┐ │  │  │
│  │  │  │  APPLICATION GATEWAY (WAF v2)                              │ │  │  │
│  │  │  │  • Path-based routing (/api/v1, /api/v2)                   │ │  │  │
│  │  │  │  • SSL termination                                         │ │  │  │
│  │  │  │  • Autoscaling (2-10 instances)                           │ │  │  │
│  │  │  └───────────────────────────┬────────────────────────────────┘ │  │  │
│  │  │                              │                                   │  │  │
│  │  │           ┌──────────────────┼──────────────────┐               │  │  │
│  │  │           ▼                  ▼                  ▼               │  │  │
│  │  │    ┌───────────┐      ┌───────────┐      ┌───────────┐         │  │  │
│  │  │    │ API v1    │      │ API v2    │      │ GraphQL   │         │  │  │
│  │  │    │ VMSS      │      │ VMSS      │      │ VMSS      │         │  │  │
│  │  │    └───────────┘      └───────────┘      └───────────┘         │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                   │                                    │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  SERVICE TIER                  │                                 │  │  │
│  │  │                                ▼                                 │  │  │
│  │  │  ┌────────────────────────────────────────────────────────────┐ │  │  │
│  │  │  │  INTERNAL LOAD BALANCER                                    │ │  │  │
│  │  │  │  10.0.2.100                                                │ │  │  │
│  │  │  └───────────────────────────┬────────────────────────────────┘ │  │  │
│  │  │                              │                                   │  │  │
│  │  │           ┌──────────────────┼──────────────────┐               │  │  │
│  │  │           ▼                  ▼                  ▼               │  │  │
│  │  │    ┌───────────┐      ┌───────────┐      ┌───────────┐         │  │  │
│  │  │    │ Order     │      │ Inventory │      │ Payment   │         │  │  │
│  │  │    │ Service   │      │ Service   │      │ Service   │         │  │  │
│  │  │    └───────────┘      └───────────┘      └───────────┘         │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                   │                                    │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  DATA TIER                     │                                 │  │  │
│  │  │                                ▼                                 │  │  │
│  │  │  ┌────────────────────────────────────────────────────────────┐ │  │  │
│  │  │  │  INTERNAL LOAD BALANCER (Read Replicas)                    │ │  │  │
│  │  │  │  10.0.3.100                                                │ │  │  │
│  │  │  └───────────────────────────┬────────────────────────────────┘ │  │  │
│  │  │                              │                                   │  │  │
│  │  │           ┌──────────────────┼──────────────────┐               │  │  │
│  │  │           ▼                  ▼                  ▼               │  │  │
│  │  │    ┌───────────┐      ┌───────────┐      ┌───────────┐         │  │  │
│  │  │    │ SQL Read  │      │ SQL Read  │      │ SQL Read  │         │  │  │
│  │  │    │ Replica 1 │      │ Replica 2 │      │ Replica 3 │         │  │  │
│  │  │    │ (Zone 1)  │      │ (Zone 2)  │      │ (Zone 3)  │         │  │  │
│  │  │    └───────────┘      └───────────┘      └───────────┘         │  │  │
│  │  │                              │                                   │  │  │
│  │  │                     ┌────────▼────────┐                         │  │  │
│  │  │                     │  SQL Primary    │                         │  │  │
│  │  │                     │  (Writes only)  │                         │  │  │
│  │  │                     └─────────────────┘                         │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Load Balancer Configuration Summary

| Tier | Load Balancer | Purpose |
|------|---------------|---------|
| Edge | Front Door | Global LB, WAF, CDN, rate limiting |
| API | App Gateway v2 | Path routing, SSL, additional WAF |
| Service | Internal LB | Microservices distribution |
| Data | Internal LB | Read replica distribution |

---

## Case Study 4: Blue-Green Deployment

### Scenario

A software company needs zero-downtime deployments with:
- Instant rollback capability
- A/B testing support
- Gradual traffic shifting

### Solution Using Front Door

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BLUE-GREEN WITH FRONT DOOR                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌────────────────────────┐                               │
│                    │      FRONT DOOR        │                               │
│                    │                        │                               │
│                    │  Traffic Split:        │                               │
│                    │  Blue: 90%             │                               │
│                    │  Green: 10%            │                               │
│                    └───────────┬────────────┘                               │
│                                │                                             │
│              ┌─────────────────┴─────────────────┐                          │
│              │                                   │                          │
│              ▼                                   ▼                          │
│  ┌───────────────────────┐         ┌───────────────────────┐              │
│  │  BLUE ENVIRONMENT     │         │  GREEN ENVIRONMENT    │              │
│  │  (Current Production) │         │  (New Version)        │              │
│  │                       │         │                       │              │
│  │  Weight: 900          │         │  Weight: 100          │              │
│  │                       │         │                       │              │
│  │  ┌─────────────────┐  │         │  ┌─────────────────┐  │              │
│  │  │  App Gateway    │  │         │  │  App Gateway    │  │              │
│  │  └────────┬────────┘  │         │  └────────┬────────┘  │              │
│  │           │           │         │           │           │              │
│  │  ┌────────▼────────┐  │         │  ┌────────▼────────┐  │              │
│  │  │   AKS v1.0      │  │         │  │   AKS v1.1      │  │              │
│  │  └─────────────────┘  │         │  └─────────────────┘  │              │
│  │                       │         │                       │              │
│  └───────────────────────┘         └───────────────────────┘              │
│                                                                              │
│  DEPLOYMENT PHASES:                                                         │
│  1. Deploy v1.1 to Green (Blue: 100%, Green: 0%)                           │
│  2. Smoke test Green directly                                               │
│  3. Shift 10% traffic (Blue: 90%, Green: 10%)                              │
│  4. Monitor metrics                                                         │
│  5. Gradually increase (50/50, then 10/90)                                 │
│  6. Complete cutover (Blue: 0%, Green: 100%)                               │
│  7. Rollback: Reverse weights instantly                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Traffic Shifting Commands

```bash
# Initial: 100% Blue
az afd origin update \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name productionOrigins \
  --origin-name blue \
  --weight 1000

az afd origin update \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name productionOrigins \
  --origin-name green \
  --weight 0

# Canary: 10% Green
az afd origin update --origin-name blue --weight 900
az afd origin update --origin-name green --weight 100

# 50/50 Split
az afd origin update --origin-name blue --weight 500
az afd origin update --origin-name green --weight 500

# Complete cutover
az afd origin update --origin-name blue --weight 0
az afd origin update --origin-name green --weight 1000

# Instant rollback
az afd origin update --origin-name blue --weight 1000
az afd origin update --origin-name green --weight 0
```

---

## Summary

| Case Study | Primary LB | Key Pattern |
|------------|------------|-------------|
| Global E-Commerce | Front Door + App GW | Multi-region, CDN, WAF |
| NVA HA | Internal LB (HA Ports) | Third-party firewall |
| Multi-Tier | Multiple LBs | Layered architecture |
| Blue-Green | Front Door | Traffic shifting |

---

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
