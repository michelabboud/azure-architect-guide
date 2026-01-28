# Security Quick Reference

## Microsoft Defender for Cloud - At a Glance

### Tiers and Coverage

| Plan | What It Protects | AWS Equivalent | Key Features |
|------|------------------|----------------|--------------|
| **CSPM (Free)** | Posture assessment | Security Hub (basic) | Secure score, recommendations |
| **CSPM (Defender)** | Advanced posture | Security Hub + Config | Attack path analysis, governance |
| **Servers** | VMs, Arc-enabled | GuardDuty + Inspector | EDR, vulnerability scanning |
| **Containers** | AKS, K8s | GuardDuty EKS | Runtime protection, scanning |
| **App Service** | Web apps | (No direct equivalent) | Threat detection for apps |
| **Storage** | Blob, Files | Macie (partial) | Malware scanning, threat detection |
| **SQL** | Azure SQL, SQL on VMs | GuardDuty RDS | SQL threat detection |
| **Key Vault** | Key Vault | (No equivalent) | Anomalous access detection |
| **ARM** | Control plane | CloudTrail + GuardDuty | Suspicious operations |
| **DNS** | DNS queries | Route53 Resolver DNS Firewall | DNS threat detection |

### Enable Defender (CLI)

```bash
# Enable Defender for Cloud (subscription level)
az security pricing create --name VirtualMachines --tier Standard
az security pricing create --name StorageAccounts --tier Standard
az security pricing create --name SqlServers --tier Standard
az security pricing create --name Containers --tier Standard

# Check status
az security pricing list --output table
```

---

## Zero Trust Quick Checklist

### Identity (Chapter 01)
- [ ] MFA enforced for all users
- [ ] Legacy authentication blocked
- [ ] Conditional Access policies active
- [ ] PIM configured for admin roles
- [ ] Access reviews scheduled
- [ ] Break-glass accounts documented

### Network
- [ ] Hub-spoke or Virtual WAN topology
- [ ] NSGs on all subnets
- [ ] Private Endpoints for PaaS services
- [ ] Azure Firewall or NVA for inspection
- [ ] No public IPs on VMs (use Bastion)
- [ ] DDoS Protection enabled

### Data
- [ ] Key Vault for all secrets/keys
- [ ] Encryption at rest enabled
- [ ] TLS 1.2+ enforced
- [ ] Purview data classification
- [ ] DLP policies configured

### Workloads
- [ ] Defender for Cloud enabled (all tiers)
- [ ] Vulnerability scanning active
- [ ] Just-in-time VM access
- [ ] Secure score > 70%

---

## Network Security Patterns

### Hub-Spoke Architecture

```
                              ┌─────────────────────────────────┐
                              │           INTERNET              │
                              └───────────────┬─────────────────┘
                                              │
                              ┌───────────────┴─────────────────┐
                              │      Azure Firewall / NVA       │
                              │      (or Azure Front Door)      │
                              └───────────────┬─────────────────┘
                                              │
┌─────────────────────────────────────────────┴─────────────────────────────────────────────┐
│                                         HUB VNET                                           │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐      │
│  │ Azure Firewall  │  │ VPN Gateway     │  │ ExpressRoute    │  │ Azure Bastion   │      │
│  │ Subnet          │  │ Subnet          │  │ Gateway Subnet  │  │ Subnet          │      │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  └─────────────────┘      │
└─────────────────────────────────┬─────────────────────────────────────────────────────────┘
                                  │ (VNet Peering)
            ┌─────────────────────┼─────────────────────┐
            │                     │                     │
            ▼                     ▼                     ▼
┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐
│    SPOKE 1 (Prod)   │ │   SPOKE 2 (Dev)     │ │   SPOKE 3 (Data)    │
│  ┌───────────────┐  │ │  ┌───────────────┐  │ │  ┌───────────────┐  │
│  │ App Tier      │  │ │  │ App Tier      │  │ │  │ Analytics     │  │
│  └───────────────┘  │ │  └───────────────┘  │ │  └───────────────┘  │
│  ┌───────────────┐  │ │  ┌───────────────┐  │ │  ┌───────────────┐  │
│  │ Data Tier     │  │ │  │ Data Tier     │  │ │  │ Storage       │  │
│  └───────────────┘  │ │  └───────────────┘  │ │  └───────────────┘  │
└─────────────────────┘ └─────────────────────┘ └─────────────────────┘
```

### Private Endpoint Pattern

