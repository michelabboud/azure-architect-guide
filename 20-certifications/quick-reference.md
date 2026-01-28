# Azure Certifications Quick Reference

## Certification Paths at a Glance

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AZURE CERTIFICATION ROADMAP                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  FUNDAMENTALS                    ASSOCIATE              EXPERT               │
│  ────────────────                ─────────              ──────               │
│                                                                              │
│      AZ-900                         AZ-104                 AZ-305           │
│    Azure Fundam.        ─────────▶ Admin          ─────────▶ Solutions      │
│    (50-100 min)         (100 min)   (100 min)     (120 min)   Architect     │
│    ~$99                 ~$165       ~$165         ~$165                     │
│                                                                              │
│                                      │                                       │
│                                      │                                       │
│                                      └─────────┬─────────────────┐          │
│                                                │                 │          │
│                                                ▼                 ▼          │
│                                            AZ-400            SPECIALTY:    │
│                                            DevOps           AZ-500 (Sec)   │
│                                            Engineer         AZ-700 (Net)   │
│                                            (~120 min)       DP-300 (DB)    │
│                                            ~$165            AI-102 (AI)    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## AWS to Azure Certification Mapping

### Direct Equivalents

```
┌─────────────────────────────────────────────────────────────────────────────┐
│               AWS → AZURE CERTIFICATION MAPPING                              │
├──────────────────────────────────┬──────────────────────────┬──────────────┤
│ AWS Certification                │ Azure Equivalent         │ Knowledge    │
│                                  │                          │ Overlap      │
├──────────────────────────────────┼──────────────────────────┼──────────────┤
│ Cloud Practitioner               │ AZ-900                   │ 85%          │
│ Solutions Architect Associate    │ AZ-104 + AZ-305 (part)   │ 70%          │
│ Solutions Architect Professional │ AZ-305                   │ 75%          │
│ DevOps Engineer Professional     │ AZ-400                   │ 68%          │
│ Security Specialty               │ AZ-500                   │ 60%          │
│ Advanced Networking Specialty    │ AZ-700                   │ 72%          │
│ Database Specialty               │ DP-300                   │ 55%          │
│ Machine Learning Specialty       │ AI-102 + DP-100          │ 50%          │
└──────────────────────────────────┴──────────────────────────┴──────────────┘
```

### Key Differences in Approach

| Area | AWS Certification | Azure Certification | Key Difference |
|------|-------------------|-------------------|-----------------|
| IAM | IAM service deep-dive | Azure AD + RBAC | Azure emphasizes identity/directory |
| Networking | VPC-centric | VNet + more PaaS options | Azure integrates networking across services |
| Databases | Database-specific exams | DP-300 is broad | Azure bundles more services together |
| DevOps | Separate exam | AZ-400 includes infrastructure | More unified approach |
| Security | Dedicated specialty | Integrated in multiple paths | Azure security is cross-cutting |

---

## AZ-104: Azure Administrator

### Exam Overview

| Attribute | Details |
|-----------|---------|
| **Duration** | 100 minutes |
| **Questions** | 40-60 (mix of question types) |
| **Passing Score** | 700/1000 (70%) |
| **Cost** | $165 USD |
| **Retake Cost** | $165 USD |
| **Valid For** | 1 year (then must recertify) |
| **Difficulty (for AWS Architects)** | Medium |

