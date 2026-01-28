# Network Security Deep Dive

## The Azure Network Security Stack

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AZURE NETWORK SECURITY LAYERS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│  LAYER 1: EDGE PROTECTION        │                                           │
│  ────────────────────────        ▼                                           │
│                    ┌─────────────────────────────┐                          │
│                    │    Azure DDoS Protection    │                          │
│                    │    (Volumetric attacks)     │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│  LAYER 2: APPLICATION PROTECTION │                                           │
│  ────────────────────────────────▼                                           │
│                    ┌─────────────────────────────┐                          │
│                    │    Azure Front Door         │                          │
│                    │    + WAF (L7 attacks)       │                          │
│                    │    OR                       │                          │
│                    │    Application Gateway      │                          │
│                    │    + WAF                    │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│  LAYER 3: PERIMETER PROTECTION   │                                           │
│  ────────────────────────────────▼                                           │
│                    ┌─────────────────────────────┐                          │
│                    │    Azure Firewall           │                          │
│                    │    (L3-L7 filtering)        │                          │
│                    │    OR NVA (Palo Alto, etc.) │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│  LAYER 4: MICROSEGMENTATION      │                                           │
│  ────────────────────────────────▼                                           │
│                    ┌─────────────────────────────┐                          │
│                    │    Network Security Groups  │                          │
│                    │    (Subnet/NIC level)       │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│  LAYER 5: PRIVATE CONNECTIVITY   │                                           │
│  ────────────────────────────────▼                                           │
│                    ┌─────────────────────────────┐                          │
│                    │    Private Endpoints        │                          │
│                    │    (No public exposure)     │                          │
│                    └─────────────────────────────┘                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## AWS to Azure Network Security Mapping

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| Security Groups | Network Security Groups (NSG) | NSG has allow AND deny rules |
| NACLs | NSG (subnet-level) | Azure uses NSGs for both |
| AWS WAF | Azure WAF | Very similar capabilities |
| AWS Shield Standard | DDoS Protection Basic | Free, always-on |
| AWS Shield Advanced | DDoS Protection Standard | ~$2,944/month + overage |
| AWS Network Firewall | Azure Firewall | Azure Firewall is fully managed |
| AWS Firewall Manager | Azure Firewall Manager | Policy management across subscriptions |
| GuardDuty (network) | Defender for DNS + NSG Flow Logs | Different approach |

---

## Network Security Groups (NSGs)

### NSG Fundamentals

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    NSG RULE PROCESSING                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  INBOUND TRAFFIC:                                                           │
│  ────────────────                                                           │
│                                                                              │
│  Traffic arrives → NSG evaluates rules by PRIORITY (lowest number first)   │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Priority │ Name              │ Action │ Source      │ Dest Port   │    │
│  ├──────────┼───────────────────┼────────┼─────────────┼─────────────┤    │
│  │ 100      │ AllowHTTPS        │ Allow  │ Internet    │ 443         │    │
│  │ 200      │ AllowSSHFromMgmt  │ Allow  │ 10.0.0.0/24 │ 22          │    │
│  │ 300      │ DenyAllInbound    │ Deny   │ Any         │ Any         │    │
│  │ 65000    │ AllowVnetInBound  │ Allow  │ VNet        │ Any         │ ←──┤
│  │ 65001    │ AllowAzureLB      │ Allow  │ AzureLB     │ Any         │    │ DEFAULT
│  │ 65500    │ DenyAllInBound    │ Deny   │ Any         │ Any         │ ←──┤ RULES
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  First matching rule wins!                                                  │
│                                                                              │
│  OUTBOUND TRAFFIC:                                                          │
│  ─────────────────                                                          │
│                                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Priority │ Name               │ Action │ Dest          │ Port     │    │
│  ├──────────┼────────────────────┼────────┼───────────────┼──────────┤    │
│  │ 100      │ AllowStorageOut    │ Allow  │ Storage       │ 443      │    │
│  │ 200      │ AllowSQLOut        │ Allow  │ Sql           │ 1433     │    │
│  │ 300      │ DenyInternetOut    │ Deny   │ Internet      │ Any      │    │
│  │ 65000    │ AllowVnetOutBound  │ Allow  │ VNet          │ Any      │ ←──┤
│  │ 65001    │ AllowInternetOut   │ Allow  │ Internet      │ Any      │    │ DEFAULT
│  │ 65500    │ DenyAllOutBound    │ Deny   │ Any           │ Any      │ ←──┤ RULES
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### NSG vs AWS Security Groups

