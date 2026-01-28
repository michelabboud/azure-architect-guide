# Chapter 11: Network Security

## Overview

This chapter covers Azure network security services including firewalls, network security groups, DDoS protection, and security best practices. As an AWS architect, you'll find many familiar concepts with Azure-specific implementations.

## AWS Architect Quick Reference

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| Security Groups | Network Security Groups (NSG) | Can attach to subnet OR NIC |
| NACLs | NSG (subnet-level) | Stateful (unlike NACLs) |
| AWS WAF | Azure WAF | Integrated with App Gateway/Front Door |
| AWS Network Firewall | Azure Firewall | Layer 3-7 with IDPS |
| AWS Shield | Azure DDoS Protection | Standard tier for SLA |
| AWS PrivateLink | Azure Private Link | Similar private connectivity |
| VPC Flow Logs | NSG Flow Logs | Stored in Storage Account |

## Chapter Contents

### [Quick Reference](quick-reference.md)
- CLI/Portal commands
- Common configurations
- Troubleshooting commands

### [01 - Network Security Groups (NSG)](01-network-security-groups.md)
- NSG fundamentals
- Rule configuration
- Application Security Groups
- Best practices

### [02 - Azure Firewall](02-azure-firewall.md)
- Firewall tiers (Basic, Standard, Premium)
- Rule configuration
- IDPS and TLS inspection
- Firewall policies

### [03 - DDoS Protection](03-ddos-protection.md)
- Protection tiers
- DDoS policies
- Attack analytics
- Response planning

### [04 - Private Endpoints & Private Link](04-private-link.md)
- Service endpoints vs private endpoints
- Private Link service
- DNS configuration
- Hybrid scenarios

### [Case Studies](case-studies.md)
- Enterprise perimeter security
- Zero-trust implementation
- Hybrid network security

## Architecture Patterns

### Defense in Depth

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Defense in Depth Layers                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Layer 1: DDoS Protection                                            │    │
│  │  ├── Azure DDoS Protection Standard                                 │    │
│  │  └── Absorbs volumetric attacks at network edge                     │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                     │                                        │
│  ┌─────────────────────────────────▼───────────────────────────────────┐    │
│  │  Layer 2: Edge Security                                              │    │
│  │  ├── Azure Front Door WAF                                           │    │
│  │  ├── Bot protection, geo-filtering                                   │    │
│  │  └── SSL offloading, rate limiting                                   │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                     │                                        │
│  ┌─────────────────────────────────▼───────────────────────────────────┐    │
│  │  Layer 3: Network Perimeter                                          │    │
│  │  ├── Azure Firewall (hub)                                           │    │
│  │  ├── Application rules, FQDN filtering                               │    │
│  │  └── Threat intelligence, IDPS                                       │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                     │                                        │
│  ┌─────────────────────────────────▼───────────────────────────────────┐    │
│  │  Layer 4: Network Segmentation                                       │    │
│  │  ├── Network Security Groups                                         │    │
│  │  ├── Application Security Groups                                     │    │
│  │  └── Subnet isolation                                                │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                     │                                        │
│  ┌─────────────────────────────────▼───────────────────────────────────┐    │
│  │  Layer 5: Host Security                                              │    │
│  │  ├── Just-in-Time VM Access                                         │    │
│  │  ├── Endpoint protection                                             │    │
│  │  └── OS hardening                                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                     │                                        │
│  ┌─────────────────────────────────▼───────────────────────────────────┐    │
│  │  Layer 6: Application Security                                       │    │
│  │  ├── Application Gateway WAF                                         │    │
│  │  ├── OWASP rule sets                                                 │    │
│  │  └── Custom rules                                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                     │                                        │
│  ┌─────────────────────────────────▼───────────────────────────────────┐    │
│  │  Layer 7: Data Security                                              │    │
│  │  ├── Private endpoints                                               │    │
│  │  ├── Encryption at rest/in transit                                   │    │
│  │  └── Key Vault integration                                           │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hub-Spoke Security Architecture