### Domain Breakdown (100%)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          AZ-104 EXAM DOMAINS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────┐                                    │
│  │ 1. Manage Azure Identities &        │ 20-25%                             │
│  │    Governance                       │                                    │
│  ├─────────────────────────────────────┤                                    │
│  │ • Azure AD users and groups         │                                    │
│  │ • Subscriptions and accounts        │                                    │
│  │ • Azure Policy                      │                                    │
│  │ • Role-Based Access Control (RBAC)  │                                    │
│  └─────────────────────────────────────┘                                    │
│                                                                              │
│  ┌─────────────────────────────────────┐                                    │
│  │ 2. Implement and Manage Storage     │ 15-20%                             │
│  ├─────────────────────────────────────┤                                    │
│  │ • Storage accounts                  │                                    │
│  │ • Blob/Files/Tables/Queues          │                                    │
│  │ • Azure File Sync                   │                                    │
│  │ • Backup and recovery               │                                    │
│  └─────────────────────────────────────┘                                    │
│                                                                              │
│  ┌─────────────────────────────────────┐                                    │
│  │ 3. Deploy and Manage Compute        │ 20-25%                             │
│  ├─────────────────────────────────────┤                                    │
│  │ • Virtual machines                  │                                    │
│  │ • App Service                       │                                    │
│  │ • Container instances               │                                    │
│  │ • Scale sets                        │                                    │
│  └─────────────────────────────────────┘                                    │
│                                                                              │
│  ┌─────────────────────────────────────┐                                    │
│  │ 4. Implement Virtual Networking     │ 15-20%                             │
│  ├─────────────────────────────────────┤                                    │
│  │ • VNets, subnets, peering           │                                    │
│  │ • Network security groups            │                                    │
│  │ • Route tables                       │                                    │
│  │ • Load balancers                     │                                    │
│  └─────────────────────────────────────┘                                    │
│                                                                              │
│  ┌─────────────────────────────────────┐                                    │
│  │ 5. Monitor and Maintain Azure       │ 10-15%                             │
│  │    Resources                        │                                    │
│  ├─────────────────────────────────────┤                                    │
│  │ • Azure Monitor                     │                                    │
│  │ • Update management                 │                                    │
│  │ • Backup and recovery               │                                    │
│  │ • Azure Advisor                     │                                    │
│  └─────────────────────────────────────┘                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Critical Topics for AWS Architects

| Topic | AWS Knowledge | Azure Difference | Focus Area |
|-------|---------------|------------------|-----------|
| RBAC | IAM Roles | Azure RBAC + Azure AD roles | Scope levels (sub, RG, resource) |
| Subscriptions | AWS Accounts | Billing + resource organization | 1 subscription = 1 billing unit |
| Storage | S3, EBS, EFS | Blob, Files, Managed Disks | Storage account types & access tiers |
| Networking | VPC, Security Groups | VNet, NSG | NSG syntax differs from SG |
| Monitoring | CloudWatch | Azure Monitor | Different metrics, KQL queries |

---

## AZ-305: Azure Solutions Architect Expert

### Exam Overview

| Attribute | Details |
|-----------|---------|
| **Duration** | 120 minutes |
| **Questions** | 40-60 (scenario-heavy) |
| **Passing Score** | 700/1000 (70%) |
| **Cost** | $165 USD |
| **Prerequisite** | AZ-104 recommended (not required) |
| **Difficulty (for AWS Architects)** | Medium-Hard |
| **Recommended Prep** | 4-8 weeks of study |

### Domain Breakdown (100%)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          AZ-305 EXAM DOMAINS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────────────────────────────┐                               │
│  │ 1. Design Identity, Governance,          │ 25-30%                        │
│  │    Monitoring Solutions                  │                               │
│  ├──────────────────────────────────────────┤                               │
│  │ • Azure AD architecture                  │                               │
│  │ • Hybrid identity solutions              │                               │
│  │ • Governance and compliance               │                               │
│  │ • Monitoring and logging                  │                               │
│  │ • Cost management                         │                               │
│  └──────────────────────────────────────────┘                               │
│                                                                              │
│  ┌──────────────────────────────────────────┐                               │
│  │ 2. Design Data Storage Solutions         │ 15-20%                        │
│  ├──────────────────────────────────────────┤                               │
│  │ • Data classification                    │                               │
│  │ • Database selection (SQL, Cosmos, etc.) │                               │
│  │ • Storage account architecture           │                               │
│  │ • Caching strategies                     │                               │
│  │ • Data protection                        │                               │
│  └──────────────────────────────────────────┘                               │
│                                                                              │
│  ┌──────────────────────────────────────────┐                               │
│  │ 3. Design Business Continuity Solutions  │ 15-20%                        │
│  ├──────────────────────────────────────────┤                               │
│  │ • High availability architecture         │                               │
│  │ • Disaster recovery planning             │                               │
│  │ • Backup and recovery                    │                               │
│  │ • RTO/RPO requirements                   │                               │
│  │ • Region redundancy                      │                               │
│  └──────────────────────────────────────────┘                               │
│                                                                              │
│  ┌──────────────────────────────────────────┐                               │
│  │ 4. Design Infrastructure Solutions       │ 30-35%                        │
│  ├──────────────────────────────────────────┤                               │
│  │ • Networking architecture (hub-spoke)    │                               │
│  │ • Compute solutions (VMs, containers)    │                               │
│  │ • Integration solutions                  │                               │
│  │ • Application architecture patterns      │                               │
│  │ • Security architecture                  │                               │
│  └──────────────────────────────────────────┘                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Scenario Topics (Most Common on AZ-305)

