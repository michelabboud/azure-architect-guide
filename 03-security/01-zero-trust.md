# Zero Trust Architecture Deep Dive

## What is Zero Trust?

Zero Trust is a security model that assumes no implicit trust. Every access request is fully authenticated, authorized, and encrypted before granting access—regardless of where the request originates or what resource it accesses.

### The Shift from Perimeter Security

```
TRADITIONAL (Castle-and-Moat):          ZERO TRUST:
──────────────────────────────          ───────────

      ┌─────────────────────┐                ┌─────────────────────┐
      │     FIREWALL        │                │  Every Request:     │
      │     (The Moat)      │                │  • Authenticated    │
      └──────────┬──────────┘                │  • Authorized       │
                 │                           │  • Encrypted        │
      ┌──────────┴──────────┐                └──────────┬──────────┘
      │   INTERNAL NETWORK  │                           │
      │   (Trusted Zone)    │                ┌──────────┴──────────┐
      │                     │                │   NO TRUSTED ZONE   │
      │   ┌─────┐ ┌─────┐   │                │                     │
      │   │ VM1 │ │ VM2 │   │                │   ┌─────┐ ┌─────┐   │
      │   └─────┘ └─────┘   │                │   │ VM1 │←┼→VM2 │   │
      │   (Trusted each     │                │   └──┬──┘ └──┬──┘   │
      │    other)           │                │      │verify │verify│
      └─────────────────────┘                │      ▼       ▼      │
                                             └─────────────────────┘
Assumption: Inside = Safe
Attack: Get in once = Free access            Assumption: Breach may exist
                                             Each hop: Re-verify
```

### The Three Principles

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ZERO TRUST PRINCIPLES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. VERIFY EXPLICITLY                                                        │
│     ───────────────────                                                      │
│     "Don't assume—verify every time"                                        │
│                                                                              │
│     Check ALL signals:                                                       │
│     • User identity (who)                                                   │
│     • Device health (from what)                                             │
│     • Location (where)                                                      │
│     • Resource requested (what)                                             │
│     • Time/behavior anomalies (when/how)                                    │
│                                                                              │
│     Azure: Entra ID + Conditional Access + Device Compliance                │
│     AWS: IAM + (limited to policy conditions)                               │
│                                                                              │
│  2. USE LEAST PRIVILEGE ACCESS                                              │
│     ──────────────────────────                                              │
│     "Grant minimum access for minimum time"                                 │
│                                                                              │
│     Implementation:                                                          │
│     • Just-In-Time access (PIM)                                             │
│     • Just-Enough-Access (minimal RBAC)                                     │
│     • Risk-based adaptive policies                                          │
│     • Time-bound permissions                                                │
│                                                                              │
│     Azure: PIM + RBAC + Access Reviews + Entitlement Management             │
│     AWS: IAM Roles + (no native JIT)                                        │
│                                                                              │
│  3. ASSUME BREACH                                                            │
│     ─────────────────                                                       │
│     "Act as if attackers are already inside"                                │
│                                                                              │
│     Implementation:                                                          │
│     • Microsegmentation                                                     │
│     • End-to-end encryption                                                 │
│     • Continuous monitoring                                                 │
│     • Anomaly detection                                                     │
│     • Minimize blast radius                                                 │
│                                                                              │
│     Azure: NSGs + Private Link + Defender + Sentinel                        │
│     AWS: Security Groups + PrivateLink + GuardDuty + (SIEM needed)         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Zero Trust Architecture in Azure