```
BEFORE (Public Endpoint):
─────────────────────────
┌─────────┐       ┌─────────────┐       ┌─────────────┐
│   VM    │──────▶│  Internet   │──────▶│ SQL Database│
│         │       │  (Public)   │       │ (Public IP) │
└─────────┘       └─────────────┘       └─────────────┘
                        ↑
                   RISK: Exposed


AFTER (Private Endpoint):
─────────────────────────
┌─────────┐       ┌─────────────┐       ┌─────────────┐
│   VM    │──────▶│ Private     │──────▶│ SQL Database│
│         │       │ Endpoint    │       │ (No public) │
└─────────┘       │ (Private IP)│       └─────────────┘
                  └─────────────┘
                        ↑
              Traffic stays in VNet
```

---

## NSG Rules Quick Reference

### Common Rule Patterns

```bash
# Allow HTTPS from anywhere
az network nsg rule create \
  --nsg-name myNSG \
  --resource-group myRG \
  --name AllowHTTPS \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 443

# Deny all inbound (explicit, after allow rules)
az network nsg rule create \
  --nsg-name myNSG \
  --resource-group myRG \
  --name DenyAllInbound \
  --priority 4096 \
  --direction Inbound \
  --access Deny \
  --protocol '*' \
  --source-address-prefixes '*' \
  --destination-port-ranges '*'

# Allow internal subnet traffic
az network nsg rule create \
  --nsg-name myNSG \
  --resource-group myRG \
  --name AllowVnetInternal \
  --priority 200 \
  --direction Inbound \
  --access Allow \
  --protocol '*' \
  --source-address-prefixes 'VirtualNetwork' \
  --destination-port-ranges '*'
```

### Service Tags (Use Instead of IPs)

| Tag | What It Covers |
|-----|----------------|
| `VirtualNetwork` | VNet + peered VNets + on-prem |
| `AzureLoadBalancer` | Azure LB health probes |
| `Internet` | Public internet (exclude VNet) |
| `AzureCloud` | All Azure public IPs |
| `AzureCloud.WestUS` | Azure IPs in specific region |
| `Storage` | Azure Storage public IPs |
| `Sql` | Azure SQL public IPs |
| `AzureActiveDirectory` | Entra ID endpoints |
| `AzureMonitor` | Azure Monitor endpoints |

---

## Key Vault Quick Reference

### Access Models

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     KEY VAULT ACCESS MODELS                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. VAULT ACCESS POLICY (Legacy)          2. AZURE RBAC (Recommended)       │
│  ─────────────────────────────────        ───────────────────────────       │
│                                                                              │
│  Key Vault has its own permission         Standard Azure RBAC applied       │
│  model separate from Azure RBAC           to data plane                     │
│                                                                              │
│  • Get, List, Set, Delete, etc.           • Key Vault Secrets User          │
│  • Per secret/key/cert                    • Key Vault Crypto User           │
│  • No inheritance                         • Key Vault Administrator         │
│  • Hard to audit                          • Consistent with Azure           │
│                                                                              │
│  When to use:                             When to use:                      │
│  • Legacy applications                    • New deployments                 │
│  • Specific compliance needs              • Standard governance             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Common Commands

```bash
# Create Key Vault with RBAC
az keyvault create \
  --name myKeyVault \
  --resource-group myRG \
  --location eastus \
  --enable-rbac-authorization true

# Grant secret read access (RBAC)
az role assignment create \
  --assignee {user-or-principal-id} \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{kv}

# Set a secret
az keyvault secret set \
  --vault-name myKeyVault \
  --name "DatabasePassword" \
  --value "SuperSecret123!"

# Get a secret
az keyvault secret show \
  --vault-name myKeyVault \
  --name "DatabasePassword" \
  --query value -o tsv
```

### Key Vault Best Practices

| Practice | Why |
|----------|-----|
| Use Managed Identity to access | No secrets for accessing secrets |
| Enable soft-delete | Recover accidentally deleted items |
| Enable purge protection | Prevent permanent deletion |
| Use Private Endpoint | No public internet exposure |
| Enable logging to Log Analytics | Audit all access |
| Separate vaults per environment | Blast radius reduction |

---

## Defender for Cloud - Secure Score

### Understanding Secure Score