```
AWS SECURITY GROUP:                     AZURE NSG:
───────────────────                     ──────────

┌─────────────────────┐                ┌─────────────────────────────────────┐
│ • Allow rules ONLY  │                │ • Allow AND Deny rules              │
│ • Implicit deny     │                │ • Explicit priority (100-4096)      │
│ • Stateful          │                │ • Stateful                          │
│ • Attached to ENI   │                │ • Attached to Subnet OR NIC         │
│ • No priorities     │                │ • Default rules included            │
│ • 60 rules/SG       │                │ • 1000 rules/NSG                    │
│ • 5 SGs per ENI     │                │ • 1 NSG per subnet/NIC              │
└─────────────────────┘                └─────────────────────────────────────┘

AWS Example:                           Azure Example:
─────────────                          ──────────────

Inbound:                               Inbound:
- Allow 443 from 0.0.0.0/0            - Priority 100: Allow 443 from Internet
- Allow 22 from 10.0.0.0/8            - Priority 200: Allow 22 from 10.0.0.0/8
(implicit deny all else)               - Priority 4096: Deny all (explicit)

To block specific IP in AWS:           To block specific IP in Azure:
- Not possible with SG alone           - Priority 50: Deny from bad-ip
- Need NACL                            - (lower priority = evaluated first)
```

### Service Tags (Critical Feature)

Service tags are Azure-managed IP address groups that automatically update.

```bash
# Instead of managing IP ranges manually:
# BAD: --source-address-prefixes 13.107.6.152/31 13.107.9.152/31 ...

# Use service tags:
# GOOD: --source-address-prefixes AzureCloud

Common Service Tags:
┌──────────────────────────┬──────────────────────────────────────────────────┐
│ Service Tag              │ What it includes                                 │
├──────────────────────────┼──────────────────────────────────────────────────┤
│ VirtualNetwork           │ VNet address space + peered VNets + on-prem     │
│ AzureLoadBalancer        │ Azure LB health probes                          │
│ Internet                 │ Public internet (excludes VNet)                 │
│ AzureCloud               │ All Azure datacenter IPs                        │
│ AzureCloud.EastUS        │ Azure IPs in East US region                     │
│ Storage                  │ Azure Storage service IPs                       │
│ Storage.EastUS           │ Storage IPs in East US                          │
│ Sql                      │ Azure SQL service IPs                           │
│ AzureActiveDirectory     │ Entra ID endpoints                              │
│ AzureMonitor             │ Azure Monitor endpoints                         │
│ AzureKeyVault            │ Key Vault endpoints                             │
│ EventHub                 │ Event Hub endpoints                             │
│ ServiceBus               │ Service Bus endpoints                           │
│ AzureContainerRegistry   │ ACR endpoints                                   │
│ MicrosoftContainerRegistry│ MCR (Microsoft images)                         │
│ AzureKubernetesService   │ AKS management IPs                              │
│ GatewayManager           │ VPN/ExpressRoute Gateway management             │
│ AzureBastion             │ Bastion service IPs                             │
└──────────────────────────┴──────────────────────────────────────────────────┘
```

### Application Security Groups (ASGs)

ASGs let you group VMs logically and write rules based on application tiers.