```
TYPICAL SCENARIO PATTERNS:
─────────────────────────

1. MULTI-TIER APPLICATION MIGRATION
   Prompt: "Migrate on-premises 3-tier app to Azure"

   ┌──────────┐
   │   Web    │  → App Service / VM scale set
   ├──────────┤
   │   API    │  → Container / App Service / AKS
   ├──────────┤
   │ Database │  → Azure SQL / Cosmos DB / MySQL
   └──────────┘

   Key Questions:
   • Lift-and-shift vs. refactor?
   • High availability needs?
   • Hybrid connectivity required?
   • Cost optimization target?

2. HYBRID IDENTITY DESIGN
   "Design identity solution for 5,000-user enterprise"

   Considerations:
   • Azure AD Connect sync strategy
   • Federation (ADFS vs Cloud auth)
   • Multi-factor authentication
   • Conditional Access policies
   • B2B guest access

3. DISASTER RECOVERY ARCHITECTURE
   "RTO 4 hours, RPO 1 hour for mission-critical app"

   Decision Tree:
   • Region redundancy (active-passive vs active-active)
   • Database replication strategy
   • Failover mechanism
   • Cost within budget

4. SECURITY AND COMPLIANCE
   "Design solution meeting PCI-DSS requirements"

   Focus Areas:
   • Network isolation
   • Data encryption (in-transit, at-rest)
   • Access control
   • Audit and monitoring
   • Compliance certification

5. COST OPTIMIZATION
   "Reduce monthly spend by 30% without reducing capacity"

   Strategies:
   • Reserved instances
   • Spot VMs for dev/test
   • Storage tier optimization
   • Database SKU right-sizing
   • Hybrid benefits (on-premises licensing)
```

---

## Study Timeline Recommendations

### For AWS Solutions Architect Associate

```
BASELINE: You know cloud fundamentals, IaaS, identity concepts

┌─────────────────────────────────────────────────────────────────────────┐
│ AGGRESSIVE PATH: 6-8 weeks to both certifications                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Week 1-2: AZ-104 Fundamentals                                         │
│  ├─ Topic: Azure AD, RBAC, Subscriptions                              │
│  ├─ Time: 15 hours (hands-on labs)                                    │
│  └─ Goal: Understand administration basics                            │
│                                                                         │
│  Week 3-4: AZ-104 Deep Dive                                            │
│  ├─ Topic: Storage, Compute, Networking                               │
│  ├─ Time: 20 hours (labs + CLI practice)                             │
│  └─ Goal: Hands-on experience with all services                      │
│                                                                         │
│  Week 5: AZ-104 Practice & Exam                                        │
│  ├─ Topic: Full practice tests, weak area review                      │
│  ├─ Time: 12 hours (2-3 practice exams)                               │
│  └─ Goal: Score 85%+ on practice tests                                │
│                                                                         │
│  Week 6-7: AZ-305 Core Domains                                         │
│  ├─ Topic: Identity, Data Storage, Business Continuity               │
│  ├─ Time: 20 hours (scenario-based study)                            │
│  └─ Goal: Understand design decisions                                │
│                                                                         │
│  Week 8: AZ-305 Practice & Exam                                        │
│  ├─ Topic: Scenario questions, case studies                           │
│  ├─ Time: 15 hours (practice exams)                                   │
│  └─ Goal: Exam ready                                                  │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### For AWS Solutions Architect Professional

```
BASELINE: You understand AWS at professional level

