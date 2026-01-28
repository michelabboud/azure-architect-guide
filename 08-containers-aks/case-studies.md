# Container Case Studies

## Case Study 1: E-Commerce Platform Migration

### Scenario

A retail company migrating from AWS EKS needs to containerize their e-commerce platform with:
- 50+ microservices
- 99.99% availability during peak sales
- Multi-region deployment
- CI/CD integration with existing pipelines

### AWS Architecture (Before)

```
Route 53 → CloudFront → ALB → EKS (Fargate) → RDS Aurora
                                    ↓
                                ElastiCache
```

### Azure Solution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE AKS ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         GLOBAL USERS                                         │
│                              │                                               │
│                              ▼                                               │
│                   ┌─────────────────────┐                                   │
│                   │    AZURE FRONT DOOR │                                   │
│                   │    (WAF + CDN)      │                                   │
│                   └──────────┬──────────┘                                   │
│                              │                                               │
│         ┌────────────────────┼────────────────────┐                         │
│         │                    │                    │                         │
│         ▼                    ▼                    ▼                         │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                 │
│  │  EAST US    │      │ WEST EUROPE │      │  EAST ASIA  │                 │
│  │             │      │             │      │             │                 │
│  │ ┌─────────┐ │      │ ┌─────────┐ │      │ ┌─────────┐ │                 │
│  │ │NGINX IC │ │      │ │NGINX IC │ │      │ │NGINX IC │ │                 │
│  │ └────┬────┘ │      │ └────┬────┘ │      │ └────┬────┘ │                 │
│  │      │      │      │      │      │      │      │      │                 │
│  │ ┌────▼────┐ │      │ ┌────▼────┐ │      │ ┌────▼────┐ │                 │
│  │ │   AKS   │ │      │ │   AKS   │ │      │ │   AKS   │ │                 │
│  │ │         │ │      │ │         │ │      │ │         │ │                 │
│  │ │┌───────┐│ │      │ │┌───────┐│ │      │ │┌───────┐│ │                 │
│  │ ││ API   ││ │      │ ││ API   ││ │      │ ││ API   ││ │                 │
│  │ │├───────┤│ │      │ │├───────┤│ │      │ │├───────┤│ │                 │
│  │ ││ Cart  ││ │      │ ││ Cart  ││ │      │ ││ Cart  ││ │                 │
│  │ │├───────┤│ │      │ │├───────┤│ │      │ │├───────┤│ │                 │
│  │ ││Catalog││ │      │ ││Catalog││ │      │ ││Catalog││ │                 │
│  │ │├───────┤│ │      │ │├───────┤│ │      │ │├───────┤│ │                 │
│  │ ││Payment││ │      │ ││Payment││ │      │ ││Payment││ │                 │
│  │ │└───────┘│ │      │ │└───────┘│ │      │ │└───────┘│ │                 │
│  │ └────┬────┘ │      │ └────┬────┘ │      │ └────┬────┘ │                 │
│  │      │      │      │      │      │      │      │      │                 │
│  │ ┌────▼────┐ │      │ ┌────▼────┐ │      │ ┌────▼────┐ │                 │
│  │ │  Redis  │ │      │ │  Redis  │ │      │ │  Redis  │ │                 │
│  │ │(Premium)│ │      │ │(Premium)│ │      │ │(Premium)│ │                 │
│  │ └─────────┘ │      │ └─────────┘ │      │ └─────────┘ │                 │
│  └─────────────┘      └─────────────┘      └─────────────┘                 │
│         │                    │                    │                         │
│         └────────────────────┼────────────────────┘                         │
│                              │                                               │
│                    ┌─────────▼─────────┐                                    │
│                    │    COSMOS DB      │                                    │
│                    │  (Global Write)   │                                    │
│                    └───────────────────┘                                    │
│                                                                              │
│  SHARED SERVICES:                                                           │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │ ACR (Premium, geo-replicated) │ Key Vault │ Azure Monitor          │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Decisions

