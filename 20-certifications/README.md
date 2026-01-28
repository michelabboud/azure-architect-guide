# Chapter 09: Azure Certifications Guide

## Overview

This chapter provides a comprehensive guide to Azure certifications, with a focus on the certifications most relevant to AWS architects transitioning to Azure. We'll cover certification paths, study strategies, and exam preparation tips.

## AWS Architect Context

As an AWS architect, you likely hold or are familiar with these certifications:

| AWS Certification | Azure Equivalent | Overlap % |
|-------------------|------------------|-----------|
| AWS Solutions Architect Associate | AZ-104 + AZ-305 (partial) | 60% |
| AWS Solutions Architect Professional | AZ-305 | 70% |
| AWS DevOps Engineer Professional | AZ-400 | 65% |
| AWS Security Specialty | AZ-500 | 55% |
| AWS Advanced Networking | AZ-700 | 60% |
| AWS Database Specialty | DP-300 | 50% |

## Certification Roadmap

```
                    ┌─────────────────────────────────────────────────────────────┐
                    │               Azure Certification Paths                      │
                    └─────────────────────────────────────────────────────────────┘

    Fundamentals                Associate                    Expert
         │                          │                           │
         ▼                          ▼                           ▼
    ┌─────────┐              ┌──────────────┐           ┌──────────────┐
    │ AZ-900  │─────────────▶│   AZ-104     │──────────▶│   AZ-305     │
    │ Azure   │              │   Azure      │           │   Azure      │
    │ Fundam. │              │   Admin.     │           │   Solutions  │
    └─────────┘              └──────────────┘           │   Architect  │
                                    │                   └──────────────┘
                                    │
                                    ▼
                             ┌──────────────┐
                             │   AZ-400     │
                             │   DevOps     │
                             │   Engineer   │
                             └──────────────┘

    Specialty Paths:
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │   AZ-500    │    │   AZ-700    │    │   DP-300    │    │   AI-102    │
    │  Security   │    │  Network    │    │  Database   │    │    AI       │
    │  Engineer   │    │  Engineer   │    │   Admin     │    │  Engineer   │
    └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

## Priority Certifications for AWS Architects

### 1. AZ-305: Azure Solutions Architect Expert (Highest Priority)

**Why This First**: Directly maps to your AWS SA Professional experience

| Domain | Weight | AWS Equivalent Knowledge |
|--------|--------|-------------------------|
| Design identity, governance, monitoring | 25-30% | IAM, Organizations, CloudWatch |
| Design data storage solutions | 15-20% | S3, RDS, DynamoDB, Redshift |
| Design business continuity solutions | 15-20% | Multi-AZ, Cross-Region, Backup |
| Design infrastructure solutions | 30-35% | EC2, VPC, ECS, Lambda |

**Prerequisite**: AZ-104 (can be waived with experience, but recommended)

### 2. AZ-104: Azure Administrator (Foundation)

**Why This Second**: Fills gaps in Azure operational knowledge

| Domain | Weight | Key Azure Services |
|--------|--------|-------------------|
| Manage Azure identities and governance | 20-25% | Azure AD, RBAC, Policy |
| Implement and manage storage | 15-20% | Storage Accounts, File Sync |
| Deploy and manage compute | 20-25% | VMs, App Service, Containers |
| Implement and manage virtual networking | 15-20% | VNet, NSG, Load Balancer |
| Monitor and maintain Azure resources | 10-15% | Monitor, Backup, Update Mgmt |

### 3. AZ-500: Azure Security Engineer (Recommended)

**Why This Third**: Critical for enterprise architecture roles

| Domain | Weight | AWS Equivalent |
|--------|--------|---------------|
| Manage identity and access | 30-35% | IAM, SSO, Directory Service |
| Secure networking | 20-25% | Security Groups, WAF, Shield |
| Secure compute, storage, databases | 25-30% | KMS, Macie, GuardDuty |
| Manage security operations | 15-20% | Security Hub, Detective |

## Quick Reference

### [AZ-305 Study Guide](az-305-study-guide.md)
- Exam objectives breakdown
- Study resources
- Practice scenarios
- Common exam topics

### [AZ-104 Study Guide](az-104-study-guide.md)
- Core concepts
- Hands-on labs
- CLI/PowerShell commands
- Practice questions

### [Certification Tips](certification-tips.md)
- Study strategies
- Exam day preparation
- Time management
- Common pitfalls

## Study Timeline (AWS Architect Background)

### Aggressive Path (2-3 months total)
```
Week 1-2:   AZ-900 (skip exam, use for learning)
Week 3-6:   AZ-104 study + exam
Week 7-12:  AZ-305 study + exam
```

### Balanced Path (4-6 months total)
```
Month 1:    AZ-900 fundamentals (optional exam)
Month 2-3:  AZ-104 study + hands-on labs + exam
Month 4-6:  AZ-305 study + case studies + exam
```

### Comprehensive Path (6-9 months)
```
Month 1:    AZ-900 fundamentals + exam
Month 2-3:  AZ-104 study + exam
Month 4-5:  AZ-305 study + exam
Month 6-7:  AZ-500 (security) + exam
Month 8-9:  AZ-400 (DevOps) + exam
```

## Cost Breakdown

| Certification | Exam Cost (USD) | Retake | Study Materials |
|--------------|-----------------|--------|-----------------|
| AZ-900 | $99 | $99 | Free (MS Learn) |
| AZ-104 | $165 | $165 | $0-300 |
| AZ-305 | $165 | $165 | $0-300 |
| AZ-500 | $165 | $165 | $0-300 |
| AZ-700 | $165 | $165 | $0-300 |

**Cost Saving Tips**:
- Microsoft Learn is free and comprehensive
- Use Azure free tier for labs ($200 credit)
- Enterprise Skills Initiative discounts
- MCP benefits include retake vouchers

## Hands-on Lab Requirements

### Essential Labs for AZ-305

| Lab | Duration | Azure Services |
|-----|----------|----------------|
| Deploy multi-tier application | 2-3 hours | VNet, App Service, SQL |
| Implement hybrid identity | 2 hours | Azure AD Connect, B2C |
| Design disaster recovery | 2 hours | Site Recovery, Backup |
| Configure monitoring | 1-2 hours | Monitor, Log Analytics |
| Implement security controls | 2 hours | Key Vault, NSG, WAF |

### Lab Environments

1. **Azure Free Account**: $200 credit for 30 days
2. **Visual Studio Subscription**: Monthly Azure credits
3. **Microsoft Learn Sandboxes**: Free, temporary environments
4. **Azure Pass**: Available through training events

## Learning Objectives

After completing this chapter, you will:

1. **Understand** the Azure certification landscape
2. **Identify** which certifications align with your career goals
3. **Create** a personalized study plan
4. **Access** quality study resources
5. **Prepare** effectively for exam day

---

*Continue to [AZ-305 Study Guide](az-305-study-guide.md)*

*Back to [Main Guide](../README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
