# Cloud Architect Interview Topics & Questions

A comprehensive guide covering common interview topics for Cloud Architect positions.

## 1. Architecture Design Principles

### Well-Architected Framework (Both AWS & Azure)

The five/six pillars you must know:

| Pillar | AWS | Azure | Key Concepts |
|--------|-----|-------|--------------|
| Operational Excellence | Yes | Yes | Monitoring, automation, continuous improvement |
| Security | Yes | Yes | Identity, encryption, compliance, threat protection |
| Reliability | Yes | Yes | Fault tolerance, disaster recovery, scaling |
| Performance Efficiency | Yes | Yes | Right-sizing, caching, optimization |
| Cost Optimization | Yes | Yes | Reserved instances, right-sizing, waste elimination |
| Sustainability | Yes | No* | Environmental impact, efficient resources |

*Azure incorporates sustainability within other pillars

### Common Interview Questions

**Q: How would you design a highly available web application?**

A: Key components to discuss:
- Multi-AZ/region deployment
- Load balancing (ALB/Application Gateway)
- Auto-scaling groups
- Database replication (Multi-AZ RDS / Azure SQL Geo-replication)
- CDN for static content
- Health checks and failover mechanisms
- DNS failover (Route 53 / Traffic Manager)

**Q: Explain the difference between high availability and disaster recovery.**

A:
- **High Availability (HA):** Minimizing downtime within a region using redundancy (multiple AZs, load balancing, auto-scaling). RTO: seconds to minutes.
- **Disaster Recovery (DR):** Recovering from regional failures using backup regions. RTO: minutes to hours depending on strategy (pilot light, warm standby, active-active).

**Q: What factors do you consider when choosing between IaaS, PaaS, and SaaS?**

A:
```
Control Level:     IaaS > PaaS > SaaS
Management Effort: IaaS > PaaS > SaaS
Flexibility:       IaaS > PaaS > SaaS
Time to Market:    SaaS > PaaS > IaaS

Choose IaaS when: Need full control, specific OS requirements, lift-and-shift
Choose PaaS when: Focus on code, want managed infrastructure, rapid development
Choose SaaS when: Standard functionality needed, minimal customization
```

## 2. Security Architecture

### Identity and Access Management

**Q: Explain the principle of least privilege and how you implement it.**

A:
- Grant minimum permissions needed for a task
- Use IAM roles instead of long-term credentials
- Implement just-in-time access
- Regular access reviews and audits
- Use managed identities (Azure) / IAM roles (AWS) for service-to-service auth

**Q: How do you secure data at rest and in transit?**

A:
```
Data at Rest:
├── Encryption using KMS/Key Vault managed keys
├── Customer-managed keys (CMK) for compliance
├── Server-side encryption for storage
└── Database encryption (TDE, encrypted EBS/Managed Disks)

Data in Transit:
├── TLS 1.2+ for all communications
├── VPN or Direct Connect/ExpressRoute for hybrid
├── Private endpoints to avoid public internet
└── Certificate management and rotation
```

**Q: Describe a Zero Trust architecture.**

A: Key principles:
1. Never trust, always verify
2. Assume breach mentality
3. Verify explicitly (identity, device, location)
4. Use least privilege access
5. Microsegmentation
6. Continuous monitoring and validation

### Network Security

**Q: How do you design a secure network architecture?**

A:
```
                    Internet
                        │
                   [WAF/DDoS]
                        │
                   [Load Balancer]
                        │
              ┌─────────┴─────────┐
              │   Public Subnet    │  (Bastion, NAT Gateway)
              └─────────┬─────────┘
                        │
              ┌─────────┴─────────┐
              │  Private Subnet    │  (Application Tier)
              └─────────┬─────────┘
                        │
              ┌─────────┴─────────┐
              │   Data Subnet      │  (Databases)
              └───────────────────┘
                        │
               [Private Endpoints]
```

Key components:
- Network segmentation with subnets
- Security groups / NSGs for instance-level firewall
- NACLs for subnet-level control (AWS)
- WAF for application protection
- DDoS protection
- Private endpoints for PaaS services
- VPN/ExpressRoute for hybrid connectivity

## 3. Reliability & Resilience

### Disaster Recovery Strategies