| Component | Choice | Rationale |
|-----------|--------|-----------|
| Orchestration | AKS (Standard tier) | Full K8s control, 99.95% SLA |
| Ingress | NGINX Ingress Controller | Familiar from EKS, full control |
| Service Mesh | Istio | Traffic management, observability |
| Registry | ACR Premium | Geo-replication, private endpoint |
| State Store | Cosmos DB | Multi-region writes |
| Cache | Redis Premium | Geo-replication, clustering |

### Migration Strategy

1. **Phase 1**: Set up ACR and push images
2. **Phase 2**: Deploy AKS clusters (infra as code)
3. **Phase 3**: Deploy stateless services first
4. **Phase 4**: Migrate databases to Cosmos DB
5. **Phase 5**: Configure Front Door routing
6. **Phase 6**: Gradual traffic cutover

---

## Case Study 2: Event-Driven Microservices

### Scenario

A financial services company needs to process millions of transactions daily with:
- Event-driven architecture
- Auto-scaling based on queue depth
- Cost optimization (scale to zero)
- Strict compliance requirements

### Solution: Container Apps with KEDA

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                  EVENT-DRIVEN CONTAINER APPS                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         EXTERNAL SYSTEMS                                     │
│                              │                                               │
│                              ▼                                               │
│                   ┌─────────────────────┐                                   │
│                   │     API MANAGEMENT  │                                   │
│                   │     (Rate limiting) │                                   │
│                   └──────────┬──────────┘                                   │
│                              │                                               │
│  ┌───────────────────────────┴───────────────────────────────────────────┐  │
│  │                 CONTAINER APPS ENVIRONMENT                             │  │
│  │                 (VNet Integrated)                                      │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  API GATEWAY APP                                                 │  │  │
│  │  │  • External ingress                                              │  │  │
│  │  │  • HTTP scaling (concurrent requests)                            │  │  │
│  │  │  • Min replicas: 2                                               │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                              │                                         │  │
│  │                              ▼                                         │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │              SERVICE BUS (Premium)                               │  │  │
│  │  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │  │  │
│  │  │  │ transactions  │  │ notifications │  │    audit      │        │  │  │
│  │  │  │    queue      │  │    queue      │  │    queue      │        │  │  │
│  │  │  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘        │  │  │
│  │  └──────────┼──────────────────┼──────────────────┼─────────────────┘  │  │
│  │             │                  │                  │                    │  │
│  │             ▼                  ▼                  ▼                    │  │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │  │
│  │  │ TRANSACTION     │  │ NOTIFICATION    │  │ AUDIT           │        │  │
│  │  │ PROCESSOR       │  │ SERVICE         │  │ SERVICE         │        │  │
│  │  │                 │  │                 │  │                 │        │  │
│  │  │ Scaling:        │  │ Scaling:        │  │ Scaling:        │        │  │
│  │  │ • Queue length  │  │ • Queue length  │  │ • Queue length  │        │  │
│  │  │ • Min: 0        │  │ • Min: 0        │  │ • Min: 1        │        │  │
│  │  │ • Max: 30       │  │ • Max: 10       │  │ • Max: 5        │        │  │
│  │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘        │  │
│  │           │                    │                    │                  │  │
│  └───────────┼────────────────────┼────────────────────┼──────────────────┘  │
│              │                    │                    │                     │
│              ▼                    ▼                    ▼                     │
│  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐       │
│  │    COSMOS DB      │  │  Email/SMS APIs   │  │   BLOB STORAGE    │       │
│  │  (Transactions)   │  │                   │  │   (Audit Logs)    │       │
│  └───────────────────┘  └───────────────────┘  └───────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### KEDA Scaling Configuration

```yaml
# Container App with Service Bus scaling
apiVersion: apps/v1
kind: Deployment
metadata:
  name: transaction-processor
spec:
  template:
    spec:
      containers:
      - name: processor
        image: myacr.azurecr.io/processor:v1
        resources:
          requests:
            cpu: "0.5"
            memory: "1Gi"
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: transaction-scaler
spec:
  scaleTargetRef:
    name: transaction-processor
  minReplicaCount: 0
  maxReplicaCount: 30
  cooldownPeriod: 60
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: transactions
      messageCount: "10"
    authenticationRef:
      name: servicebus-auth
```