```
WITHOUT ASGs (IP-based):               WITH ASGs (Application-based):
────────────────────────               ────────────────────────────

NSG Rule:                              NSG Rule:
Allow 8080                             Allow 8080
From: 10.0.1.10, 10.0.1.11, 10.0.1.12 From: ASG-WebServers
To: 10.0.2.20, 10.0.2.21              To: ASG-AppServers

Problem: Must update IPs when          Benefit: Just add VM to ASG
         VMs change                             Rule auto-applies

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  ASG: WebServers                     ASG: AppServers                │   │
│  │  ┌─────┐ ┌─────┐ ┌─────┐            ┌─────┐ ┌─────┐                │   │
│  │  │Web1 │ │Web2 │ │Web3 │    ────►   │App1 │ │App2 │                │   │
│  │  └─────┘ └─────┘ └─────┘    8080    └─────┘ └─────┘                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  NSG Rule: Allow TCP/8080 from ASG-WebServers to ASG-AppServers            │
│                                                                              │
│  When you add Web4:                                                         │
│  1. Attach Web4's NIC to ASG-WebServers                                    │
│  2. Rule automatically applies (no NSG change needed)                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Azure Firewall

### Azure Firewall vs AWS Network Firewall

| Feature | Azure Firewall | AWS Network Firewall |
|---------|----------------|---------------------|
| Type | Cloud-native, fully managed | Cloud-native, managed |
| Deployment | Single resource in VNet | Endpoint per AZ |
| Throughput | Up to 100 Gbps | Up to 100 Gbps |
| FQDN filtering | Yes (built-in) | Yes |
| TLS inspection | Yes (Premium) | Yes |
| Threat intelligence | Yes (Premium) | With GuardDuty |
| IDPS | Yes (Premium) | Yes |
| URL filtering | Yes | Via domain lists |
| Pricing | ~$1.25/hour + data | ~$0.395/hour + data |

### Azure Firewall Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AZURE FIREWALL ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│                    ┌─────────────┴─────────────┐                            │
│                    │    Azure Firewall          │                            │
│                    │    Public IP               │                            │
│                    └─────────────┬─────────────┘                            │
│                                  │                                           │
│  ┌───────────────────────────────┴───────────────────────────────────────┐  │
│  │                       HUB VNET                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │              AzureFirewallSubnet (min /26)                       │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌───────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │                 AZURE FIREWALL                            │  │  │  │
│  │  │  │                                                            │  │  │  │
│  │  │  │  RULE COLLECTIONS (processed in order):                   │  │  │  │
│  │  │  │                                                            │  │  │  │
│  │  │  │  1. NAT Rules (DNAT - inbound)                            │  │  │  │
│  │  │  │     └─ Public IP:443 → Internal LB:443                    │  │  │  │
│  │  │  │                                                            │  │  │  │
│  │  │  │  2. Network Rules (L3/L4)                                 │  │  │  │
│  │  │  │     └─ Allow 10.1.0.0/16 → 10.2.0.0/16 : 1433            │  │  │  │
│  │  │  │     └─ Allow * → SQL (service tag) : 1433                 │  │  │  │
│  │  │  │                                                            │  │  │  │
│  │  │  │  3. Application Rules (L7 - FQDN)                         │  │  │  │
│  │  │  │     └─ Allow *.microsoft.com : HTTPS                      │  │  │  │
│  │  │  │     └─ Allow github.com : HTTPS                           │  │  │  │
│  │  │  │     └─ Deny * (implicit)                                  │  │  │  │
│  │  │  │                                                            │  │  │  │
│  │  │  └───────────────────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                         │                                    │
│            ┌────────────────────────────┼────────────────────────────┐      │
│            │ (VNet Peering + UDR)       │                            │      │
│            ▼                            ▼                            ▼      │
│  ┌─────────────────┐          ┌─────────────────┐          ┌─────────────┐ │
│  │   SPOKE 1       │          │   SPOKE 2       │          │   SPOKE 3   │ │
│  │   Route Table:  │          │   Route Table:  │          │             │ │
│  │   0.0.0.0/0 →   │          │   0.0.0.0/0 →   │          │             │ │
│  │   Firewall IP   │          │   Firewall IP   │          │             │ │
│  └─────────────────┘          └─────────────────┘          └─────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Azure Firewall Tiers

| Feature | Standard | Premium |
|---------|----------|---------|
| L3-L4 filtering | ✓ | ✓ |
| FQDN filtering | ✓ | ✓ |
| FQDN tags (Azure services) | ✓ | ✓ |
| Service tags | ✓ | ✓ |
| Threat intelligence | ✓ | ✓ |
| DNS proxy | ✓ | ✓ |
| TLS inspection | ✗ | ✓ |
| IDPS (signatures) | ✗ | ✓ |
| URL filtering | ✗ | ✓ |
| Web categories | ✗ | ✓ |
| Cost | ~$1.25/hr | ~$1.75/hr |

### Azure Firewall Configuration

```bash
# Create Azure Firewall
az network firewall create \
  --resource-group myRG \
  --name myFirewall \
  --location eastus \
  --vnet-name hub-vnet \
  --sku AZFW_VNet \
  --tier Premium \
  --enable-dns-proxy true