| Strategy | RTO | RPO | Cost | Use Case |
|----------|-----|-----|------|----------|
| Backup & Restore | Hours | Hours | $ | Non-critical systems |
| Pilot Light | 10s of min | Minutes | $$ | Core systems ready |
| Warm Standby | Minutes | Seconds | $$$ | Scaled-down copy running |
| Multi-Site Active-Active | Real-time | Near zero | $$$$ | Mission-critical |

**Q: How do you calculate and achieve a specific SLA?**

A:
```
Composite SLA = SLA1 × SLA2 × SLA3 × ...

Example: Web App (99.95%) × Database (99.99%) × Storage (99.9%)
= 0.9995 × 0.9999 × 0.999 = 99.84%

To improve availability:
1. Add redundancy (multiple instances)
2. Use availability zones
3. Implement health checks and auto-healing
4. Design for failure with circuit breakers
5. Use managed services with higher SLAs
```

### Auto-Scaling Patterns

**Q: Explain different auto-scaling strategies.**

A:
1. **Reactive Scaling:** Based on current metrics (CPU, memory, requests)
2. **Scheduled Scaling:** Based on known patterns (business hours)
3. **Predictive Scaling:** ML-based prediction of future demand
4. **Target Tracking:** Maintain a specific metric value

Best practices:
- Set appropriate cooldown periods
- Use multiple metrics
- Test scaling policies
- Monitor for thrashing
- Consider step scaling for gradual changes

## 4. Cost Optimization

### Cost Management Strategies

**Q: How do you optimize cloud costs?**

A:
```
Immediate Wins:
├── Right-size underutilized resources
├── Delete unused resources (orphaned disks, old snapshots)
├── Use reserved instances for steady-state workloads
└── Spot/preemptible instances for fault-tolerant workloads

Architectural:
├── Use serverless for variable workloads
├── Implement auto-scaling
├── Choose appropriate storage tiers
├── Use CDN to reduce egress costs
└── Optimize data transfer paths

Governance:
├── Implement tagging strategy
├── Set up budgets and alerts
├── Regular cost reviews
├── Showback/chargeback to teams
└── Use cost management tools
```

**Q: When would you use reserved instances vs spot instances vs on-demand?**

A:
| Type | Use Case | Savings | Risk |
|------|----------|---------|------|
| On-Demand | Short-term, unpredictable | 0% | None |
| Reserved (1yr) | Steady-state workloads | 30-40% | Commitment |
| Reserved (3yr) | Long-term predictable | 50-60% | Long commitment |
| Spot/Preemptible | Batch, fault-tolerant | 60-90% | Interruption |
| Savings Plans | Flexible commitment | 20-50% | Commitment |

## 5. Migration Strategies

### The 6 R's of Migration

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Rehost | Lift and shift | Quick migration, minimal changes |
| Replatform | Lift and optimize | Minor cloud optimizations |
| Repurchase | Replace with SaaS | When SaaS meets needs |
| Refactor | Re-architect | Cloud-native benefits needed |
| Retire | Decommission | No longer needed |
| Retain | Keep on-premises | Cannot migrate (compliance, tech) |

**Q: How would you approach migrating a legacy monolithic application to the cloud?**

A:
1. **Assess:** Understand dependencies, performance, data flows
2. **Plan:** Choose migration strategy based on business goals
3. **Pilot:** Start with non-critical components
4. **Migrate:** Execute in phases
5. **Optimize:** Modernize post-migration

Considerations:
- Database migration strategy (homogeneous vs heterogeneous)
- Network connectivity during migration
- Data sync and cutover planning
- Testing and validation
- Rollback procedures

## 6. DevOps & Automation

### Infrastructure as Code

**Q: Compare CloudFormation, Terraform, and ARM/Bicep.**

A:
| Feature | CloudFormation | Terraform | ARM/Bicep |
|---------|---------------|-----------|-----------|
| Cloud | AWS only | Multi-cloud | Azure only |
| Language | JSON/YAML | HCL | JSON/Bicep |
| State | Managed by AWS | External state file | Managed by Azure |
| Modularity | Nested stacks | Modules | Linked templates/modules |
| Drift Detection | Yes | Yes | Limited |

**Q: Describe a CI/CD pipeline for cloud infrastructure.**