### Cost Analysis

| Scenario | AKS Cost | Container Apps Cost | Savings |
|----------|----------|---------------------|---------|
| Baseline (24/7) | $2,400/month | $1,800/month | 25% |
| Off-peak (scale to 0) | $2,400/month | $600/month | 75% |
| Burst (10x traffic) | $8,000/month | $3,000/month | 63% |

---

## Case Study 3: CI/CD Pipeline with ACI

### Scenario

A software company needs scalable CI/CD agents with:
- On-demand build agents
- VNet integration for private repositories
- Cost optimization (no idle resources)
- Quick spin-up time

### Solution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     CI/CD WITH ACI AGENTS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    ┌────────────────────────┐                               │
│                    │    AZURE DEVOPS        │                               │
│                    │    (Agent Pool)        │                               │
│                    └───────────┬────────────┘                               │
│                                │                                             │
│              Trigger: New Build Request                                      │
│                                │                                             │
│                                ▼                                             │
│                    ┌────────────────────────┐                               │
│                    │    AZURE FUNCTION      │                               │
│                    │    (Agent Orchestrator)│                               │
│                    │                        │                               │
│                    │  • Receive webhook     │                               │
│                    │  • Create ACI instance │                               │
│                    │  • Register with pool  │                               │
│                    │  • Cleanup on complete │                               │
│                    └───────────┬────────────┘                               │
│                                │                                             │
│                    Creates Container Instance                                │
│                                │                                             │
│  ┌─────────────────────────────▼─────────────────────────────────────────┐  │
│  │                         VNET                                           │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  ACI SUBNET (Delegated)                                         │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │  │  │
│  │  │  │ Build Agent 1 │  │ Build Agent 2 │  │ Build Agent N │        │  │  │
│  │  │  │               │  │               │  │               │        │  │  │
│  │  │  │ • 4 vCPU      │  │ • 4 vCPU      │  │ • 4 vCPU      │        │  │  │
│  │  │  │ • 16 GB RAM   │  │ • 16 GB RAM   │  │ • 16 GB RAM   │        │  │  │
│  │  │  │ • Ephemeral   │  │ • Ephemeral   │  │ • Ephemeral   │        │  │  │
│  │  │  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘        │  │  │
│  │  │          │                  │                  │                 │  │  │
│  │  └──────────┼──────────────────┼──────────────────┼─────────────────┘  │  │
│  │             │                  │                  │                    │  │
│  │             ▼                  ▼                  ▼                    │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                    PRIVATE RESOURCES                             │  │  │
│  │  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐        │  │  │
│  │  │  │ ACR (Private) │  │ Key Vault     │  │ AKS Cluster   │        │  │  │
│  │  │  │               │  │ (Secrets)     │  │ (Deploy)      │        │  │  │
│  │  │  └───────────────┘  └───────────────┘  └───────────────┘        │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  LIFECYCLE:                                                                 │
│  1. Build triggered → Function creates ACI                                  │
│  2. ACI starts (~30 seconds)                                               │
│  3. Agent registers with Azure DevOps                                       │
│  4. Build runs                                                              │
│  5. Build completes → Function deletes ACI                                 │
│  6. Pay only for build duration                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ACI Agent Configuration

```bash
# Create ACI build agent
az container create \
  --resource-group cicd-rg \
  --name build-agent-001 \
  --image myacr.azurecr.io/devops-agent:latest \
  --cpu 4 \
  --memory 16 \
  --vnet cicd-vnet \
  --subnet aci-subnet \
  --assign-identity [system] \
  --environment-variables \
    AZP_URL=https://dev.azure.com/myorg \
    AZP_POOL=ACI-Pool \
    AZP_AGENT_NAME=build-agent-001 \
  --secure-environment-variables \
    AZP_TOKEN=$DEVOPS_PAT \
  --restart-policy Never
```

### Cost Comparison

