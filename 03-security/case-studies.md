# Security Case Studies

Real-world security architecture scenarios with decision analysis.

---

## Case Study 1: Securing a Public-Facing Web Application (Beginner)

### Scenario

**Company**: E-commerce startup
**Application**: Web store with payment processing
**Current State**: Single VNet, VMs with public IPs, no WAF
**Goal**: Production-ready security without enterprise budget

### The Debate

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION A: Minimal Security                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  • NSGs only                                                                │
│  • No WAF (rely on app-level security)                                     │
│  • Public IPs on VMs                                                        │
│  • DDoS Basic (free)                                                        │
│                                                                              │
│  Cost: ~$0/month (just compute)                                            │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Cheapest option                  • No L7 protection                     │
│  • Simple to implement              • VMs directly exposed                  │
│  • Fast to deploy                   • No DDoS visibility                   │
│                                     • PCI compliance at risk               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION B: Application Gateway + WAF                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  • Application Gateway v2 with WAF                                         │
│  • NSGs on all subnets                                                     │
│  • No public IPs on VMs                                                    │
│  • Bastion for admin access                                                │
│                                                                              │
│  Cost: ~$400-600/month                                                     │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • L7 protection (OWASP rules)      • Higher cost                          │
│  • VMs not directly exposed         • More complexity                      │
│  • SSL offload                      • Still regional only                  │
│  • PCI compliant architecture       • No global load balancing             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION C: Front Door + WAF + Private Backend             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  • Azure Front Door with WAF                                               │
│  • Private endpoints for backend                                           │
│  • NSGs on all subnets                                                     │
│  • Bastion for admin access                                                │
│                                                                              │
│  Cost: ~$600-900/month                                                     │
│                                                                              │
│  PROS:                              CONS:                                   │
│  • Global anycast (faster for users)• Highest cost                        │
│  • DDoS protection at edge          • Most complex                         │
│  • CDN included                     • May be overkill for startup          │
│  • Full zero trust backend          •                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Decision

**Chosen**: Option B (Application Gateway + WAF)

**Reasoning**:
> 1. **PCI requirement**: Payment processing requires WAF protection - non-negotiable
> 2. **Cost-effective**: $400-600/month is reasonable for production security
> 3. **Right-sized**: Global CDN not needed for initial regional customer base
> 4. **Growth path**: Can add Front Door later when expanding globally

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    BEGINNER SECURE WEB APP ARCHITECTURE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│                    ┌─────────────┴─────────────┐                            │
│                    │    Public IP (AppGW)      │                            │
│                    └─────────────┬─────────────┘                            │
│                                  │                                           │
│  ┌───────────────────────────────┴───────────────────────────────────────┐  │
│  │                           VNET: 10.0.0.0/16                            │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  AppGW Subnet: 10.0.0.0/24                                      │  │  │
│  │  │  NSG: Allow 443 from Internet to AppGW                          │  │  │
│  │  │  ┌──────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │  APPLICATION GATEWAY v2 + WAF                            │  │  │  │
│  │  │  │  • WAF Policy: OWASP 3.2, Prevention mode                │  │  │  │
│  │  │  │  • SSL Certificate (Key Vault reference)                 │  │  │  │
│  │  │  │  • Health probes to backend                              │  │  │  │
│  │  │  └──────────────────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                  │                                     │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Web Subnet: 10.0.1.0/24                                        │  │  │
│  │  │  NSG: Allow 80/443 from AppGW subnet ONLY                       │  │  │
│  │  │  ┌───────────────┐  ┌───────────────┐                          │  │  │
│  │  │  │  Web VM 1     │  │  Web VM 2     │  (No public IPs!)        │  │  │
│  │  │  │  10.0.1.10    │  │  10.0.1.11    │                          │  │  │
│  │  │  └───────────────┘  └───────────────┘                          │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                  │                                     │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Data Subnet: 10.0.2.0/24                                       │  │  │
│  │  │  NSG: Allow 1433 from Web subnet ONLY                           │  │  │
│  │  │  ┌───────────────────────────────────────────────────────────┐ │  │  │
│  │  │  │  Private Endpoint (Azure SQL)                             │ │  │  │
│  │  │  │  10.0.2.10                                                 │ │  │  │
│  │  │  └───────────────────────────────────────────────────────────┘ │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  Bastion Subnet: 10.0.255.0/27                                  │  │  │
│  │  │  Azure Bastion (for admin access)                               │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  SECURITY CONTROLS:                                                         │
│  ✓ WAF with OWASP rules (SQL injection, XSS protection)                    │
│  ✓ No public IPs on VMs (attack surface minimized)                         │
│  ✓ Private Endpoint for database (no public SQL exposure)                  │
│  ✓ NSG microsegmentation (traffic flows controlled)                        │
│  ✓ Bastion for admin (no SSH/RDP from internet)                           │
│  ✓ Defender for Cloud enabled (vulnerability scanning)                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Case Study 2: Hub-Spoke with Centralized Security (Intermediate)