A:
```
Code Commit → Lint/Validate → Security Scan → Plan/Preview →
Manual Approval → Deploy to Dev → Test → Deploy to Staging →
Integration Test → Manual Approval → Deploy to Prod → Smoke Test
```

Key practices:
- Version control all infrastructure code
- Use pull requests with reviews
- Automated testing (unit, integration)
- Security scanning (checkov, tfsec)
- Immutable infrastructure
- Blue/green or canary deployments

## 7. Containerization & Kubernetes

### Container Architecture

**Q: When would you choose containers over serverless or VMs?**

A:
```
Choose VMs when:
├── Legacy applications requiring specific OS
├── Stateful applications with complex storage needs
├── Licensing tied to hardware/VM
└── Full control over infrastructure needed

Choose Containers when:
├── Microservices architecture
├── Consistent dev/prod environments
├── Rapid scaling requirements
├── Resource efficiency important
└── Portability between clouds needed

Choose Serverless when:
├── Event-driven workloads
├── Highly variable traffic
├── Want zero infrastructure management
├── Cost optimization for sporadic usage
└── Rapid development priority
```

**Q: Explain Kubernetes architecture components.**

A:
```
Control Plane:
├── API Server (kubectl, SDKs interact here)
├── etcd (cluster state store)
├── Scheduler (places pods on nodes)
└── Controller Manager (maintains desired state)

Worker Nodes:
├── Kubelet (node agent)
├── Container Runtime (containerd, CRI-O)
└── Kube-proxy (network rules)

Key Objects:
├── Pods (smallest deployable unit)
├── Deployments (declarative pod management)
├── Services (network abstraction)
├── Ingress (external access)
└── ConfigMaps/Secrets (configuration)
```

## 8. Data Architecture

### Data Storage Decisions

**Q: How do you choose between SQL and NoSQL databases?**

A:
| Factor | SQL | NoSQL |
|--------|-----|-------|
| Data Structure | Structured, relationships | Flexible, denormalized |
| Consistency | Strong (ACID) | Eventual (BASE) |
| Scalability | Vertical primarily | Horizontal |
| Query Complexity | Complex joins | Simple lookups |
| Use Cases | Financial, ERP | IoT, content, gaming |

**Q: Explain data lake vs data warehouse.**

A:
```
Data Lake:
├── Raw, unprocessed data
├── Schema-on-read
├── All data types (structured, unstructured)
├── Exploration and ML
└── Lower cost storage

Data Warehouse:
├── Processed, curated data
├── Schema-on-write
├── Structured data
├── BI and reporting
└── Optimized for queries
```

## 9. Behavioral Questions

### STAR Method Examples

**Q: Tell me about a time you had to make a difficult architectural decision.**

Structure your answer:
- **Situation:** Set the context
- **Task:** What was your responsibility
- **Action:** What you specifically did
- **Result:** Quantifiable outcome

Example topics:
- Choosing between build vs buy
- Migrating to microservices
- Handling technical debt
- Cost optimization initiatives
- Security incident response

**Q: How do you handle disagreements with team members about architecture?**

Key points:
- Listen to understand their perspective
- Focus on requirements and constraints
- Use data and evidence
- Prototype when possible
- Know when to escalate
- Document decisions (ADRs)

## 10. Scenario-Based Questions

### Design Exercises

**Q: Design a scalable e-commerce platform.**

Key components to discuss:
```
┌─────────────────────────────────────────────────────────────┐
│                        CDN (CloudFront/Azure CDN)           │
└─────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────┐
│                    API Gateway / Load Balancer              │
└─────────────────────────────────────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
   [Product Service]    [Order Service]      [User Service]
        │                     │                     │
   [Product DB]          [Order DB]           [User DB]
   (Read replicas)     (Write/Read split)    (Global table)
        │                     │                     │
        └─────────────────────┼─────────────────────┘
                              │
                    [Cache Layer - Redis]
                              │
                    [Search - Elasticsearch]
                              │
                    [Queue - SQS/Service Bus]
                              │
                    [Async Workers]
```

Considerations:
- Caching strategy (product catalog, sessions)
- Search functionality
- Payment processing (PCI compliance)
- Inventory management
- Order processing queues
- Analytics and recommendations

---

*Next: See [03-architecture-patterns.md](03-architecture-patterns.md) for common design patterns*
