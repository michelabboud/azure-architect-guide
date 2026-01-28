# Chapter 12: Load Balancing, CDN, and Auto-Scaling

## AWS to Azure Load Balancing Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 AWS vs AZURE LOAD BALANCING SERVICES                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS                              Azure                                      │
│  ───                              ─────                                      │
│                                                                              │
│  LAYER 7 (HTTP/HTTPS):                                                       │
│  ┌─────────────────────┐         ┌──────────────────────────────────────┐   │
│  │ Application Load    │         │ Application Gateway                  │   │
│  │ Balancer (ALB)      │  ←───→  │ • Path-based routing                 │   │
│  │                     │         │ • WAF v2 integration                 │   │
│  └─────────────────────┘         │ • SSL termination                    │   │
│                                  │ • Autoscaling                        │   │
│                                  └──────────────────────────────────────┘   │
│                                                                              │
│  LAYER 4 (TCP/UDP):                                                          │
│  ┌─────────────────────┐         ┌──────────────────────────────────────┐   │
│  │ Network Load        │         │ Azure Load Balancer                  │   │
│  │ Balancer (NLB)      │  ←───→  │ • Standard SKU (zone-redundant)      │   │
│  │                     │         │ • Basic SKU (single zone)            │   │
│  └─────────────────────┘         │ • HA Ports for NVAs                  │   │
│                                  └──────────────────────────────────────┘   │
│                                                                              │
│  GLOBAL (Anycast):                                                           │
│  ┌─────────────────────┐         ┌──────────────────────────────────────┐   │
│  │ Global Accelerator  │         │ Azure Front Door                     │   │
│  │ CloudFront          │  ←───→  │ • Global HTTP load balancing         │   │
│  │                     │         │ • CDN built-in                       │   │
│  │                     │         │ • WAF integration                    │   │
│  └─────────────────────┘         │ • URL rewrite/redirect               │   │
│                                  └──────────────────────────────────────┘   │
│                                                                              │
│  CDN:                                                                        │
│  ┌─────────────────────┐         ┌──────────────────────────────────────┐   │
│  │ CloudFront          │         │ Azure CDN                            │   │
│  │                     │  ←───→  │ • Microsoft, Akamai, Verizon options │   │
│  │                     │         │ • OR use Front Door                  │   │
│  └─────────────────────┘         └──────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## When to Use What