### Scenario

**Company**: Financial services firm (500 employees)
**Requirement**: Multiple applications, centralized security inspection, compliance
**Current State**: Siloed VNets per application, inconsistent security
**Goal**: Unified security architecture with full traffic inspection

### The Debate

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION A: Azure Firewall Standard                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Cost: ~$1.25/hour × 730 = ~$912/month + data                              │
│                                                                              │
│  Features:                                                                  │
│  ✓ L3-L7 filtering                                                         │
│  ✓ FQDN filtering                                                          │
│  ✓ Threat intelligence                                                     │
│  ✓ DNS proxy                                                               │
│  ✗ TLS inspection                                                          │
│  ✗ IDPS signatures                                                         │
│  ✗ URL categories                                                          │
│                                                                              │
│  Good for: General enterprise traffic filtering                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION B: Azure Firewall Premium                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Cost: ~$1.75/hour × 730 = ~$1,277/month + data                            │
│                                                                              │
│  Features:                                                                  │
│  ✓ All Standard features                                                   │
│  ✓ TLS inspection                                                          │
│  ✓ IDPS (50,000+ signatures)                                               │
│  ✓ URL filtering                                                           │
│  ✓ Web categories                                                          │
│                                                                              │
│  Good for: Financial services, regulated industries                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                    OPTION C: Third-Party NVA (Palo Alto, Fortinet)          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Cost: ~$2,000-5,000/month (varies by vendor/throughput)                   │
│                                                                              │
│  Features:                                                                  │
│  ✓ Full enterprise firewall features                                       │
│  ✓ Existing team expertise (if using on-prem)                             │
│  ✓ Advanced threat prevention                                              │
│  ✓ Unified policy with on-prem                                            │
│  ✗ Not Azure-native (separate management)                                  │
│  ✗ HA complexity (you manage failover)                                     │
│                                                                              │
│  Good for: Existing vendor investment, specific compliance requirements    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Decision

**Chosen**: Option B (Azure Firewall Premium)

**Reasoning**:
> 1. **Compliance**: Financial services require TLS inspection for DLP - Premium only
> 2. **IDPS**: Regulatory requirement for intrusion detection - Premium has 50K+ signatures
> 3. **TCO**: Azure-native = no HA management overhead, automatic updates
> 4. **Integration**: Native integration with Azure Monitor, Sentinel, Policy
> 5. **Skills**: Team is Azure-focused, not firewall specialists

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              INTERMEDIATE: HUB-SPOKE WITH CENTRALIZED SECURITY              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│                    ┌─────────────┴─────────────┐                            │
│                    │    Azure Front Door       │                            │
│                    │    + WAF (Global edge)    │                            │
│                    └─────────────┬─────────────┘                            │
│                                  │                                           │
│  ┌───────────────────────────────┴───────────────────────────────────────┐  │
│  │                           HUB VNET: 10.0.0.0/16                        │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  AzureFirewallSubnet: 10.0.1.0/26                               │  │  │
│  │  │  ┌──────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │  AZURE FIREWALL PREMIUM                                  │  │  │  │
│  │  │  │  • TLS inspection (decrypt/inspect/re-encrypt)          │  │  │  │
│  │  │  │  • IDPS in Alert+Deny mode                              │  │  │  │
│  │  │  │  • FQDN filtering (allowlist approach)                  │  │  │  │
│  │  │  │  • Threat intelligence: Block known bad                 │  │  │  │
│  │  │  └──────────────────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │  │
│  │  │ VPN Gateway │  │ExpressRoute │  │   Bastion   │  │  Log        │  │  │
│  │  │ (Remote)    │  │ (On-prem)   │  │  (Admin)    │  │  Analytics  │  │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘  │  │
│  └───────────────────────────────┬────────────────────────────────────────┘  │
│                                  │ (VNet Peering)                            │
│            ┌─────────────────────┼─────────────────────┐                    │
│            │                     │                     │                    │
│            ▼                     ▼                     ▼                    │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│  │  SPOKE: Trading │   │  SPOKE: Banking │   │  SPOKE: Data    │          │
│  │  10.1.0.0/16    │   │  10.2.0.0/16    │   │  10.3.0.0/16    │          │
│  │                 │   │                 │   │                 │          │
│  │  Route Table:   │   │  Route Table:   │   │  Route Table:   │          │
│  │  0.0.0.0/0 →    │   │  0.0.0.0/0 →    │   │  0.0.0.0/0 →    │          │
│  │  10.0.1.4 (FW)  │   │  10.0.1.4 (FW)  │   │  10.0.1.4 (FW)  │          │
│  │                 │   │                 │   │                 │          │
│  │  NSG: Segment   │   │  NSG: Segment   │   │  NSG: Segment   │          │
│  │  by app tier    │   │  by app tier    │   │  by app tier    │          │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘          │
│                                                                              │
│  TRAFFIC FLOWS:                                                             │
│  ─────────────                                                              │
│  1. Internet → Front Door → WAF inspection → Firewall → Spoke             │
│  2. Spoke → Firewall (TLS inspect) → Internet                              │
│  3. Spoke → Firewall (inspect) → Other Spoke (east-west)                  │
│  4. Spoke → Firewall (log) → On-premises (via ExpressRoute)               │
│  5. Admin → Bastion → Spoke VMs (no direct access)                        │
│                                                                              │
│  ALL TRAFFIC INSPECTED AT FIREWALL (except Bastion management)             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Implementation Details