# Create public IP
az network public-ip create \
  --resource-group myRG \
  --name fw-pip \
  --sku Standard \
  --allocation-method Static

# Associate public IP
az network firewall ip-config create \
  --firewall-name myFirewall \
  --resource-group myRG \
  --name fw-config \
  --public-ip-address fw-pip \
  --vnet-name hub-vnet

# Create application rule (allow Windows Update)
az network firewall application-rule create \
  --firewall-name myFirewall \
  --resource-group myRG \
  --collection-name "AllowWindowsUpdate" \
  --name "WindowsUpdate" \
  --protocols Https=443 \
  --source-addresses "10.0.0.0/8" \
  --target-fqdns "*.windowsupdate.com" "*.microsoft.com" \
  --action Allow \
  --priority 100

# Create network rule (allow SQL)
az network firewall network-rule create \
  --firewall-name myFirewall \
  --resource-group myRG \
  --collection-name "AllowSQL" \
  --name "AllowAzureSQL" \
  --protocols TCP \
  --source-addresses "10.1.0.0/16" \
  --destination-addresses "Sql" \
  --destination-ports 1433 \
  --action Allow \
  --priority 200
```

---

## Web Application Firewall (WAF)

### WAF Deployment Options

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WAF DEPLOYMENT OPTIONS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  OPTION 1: Application Gateway + WAF v2                                     │
│  ───────────────────────────────────────                                    │
│  • Regional (single region)                                                 │
│  • Behind Azure Front Door or standalone                                    │
│  • Best for: Traditional web apps, internal apps                           │
│                                                                              │
│  OPTION 2: Azure Front Door + WAF                                          │
│  ─────────────────────────────────                                          │
│  • Global (all edge locations)                                              │
│  • DDoS protection included                                                 │
│  • Best for: Global apps, CDN + security                                   │
│                                                                              │
│  OPTION 3: Azure CDN + WAF                                                 │
│  ──────────────────────────                                                 │
│  • CDN with WAF (Microsoft tier)                                           │
│  • Best for: Static content with basic protection                          │
│                                                                              │
│                                                                              │
│  WAF RULE SETS:                                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │ OWASP Core Rule Set (CRS)                                              ││
│  │ • CRS 3.2 (default, recommended)                                       ││
│  │ • CRS 3.1, 3.0, 2.2.9 (legacy)                                         ││
│  │                                                                         ││
│  │ Protects against:                                                       ││
│  │ • SQL Injection                                                         ││
│  │ • Cross-Site Scripting (XSS)                                           ││
│  │ • Command Injection                                                     ││
│  │ • Local File Inclusion                                                  ││
│  │ • Remote Code Execution                                                 ││
│  │ • Protocol violations                                                   ││
│  │ • And more...                                                           ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  BOT PROTECTION (Add-on):                                                   │
│  • Good bots (search engines) - Allow                                       │
│  • Bad bots (scrapers, attacks) - Block                                    │
│  • Unknown bots - Challenge (CAPTCHA)                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### WAF Modes

```
DETECTION MODE:                         PREVENTION MODE:
───────────────                         ────────────────

• Logs all rule matches                 • Blocks matching requests
• Does NOT block traffic                • Logs all blocked requests
• Use for: Testing, tuning              • Use for: Production

Request → WAF evaluates → Log only      Request → WAF evaluates → Block + Log
                       ↓                                        ↓
                    Backend                                  403 Forbidden