| Approach | Monthly Cost | Pros | Cons |
|----------|--------------|------|------|
| VMSS Agents | $500+ | Always available | Idle costs |
| AKS Agents | $300+ | Shared resources | Complexity |
| ACI Agents | $50-100 | Pay per build | 30s startup |

---

## Case Study 4: Hybrid AKS with Virtual Nodes

### Scenario

A media company needs to handle unpredictable rendering workloads:
- Burst to 1000s of containers
- No node management overhead
- Mixed persistent and burst workloads

### Solution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AKS WITH VIRTUAL NODES                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                           AKS CLUSTER                                  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  STANDARD NODE POOL                                              │  │  │
│  │  │  (Persistent workloads - Web API, Databases)                     │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │  │  │
│  │  │  │  Node 1  │  │  Node 2  │  │  Node 3  │                       │  │  │
│  │  │  │ D8s_v5   │  │ D8s_v5   │  │ D8s_v5   │                       │  │  │
│  │  │  │          │  │          │  │          │                       │  │  │
│  │  │  │[API Pod] │  │[API Pod] │  │[API Pod] │                       │  │  │
│  │  │  │[DB Pod]  │  │[DB Pod]  │  │[Worker]  │                       │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘                       │  │  │
│  │  │                                                                  │  │  │
│  │  │  Autoscaler: min=3, max=10                                      │  │  │
│  │  │  Tolerations: None (standard workloads)                         │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  VIRTUAL NODE (ACI Backend)                                      │  │  │
│  │  │  (Burst workloads - Rendering, Processing)                       │  │  │
│  │  │                                                                  │  │  │
│  │  │         ┌────────────┐  ┌────────────┐  ┌────────────┐          │  │  │
│  │  │         │ Render Job │  │ Render Job │  │ Render Job │          │  │  │
│  │  │         │     1      │  │     2      │  │     N      │          │  │  │
│  │  │         │            │  │            │  │   ...      │          │  │  │
│  │  │         │ 4 vCPU     │  │ 4 vCPU     │  │ 4 vCPU     │          │  │  │
│  │  │         │ 16GB RAM   │  │ 16GB RAM   │  │ 16GB RAM   │          │  │  │
│  │  │         └────────────┘  └────────────┘  └────────────┘          │  │  │
│  │  │                                                                  │  │  │
│  │  │  Scaling: Unlimited (ACI capacity)                              │  │  │
│  │  │  Toleration: virtual-kubelet.io/provider=azure                  │  │  │
│  │  │  NodeSelector: kubernetes.io/role=agent,type=virtual            │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│  TRAFFIC PATTERN:                                                           │
│                                                                              │
│  Jobs/hour │                                    ████                        │
│            │                               █████████                        │
│            │                          ██████████████                        │
│            │    ████████████████████████████████████                        │
│            └────────────────────────────────────────────                    │
│                 06:00        12:00       18:00      24:00                   │
│                                                                              │
│  COST OPTIMIZATION:                                                         │
│  • Base load on dedicated nodes (predictable cost)                         │
│  • Burst load on virtual nodes (pay per second)                            │
│  • No over-provisioning required                                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Virtual Node Deployment

```yaml
# Render job targeting virtual nodes
apiVersion: batch/v1
kind: Job
metadata:
  name: render-job
spec:
  parallelism: 100
  template:
    spec:
      nodeSelector:
        kubernetes.io/role: agent
        type: virtual
      tolerations:
      - key: virtual-kubelet.io/provider
        operator: Equal
        value: azure
        effect: NoSchedule
      containers:
      - name: renderer
        image: myacr.azurecr.io/renderer:v1
        resources:
          requests:
            cpu: "4"
            memory: "16Gi"
          limits:
            cpu: "4"
            memory: "16Gi"
      restartPolicy: Never
```

---

## Summary

| Case Study | Container Service | Key Pattern |
|------------|-------------------|-------------|
| E-Commerce | AKS (Multi-region) | Microservices, global distribution |
| Event-Driven | Container Apps | KEDA scaling, scale-to-zero |
| CI/CD | ACI | Ephemeral agents, VNet integration |
| Burst Workloads | AKS + Virtual Nodes | Hybrid scaling |

---

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