```
                    ┌─────────────────────────┐
                    │ What type of traffic?   │
                    └───────────┬─────────────┘
                                │
         ┌──────────────────────┼──────────────────────┐
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ HTTP/HTTPS      │    │ TCP/UDP         │    │ Global HTTP     │
│ (Web apps)      │    │ (Non-HTTP)      │    │ Multi-region    │
└────────┬────────┘    └────────┬────────┘    └────────┬────────┘
         │                      │                      │
         ▼                      ▼                      ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│ Single region?  │    │ Azure Load      │    │ Azure Front     │
│                 │    │ Balancer        │    │ Door            │
│ Yes → App       │    │ (Standard)      │    │                 │
│       Gateway   │    │                 │    │ + CDN           │
│                 │    │ + HA Ports for  │    │ + WAF           │
│ No → Front Door │    │   NVAs          │    │ + SSL           │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

---

## Chapter Contents

| File | Topic | Description |
|------|-------|-------------|
| [Quick Reference](quick-reference.md) | Cheat Sheet | CLI commands, decision matrix, quick configs |
| [01-application-gateway.md](01-application-gateway.md) | Application Gateway | L7 load balancing, WAF, SSL termination, path routing |
| [02-load-balancer.md](02-load-balancer.md) | Azure Load Balancer | L4 load balancing, HA ports, health probes |
| [03-front-door.md](03-front-door.md) | Azure Front Door | Global load balancing, CDN, WAF, anycast |
| [case-studies.md](case-studies.md) | Case Studies | Real-world implementations and patterns |
| [extra-resources.md](extra-resources.md) | Resources | Official docs, videos, learning paths |

---

## Application Gateway Deep Dive

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     APPLICATION GATEWAY ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│                                  ▼                                           │
│                    ┌─────────────────────────────┐                          │
│                    │    Azure Front Door         │  (Optional global LB)    │
│                    │    OR                       │                          │
│                    │    Public IP                │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│  ┌───────────────────────────────┴───────────────────────────────────────┐  │
│  │                    APPLICATION GATEWAY                                 │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  FRONTEND                                                        │  │  │
│  │  │  • Public IP (or Internal IP for private apps)                  │  │  │
│  │  │  • Listeners (HTTP/HTTPS, multi-site)                           │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  WAF v2 (Optional but recommended)                              │  │  │
│  │  │  • OWASP Core Rule Set                                          │  │  │
│  │  │  • Bot protection                                               │  │  │
│  │  │  • Custom rules                                                 │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  ROUTING RULES                                                   │  │  │
│  │  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │  │  │
│  │  │  │ /api/*         │  │ /images/*       │  │ /*              │  │  │  │
│  │  │  │ → API Pool     │  │ → Static Pool   │  │ → Web Pool      │  │  │  │
│  │  │  └─────────────────┘  └─────────────────┘  └─────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  BACKEND POOLS                                                   │  │  │
│  │  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │  │  │
│  │  │  │ API Pool        │  │ Static Pool     │  │ Web Pool        │  │  │  │
│  │  │  │ • VM Scale Set  │  │ • Storage Blob  │  │ • App Service   │  │  │  │
│  │  │  │ • AKS           │  │ • CDN           │  │ • VMs           │  │  │  │
│  │  │  └─────────────────┘  └─────────────────┘  └─────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Configuration Example

```bash
# Create Application Gateway with WAF
az network application-gateway create \
  --resource-group myRG \
  --name myAppGW \
  --location eastus \
  --sku WAF_v2 \
  --capacity 2 \
  --vnet-name myVNet \
  --subnet appgw-subnet \
  --public-ip-address appgw-pip \
  --http-settings-cookie-based-affinity Disabled \
  --frontend-port 443 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --servers 10.0.2.10 10.0.2.11

# Enable WAF
az network application-gateway waf-config set \
  --resource-group myRG \
  --gateway-name myAppGW \
  --enabled true \
  --firewall-mode Prevention \
  --rule-set-version 3.2
```

---

## Azure Front Door

### Global Load Balancing with CDN

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AZURE FRONT DOOR ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              USERS WORLDWIDE                                 │
│                    Americas    │    Europe    │    Asia                     │
│                        │            │            │                           │
│                        ▼            ▼            ▼                           │
│            ┌─────────────────────────────────────────────────────┐          │
│            │              AZURE FRONT DOOR                       │          │
│            │              (Anycast at Edge)                      │          │
│            │                                                     │          │
│            │  ┌─────────────────────────────────────────────┐   │          │
│            │  │  EDGE LOCATIONS (200+ POPs globally)        │   │          │
│            │  │  • SSL termination at edge                  │   │          │
│            │  │  • WAF at edge                              │   │          │
│            │  │  • Caching at edge                          │   │          │
│            │  │  • Compression                              │   │          │
│            │  └─────────────────────────────────────────────┘   │          │
│            │                         │                           │          │
│            │  ┌─────────────────────────────────────────────┐   │          │
│            │  │  ROUTING                                    │   │          │
│            │  │  • Latency-based (fastest backend)          │   │          │
│            │  │  • Priority-based (active-passive)          │   │          │
│            │  │  • Weighted (A/B testing)                   │   │          │
│            │  │  • Session affinity                         │   │          │
│            │  └─────────────────────────────────────────────┘   │          │
│            └─────────────────────────────┬───────────────────────┘          │
│                                          │                                   │
│                    ┌─────────────────────┼─────────────────────┐            │
│                    │                     │                     │            │
│                    ▼                     ▼                     ▼            │
│          ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐   │
│          │  ORIGIN GROUP 1 │   │  ORIGIN GROUP 2 │   │  ORIGIN GROUP 3 │   │
│          │  East US        │   │  West Europe    │   │  East Asia      │   │
│          │  • App Service  │   │  • App Service  │   │  • App Service  │   │
│          │  • VMSS         │   │  • AKS          │   │  • Blob         │   │
│          └─────────────────┘   └─────────────────┘   └─────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

User in Paris → Front Door edge in Paris → Routed to West Europe origin
User in Tokyo → Front Door edge in Tokyo → Routed to East Asia origin
```