```
                                    Internet
                                        │
                                        ▼
                              ┌─────────────────┐
                              │  Azure Front    │
                              │  Door (WAF)     │
                              └────────┬────────┘
                                       │
        ┌──────────────────────────────┼──────────────────────────────┐
        │                              │                              │
        │                    ┌─────────▼─────────┐                    │
        │                    │      Hub VNet     │                    │
        │                    │                   │                    │
        │    ┌───────────────┼───────────────┐   │                    │
        │    │               │               │   │                    │
        │    ▼               ▼               ▼   │                    │
        │ ┌──────┐     ┌──────────┐    ┌──────┐ │                    │
        │ │Azure │     │  Azure   │    │Azure │ │                    │
        │ │Bastion│    │ Firewall │    │ VPN  │ │                    │
        │ └──────┘     └────┬─────┘    │Gateway│ │                    │
        │                   │          └───┬──┘ │                    │
        │                   │              │    │                    │
        └───────────────────┼──────────────┼────┘                    │
                            │              │                          │
           ┌────────────────┼──────────────┼────────────────┐        │
           │                │              │                │        │
    ┌──────▼──────┐  ┌──────▼──────┐  ┌───▼────────┐       │        │
    │  Spoke 1    │  │  Spoke 2    │  │   On-Prem  │       │        │
    │  (Web)      │  │  (App)      │  │   Network  │       │        │
    │             │  │             │  │            │       │        │
    │ ┌─────────┐ │  │ ┌─────────┐ │  └────────────┘       │        │
    │ │  NSG    │ │  │ │  NSG    │ │                       │        │
    │ │ (Web)   │ │  │ │ (App)   │ │                       │        │
    │ └─────────┘ │  │ └─────────┘ │                       │        │
    │             │  │             │                       │        │
    │ ┌─────────┐ │  │ ┌─────────┐ │       ┌──────────────▼──────┐  │
    │ │ App GW  │ │  │ │Internal │ │       │      Spoke 3        │  │
    │ │ (WAF)   │ │  │ │   LB    │ │       │      (Data)         │  │
    │ └─────────┘ │  │ └─────────┘ │       │                     │  │
    └─────────────┘  └─────────────┘       │  ┌───────────────┐  │  │
                                           │  │Private        │  │  │
                                           │  │Endpoints      │  │  │
                                           │  └───────────────┘  │  │
                                           └─────────────────────┘  │
```

## Service Comparison Matrix

| Feature | NSG | Azure Firewall | WAF | DDoS Protection |
|---------|-----|----------------|-----|-----------------|
| Layer | 3-4 | 3-7 | 7 | 3-4 |
| Stateful | Yes | Yes | N/A | N/A |
| FQDN Filtering | No | Yes | Yes | No |
| TLS Inspection | No | Premium only | No | No |
| Threat Intel | No | Yes | Yes | No |
| Cost | Free | $$$ | $$ | $$$ |
| Scale | Subnet/NIC | Central | Per App | Per VNet |

## Learning Objectives

After completing this chapter, you will:

1. **Design** comprehensive network security architecture
2. **Configure** NSGs, ASGs, and Azure Firewall
3. **Implement** DDoS protection strategies
4. **Deploy** Private Link for secure service access
5. **Monitor** network security with flow logs and analytics
6. **Apply** zero-trust networking principles

## Quick Wins for AWS Architects

### Immediate Knowledge Transfer

| AWS Concept | Azure Equivalent | Notes |
|-------------|------------------|-------|
| Security Group rules | NSG rules | Nearly identical syntax |
| Inbound/Outbound | Inbound/Outbound | Same concept |
| ALLOW/DENY | Allow/Deny | Same |
| VPC Flow Logs | NSG Flow Logs | Similar analysis |

### Key Differences to Note

1. **NSGs are stateful** (unlike NACLs)
2. **NSGs can attach to subnets OR NICs** (not both like AWS)
3. **Default rules cannot be deleted** (only overridden)
4. **Service tags** simplify rules (like AWS prefix lists)
5. **Application Security Groups** provide logical grouping

---

*Continue to [Quick Reference](quick-reference.md)*

*Back to [Main Guide](../README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