┌─────────────────────────────────────────────────────────────────────────┐
│ BALANCED PATH: 10-12 weeks to both certifications                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  Month 1: AZ-104 Study                                                  │
│  ├─ Week 1-2: Fundamentals (Azure AD, RBAC, Subscriptions)           │
│  ├─ Week 3-4: Services (Storage, Compute, Networking)                │
│  └─ Focus: Build confidence with Azure tools                          │
│                                                                         │
│  Month 2: AZ-104 Exam + AZ-305 Prep                                    │
│  ├─ Week 1: AZ-104 exam (practice tests)                              │
│  ├─ Week 2-4: Start AZ-305 (identity, governance, monitoring)       │
│  └─ Focus: Begin scenario-based thinking                             │
│                                                                         │
│  Month 3: AZ-305 Deep Dive                                             │
│  ├─ Week 1-2: Data storage and business continuity                   │
│  ├─ Week 3-4: Infrastructure and security design                     │
│  └─ Focus: Design patterns and trade-offs                            │
│                                                                         │
│  Month 4: AZ-305 Exam Prep                                             │
│  ├─ Week 1-3: Practice exams, weak area review                       │
│  ├─ Week 4: Final review, exam prep                                   │
│  └─ Focus: Scenario mastery                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Study Resource Matrix

### Recommended Resources by Type

| Resource | Type | Cost | Best For | Link |
|----------|------|------|----------|------|
| Microsoft Learn | Documentation | Free | Official, comprehensive | learn.microsoft.com |
| Pluralsight AZ-104 | Video course | $35/month | Visual learners | pluralsight.com |
| Udemy AZ courses | Video course | $15-30 | Affordable, detailed | udemy.com |
| MeasureUp | Practice tests | $150-200 | Official practice tests | measureup.com |
| Whizlabs | Practice tests | $50-100 | Good quality, cheaper | whizlabs.com |
| Azure Free Account | Hands-on | Free ($200 credit) | Critical for practice | azure.microsoft.com/free |
| MS Learn Sandboxes | Hands-on | Free | Safe practice environment | learn.microsoft.com |
| CloudAcademy | Video + labs | $30/month | Hands-on labs | cloudacademy.com |

### Microsoft Learn Learning Paths (Free, Official)

**For AZ-104:**
- Microsoft Azure Administrator: 20-25 hours
- Azure Fundamentals: 4 hours
- Azure Administration prerequisites: 15 hours

**For AZ-305:**
- Design Azure solutions: 30-40 hours
- Design identity and governance: 8 hours
- Design data storage solutions: 8 hours
- Design business continuity solutions: 6 hours
- Design infrastructure solutions: 10 hours

---

## Study Tips by Learning Style

### Visual Learners
```
✓ Watch Microsoft Azure Architecture Center videos
✓ Use Azure diagram tools (Lucidchart, Draw.io)
✓ Draw architecture diagrams for each scenario
✓ Use visual practice test platforms
✓ Watch YouTube explanations (John Savill, Tim Warner)
```

### Hands-on Learners
```
✓ Build in Azure Free Account (critical!)
✓ Replicate Microsoft Learn labs 2-3 times
✓ Create personal projects (host website, migrate DB)
✓ Use Azure CLI/PowerShell, not portal (better muscle memory)
✓ Practice Infrastructure as Code (ARM, Bicep, Terraform)
```