### The Complete Picture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AZURE ZERO TRUST REFERENCE ARCHITECTURE                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              ┌─────────────────┐                            │
│                              │    USER         │                            │
│                              │    (Any Device) │                            │
│                              └────────┬────────┘                            │
│                                       │                                      │
│  ┌────────────────────────────────────┼────────────────────────────────────┐│
│  │                          IDENTITY LAYER                                  ││
│  │  ┌─────────────┐    ┌─────────────────────┐    ┌──────────────────────┐ ││
│  │  │  Entra ID   │───▶│ Conditional Access  │───▶│  MFA / Passwordless  │ ││
│  │  │  (AuthN)    │    │ (Policy Engine)     │    │  (Strong Auth)       │ ││
│  │  └─────────────┘    └─────────────────────┘    └──────────────────────┘ ││
│  └────────────────────────────────────┼────────────────────────────────────┘│
│                                       │ (Token with claims)                  │
│  ┌────────────────────────────────────┼────────────────────────────────────┐│
│  │                          ENDPOINT LAYER                                  ││
│  │  ┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────┐  ││
│  │  │ Intune (MDM)    │───▶│ Compliance Check    │───▶│ Defender for    │  ││
│  │  │ Device Mgmt     │    │ (Is device healthy?)│    │ Endpoint (EDR)  │  ││
│  │  └─────────────────┘    └─────────────────────┘    └─────────────────┘  ││
│  └────────────────────────────────────┼────────────────────────────────────┘│
│                                       │ (Device state in token)              │
│  ┌────────────────────────────────────┼────────────────────────────────────┐│
│  │                          NETWORK LAYER                                   ││
│  │                                                                          ││
│  │     ┌─────────────┐      ┌─────────────┐      ┌─────────────┐          ││
│  │     │   Azure     │      │  Private    │      │    NSG      │          ││
│  │     │   Firewall  │─────▶│  Endpoints  │─────▶│  (Micro-    │          ││
│  │     │   / WAF     │      │             │      │  segmented) │          ││
│  │     └─────────────┘      └─────────────┘      └─────────────┘          ││
│  │                                                                          ││
│  └────────────────────────────────────┼────────────────────────────────────┘│
│                                       │                                      │
│  ┌────────────────────────────────────┼────────────────────────────────────┐│
│  │                        APPLICATION LAYER                                 ││
│  │  ┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────┐  ││
│  │  │  Managed        │───▶│  RBAC               │───▶│  App-level      │  ││
│  │  │  Identity       │    │  (Authorization)    │    │  AuthZ          │  ││
│  │  └─────────────────┘    └─────────────────────┘    └─────────────────┘  ││
│  └────────────────────────────────────┼────────────────────────────────────┘│
│                                       │                                      │
│  ┌────────────────────────────────────┼────────────────────────────────────┐│
│  │                           DATA LAYER                                     ││
│  │  ┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────┐  ││
│  │  │  Purview        │───▶│  Key Vault          │───▶│  Encryption     │  ││
│  │  │  Classification │    │  (Secrets/Keys)     │    │  (At rest/      │  ││
│  │  │  + DLP          │    │                     │    │   In transit)   │  ││
│  │  └─────────────────┘    └─────────────────────┘    └─────────────────┘  ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                     VISIBILITY & ANALYTICS                               ││
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐               ││
│  │  │ Defender for  │  │   Sentinel    │  │ Azure Monitor │               ││
│  │  │ Cloud         │  │   (SIEM)      │  │ (Telemetry)   │               ││
│  │  └───────────────┘  └───────────────┘  └───────────────┘               ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Case Studies: Beginner → Expert

### Beginner: Basic Zero Trust for a Web Application

**Scenario**: Simple web app needs Zero Trust without major infrastructure changes.

**Current State**:
- App Service running a web application
- SQL Database backend
- Blob Storage for files
- All using public endpoints

**Target State**:
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      BASIC ZERO TRUST WEB APP                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  USERS                                                                       │
│    │                                                                         │
│    │ 1. Authenticate to Entra ID                                            │
│    │ 2. Conditional Access: Require MFA                                     │
│    │ 3. Get token with identity claims                                      │
│    │                                                                         │
│    ▼                                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                      APP SERVICE                                     │    │
│  │  • Entra ID authentication enabled                                  │    │
│  │  • Managed Identity assigned                                        │    │
│  │  • HTTPS only, TLS 1.2                                              │    │
│  └───────────────────────────────┬─────────────────────────────────────┘    │
│                                  │                                           │
│                    4. App uses Managed Identity                              │
│                       (no stored credentials)                                │
│                                  │                                           │
│         ┌────────────────────────┼────────────────────────────┐             │
│         │                        │                            │             │
│         ▼                        ▼                            ▼             │
│  ┌─────────────┐         ┌─────────────┐         ┌─────────────────┐       │
│  │ SQL Database│         │ Key Vault   │         │ Blob Storage    │       │
│  │             │         │             │         │                 │       │
│  │ • Entra auth│         │ • RBAC      │         │ • RBAC          │       │
│  │ • MI access │         │ • MI access │         │ • MI access     │       │
│  │ • TLS       │         │ • Private   │         │ • Private       │       │
│  │             │         │   endpoint  │         │   endpoint      │       │
│  └─────────────┘         └─────────────┘         └─────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Implementation Steps**:

```bash
# 1. Enable Entra ID authentication on App Service
az webapp auth update \
  --resource-group myRG \
  --name myWebApp \
  --enabled true \
  --action LoginWithAzureActiveDirectory \
  --aad-client-id {app-registration-client-id}

# 2. Enable Managed Identity
az webapp identity assign \
  --resource-group myRG \
  --name myWebApp

# 3. Grant MI access to SQL
az sql server ad-admin create \
  --resource-group myRG \
  --server-name myserver \
  --display-name "App Service MI" \
  --object-id {managed-identity-principal-id}

# 4. Grant MI access to Storage
az role assignment create \
  --assignee {managed-identity-principal-id} \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{sa}

# 5. Grant MI access to Key Vault
az role assignment create \
  --assignee {managed-identity-principal-id} \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{kv}
```