---

## Auto-Scaling

### VM Scale Sets (VMSS)

**AWS Equivalent**: Auto Scaling Groups

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    VM SCALE SET ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      LOAD BALANCER                                   │    │
│  └─────────────────────────────────────┬───────────────────────────────┘    │
│                                        │                                     │
│  ┌─────────────────────────────────────┴───────────────────────────────┐    │
│  │                      VM SCALE SET                                    │    │
│  │                                                                      │    │
│  │  INSTANCES (Auto-managed):                                          │    │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐  . . .          │    │
│  │  │ VM 0 │  │ VM 1 │  │ VM 2 │  │ VM 3 │  │ VM 4 │                  │    │
│  │  │ AZ 1 │  │ AZ 2 │  │ AZ 3 │  │ AZ 1 │  │ AZ 2 │                  │    │
│  │  └──────┘  └──────┘  └──────┘  └──────┘  └──────┘                  │    │
│  │                                                                      │    │
│  │  SCALING RULES:                                                     │    │
│  │  ┌────────────────────────────────────────────────────────────┐    │    │
│  │  │ IF CPU > 70% for 5 min → Scale OUT by 2 instances         │    │    │
│  │  │ IF CPU < 30% for 10 min → Scale IN by 1 instance          │    │    │
│  │  │ Min: 2, Max: 10, Default: 3                               │    │    │
│  │  └────────────────────────────────────────────────────────────┘    │    │
│  │                                                                      │    │
│  │  UPGRADE POLICY:                                                    │    │
│  │  • Automatic (update all at once)                                   │    │
│  │  • Rolling (update in batches)                                      │    │
│  │  • Manual (you control when)                                        │    │
│  │                                                                      │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Create VMSS with autoscaling
az vmss create \
  --resource-group myRG \
  --name myScaleSet \
  --image Ubuntu2204 \
  --vm-sku Standard_DS2_v2 \
  --instance-count 2 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --load-balancer myLB \
  --zones 1 2 3

# Create autoscale settings
az monitor autoscale create \
  --resource-group myRG \
  --resource myScaleSet \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name autoscale \
  --min-count 2 \
  --max-count 10 \
  --count 3

# Scale out rule
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name autoscale \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2

# Scale in rule
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name autoscale \
  --condition "Percentage CPU < 30 avg 10m" \
  --scale in 1
```

---

## App Service Auto-Scale

```bash
# Scale App Service Plan
az appservice plan update \
  --resource-group myRG \
  --name myPlan \
  --sku P1V3 \
  --number-of-workers 3

# Create autoscale for App Service
az monitor autoscale create \
  --resource-group myRG \
  --resource myPlan \
  --resource-type Microsoft.Web/serverfarms \
  --name app-autoscale \
  --min-count 1 \
  --max-count 10 \
  --count 2

# Scale based on HTTP queue length
az monitor autoscale rule create \
  --resource-group myRG \
  --autoscale-name app-autoscale \
  --condition "HttpQueueLength > 100 avg 5m" \
  --scale out 1
```

---

## Service Comparison Summary

| Feature | AWS | Azure | Notes |
|---------|-----|-------|-------|
| Layer 7 Regional | ALB | Application Gateway | AppGW has built-in WAF |
| Layer 4 Regional | NLB | Azure Load Balancer | Standard SKU is zone-redundant |
| Global HTTP | CloudFront + ALB | Azure Front Door | Front Door includes CDN |
| Global TCP/UDP | Global Accelerator | Front Door (TCP preview) | |
| CDN | CloudFront | Azure CDN or Front Door | Multiple providers available |
| Auto Scaling (VMs) | Auto Scaling Groups | VM Scale Sets | Very similar |
| Auto Scaling (Containers) | ECS Service Auto Scaling | AKS HPA/KEDA | |
| Auto Scaling (PaaS) | Lambda concurrency | App Service autoscale | |

---

*Next Chapter: [Containers & AKS](../13-containers-aks/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
