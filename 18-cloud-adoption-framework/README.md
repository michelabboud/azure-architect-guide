# Chapter 18: Cloud Adoption Framework & Landing Zones

## Overview

The Microsoft Cloud Adoption Framework (CAF) is a comprehensive methodology for cloud adoption that covers the entire lifecycle from initial strategy to ongoing management. For AWS architects transitioning to Azure, understanding CAF is essential as it provides the foundation for enterprise-scale Azure deployments.

## AWS Architect Quick Reference

| AWS Concept | Azure CAF Equivalent | Key Differences |
|-------------|---------------------|-----------------|
| AWS Well-Architected | Azure Well-Architected | Similar pillars, Azure includes sustainability |
| AWS Landing Zone | Azure Landing Zone | More prescriptive, includes governance |
| Control Tower | CAF + Landing Zone Accelerator | Comprehensive governance framework |
| AWS Organizations | Management Groups | Hierarchical structure for policies |
| Service Control Policies | Azure Policy | Policy inheritance and compliance |
| AWS Config | Azure Policy + Defender | Integrated compliance monitoring |

## Chapter Contents

### [01 - Cloud Adoption Framework Overview](01-caf-overview.md)
- Framework phases
- Methodology deep dive
- Migration strategies

### [02 - Azure Landing Zones](02-landing-zones.md)
- Landing zone architecture
- Design areas
- Implementation options

### [03 - Governance](03-governance.md)
- Policy management
- Cost management
- Security baseline

### [Quick Reference](quick-reference.md)
- Decision trees
- CLI commands
- Bicep templates

---

## Cloud Adoption Framework Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLOUD ADOPTION FRAMEWORK LIFECYCLE                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                           ┌───────────────┐                                 │
│                           │   STRATEGY    │                                 │
│                           │ Define motives│                                 │
│                           │ Business goals│                                 │
│                           └───────┬───────┘                                 │
│                                   │                                          │
│                                   ▼                                          │
│                           ┌───────────────┐                                 │
│                           │     PLAN      │                                 │
│                           │ Digital estate│                                 │
│                           │ Skills, team  │                                 │
│                           └───────┬───────┘                                 │
│                                   │                                          │
│                                   ▼                                          │
│                           ┌───────────────┐                                 │
│                           │    READY      │                                 │
│                           │ Landing zones │                                 │
│                           │ Foundation    │                                 │
│                           └───────┬───────┘                                 │
│                                   │                                          │
│                    ┌──────────────┼──────────────┐                          │
│                    ▼              ▼              ▼                          │
│             ┌───────────┐  ┌───────────┐  ┌───────────┐                     │
│             │  MIGRATE  │  │  INNOVATE │  │  SECURE   │                     │
│             │ Lift/shift│  │ New apps  │  │ Zero Trust│                     │
│             └─────┬─────┘  └─────┬─────┘  └─────┬─────┘                     │
│                   │              │              │                            │
│                   └──────────────┼──────────────┘                            │
│                                  │                                           │
│                    ┌─────────────┼─────────────┐                            │
│                    ▼             ▼             ▼                            │
│             ┌───────────┐ ┌───────────┐ ┌───────────┐                       │
│             │  GOVERN   │ │  MANAGE   │ │  ORGANIZE │                       │
│             │ Policies  │ │ Operations│ │   Teams   │                       │
│             └───────────┘ └───────────┘ └───────────┘                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## CAF Phases Explained

### 1. Strategy Phase

Define motivations and business outcomes for cloud adoption.

| Motivation | Description | Example Outcomes |
|------------|-------------|------------------|
| Migration | Move existing workloads | Reduced datacenter costs |
| Innovation | Build new cloud-native apps | Faster time to market |
| Optimization | Improve existing systems | Better performance, lower TCO |

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         STRATEGY CONSIDERATIONS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Business Drivers                     Technical Drivers                    │
│   ────────────────                     ─────────────────                    │
│   • Cost reduction                     • End of support (Windows 2012)      │
│   • Business agility                   • Datacenter exit                    │
│   • Market expansion                   • Security compliance                │
│   • M&A integration                    • Scalability requirements           │
│   • Competitive pressure               • Modernization needs                │
│                                                                              │
│   Key Questions:                                                            │
│   1. Why are we moving to the cloud?                                        │
│   2. What business outcomes do we expect?                                   │
│   3. How will we measure success?                                           │
│   4. What is our risk tolerance?                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2. Plan Phase

Create actionable cloud adoption plan.

**Digital Estate Assessment:**
- Inventory all workloads
- Classify by criticality
- Identify dependencies
- Estimate costs