```
Secure Score = (Points Achieved / Maximum Points) × 100%

Example:
┌─────────────────────────────────────────────────────────────────────────────┐
│                        SECURE SCORE: 72%                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CONTROL                           SCORE      MAX     STATUS                │
│  ────────────────────────────────────────────────────────────               │
│  Enable MFA                        10/10      10      ✓ Complete            │
│  Enable endpoint protection         6/10      10      ⚠ Partial             │
│  Encrypt data in transit           8/8        8       ✓ Complete            │
│  Restrict unauthorized access      12/15      15      ⚠ Partial             │
│  Enable auditing and logging       7/7        7       ✓ Complete            │
│  Apply system updates              15/20      20      ⚠ Partial             │
│  ...                                                                         │
│                                                                              │
│  TOTAL                            72/100     100                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Priority Recommendations (High Impact)

| Recommendation | Impact | Effort |
|----------------|--------|--------|
| Enable MFA for accounts with owner permissions | +10 | Low |
| Enable Defender for Servers | +5 | Medium |
| Enable encryption for storage accounts | +4 | Low |
| Configure NSGs on subnets | +3 | Medium |
| Enable Defender for SQL | +4 | Low |
| Enable Just-in-time VM access | +3 | Medium |

---

## Just-in-Time (JIT) VM Access

### How It Works

```
WITHOUT JIT:                          WITH JIT:
────────────                          ─────────

NSG Rule:                             NSG Rule:
Allow SSH/RDP from Any                Deny SSH/RDP from Any
    │                                     │
    ▼                                     ▼
┌─────────────┐                      ┌─────────────┐
│ Always Open │                      │Always Closed│
│ to Attack   │                      │ by Default  │
└─────────────┘                      └─────────────┘
                                          │
                                     Request Access
                                          │
                                          ▼
                                     ┌─────────────┐
                                     │ Approved    │
                                     │ Time-limited│
                                     │ IP-specific │
                                     └─────────────┘
                                          │
                                     Access for 3 hours
                                     from specific IP
```

### Enable JIT

```bash
# Via Azure CLI (simplified)
# Full implementation typically via Portal or ARM

# Request JIT access
az security jit-policy set \
  --resource-group myRG \
  --name myVM \
  --virtual-machines '[{
    "id": "/subscriptions/.../virtualMachines/myVM",
    "ports": [{
      "number": 22,
      "protocol": "TCP",
      "allowedSourceAddressPrefix": "1.2.3.4",
      "maxRequestAccessDuration": "PT3H"
    }]
  }]'
```

---

## Security Alerts - Quick Response

### Common Alert Types

| Alert | Severity | Response |
|-------|----------|----------|
| Unusual sign-in activity | Medium | Check user, device, location |
| Brute force attack | High | Block IP, reset password |
| Malware detected | High | Isolate VM, investigate |
| Suspicious process execution | High | Investigate, contain |
| SQL injection attempt | Medium | Review WAF logs, block source |
| Unusual data access | Medium | Review who/what/when |
| Crypto mining detected | High | Isolate, investigate compromise |

### KQL for Security Investigation

```kusto
// Failed sign-ins from unusual locations
SigninLogs
| where ResultType != 0
| where Location !in ("United States", "Canada")
| summarize FailedAttempts = count() by UserPrincipalName, Location, IPAddress
| where FailedAttempts > 5
| order by FailedAttempts desc

// Defender alerts in last 24 hours
SecurityAlert
| where TimeGenerated > ago(24h)
| project TimeGenerated, AlertName, Severity, Description, Entities
| order by Severity, TimeGenerated desc

// VM with highest alert count
SecurityAlert
| where TimeGenerated > ago(7d)
| extend VM = tostring(parse_json(Entities)[0].Name)
| summarize AlertCount = count() by VM, Severity
| order by AlertCount desc
```

---

## Decision Tree: Security Service Selection

```
                    ┌─────────────────────────┐
                    │ What needs protection?  │
                    └───────────┬─────────────┘
                                │
     ┌──────────────────────────┼──────────────────────────┐
     │                          │                          │
     ▼                          ▼                          ▼
┌─────────────┐          ┌─────────────┐          ┌─────────────┐
│ Identity    │          │  Network    │          │  Workloads  │
│ attacks     │          │  threats    │          │  & Data     │
└──────┬──────┘          └──────┬──────┘          └──────┬──────┘
       │                        │                        │
       ▼                        ▼                        ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────────┐
│ Defender for    │    │ Azure Firewall  │    │ Defender for Cloud  │
│ Identity        │    │ + WAF           │    │ (workload-specific) │
│ + Entra ID      │    │ + DDoS          │    │                     │
│   Protection    │    │ + NSGs          │    │ • Servers           │
└─────────────────┘    └─────────────────┘    │ • Containers        │
                                              │ • Storage           │
                                              │ • SQL               │
                                              │ • Key Vault         │
                                              └─────────────────────┘
```

---

*Deep Dive: [Zero Trust](01-zero-trust.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