```bash
# Force all traffic through firewall
az network route-table create \
  --name spoke-trading-rt \
  --resource-group hub-rg

az network route-table route create \
  --route-table-name spoke-trading-rt \
  --resource-group hub-rg \
  --name to-internet \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4  # Firewall private IP

az network route-table route create \
  --route-table-name spoke-trading-rt \
  --resource-group hub-rg \
  --name to-other-spokes \
  --address-prefix 10.0.0.0/8 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address 10.0.1.4  # Force east-west through FW
```

---

## Case Study 3: Multi-Region Zero Trust (Expert)

### Scenario

**Company**: Global SaaS provider
**Requirement**: Multi-region HA, zero trust everywhere, SOC 2 compliance
**Scale**: 10,000+ users, millions of API calls daily
**Goal**: No implicit trust, even for internal traffic

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              EXPERT: MULTI-REGION ZERO TRUST ARCHITECTURE                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              GLOBAL USERS                                    │
│                                  │                                           │
│                    ┌─────────────┴─────────────┐                            │
│                    │    AZURE FRONT DOOR       │                            │
│                    │    (Global Anycast)       │                            │
│                    │    • WAF (OWASP + Bot)    │                            │
│                    │    • DDoS at edge         │                            │
│                    │    • Latency-based routing│                            │
│                    └─────────────┬─────────────┘                            │
│                                  │                                           │
│         ┌────────────────────────┼────────────────────────┐                 │
│         │                        │                        │                 │
│         ▼                        ▼                        ▼                 │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│  │  REGION: US     │   │ REGION: Europe  │   │ REGION: Asia    │          │
│  │                 │   │                 │   │                 │          │
│  │  ┌───────────┐  │   │  ┌───────────┐  │   │  ┌───────────┐  │          │
│  │  │ Hub VNet  │  │   │  │ Hub VNet  │  │   │  │ Hub VNet  │  │          │
│  │  │ • FW Prem │  │   │  │ • FW Prem │  │   │  │ • FW Prem │  │          │
│  │  │ • Bastion │  │   │  │ • Bastion │  │   │  │ • Bastion │  │          │
│  │  └─────┬─────┘  │   │  └─────┬─────┘  │   │  └─────┬─────┘  │          │
│  │        │        │   │        │        │   │        │        │          │
│  │  ┌─────┴─────┐  │   │  ┌─────┴─────┐  │   │  ┌─────┴─────┐  │          │
│  │  │ Spoke:    │  │   │  │ Spoke:    │  │   │  │ Spoke:    │  │          │
│  │  │ AKS       │  │   │  │ AKS       │  │   │  │ AKS       │  │          │
│  │  │ • Workload│  │   │  │ • Workload│  │   │  │ • Workload│  │          │
│  │  │   Identity│  │   │  │   Identity│  │   │  │   Identity│  │          │
│  │  │ • mTLS    │  │   │  │ • mTLS    │  │   │  │ • mTLS    │  │          │
│  │  │ • Network │  │   │  │ • Network │  │   │  │ • Network │  │          │
│  │  │   Policy  │  │   │  │   Policy  │  │   │  │   Policy  │  │          │
│  │  └───────────┘  │   │  └───────────┘  │   │  └───────────┘  │          │
│  │        │        │   │        │        │   │        │        │          │
│  │  Private Endpoints (All PaaS via Private Link)          │              │
│  │        │        │   │        │        │   │        │        │          │
│  │  ┌─────┴─────┐  │   │  ┌─────┴─────┐  │   │  ┌─────┴─────┐  │          │
│  │  │Cosmos DB  │◄─┼───┼──┤Cosmos DB  │◄─┼───┼──┤Cosmos DB  │  │          │
│  │  │(Multi-    │  │   │  │(Global    │  │   │  │(Geo-      │  │          │
│  │  │ region)   │──┼───┼──┤ replication│──┼───┼──┤replication│  │          │
│  │  └───────────┘  │   │  └───────────┘  │   │  └───────────┘  │          │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘          │
│                                                                              │
│  ZERO TRUST CONTROLS AT EVERY LAYER:                                       │
│  ────────────────────────────────────                                       │
│                                                                              │
│  IDENTITY:                                                                  │
│  • All users: Entra ID + Conditional Access + MFA                          │
│  • All workloads: Workload Identity (no stored credentials)                │
│  • All admins: PIM with 4-hour max activation                              │
│                                                                              │
│  NETWORK:                                                                   │
│  • All traffic through regional firewall (no direct internet)              │
│  • All PaaS via Private Endpoints (no public endpoints)                    │
│  • All pod-to-pod: Kubernetes Network Policy + mTLS (service mesh)         │
│  • All east-west: Inspected at firewall                                    │
│                                                                              │
│  DATA:                                                                      │
│  • All secrets: Azure Key Vault (accessed via Managed Identity)            │
│  • All storage: Encryption at rest (CMK)                                   │
│  • All transit: TLS 1.3                                                    │
│  • Classification: Purview labels on sensitive data                        │
│                                                                              │
│  MONITORING:                                                                │
│  • All logs: Centralized Log Analytics                                     │
│  • All alerts: Microsoft Sentinel                                          │
│  • All security: Defender for Cloud (all workloads)                       │
│  • All access: Entra ID sign-in logs + audit logs                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Zero Trust Implementation Details

**1. Kubernetes Zero Trust (AKS)**

```yaml
# Network Policy: Default deny all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress

---
# Network Policy: Allow specific traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: api
    ports:
    - protocol: TCP
      port: 5432
```

**2. Workload Identity (No Secrets)**

```yaml
# Service Account with Workload Identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: api-service
  namespace: production
  annotations:
    azure.workload.identity/client-id: "{managed-identity-client-id}"

---
# Pod using Workload Identity
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  namespace: production
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: api-service
  containers:
  - name: api
    image: myapp:latest
    # No secrets mounted - identity from Azure
```

**3. Private Endpoints for Everything**

```bash
# No public endpoints - all PaaS via Private Link
az cosmosdb update \
  --name mycosmosdb \
  --resource-group myRG \
  --enable-public-network false

az keyvault update \
  --name mykv \
  --resource-group myRG \
  --public-network-access Disabled

az storage account update \
  --name mystorageacct \
  --resource-group myRG \
  --public-network-access Disabled
```

---

## Summary: Security Architecture Patterns

### Pattern Selection Guide

| Scenario | Recommended Pattern | Key Components |
|----------|--------------------|--------------------|
| **Startup/Small** | AppGW + WAF + NSGs | Application Gateway, WAF v2, NSGs, Bastion |
| **Mid-size** | Hub-Spoke + Azure FW | Azure Firewall Standard/Premium, Hub VNet, UDRs |
| **Enterprise** | Multi-region Zero Trust | Front Door, Regional FW Premium, Private Link everywhere, Sentinel |
| **Regulated** | Full inspection + mTLS | TLS inspection, IDPS, Service Mesh, Workload Identity |

### Cost Comparison

| Pattern | Monthly Cost Estimate | Security Level |
|---------|----------------------|----------------|
| Minimal (NSGs only) | ~$0 | Basic |
| AppGW + WAF | ~$400-600 | Good |
| Hub-Spoke + FW Standard | ~$1,500-2,500 | Strong |
| Hub-Spoke + FW Premium | ~$2,500-4,000 | Very Strong |
| Multi-region Zero Trust | ~$8,000-15,000 | Maximum |

### Decision Checklist

```
□ Do you handle payment data? → WAF required (PCI)
□ Do you need TLS inspection? → Azure Firewall Premium
□ Do you need intrusion detection? → Azure Firewall Premium or NVA
□ Are you multi-region? → Front Door + regional firewalls
□ Do you have compliance requirements? → Full zero trust
□ Is cost the primary constraint? → Start with AppGW + WAF
□ Do you have existing firewall vendor? → Consider NVA
```

---

*Back to: [Chapter Overview](README.md)* | *Next Chapter: [Compliance](../03-compliance/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