**The 5 Rs of Migration:**

| Strategy | Description | When to Use |
|----------|-------------|-------------|
| Rehost | Lift and shift | Quick migration, minimal changes |
| Refactor | Minimal changes for PaaS | Optimize for cloud services |
| Rearchitect | Significant changes | Modernize architecture |
| Rebuild | Start from scratch | Replace with cloud-native |
| Replace | Use SaaS | When SaaS meets needs better |

### 3. Ready Phase

Prepare your Azure environment with Landing Zones.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AZURE LANDING ZONES                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Tenant Root Group                                 │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │              Platform (Central IT)                           │   │   │
│   │   │                                                              │   │   │
│   │   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │   │   │
│   │   │   │ Management  │  │ Connectivity│  │  Identity   │        │   │   │
│   │   │   │             │  │             │  │             │        │   │   │
│   │   │   │ • Log       │  │ • Hub VNet  │  │ • Entra ID  │        │   │   │
│   │   │   │   Analytics │  │ • ExpressR  │  │ • Domain    │        │   │   │
│   │   │   │ • Automation│  │ • Firewall  │  │   Services  │        │   │   │
│   │   │   │ • Monitor   │  │ • DNS       │  │ • PIM       │        │   │   │
│   │   │   └─────────────┘  └─────────────┘  └─────────────┘        │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │              Landing Zones (Workloads)                       │   │   │
│   │   │                                                              │   │   │
│   │   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │   │   │
│   │   │   │    Corp     │  │   Online    │  │  Sandbox    │        │   │   │
│   │   │   │             │  │             │  │             │        │   │   │
│   │   │   │ Internal    │  │ Internet-   │  │ Dev/Test    │        │   │   │
│   │   │   │ workloads   │  │ facing apps │  │ experiments │        │   │   │
│   │   │   └─────────────┘  └─────────────┘  └─────────────┘        │   │   │
│   │   │                                                              │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4. Migrate Phase

Execute migration using proven patterns.

**Migration Approach:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         MIGRATION WAVES                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Wave 0: Foundation                                                        │
│   ├── Landing zone deployment                                               │
│   ├── Network connectivity                                                  │
│   └── Identity integration                                                  │
│                                                                              │
│   Wave 1: Quick Wins                                                        │
│   ├── Simple VMs (no dependencies)                                          │
│   ├── Dev/test environments                                                 │
│   └── Non-critical workloads                                                │
│                                                                              │
│   Wave 2: Business Systems                                                  │
│   ├── Line-of-business apps                                                 │
│   ├── Databases (SQL migration)                                             │
│   └── File servers                                                          │
│                                                                              │
│   Wave 3: Complex/Critical                                                  │
│   ├── ERP systems                                                           │
│   ├── Legacy applications                                                   │
│   └── High-availability workloads                                           │
│                                                                              │
│   Wave 4: Optimization                                                      │
│   ├── Modernization projects                                                │
│   ├── Performance tuning                                                    │
│   └── Cost optimization                                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 5. Innovate Phase

Build cloud-native applications.

| Pattern | Azure Services | Use Case |
|---------|---------------|----------|
| Microservices | AKS, Container Apps | Scalable, independent services |
| Event-Driven | Event Grid, Functions | Real-time processing |
| AI/ML | AI Foundry, Cognitive Services | Intelligent applications |
| IoT | IoT Hub, Digital Twins | Connected devices |

### 6. Secure Phase

Implement Zero Trust security model.

**Zero Trust Principles:**
1. Verify explicitly
2. Use least privilege access
3. Assume breach

**Security Layers:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         DEFENSE IN DEPTH                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                        Identity                                      │   │
│   │   Entra ID • Conditional Access • PIM • MFA                         │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │                     Perimeter                                │   │   │
│   │   │   DDoS Protection • Azure Firewall • WAF                    │   │   │
│   │   │   ┌─────────────────────────────────────────────────────┐   │   │   │
│   │   │   │                    Network                           │   │   │   │
│   │   │   │   NSGs • Private Link • Service Endpoints            │   │   │   │
│   │   │   │   ┌─────────────────────────────────────────────┐   │   │   │   │
│   │   │   │   │              Compute                         │   │   │   │   │
│   │   │   │   │   VMs • Containers • Functions               │   │   │   │   │
│   │   │   │   │   ┌─────────────────────────────────────┐   │   │   │   │   │
│   │   │   │   │   │          Application                │   │   │   │   │   │
│   │   │   │   │   │   Code • Dependencies • Config     │   │   │   │   │   │
│   │   │   │   │   │   ┌─────────────────────────────┐   │   │   │   │   │   │
│   │   │   │   │   │   │         Data                │   │   │   │   │   │   │
│   │   │   │   │   │   │  Encryption • Classification│   │   │   │   │   │   │
│   │   │   │   │   │   └─────────────────────────────┘   │   │   │   │   │   │
│   │   │   │   │   └─────────────────────────────────────┘   │   │   │   │   │
│   │   │   │   └─────────────────────────────────────────────┘   │   │   │   │
│   │   │   └─────────────────────────────────────────────────────┘   │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 7. Govern Phase