RECOMMENDATION:
1. Start in Detection mode
2. Review logs for false positives
3. Create exclusions for legitimate traffic
4. Switch to Prevention mode
```

### WAF Configuration

```bash
# Create WAF policy
az network application-gateway waf-policy create \
  --resource-group myRG \
  --name myWAFPolicy

# Configure managed rules (OWASP CRS)
az network application-gateway waf-policy managed-rule rule-set add \
  --policy-name myWAFPolicy \
  --resource-group myRG \
  --type OWASP \
  --version 3.2

# Add bot protection
az network application-gateway waf-policy managed-rule rule-set add \
  --policy-name myWAFPolicy \
  --resource-group myRG \
  --type Microsoft_BotManagerRuleSet \
  --version 1.0

# Create custom rule (block specific IPs)
az network application-gateway waf-policy custom-rule create \
  --policy-name myWAFPolicy \
  --resource-group myRG \
  --name BlockBadIPs \
  --priority 10 \
  --rule-type MatchRule \
  --action Block \
  --match-conditions '[{
    "matchVariables": [{"variableName": "RemoteAddr"}],
    "operator": "IPMatch",
    "matchValues": ["192.0.2.1", "198.51.100.0/24"]
  }]'

# Create custom rule (rate limiting)
az network application-gateway waf-policy custom-rule create \
  --policy-name myWAFPolicy \
  --resource-group myRG \
  --name RateLimitRule \
  --priority 20 \
  --rule-type RateLimitRule \
  --rate-limit-threshold 1000 \
  --rate-limit-duration FiveMins \
  --action Block \
  --match-conditions '[{
    "matchVariables": [{"variableName": "RequestUri"}],
    "operator": "Contains",
    "matchValues": ["/api/"]
  }]'

# Enable WAF in prevention mode
az network application-gateway waf-policy update \
  --resource-group myRG \
  --name myWAFPolicy \
  --mode Prevention \
  --state Enabled
```

---

## DDoS Protection

### DDoS Tiers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DDOS PROTECTION TIERS                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  BASIC (Free - Always On):                                                  │
│  ─────────────────────────                                                  │
│  • Enabled by default for all Azure services                               │
│  • Protects against common network-layer attacks                           │
│  • No configuration needed                                                  │
│  • No SLA, no telemetry, no alerting                                       │
│                                                                              │
│  STANDARD ($2,944/month + overage):                                         │
│  ──────────────────────────────────                                         │
│  • Tuned specifically for your Azure resources                             │
│  • Real-time metrics and alerts                                            │
│  • Attack mitigation reports                                               │
│  • DDoS Rapid Response (DRR) team access                                   │
│  • Cost protection (credit for scale-out during attack)                    │
│  • Covers up to 100 public IPs                                             │
│  • SLA: 99.99%                                                              │
│                                                                              │
│  WHAT IT PROTECTS:                                                          │
│  ┌────────────────────────────────────────────────────────────────────────┐│
│  │ Attack Type              │ Basic │ Standard │                          ││
│  ├──────────────────────────┼───────┼──────────┤                          ││
│  │ Volumetric (floods)      │  ✓    │    ✓     │ UDP, ICMP, TCP flood    ││
│  │ Protocol attacks         │  ✓    │    ✓     │ SYN flood, Ping of Death││
│  │ Application layer (L7)   │  ✗    │    ✗*    │ Use WAF for L7          ││
│  └────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  * DDoS Standard protects the network layer; WAF protects application layer│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Enable DDoS Protection

```bash
# Create DDoS protection plan
az network ddos-protection create \
  --resource-group myRG \
  --name myDDoSPlan \
  --location eastus

# Associate with VNet
az network vnet update \
  --resource-group myRG \
  --name myVNet \
  --ddos-protection-plan myDDoSPlan \
  --ddos-protection true