### Reading/Writing Learners
```
✓ Take detailed notes from Microsoft Learn modules
✓ Summarize each domain in your own words
✓ Create flashcards for Azure AD, RBAC, SKUs
✓ Read Azure whitepapers
✓ Write blog posts explaining concepts
```

### Auditory Learners
```
✓ Listen to Azure podcasts (Azure Weekly, Azure Friday)
✓ Study with others (study groups, Discord communities)
✓ Record yourself explaining topics
✓ Attend Azure user group meetings
✓ Watch Ignite conference sessions
```

---

## Common Study Mistakes (and How to Avoid Them)

```
❌ MISTAKE                              ✓ SOLUTION
──────────────────────────────────────────────────────────────────────────

No hands-on practice                   → Spend 60-70% time in Azure portal
                                         Build at least 3 complete labs

Memorizing exam brain dumps            → Learn concepts, understand WHY
                                         Brain dumps become outdated, fail real exam

Skipping AZ-104 (jumping to AZ-305)   → AZ-104 fills critical gaps
                                         Most AWS architects lack Azure operations

Studying AWS parallels only            → Learn Azure's unique approach
                                         Some concepts don't map 1:1

Taking practice tests too early        → Test at 50-70% study completion
                                         Use tests to identify gaps, not measure progress

Not reading scenario carefully         → Read ENTIRE scenario before answering
                                         Key constraints hidden in details

Cramming week before exam              → Study consistently over 6-8 weeks
                                         Last week should be light review only

Ignoring weak domains                  → Identify weak areas after practice tests
                                         Spend 30% of final week on weak areas
```

---

## Exam Day Checklist