**Conditional Access Policy**:
```
Name: MFA for Web App Access

Users: All users
Apps: [App Registration for Web App]
Grant: Require MFA
```

**Why This Design**:
> - **Managed Identity**: Zero credentials stored in app = zero credentials to steal
> - **Entra authentication**: Every user verified through corporate identity
> - **MFA**: Even with password compromise, attacker blocked
> - **Private Endpoints (optional next step)**: Move data plane traffic off internet

---

### Intermediate: Network Microsegmentation

**Scenario**: Multi-tier application needs network isolation between tiers.

**Architecture**:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      MICROSEGMENTED WEB APPLICATION                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│                                  ▼                                           │
│                    ┌─────────────────────────────┐                          │
│                    │    Application Gateway     │                          │
│                    │    + WAF                   │                          │
│                    │    (NSG: Allow 443 from *) │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│  ┌───────────────────────────────┼───────────────────────────────────────┐  │
│  │                     WEB TIER SUBNET                                    │  │
│  │  NSG Rules:                                                            │  │
│  │  • Inbound: Allow 443 from AppGateway only                            │  │
│  │  • Outbound: Allow 443 to API Tier only                               │  │
│  │  • Deny all else                                                       │  │
│  │                                                                        │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                   │  │
│  │  │   Web VM 1  │  │   Web VM 2  │  │   Web VM 3  │                   │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                   │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │                                           │
│  ┌───────────────────────────────┼───────────────────────────────────────┐  │
│  │                     API TIER SUBNET                                    │  │
│  │  NSG Rules:                                                            │  │
│  │  • Inbound: Allow 8080 from Web Tier only                             │  │
│  │  • Outbound: Allow 1433 to Data Tier only                             │  │
│  │  • Deny all else                                                       │  │
│  │                                                                        │  │
│  │  ┌─────────────┐  ┌─────────────┐                                     │  │
│  │  │   API VM 1  │  │   API VM 2  │                                     │  │
│  │  └─────────────┘  └─────────────┘                                     │  │
│  └───────────────────────────────┬───────────────────────────────────────┘  │
│                                  │                                           │
│  ┌───────────────────────────────┼───────────────────────────────────────┐  │
│  │                    DATA TIER SUBNET                                    │  │
│  │  NSG Rules:                                                            │  │
│  │  • Inbound: Allow 1433 from API Tier only                             │  │
│  │  • Outbound: Deny all (or specific Azure services)                    │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────┐                                      │  │
│  │  │     SQL Server               │                                      │  │
│  │  │     (Private Endpoint)       │                                      │  │
│  │  └─────────────────────────────┘                                      │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  RESULT: Attacker compromising Web tier CANNOT directly reach Data tier    │
│          Must pivot through API tier (which can have additional controls)  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**NSG Implementation**:

```bash
# Web Tier NSG
az network nsg create --name nsg-web-tier --resource-group myRG

# Allow from AppGateway only
az network nsg rule create \
  --nsg-name nsg-web-tier --resource-group myRG \
  --name AllowAppGateway --priority 100 \
  --direction Inbound --access Allow --protocol Tcp \
  --source-address-prefixes 10.0.0.0/24 \  # AppGW subnet
  --destination-port-ranges 443

# Allow to API Tier
az network nsg rule create \
  --nsg-name nsg-web-tier --resource-group myRG \
  --name AllowToAPI --priority 100 \
  --direction Outbound --access Allow --protocol Tcp \
  --destination-address-prefixes 10.0.2.0/24 \  # API subnet
  --destination-port-ranges 8080

# Explicit deny all other outbound
az network nsg rule create \
  --nsg-name nsg-web-tier --resource-group myRG \
  --name DenyAllOutbound --priority 4096 \
  --direction Outbound --access Deny --protocol '*' \
  --destination-address-prefixes '*' --destination-port-ranges '*'
```

**Why This Design**:
> - **Blast radius reduction**: Compromise of web tier limits attacker to that tier
> - **Audit trail**: All traffic between tiers must pass through NSG logging
> - **Lateral movement prevention**: Can't jump from web → data directly
> - **Defense in depth**: Even if one control fails, others remain

---

### Expert: Full Zero Trust with Conditional Access App Control

**Scenario**: Enterprise needs to protect sensitive data even from authorized users on managed devices.

**Challenge**: Authorized user copies sensitive file to personal OneDrive.