```

---

## Azure Bastion

### Secure VM Access Without Public IPs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AZURE BASTION ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  WITHOUT BASTION:                      WITH BASTION:                        │
│  ────────────────                      ─────────────                        │
│                                                                              │
│  User → Internet → Public IP → VM      User → Browser → Bastion → VM       │
│         (SSH/RDP exposed!)                    (HTTPS only, no public IP)   │
│                                                                              │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          VNET                                        │   │
│  │                                                                      │   │
│  │  ┌───────────────────────────────────────────────────────────────┐  │   │
│  │  │              AzureBastionSubnet (min /26)                      │  │   │
│  │  │                                                                 │  │   │
│  │  │  ┌─────────────────────────────────────────────────────────┐  │  │   │
│  │  │  │              AZURE BASTION                               │  │  │   │
│  │  │  │              (Managed PaaS)                              │  │  │   │
│  │  │  │                                                          │  │  │   │
│  │  │  │  • Browser-based RDP/SSH (HTML5)                        │  │  │   │
│  │  │  │  • No public IP on VMs needed                           │  │  │   │
│  │  │  │  • Integrated with Entra ID                             │  │  │   │
│  │  │  │  • Session recording (Premium)                          │  │  │   │
│  │  │  │  • Copy/paste control                                   │  │  │   │
│  │  │  └─────────────────────────────────────────────────────────┘  │  │   │
│  │  └───────────────────────────────────────────────────────────────┘  │   │
│  │                                │                                    │   │
│  │                    Connects via private IP                          │   │
│  │                                │                                    │   │
│  │  ┌──────────────────────┐  ┌──────────────────────┐                │   │
│  │  │        VM 1          │  │        VM 2          │                │   │
│  │  │  (No public IP)      │  │  (No public IP)      │                │   │
│  │  │  Private: 10.0.1.10  │  │  Private: 10.0.1.11  │                │   │
│  │  └──────────────────────┘  └──────────────────────┘                │   │
│  │                                                                      │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bastion SKUs

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Connect to VMs in same VNet | ✓ | ✓ | ✓ |
| Connect to peered VNets | ✗ | ✓ | ✓ |
| Connect via IP address | ✗ | ✓ | ✓ |
| Native client support (SSH/RDP) | ✗ | ✓ | ✓ |
| File transfer | ✗ | ✓ | ✓ |
| Shareable link | ✗ | ✓ | ✓ |
| Kerberos authentication | ✗ | ✗ | ✓ |
| Session recording | ✗ | ✗ | ✓ |
| Private-only deployment | ✗ | ✗ | ✓ |
| Cost | ~$0.19/hr | ~$0.35/hr | ~$0.70/hr |

```bash
# Create Bastion
az network bastion create \
  --resource-group myRG \
  --name myBastion \
  --vnet-name myVNet \
  --public-ip-address bastion-pip \
  --sku Standard \
  --location eastus
```

---

## Network Security Best Practices Summary

### Defense in Depth Checklist

```
□ EDGE
  ├── □ DDoS Protection Standard for production workloads
  ├── □ Azure Front Door with WAF for global apps
  └── □ Application Gateway with WAF for regional apps

□ PERIMETER
  ├── □ Azure Firewall in hub VNet
  ├── □ All spoke traffic routed through firewall (UDRs)
  ├── □ FQDN filtering for outbound traffic
  └── □ TLS inspection for sensitive workloads (Premium)

□ NETWORK SEGMENTATION
  ├── □ NSGs on every subnet
  ├── □ Deny by default, allow by exception
  ├── □ Use service tags (not IP ranges)
  ├── □ Use ASGs for application-tier grouping
  └── □ NSG flow logs enabled

□ PRIVATE CONNECTIVITY
  ├── □ Private Endpoints for all PaaS services
  ├── □ Disable public endpoints where possible
  ├── □ Private DNS zones for name resolution
  └── □ No public IPs on VMs (use Bastion)

□ MONITORING
  ├── □ NSG flow logs → Log Analytics
  ├── □ Azure Firewall logs → Log Analytics
  ├── □ Defender for DNS enabled
  ├── □ Alerts for suspicious traffic patterns
  └── □ Regular review of denied traffic
```

---

*Next: [Security Case Studies](case-studies.md)* | *Back to [Quick Reference](quick-reference.md)*