Establish cloud governance.

**Governance Disciplines:**

| Discipline | Purpose | Azure Tools |
|------------|---------|-------------|
| Cost Management | Control spending | Cost Management, Budgets |
| Security Baseline | Enforce security | Azure Policy, Defender |
| Identity Baseline | Manage access | Entra ID, RBAC |
| Resource Consistency | Standardize resources | Azure Policy, Blueprints |
| Deployment Acceleration | Automate deployments | ARM, Bicep, DevOps |

### 8. Manage Phase

Operate cloud workloads.

**Operations Baseline:**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         OPERATIONS MANAGEMENT                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Inventory & Visibility                                                    │
│   ├── Azure Resource Graph                                                  │
│   ├── Service Health                                                        │
│   └── Azure Monitor                                                         │
│                                                                              │
│   Operational Compliance                                                    │
│   ├── Update Management                                                     │
│   ├── Azure Automation                                                      │
│   └── Azure Policy                                                          │
│                                                                              │
│   Protect & Recover                                                         │
│   ├── Azure Backup                                                          │
│   ├── Azure Site Recovery                                                   │
│   └── Disaster Recovery                                                     │
│                                                                              │
│   Platform Operations                                                       │
│   ├── Azure Lighthouse                                                      │
│   ├── Azure Arc                                                             │
│   └── Cost Management                                                       │
│                                                                              │
│   Workload Operations                                                       │
│   ├── Application Insights                                                  │
│   ├── Log Analytics                                                         │
│   └── Workbooks & Dashboards                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Landing Zone Design Areas

Azure Landing Zones have 8 design areas:

| Design Area | Description | Key Decisions |
|-------------|-------------|---------------|
| Billing & Enrollment | EA/CSP setup | Billing accounts, departments |
| Identity | Authentication | Entra ID, hybrid identity |
| Network Topology | Connectivity | Hub-spoke, Virtual WAN |
| Resource Organization | Structure | Management groups, subscriptions |
| Governance | Policies | Azure Policy, naming, tagging |
| Security | Protection | Defender, Sentinel, encryption |
| Management | Operations | Monitor, backup, patching |
| Platform Automation | DevOps | IaC, CI/CD, GitOps |

## Landing Zone Implementation Options

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LANDING ZONE IMPLEMENTATION OPTIONS                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Option 1: Start Small                                                     │
│   ─────────────────────                                                     │
│   • Single subscription                                                     │
│   • Basic networking                                                        │
│   • Manual governance                                                       │
│   • Best for: Small teams, POCs                                            │
│                                                                              │
│   Option 2: Enterprise-Scale (CAF Foundation)                               │
│   ───────────────────────────────────────────                               │
│   • Management group hierarchy                                              │
│   • Hub-spoke networking                                                    │
│   • Azure Policy guardrails                                                 │
│   • Best for: Growing organizations                                         │
│                                                                              │
│   Option 3: Azure Landing Zone Accelerator                                  │
│   ───────────────────────────────────────────                               │
│   • Full reference architecture                                             │
│   • Automated deployment                                                    │
│   • Enterprise governance                                                   │
│   • Best for: Large enterprises                                             │
│                                                                              │
│   AWS Comparison:                                                           │
│   • Option 1 ≈ AWS Account + basic VPC                                     │
│   • Option 2 ≈ AWS Organizations + Control Tower basics                    │
│   • Option 3 ≈ Full Control Tower + Landing Zone                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Learning Objectives

After completing this chapter, you will:

1. **Understand** the Cloud Adoption Framework phases
2. **Design** Azure Landing Zones for enterprise deployments
3. **Implement** governance using Azure Policy
4. **Plan** migration strategies using the 5 Rs
5. **Apply** Zero Trust security principles
6. **Manage** cloud operations at scale

---

*Continue to [CAF Overview](01-caf-overview.md)*

*Back to [Main Guide](../README.md)*
