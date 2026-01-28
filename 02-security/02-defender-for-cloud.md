# Microsoft Defender for Cloud Deep Dive

## What is Defender for Cloud?

Microsoft Defender for Cloud is a **Cloud-Native Application Protection Platform (CNAPP)** that combines:
- **CSPM** (Cloud Security Posture Management): Find and fix misconfigurations
- **CWPP** (Cloud Workload Protection Platform): Protect running workloads

### AWS Equivalent Mapping

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DEFENDER FOR CLOUD vs AWS SECURITY                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS (Multiple Services):                Azure (Unified):                   │
│  ─────────────────────────               ─────────────────                  │
│                                                                              │
│  ┌─────────────────────┐                ┌─────────────────────────────────┐ │
│  │   Security Hub     │                │                                  │ │
│  │   (Aggregation)    │                │     DEFENDER FOR CLOUD           │ │
│  └─────────────────────┘                │                                  │ │
│  ┌─────────────────────┐                │  ┌─────────────────────────┐    │ │
│  │   GuardDuty        │                │  │  Threat Detection       │    │ │
│  │   (Threat Detection│                │  │  (All workloads)        │    │ │
│  └─────────────────────┘                │  └─────────────────────────┘    │ │
│  ┌─────────────────────┐                │  ┌─────────────────────────┐    │ │
│  │   Inspector        │                │  │  Vulnerability          │    │ │
│  │   (Vulnerability)  │                │  │  Assessment             │    │ │
│  └─────────────────────┘                │  └─────────────────────────┘    │ │
│  ┌─────────────────────┐                │  ┌─────────────────────────┐    │ │
│  │   Config Rules     │                │  │  Secure Score &         │    │ │
│  │   (Compliance)     │                │  │  Recommendations        │    │ │
│  └─────────────────────┘                │  └─────────────────────────┘    │ │
│  ┌─────────────────────┐                │  ┌─────────────────────────┐    │ │
│  │   Trusted Advisor  │                │  │  Regulatory             │    │ │
│  │   (Best Practices) │                │  │  Compliance             │    │ │
│  └─────────────────────┘                └──┴─────────────────────────┴────┘ │
│                                                                              │
│  Integration: Manual/Custom                Integration: Native, unified     │
│  Learning curve: High (many consoles)     Learning curve: Medium (one)     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## CSPM: Free vs Defender Tiers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CSPM TIER COMPARISON                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  FREE CSPM (Foundational)              DEFENDER CSPM (Enhanced)             │
│  ─────────────────────────             ────────────────────────             │
│                                                                              │
│  ✓ Secure Score                        ✓ Everything in Free                 │
│  ✓ Basic recommendations               ✓ Attack path analysis               │
│  ✓ Azure Security Benchmark            ✓ Cloud security explorer           │
│  ✓ Asset inventory                     ✓ Agentless scanning                │
│  ✓ Multi-cloud (basic)                 ✓ Governance rules                  │
│                                        ✓ Risk prioritization               │
│  Cost: Free                            ✓ Data-aware security posture       │
│                                        ✓ External attack surface           │
│                                                                              │
│                                        Cost: ~$5/server/month               │
│                                              (varies by workload)           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Workload Protection Plans

### Enable Defender Plans

```bash
# Enable all Defender plans (subscription scope)
az security pricing create --name VirtualMachines --tier Standard
az security pricing create --name StorageAccounts --tier Standard
az security pricing create --name SqlServers --tier Standard
az security pricing create --name SqlServerVirtualMachines --tier Standard
az security pricing create --name AppServices --tier Standard
az security pricing create --name Containers --tier Standard
az security pricing create --name KeyVaults --tier Standard
az security pricing create --name Dns --tier Standard
az security pricing create --name Arm --tier Standard
az security pricing create --name OpenSourceRelationalDatabases --tier Standard
az security pricing create --name CosmosDbs --tier Standard
az security pricing create --name Api --tier Standard

# Check status
az security pricing list --output table
```

### What Each Plan Protects

| Plan | Protects | Key Capabilities | Monthly Cost |
|------|----------|------------------|--------------|
| **Servers** | VMs, Arc-enabled | EDR (Defender for Endpoint), vulnerability scanning, file integrity | ~$15/server |
| **Containers** | AKS, K8s, registries | Runtime protection, image scanning, K8s alerts | ~$7/vCore |
| **App Service** | Web apps | Threat detection, anomaly detection | ~$15/app |
| **Storage** | Blob, Files, Data Lake | Malware scanning, anomalous access | ~$10/storage account |
| **SQL** | Azure SQL, SQL MI | SQL threat detection, vulnerability assessment | ~$15/instance |
| **Key Vault** | Key Vault | Anomalous access, unusual patterns | ~$0.02/10K operations |
| **DNS** | DNS queries | Domain reputation, DGA detection | ~$0.70/million queries |
| **ARM** | Control plane | Suspicious operations, risky resource configs | ~$4/subscription |
| **APIs** | API Management | API-specific threats, anomalous patterns | Varies |

---

## Case Study: Implementing Defender for a Multi-Cloud Environment

### Scenario

**Company**: Retail company with workloads in Azure + AWS
**Goal**: Unified security visibility and protection across both clouds

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│              DEFENDER FOR CLOUD: MULTI-CLOUD ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌──────────────────────────────────┐                     │
│                    │   DEFENDER FOR CLOUD             │                     │
│                    │   (Azure Portal - Single Pane)   │                     │
│                    └───────────────┬──────────────────┘                     │
│                                    │                                         │
│                    ┌───────────────┼───────────────┐                        │
│                    │               │               │                        │
│                    ▼               ▼               ▼                        │
│          ┌─────────────────┐ ┌─────────────┐ ┌─────────────────┐           │
│          │     AZURE       │ │    AWS      │ │     GCP         │           │
│          │                 │ │             │ │                 │           │
│          │ Native          │ │ Connector   │ │ Connector       │           │
│          │ Integration     │ │ via Arc     │ │ via Arc         │           │
│          │                 │ │             │ │                 │           │
│          │ • VMs           │ │ • EC2       │ │ • Compute       │           │
│          │ • AKS           │ │ • EKS       │ │ • GKE           │           │
│          │ • SQL           │ │ • RDS       │ │ • Cloud SQL     │           │
│          │ • Storage       │ │ • S3        │ │ • GCS           │           │
│          └─────────────────┘ └─────────────┘ └─────────────────┘           │
│                                                                              │
│  UNIFIED CAPABILITIES:                                                      │
│  • Single Secure Score across all clouds                                    │
│  • Consistent recommendations                                               │
│  • Unified alerts and incidents                                             │
│  • Cross-cloud attack path analysis                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation: Connect AWS to Defender for Cloud

```
Step 1: Create AWS Connector in Azure Portal
─────────────────────────────────────────────
Defender for Cloud → Environment Settings → Add Environment → AWS

Step 2: Configure AWS Account
─────────────────────────────
• Provide AWS Account ID
• Choose onboarding type:
  - Single account
  - Management account (all accounts in org)

Step 3: Deploy CloudFormation Stack
───────────────────────────────────
Defender for Cloud provides a CloudFormation template that creates:
• IAM Role for Defender to assume
• Required permissions for scanning
• EventBridge rules for real-time alerts

Step 4: Select Plans
────────────────────
• CSPM (posture assessment)
• Defender for Servers (EC2 instances)
• Defender for Containers (EKS)
• Defender for SQL (RDS)
```

### Results After 30 Days

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MULTI-CLOUD SECURITY DASHBOARD                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SECURE SCORE:                                                              │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ Overall: 72%                                                        │    │
│  │ ████████████████████████████████░░░░░░░░░░░░                        │    │
│  ├────────────────────────────────────────────────────────────────────┤    │
│  │ Azure:  78%  ██████████████████████████████████░░░░░░░             │    │
│  │ AWS:    65%  ██████████████████████████░░░░░░░░░░░░░░░             │    │
│  │ GCP:    71%  █████████████████████████████░░░░░░░░░░░░             │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  TOP RECOMMENDATIONS:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ 1. [AWS] Enable MFA for root account          Impact: +5%    High  │   │
│  │ 2. [Azure] Enable encryption for SQL DBs      Impact: +3%    High  │   │
│  │ 3. [AWS] Restrict S3 bucket public access     Impact: +4%    High  │   │
│  │ 4. [GCP] Enable VPC flow logs                 Impact: +2%    Med   │   │
│  │ 5. [Azure] Enable Defender for Storage        Impact: +3%    Med   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ACTIVE ALERTS (Last 7 Days):                                              │
│  Azure: 12 (3 High, 5 Medium, 4 Low)                                       │
│  AWS:   8  (2 High, 4 Medium, 2 Low)                                       │
│  GCP:   3  (0 High, 2 Medium, 1 Low)                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Attack Path Analysis (Defender CSPM)

### What It Does

Attack path analysis identifies potential paths an attacker could take to compromise critical assets.

```
EXAMPLE ATTACK PATH:
────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│  INTERNET                                                                    │
│      │                                                                       │
│      │ 1. Internet-facing VM with public IP                                 │
│      ▼    (HIGH RISK: exposed to internet)                                  │
│  ┌───────────────────────────────────────────────────────────┐              │
│  │  VM: web-server-01                                        │              │
│  │  Risk: Has vulnerabilities (CVE-2023-xxxx)                │              │
│  │  Risk: Running as admin                                   │              │
│  └───────────────────────┬───────────────────────────────────┘              │
│                          │                                                   │
│                          │ 2. VM has access to Key Vault                    │
│                          │    (via overly permissive RBAC)                  │
│                          ▼                                                   │
│  ┌───────────────────────────────────────────────────────────┐              │
│  │  Key Vault: kv-secrets-prod                               │              │
│  │  Contains: Database connection strings                    │              │
│  │  Contains: API keys for payment gateway                   │              │
│  └───────────────────────┬───────────────────────────────────┘              │
│                          │                                                   │
│                          │ 3. Connection string provides access to          │
│                          │    production database                           │
│                          ▼                                                   │
│  ┌───────────────────────────────────────────────────────────┐              │
│  │  SQL Database: sqldb-customers-prod                       │              │
│  │  Contains: 500K customer records (PII)                    │              │
│  │  CRITICAL ASSET                                           │              │
│  └───────────────────────────────────────────────────────────┘              │
│                                                                              │
│  RECOMMENDATION: Fix vulnerabilities on VM, restrict Key Vault access,     │
│                  remove public IP, use Private Endpoint                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Implementing Defender: Beginner → Expert

### Beginner: Enable Free CSPM

```bash
# Verify Defender status
az security pricing list --output table

# The free tier is automatically enabled
# Just ensure you review the recommendations
az security assessment list --query "[?status.code=='Unhealthy']" --output table
```

### Intermediate: Enable Workload Protection

```bash
# Enable Defender for Servers
az security pricing create \
  --name VirtualMachines \
  --tier Standard

# Enable auto-provisioning of Defender for Endpoint
az security auto-provisioning-setting update \
  --name default \
  --auto-provision on

# Enable vulnerability assessment
az security setting update \
  --name MCAS \
  --setting-name DefenderForServersVulnerabilityAssessmentProvider \
  --setting-value MdeTvm
```

### Expert: Full CSPM with Governance

```bash
# Enable Defender CSPM
az security pricing create \
  --name CloudPosture \
  --tier Standard \
  --extensions '[{"name":"SensitiveDataDiscovery","isEnabled":"true"},
                 {"name":"AgentlessVmScanning","isEnabled":"true"},
                 {"name":"AgentlessDiscoveryForKubernetes","isEnabled":"true"}]'

# Create governance rule
# (Typically done via Portal or ARM template)
# Governance rule: Auto-assign owners to recommendations
# Governance rule: Set due dates based on severity
```

---

## KQL Queries for Defender Analysis

```kusto
// High severity alerts in last 7 days
SecurityAlert
| where TimeGenerated > ago(7d)
| where AlertSeverity == "High"
| project TimeGenerated, AlertName, Description, RemediationSteps
| order by TimeGenerated desc

// Recommendations by resource type
SecurityRecommendation
| summarize Count = count() by RecommendationName, ResourceType
| order by Count desc
| take 20

// Secure Score trend
SecureScoreControls
| summarize Score = avg(CurrentScore) by bin(TimeGenerated, 1d)
| render timechart

// Attack paths to critical assets
SecurityResources
| where type == "microsoft.security/attackpaths"
| extend path = parse_json(properties)
| project PathName = path.attackPathDisplayName,
          RiskLevel = path.riskLevel,
          EntryPoint = path.entryPointResourceId
```

---

*Next: [Network Security](03-network-security.md)* | *Back to [Quick Reference](quick-reference.md)*