**Solution**: Conditional Access App Control (via Defender for Cloud Apps)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              ZERO TRUST WITH INLINE SESSION CONTROL                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  USER ACCESSES SHAREPOINT:                                                  │
│                                                                              │
│  ┌─────────────┐      ┌─────────────────────────────────────────────────┐   │
│  │    User     │─────▶│                 ENTRA ID                         │   │
│  │             │      │  1. Authenticate                                 │   │
│  └─────────────┘      │  2. Conditional Access: Session Control         │   │
│                       │     "Use Conditional Access App Control"        │   │
│                       └───────────────────────────┬─────────────────────┘   │
│                                                   │                          │
│                                                   ▼                          │
│                       ┌─────────────────────────────────────────────────┐   │
│                       │         DEFENDER FOR CLOUD APPS                  │   │
│                       │         (Reverse Proxy)                          │   │
│                       │                                                   │   │
│                       │  Session Policy: "Block download of             │   │
│                       │                   files labeled Confidential    │   │
│                       │                   from unmanaged devices"       │   │
│                       │                                                   │   │
│                       │  Actions:                                        │   │
│                       │  • Monitor all file access                       │   │
│                       │  • Block download of labeled files               │   │
│                       │  • Block copy/paste of sensitive content        │   │
│                       │  • Block print of sensitive documents           │   │
│                       │  • Watermark downloaded files                   │   │
│                       └───────────────────────────┬─────────────────────┘   │
│                                                   │                          │
│                                                   ▼                          │
│                       ┌─────────────────────────────────────────────────┐   │
│                       │               SHAREPOINT                         │   │
│                       │                                                   │   │
│                       │  File: Financial_Report.xlsx                    │   │
│                       │  Label: Confidential                            │   │
│                       │                                                   │   │
│                       │  User CAN: View in browser                      │   │
│                       │  User CANNOT: Download, copy content            │   │
│                       └─────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Implementation**:

1. **Conditional Access Policy**:
```
Name: Session Control for SharePoint

Users: All users
Apps: SharePoint Online
Conditions: Any

Session Controls:
  Use Conditional Access App Control:
    - Monitor only (initial)
    - Custom policy (after testing)
```

2. **Defender for Cloud Apps Session Policy**:
```
Policy Name: Block Confidential Downloads

Filters:
  Activity type: Download
  Sensitivity label: Confidential, Highly Confidential

Actions:
  Block download
  Notify user: "This file cannot be downloaded due to its classification"
  Alert: Send to security team
```

**Why This Design**:
> - **Data-centric**: Protection follows the data, not just the network
> - **Granular control**: Allow read but block download
> - **Label-based**: Automatically applies to any file with sensitivity label
> - **Audit**: All access logged regardless of action
> - **User education**: Message tells user why blocked

---

## AWS Comparison: Zero Trust Capabilities

| Capability | Azure Implementation | AWS Approach | Gap |
|------------|---------------------|--------------|-----|
| Identity verification | Entra ID + Conditional Access | IAM Identity Center | CA more granular |
| Device compliance | Intune + CA device conditions | Third-party MDM required | Native vs. integrated |
| Risk-based access | Identity Protection | Custom/third-party | No native equivalent |
| Session control | MCAS inline proxy | Custom WAF rules | Major gap |
| Microsegmentation | NSG + Private Endpoints | Security Groups + PrivateLink | Similar |
| SIEM/SOAR | Sentinel (native) | Third-party (Splunk, etc.) | Additional cost/complexity |
| Data classification | Purview (native) | Macie (limited) | Purview more comprehensive |

---

## Zero Trust Maturity Model

### Assessment Criteria

```
LEVEL 1: TRADITIONAL                     LEVEL 2: INITIAL ZERO TRUST
──────────────────────                   ──────────────────────────────
• Password-only authentication           • MFA enabled for admins
• Network perimeter focus               • Basic Conditional Access
• Flat internal network                 • Some network segmentation
• Manual access reviews                 • Defender for Cloud enabled
• Limited visibility                    • Basic logging to SIEM

LEVEL 3: ADVANCED ZERO TRUST            LEVEL 4: OPTIMAL ZERO TRUST
────────────────────────────            ───────────────────────────────
• MFA for all users                     • Passwordless authentication
• Risk-based Conditional Access        • Continuous access evaluation
• Microsegmentation                    • Full microsegmentation
• PIM for privileged access            • JIT/JEA everywhere
• Managed Identities for workloads     • Zero standing privileges
• Private Endpoints for PaaS           • Full data classification
• Automated threat response            • Automated remediation
```

### Self-Assessment Questions

| Area | Question | Level |
|------|----------|-------|
| Identity | Is MFA enforced for 100% of users? | 2+ |
| Identity | Are admin roles JIT with PIM? | 3+ |
| Identity | Is passwordless available? | 4 |
| Network | Are all PaaS services on Private Endpoints? | 3+ |
| Network | Is east-west traffic inspected? | 4 |
| Data | Is sensitive data classified and labeled? | 3+ |
| Data | Are DLP policies blocking exfiltration? | 4 |
| Workload | Do all Azure resources use Managed Identity? | 3+ |
| Workload | Is Defender for Cloud enabled (all plans)? | 2+ |

---

*Next: [Defender for Cloud](02-defender-for-cloud.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
