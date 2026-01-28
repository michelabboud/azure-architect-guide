# Chapter 01: Identity - The Security Perimeter

## Why Identity First?

In Azure (and modern cloud architecture), **identity is the new perimeter**. Unlike traditional network-based security, Azure's security model starts with "Who are you?" before "What are you trying to access?"

```
Traditional Security:                  Azure Zero Trust:
──────────────────────                 ─────────────────

    ┌─────────────┐                         ┌─────────────┐
    │  Firewall   │                         │  Identity   │
    └──────┬──────┘                         │  (Entra ID) │
           │                                └──────┬──────┘
    ┌──────┴──────┐                                │
    │   Network   │                         ┌──────┴──────┐
    │   (Trusted) │                         │ Conditional │
    └──────┬──────┘                         │   Access    │
           │                                └──────┬──────┘
    ┌──────┴──────┐                                │
    │  Resources  │                         ┌──────┴──────┐
    └─────────────┘                         │    RBAC     │
                                            └──────┬──────┘
    "Inside = Trusted"                             │
                                            ┌──────┴──────┐
                                            │  Resources  │
                                            └─────────────┘

                                            "Never Trust, Always Verify"
```

## Chapter Contents

| Document | Purpose | Time |
|----------|---------|------|
| [Quick Reference](quick-reference.md) | Cheat sheet for identity concepts | 5 min |
| [01 - Entra ID](01-entra-id.md) | Deep dive into Microsoft Entra ID | 30 min |
| [02 - Conditional Access](02-conditional-access.md) | Policy engine for access control | 30 min |
| [03 - Governance](03-governance.md) | Lifecycle, access reviews, entitlements | 30 min |
| [Case Studies](case-studies.md) | Real-world scenarios with decisions | 20 min |

## Key Concepts Overview

### Microsoft Entra ID (formerly Azure AD)

The cloud identity and access management service. Not just a directory—it's the foundation of Azure security.

```
Entra ID Components:
┌─────────────────────────────────────────────────────────────────────┐
│                        ENTRA ID TENANT                               │
├─────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │   Users     │  │   Groups    │  │    Apps     │  │  Devices   │ │
│  │             │  │             │  │             │  │            │ │
│  │ • Internal  │  │ • Security  │  │ • Enterprise│  │ • Managed  │ │
│  │ • Guest     │  │ • M365      │  │ • App Reg   │  │ • BYOD     │ │
│  │ • Service   │  │ • Dynamic   │  │ • Service   │  │ • Compliant│ │
│  │   Principals│  │             │  │   Principals│  │            │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └────────────┘ │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    CONDITIONAL ACCESS                        │    │
│  │  Policies that determine IF authentication should succeed    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                       GOVERNANCE                             │    │
│  │  • Access Reviews  • Entitlement Management  • Lifecycle     │    │
│  └─────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘
```

### Conditional Access

The "if-then" policy engine that sits between authentication and access:

```
IF:                              THEN:
───                              ─────
• User/Group                     • Grant access
• Cloud app                        - Require MFA
• Device platform                  - Require compliant device
• Location                         - Require app protection
• Risk level                       - Require password change
• Client app                     • Block access
• Device state                   • Session controls
                                   - App enforced restrictions
                                   - Sign-in frequency
                                   - Persistent browser
```

### Governance

Managing the identity lifecycle:

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Joiner    │ →  │    Mover    │ →  │   Leaver    │ →  │   Review    │
├─────────────┤    ├─────────────┤    ├─────────────┤    ├─────────────┤
│ • Provision │    │ • Update    │    │ • Deprovision│   │ • Access    │
│   accounts  │    │   access    │    │   accounts  │    │   reviews   │
│ • Assign    │    │ • Role      │    │ • Revoke    │    │ • Recertify │
│   initial   │    │   changes   │    │   access    │    │ • Remove    │
│   access    │    │             │    │             │    │   unused    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

## AWS to Azure Identity Mapping

| AWS Concept | Azure Equivalent | Key Difference |
|-------------|------------------|----------------|
| IAM Users | Entra ID Users | Full identity platform vs permission system |
| IAM Groups | Entra ID Groups | Groups can be dynamic based on attributes |
| IAM Roles | Service Principals + Managed Identities | Separate concepts in Azure |
| IAM Policies | RBAC + Conditional Access | Split: RBAC for permissions, CA for access |
| AWS SSO | Entra ID | Native SSO in Entra |
| Cognito | Entra ID B2C | Customer identity management |
| STS AssumeRole | Managed Identity | No credential management needed |
| Resource Policies | Conditional Access + RBAC | More granular in Azure |

## Job Relevance: NICE Cloud Architect

This chapter directly addresses these job requirements:

✅ **Identity-first security (Entra ID + Conditional Access)**
- Design enterprise identity architecture
- Implement Zero Trust with Conditional Access
- Configure MFA and passwordless authentication

✅ **Zero Trust + Governance**
- Least privilege access with access reviews
- Entitlement management for access packages
- Lifecycle management automation

## Learning Objectives

After completing this chapter, you will be able to:

1. **Design** enterprise identity architecture using Entra ID
2. **Implement** Conditional Access policies for Zero Trust
3. **Configure** governance controls (access reviews, entitlements)
4. **Compare** AWS IAM patterns to Azure identity patterns
5. **Troubleshoot** common identity and access issues
6. **Explain** architectural decisions in interviews

## Prerequisites

- Understanding of AWS IAM (users, groups, roles, policies)
- Basic understanding of authentication protocols (OAuth, SAML, OIDC)
- Familiarity with MFA concepts

---

*Start with: [Quick Reference](quick-reference.md)* | *Or dive into: [Entra ID](01-entra-id.md)*