### 24 Hours Before
- [ ] System test completed at [pearsonvue.com/systest](https://www.pearsonvue.com/testtaker/home/systemTest)
- [ ] Room/environment checked (clear desk, quiet space)
- [ ] All notifications disabled (phone, browser, email)
- [ ] Backup internet connection identified (mobile hotspot)
- [ ] ID and credentials prepared
- [ ] Light review of weak areas only
- [ ] Good meal and early sleep planned

### During Exam
- [ ] Read each question completely (2x if needed)
- [ ] Identify keywords: "minimize", "maximize", "most secure", etc.
- [ ] Mark difficult questions for later review
- [ ] Budget time: 1.5-2.5 min per question
- [ ] Check progress: 25%, 50%, 75% through exam
- [ ] Don't leave questions blank
- [ ] Review marked questions if time permits

### After Exam
- [ ] Screenshot results page (proof for resume/LinkedIn)
- [ ] Note weak domains from score report
- [ ] Download digital badge from Credly
- [ ] Add to LinkedIn within 24 hours
- [ ] Plan next certification
- [ ] Note any updated exam topics for future takers

---

## Cost Analysis

### Exam Costs

| Certification | Exam Cost | Retake Cost | Avg Prep Cost | Total Budget |
|---------------|-----------|-------------|---------------|--------------|
| AZ-900 | $99 | $99 | $0-100 | $99-199 |
| AZ-104 | $165 | $165 | $0-200 | $165-365 |
| AZ-305 | $165 | $165 | $0-200 | $165-365 |
| AZ-500 | $165 | $165 | $0-200 | $165-365 |

### Cost Optimization Tips

```
REDUCE EXAM COSTS:
──────────────────
1. Enterprise Skills Initiative: Free exams for employees
2. Certifications for Healthcare: Discounted exams
3. Microsoft Learn: 100% free study material
4. Azure Free Account: $200 credit for labs (not paid)

PREPARATION COSTS:
──────────────────
Budget: $0-500 total for both AZ-104 and AZ-305

Free Options:
  • Microsoft Learn modules
  • Azure Free Account sandbox
  • Microsoft Learn practice labs
  Estimated: $0

Budget Options ($50-100):
  • One practice test platform (Whizlabs)
  • One video course (Udemy sale)
  Estimated: $50-100

Premium Options ($300-500):
  • MeasureUp official practice tests ($200)
  • Pluralsight subscription 1 month ($35)
  • Udemy video course ($50)
  • CloudAcademy labs 1 month ($30)
  Estimated: $300-500
```

---

## Success Metrics

### AZ-104 Practice Test Benchmarks

| Score Range | Status | Recommendation |
|-------------|--------|-----------------|
| 85-100% | Ready | Schedule exam |
| 70-84% | Nearly ready | Review weak domains |
| 50-69% | More study needed | 2-3 more weeks of study |
| <50% | Start over | Focus on fundamentals |

### AZ-305 Practice Test Benchmarks

| Score Range | Status | Recommendation |
|-------------|--------|-----------------|
| 80-100% | Ready | Schedule exam |
| 65-79% | Nearly ready | Focus on scenario questions |
| 50-64% | More study needed | Build more scenarios |
| <50% | Start over | Complete AZ-104 first |

---

## AWS to Azure Terminology Quick Reference

| Concept | AWS | Azure | Key Difference |
|---------|-----|-------|-----------------|
| Account isolation | AWS Account | Subscription | Azure is more flexible (multiple subscriptions per account) |
| Resource grouping | Tags | Tags + Resource Groups | Azure Resource Groups are mandatory |
| Organizational structure | AWS Organizations | Management Groups | Azure has hierarchical management |
| IAM | IAM service | Azure AD + RBAC | Azure separates identity from authorization |
| Virtual network | VPC | VNet | Same concept, Azure simpler model |
| Firewall | Security Groups | NSG | Similar but different syntax |
| Cloud firewall | AWS WAF | Azure WAF | Same purpose, different services |
| Load balancing | ELB/ALB/NLB | Load Balancer/App Gateway | Azure simpler, fewer SKUs |
| DNS | Route 53 | Azure DNS | Azure DNS cheaper, simpler |
| CDN | CloudFront | Azure CDN | Different providers, similar pricing |

---

## Next Steps After Certification

```
IMMEDIATE (Week 1):
├─ Download and share digital badge
├─ Update LinkedIn with certification
├─ Update resume with exam date/score
└─ Share news with team/manager

SHORT-TERM (Month 1-2):
├─ Plan next certification
├─ Apply certified knowledge in real projects
├─ Write blog post about exam experience
└─ Join Azure certification study groups

MEDIUM-TERM (Month 3-6):
├─ Earn specialty certifications (AZ-500, AZ-700, DP-300)
├─ Contribute to Azure community
├─ Mentor others preparing for exams
└─ Stay current with Azure updates

LONG-TERM (Year 1+):
├─ Maintain certification (annual renewal)
├─ Keep Azure skills current
├─ Pursue advanced certifications
└─ Build portfolio of certified expertise
```

---

## Quick Decision Guide: Which Cert Should You Pursue?

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CERTIFICATION DECISION TREE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  "What is your goal?"                                                        │
│                                                                              │
│  ├─ I need foundational knowledge                    → AZ-900               │
│  │                                                                            │
│  ├─ I need to manage Azure resources (ops role)     → AZ-104               │
│  │  (Also: if you already have AWS SA Associate, this fills gaps)           │
│  │                                                                            │
│  ├─ I need to design Azure solutions (architect)    → AZ-305               │
│  │  (Prerequisite: AZ-104 recommended)                                      │
│  │                                                                            │
│  ├─ I specialize in security                         → AZ-500               │
│  │  (Prerequisite: AZ-104 or equivalent experience)                         │
│  │                                                                            │
│  ├─ I specialize in networking                       → AZ-700               │
│  │  (Prerequisite: AZ-104 strongly recommended)                             │
│  │                                                                            │
│  └─ I specialize in databases                        → DP-300               │
│     (Prerequisite: SQL Server/database knowledge required)                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Deep Dive: [AZ-104 Study Guide](az-104-study-guide.md)* | *[AZ-305 Study Guide](az-305-study-guide.md)* | *[Certification Tips](certification-tips.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
