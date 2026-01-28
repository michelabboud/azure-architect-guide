# Chapter 02: Security - Microsoft Defender & Zero Trust

## Why Security is Second?

Security builds on the identity foundation. Without solid identity (Chapter 01), security controls lack a foundation to attach to. The Azure security model is "identity-first" but needs the security layer to be complete.

```
Chapter 01: Identity                    Chapter 02: Security
─────────────────────                   ────────────────────

"Who are you?"                          "What threats exist?"
"Should you be here?"                   "How do we protect?"

┌─────────────────────┐                 ┌─────────────────────┐
│    Entra ID         │                 │    Defender for     │
│    Conditional      │───────────────→│    Cloud (CNAPP)    │
│    Access           │                 │                     │
└─────────────────────┘                 └─────────────────────┘
                                                  │
                                                  ▼
                                        ┌─────────────────────┐
                                        │   Zero Trust        │
                                        │   Network Security  │
                                        │   Key Vault         │
                                        └─────────────────────┘
```

## Chapter Contents

| Document | Purpose | Time |
|----------|---------|------|
| [Quick Reference](quick-reference.md) | Security services cheat sheet | 5 min |
| [01 - Zero Trust](01-zero-trust.md) | Zero Trust architecture principles | 30 min |
| [02 - Defender for Cloud](02-defender-for-cloud.md) | CNAPP deep dive | 30 min |
| [03 - Network Security](03-network-security.md) | Network security patterns | 30 min |
| [Case Studies](case-studies.md) | Security architecture scenarios | 20 min |

## AWS to Azure Security Mapping

| AWS Service | Azure Equivalent | Key Difference |
|-------------|------------------|----------------|
| Security Hub | Defender for Cloud | Defender is unified CNAPP |
| GuardDuty | Defender threat detection | Part of unified platform |
| Inspector | Defender vulnerability assessment | Integrated, not separate |
| WAF | Azure WAF (+ Front Door) | Similar capabilities |
| Shield | DDoS Protection | Standard vs Advanced tiers |
| KMS | Key Vault | Key Vault adds secrets + certs |
| Secrets Manager | Key Vault | Combined in Key Vault |
| Macie | Purview (Chapter 03) | Broader scope in Purview |
| Config | Azure Policy | More effects in Policy |
| CloudTrail | Activity Log | Split: resource vs identity logs |

## The Microsoft Defender Family

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     MICROSOFT DEFENDER ECOSYSTEM                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                    DEFENDER FOR CLOUD (CNAPP)                           ││
│  │    Cloud Security Posture Management (CSPM) + Cloud Workload Protection ││
│  │                                                                          ││
│  │    Coverage: Azure + AWS + GCP + On-premises                            ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│  WORKLOAD-SPECIFIC DEFENDERS:                                               │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐               │
│  │ Defender for    │ │ Defender for    │ │ Defender for    │               │
│  │ Servers         │ │ Containers      │ │ App Service     │               │
│  │ (VMs)           │ │ (AKS, K8s)      │ │ (Web apps)      │               │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘               │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐               │
│  │ Defender for    │ │ Defender for    │ │ Defender for    │               │
│  │ Storage         │ │ SQL             │ │ Key Vault       │               │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘               │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐               │
│  │ Defender for    │ │ Defender for    │ │ Defender for    │               │
│  │ DNS             │ │ ARM             │ │ APIs            │               │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘               │
│                                                                              │
│  IDENTITY + ENDPOINT:                                                       │
│  ┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐               │
│  │ Defender for    │ │ Defender for    │ │ Defender for    │               │
│  │ Identity        │ │ Endpoint        │ │ Cloud Apps      │               │
│  │ (AD threats)    │ │ (EDR)           │ │ (CASB)          │               │
│  └─────────────────┘ └─────────────────┘ └─────────────────┘               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Zero Trust Principles

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        ZERO TRUST PILLARS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. VERIFY EXPLICITLY                                                       │
│      Always authenticate and authorize based on all available data          │
│      • User identity (Entra ID)                                             │
│      • Device health (Intune)                                               │
│      • Location (Named locations)                                           │
│      • Service/workload (Managed Identity)                                  │
│                                                                              │
│   2. USE LEAST PRIVILEGE ACCESS                                             │
│      Limit user access with Just-In-Time and Just-Enough-Access            │
│      • PIM for admin access                                                 │
│      • RBAC with minimal scopes                                             │
│      • Access reviews                                                        │
│      • Time-limited access                                                  │
│                                                                              │
│   3. ASSUME BREACH                                                           │
│      Minimize blast radius and segment access                               │
│      • Network microsegmentation                                            │
│      • End-to-end encryption                                                │
│      • Continuous monitoring                                                │
│      • Automated threat detection                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘

Applied to Azure:
─────────────────

     ┌────────────────────────────────────────────────────────────────────┐
     │                         ZERO TRUST LAYERS                          │
     └────────────────────────────────────────────────────────────────────┘
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         │                            │                            │
         ▼                            ▼                            ▼
┌─────────────────┐         ┌─────────────────┐         ┌─────────────────┐
│    IDENTITY     │         │     NETWORK     │         │      DATA       │
│                 │         │                 │         │                 │
│ • Entra ID      │         │ • NSGs          │         │ • Purview DLP   │
│ • Conditional   │         │ • Private Link  │         │ • Key Vault     │
│   Access        │         │ • Firewall      │         │ • Encryption    │
│ • MFA           │         │ • WAF           │         │ • Labels        │
│ • PIM           │         │ • DDoS          │         │                 │
└─────────────────┘         └─────────────────┘         └─────────────────┘
         │                            │                            │
         └────────────────────────────┼────────────────────────────┘
                                      │
                                      ▼
                          ┌─────────────────────┐
                          │    VISIBILITY &     │
                          │    ANALYTICS        │
                          │                     │
                          │ • Defender for Cloud│
                          │ • Sentinel          │
                          │ • Azure Monitor     │
                          └─────────────────────┘
```

## Key Skills Covered

This chapter addresses these essential Cloud Architect skills:

✅ **Zero Trust + Governance**
- Design and implement Zero Trust architecture
- Network microsegmentation
- Private endpoints and secure connectivity

✅ **Security + Compliance (Defender/Purview)**
- Configure Defender for Cloud across workloads
- Set up security alerts and automation
- Integrate with SIEM (Sentinel)

## Learning Objectives

After completing this chapter:

1. **Design** Zero Trust architecture for Azure environments
2. **Configure** Microsoft Defender for Cloud (all tiers)
3. **Implement** network security patterns (hub-spoke, Private Link)
4. **Compare** AWS security tools to Azure equivalents
5. **Build** security monitoring and alerting
6. **Explain** security architectural decisions in interviews

---

*Start with: [Quick Reference](quick-reference.md)* | *Or dive into: [Zero Trust](01-zero-trust.md)*
