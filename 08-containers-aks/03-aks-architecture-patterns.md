# AKS Architecture Patterns: The Architect's Guide

*Good designs, bad designs, and everything in between*

---

## The Architecture Decision Framework

Before diving into patterns, every AKS architecture decision should answer:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AKS ARCHITECTURE DECISION FRAMEWORK                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   1. WORKLOAD CHARACTERISTICS                                               │
│      ├── Stateless or stateful?                                             │
│      ├── CPU-bound, memory-bound, or I/O-bound?                            │
│      ├── Consistent load or spiky traffic?                                 │
│      └── Latency-sensitive or throughput-optimized?                        │
│                                                                              │
│   2. RELIABILITY REQUIREMENTS                                               │
│      ├── What's the acceptable downtime? (99.9% = 8.76 hrs/year)          │
│      ├── RPO/RTO for disaster recovery?                                    │
│      └── Blast radius tolerance?                                            │
│                                                                              │
│   3. SECURITY & COMPLIANCE                                                  │
│      ├── Network isolation requirements?                                    │
│      ├── Data residency constraints?                                        │
│      └── Audit and compliance needs?                                        │
│                                                                              │
│   4. OPERATIONAL MATURITY                                                   │
│      ├── Team's Kubernetes expertise?                                       │
│      ├── Existing tooling and processes?                                   │
│      └── On-call and incident response capabilities?                       │
│                                                                              │
│   5. COST CONSTRAINTS                                                       │
│      ├── Budget for infrastructure?                                         │
│      ├── Cost optimization priorities?                                      │
│      └── Reserved capacity vs pay-as-you-go?                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Pattern 1: Node Pool Strategy

### ❌ Anti-Pattern: Single Node Pool for Everything

```
THE "ONE POOL TO RULE THEM ALL" MISTAKE:
────────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │                     SINGLE NODE POOL                                   ││
│   │                     (Standard_D4s_v3)                                  ││
│   │                                                                        ││
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       ││
│   │   │ CoreDNS │ │ Web App │ │ ML Job  │ │ Redis   │ │ Cronjob │       ││
│   │   │ metrics │ │ (light) │ │ (heavy) │ │ (memory)│ │ (burst) │       ││
│   │   └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       ││
│   │                                                                        ││
│   │   PROBLEMS:                                                           ││
│   │   ├── ML job starves CoreDNS → cluster-wide DNS failures             ││
│   │   ├── Can't scale ML independently from web                          ││
│   │   ├── Redis needs memory-optimized VMs, web needs balanced           ││
│   │   ├── System components compete with user workloads                  ││
│   │   ├── One bad actor affects everything                               ││
│   │   └── Can't use spot instances for batch jobs safely                 ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   REAL INCIDENT:                                                            │
│   "Our entire platform went down because a memory-leaking batch job         │
│    caused OOM kills on nodes running CoreDNS. DNS failures cascaded         │
│    to all services. Outage: 47 minutes."                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ✅ Good Pattern: Purpose-Built Node Pools

```
WELL-ARCHITECTED NODE POOL STRATEGY:
────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  SYSTEM POOL (Required - Isolated)                                   │  │
│   │  VM: Standard_D4s_v5 | Zones: 1,2,3 | Count: 3 (fixed)              │  │
│   │  Taint: CriticalAddonsOnly=true:NoSchedule                          │  │
│   │                                                                      │  │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐               │  │
│   │  │ CoreDNS  │ │ metrics- │ │ kube-    │ │ Azure    │               │  │
│   │  │          │ │ server   │ │ proxy    │ │ Policy   │               │  │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘               │  │
│   │                                                                      │  │
│   │  WHY: System components are NEVER affected by user workloads        │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  GENERAL POOL (User workloads)                                       │  │
│   │  VM: Standard_D8s_v5 | Zones: 1,2,3 | Autoscale: 3-50               │  │
│   │  Labels: workload=general                                            │  │
│   │                                                                      │  │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐               │  │
│   │  │ API      │ │ Web      │ │ Worker   │ │ Sidecar  │               │  │
│   │  │ Services │ │ Frontend │ │ Services │ │ Proxies  │               │  │
│   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘               │  │
│   │                                                                      │  │
│   │  WHY: Standard workloads with predictable resource patterns         │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  MEMORY POOL (Memory-intensive)                                      │  │
│   │  VM: Standard_E8s_v5 | Zones: 1,2,3 | Autoscale: 2-20               │  │
│   │  Taint: workload=memory:NoSchedule                                   │  │
│   │  Labels: workload=memory                                             │  │
│   │                                                                      │  │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐                             │  │
│   │  │ Redis    │ │ In-mem   │ │ Cache    │                             │  │
│   │  │ Clusters │ │ DBs      │ │ Layers   │                             │  │
│   │  └──────────┘ └──────────┘ └──────────┘                             │  │
│   │                                                                      │  │
│   │  WHY: 1:8 vCPU:memory ratio for memory-bound workloads              │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  SPOT POOL (Batch/Interruptible)                                     │  │
│   │  VM: Standard_D16s_v5 (Spot) | Zones: 1,2,3 | Autoscale: 0-100      │  │
│   │  Taint: kubernetes.azure.com/scalesetpriority=spot:NoSchedule       │  │
│   │  Labels: workload=batch                                              │  │
│   │                                                                      │  │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐                             │  │
│   │  │ ML       │ │ Data     │ │ Report   │                             │  │
│   │  │ Training │ │ Pipeline │ │ Generate │                             │  │
│   │  └──────────┘ └──────────┘ └──────────┘                             │  │
│   │                                                                      │  │
│   │  WHY: Up to 90% cost savings for fault-tolerant batch jobs          │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐  │
│   │  GPU POOL (ML Inference)                                             │  │
│   │  VM: Standard_NC6s_v3 | Zones: 1 | Autoscale: 0-10                  │  │
│   │  Taint: sku=gpu:NoSchedule                                           │  │
│   │  Labels: hardware=gpu                                                │  │
│   │                                                                      │  │
│   │  ┌──────────┐ ┌──────────┐                                          │  │
│   │  │ Model    │ │ Vision   │                                          │  │
│   │  │ Serving  │ │ API      │                                          │  │
│   │  └──────────┘ └──────────┘                                          │  │
│   │                                                                      │  │
│   │  WHY: Expensive GPUs only used by workloads that need them          │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```bash
# Create cluster with system pool
az aks create \
  --resource-group aks-rg \
  --name prod-aks \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --nodepool-name system \
  --nodepool-taints CriticalAddonsOnly=true:NoSchedule \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 5

# Add general user pool
az aks nodepool add \
  --resource-group aks-rg \
  --cluster-name prod-aks \
  --name general \
  --node-count 3 \
  --node-vm-size Standard_D8s_v5 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 50 \
  --labels workload=general \
  --mode User

# Add memory-optimized pool
az aks nodepool add \
  --resource-group aks-rg \
  --cluster-name prod-aks \
  --name mempool \
  --node-count 2 \
  --node-vm-size Standard_E8s_v5 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 2 \
  --max-count 20 \
  --node-taints workload=memory:NoSchedule \
  --labels workload=memory \
  --mode User

# Add spot pool for batch
az aks nodepool add \
  --resource-group aks-rg \
  --cluster-name prod-aks \
  --name spotpool \
  --node-count 0 \
  --node-vm-size Standard_D16s_v5 \
  --priority Spot \
  --eviction-policy Delete \
  --spot-max-price -1 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 100 \
  --labels workload=batch \
  --mode User
```

### Pod Scheduling to Target Pools

```yaml
# Schedule to memory pool
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis-cluster
spec:
  template:
    spec:
      nodeSelector:
        workload: memory
      tolerations:
        - key: workload
          operator: Equal
          value: memory
          effect: NoSchedule
      containers:
        - name: redis
          resources:
            requests:
              memory: "8Gi"
              cpu: "1"
            limits:
              memory: "16Gi"
              cpu: "2"
---
# Schedule batch job to spot pool
apiVersion: batch/v1
kind: Job
metadata:
  name: ml-training
spec:
  template:
    spec:
      nodeSelector:
        workload: batch
      tolerations:
        - key: kubernetes.azure.com/scalesetpriority
          operator: Equal
          value: spot
          effect: NoSchedule
      restartPolicy: OnFailure  # Critical for spot!
      containers:
        - name: trainer
          image: myacr.azurecr.io/ml-trainer:v1
```

---

## Pattern 2: Availability Zone Design

### ❌ Anti-Pattern: Single Zone Deployment

```
THE "WE'LL BE FINE" SINGLE ZONE:
────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   AVAILABILITY ZONE 1 ONLY                                                  │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │                                                                        ││
│   │   ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐        ││
│   │   │ Node 1 │  │ Node 2 │  │ Node 3 │  │ Node 4 │  │ Node 5 │        ││
│   │   └────────┘  └────────┘  └────────┘  └────────┘  └────────┘        ││
│   │                                                                        ││
│   │   ┌────────────────────────────────────────────────────────────────┐ ││
│   │   │ Azure Disk (LRS) │ Azure Disk (LRS) │ Azure Disk (LRS)        │ ││
│   │   │ Zone 1 only      │ Zone 1 only      │ Zone 1 only             │ ││
│   │   └────────────────────────────────────────────────────────────────┘ ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   ⚡ ZONE 1 GOES DOWN ⚡                                                     │
│                                                                              │
│   RESULT:                                                                    │
│   ├── ALL nodes unavailable                                                 │
│   ├── ALL persistent volumes inaccessible                                   │
│   ├── Complete application outage                                           │
│   ├── Manual intervention required to recover                              │
│   └── Data potentially lost if not replicated                              │
│                                                                              │
│   REAL INCIDENT (Azure, September 2023):                                    │
│   "Zone 1 cooling failure in South Central US affected all workloads       │
│    not distributed across zones. Multi-hour outage for single-zone         │
│    deployments."                                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ✅ Good Pattern: Zone-Redundant Architecture

```
ZONE-REDUNDANT AKS ARCHITECTURE:
────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                           AZURE REGION (East US)                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐         │
│   │ AVAILABILITY     │  │ AVAILABILITY     │  │ AVAILABILITY     │         │
│   │ ZONE 1           │  │ ZONE 2           │  │ ZONE 3           │         │
│   │                  │  │                  │  │                  │         │
│   │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │         │
│   │  │  Node 1    │  │  │  │  Node 2    │  │  │  │  Node 3    │  │         │
│   │  │  (system)  │  │  │  │  (system)  │  │  │  │  (system)  │  │         │
│   │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │         │
│   │                  │  │                  │  │                  │         │
│   │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │         │
│   │  │  Node 4    │  │  │  │  Node 5    │  │  │  │  Node 6    │  │         │
│   │  │  (user)    │  │  │  │  (user)    │  │  │  │  (user)    │  │         │
│   │  │            │  │  │  │            │  │  │  │            │  │         │
│   │  │ ┌────────┐ │  │  │  │ ┌────────┐ │  │  │  │ ┌────────┐ │  │         │
│   │  │ │ Pod A  │ │  │  │  │ │ Pod A  │ │  │  │  │ │ Pod A  │ │  │         │
│   │  │ │(replica│ │  │  │  │ │(replica│ │  │  │  │ │(replica│ │  │         │
│   │  │ │  1)    │ │  │  │  │ │  2)    │ │  │  │  │ │  3)    │ │  │         │
│   │  │ └────────┘ │  │  │  │ └────────┘ │  │  │  │ └────────┘ │  │         │
│   │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │         │
│   │                  │  │                  │  │                  │         │
│   │  ┌────────────┐  │  │  ┌────────────┐  │  │  ┌────────────┐  │         │
│   │  │ ZRS Disk   │  │  │  │ ZRS Disk   │  │  │  │ ZRS Disk   │  │         │
│   │  │ (replicated│  │  │  │ (replicated│  │  │  │ (replicated│  │         │
│   │  │ across all │  │  │  │ across all │  │  │  │ across all │  │         │
│   │  │ zones)     │  │  │  │ zones)     │  │  │  │ zones)     │  │         │
│   │  └────────────┘  │  │  └────────────┘  │  │  └────────────┘  │         │
│   │                  │  │                  │  │                  │         │
│   └──────────────────┘  └──────────────────┘  └──────────────────┘         │
│                                                                              │
│   ⚡ ZONE 1 GOES DOWN ⚡                                                     │
│                                                                              │
│   RESULT:                                                                    │
│   ├── Zones 2 & 3 continue serving traffic                                 │
│   ├── Load balancer routes around failed zone                              │
│   ├── Stateful workloads fail over to healthy zone                        │
│   ├── Automatic recovery when zone returns                                 │
│   └── Zero data loss with ZRS storage                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Zone-Aware Pod Distribution

```yaml
# Topology spread constraints - distribute pods across zones
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-api
spec:
  replicas: 6  # Multiple of 3 for even distribution
  selector:
    matchLabels:
      app: web-api
  template:
    metadata:
      labels:
        app: web-api
    spec:
      topologySpreadConstraints:
        # Spread across zones (CRITICAL for HA)
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: web-api
        # Also spread across nodes within zones
        - maxSkew: 1
          topologyKey: kubernetes.io/hostname
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: web-api
      containers:
        - name: api
          image: myacr.azurecr.io/web-api:v1
          resources:
            requests:
              cpu: "500m"
              memory: "512Mi"
---
# Zone-redundant storage class
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi-premium-zrs
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_ZRS  # Zone-redundant storage
  cachingMode: ReadOnly
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

---

## Pattern 3: Autoscaling Strategy

### ❌ Anti-Pattern: No Autoscaling or Misconfigured

```
AUTOSCALING ANTI-PATTERNS:
──────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   ANTI-PATTERN 1: No Autoscaling                                           │
│   ───────────────────────────────                                          │
│   "We'll just provision for peak and leave it"                             │
│                                                                              │
│   Peak Traffic ────────────────────────┐                                    │
│                                        │                                    │
│   Provisioned  ═══════════════════════════════════════════                 │
│   Capacity                             │                                    │
│                                        │                                    │
│   Actual       ──────/\──────/\───────/\────────────────                   │
│   Traffic          /  \    /  \     /  \                                   │
│               ────/    \──/    \───/    \────────────                      │
│                                                                              │
│   RESULT: 60-70% of capacity wasted, $$$$ thrown away                      │
│                                                                              │
│   ─────────────────────────────────────────────────────────────────────    │
│                                                                              │
│   ANTI-PATTERN 2: HPA Only (No Cluster Autoscaler)                         │
│   ─────────────────────────────────────────────────                        │
│                                                                              │
│   Pods:    [Pod][Pod][Pod][PENDING][PENDING][PENDING]                      │
│   Nodes:   [Node 1    ][Node 2    ][Node 3    ]                            │
│                         ↑                                                   │
│            "No room for more pods, but cluster doesn't scale!"             │
│                                                                              │
│   ─────────────────────────────────────────────────────────────────────    │
│                                                                              │
│   ANTI-PATTERN 3: Aggressive Scale-Down                                    │
│   ─────────────────────────────────────                                    │
│                                                                              │
│   10:00 - Traffic spike → Scale up to 20 pods                              │
│   10:05 - Traffic drops → Scale down to 3 pods                             │
│   10:06 - Traffic returns → Scale up (cold start latency!)                 │
│   10:10 - Traffic drops → Scale down again                                 │
│   ...repeat... (thrashing)                                                  │
│                                                                              │
│   RESULT: User-visible latency spikes, wasted scale operations             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ✅ Good Pattern: Layered Autoscaling

```
THREE-LAYER AUTOSCALING ARCHITECTURE:
─────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   LAYER 1: KEDA (Event-Driven Pod Autoscaling)                             │
│   ────────────────────────────────────────────                             │
│   Scales based on external metrics (queue depth, custom metrics)           │
│                                                                              │
│   ┌────────────────────────────────────────────────────────────────────┐   │
│   │                                                                     │   │
│   │  Service Bus Queue: 1000 messages                                  │   │
│   │         │                                                           │   │
│   │         ▼                                                           │   │
│   │  ┌──────────────┐                                                  │   │
│   │  │     KEDA     │ → "Queue > 100 messages? Scale to process!"     │   │
│   │  │  ScaledObject│                                                  │   │
│   │  └──────┬───────┘                                                  │   │
│   │         │                                                           │   │
│   │         ▼                                                           │   │
│   │  [Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod][Pod]               │   │
│   │  (Scaled from 1 to 10 based on queue depth)                        │   │
│   │                                                                     │   │
│   └────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   LAYER 2: HPA (Horizontal Pod Autoscaler)                                 │
│   ────────────────────────────────────────                                 │
│   Scales based on CPU/Memory or custom metrics                             │
│                                                                              │
│   ┌────────────────────────────────────────────────────────────────────┐   │
│   │                                                                     │   │
│   │  CPU Utilization: 85%                                              │   │
│   │         │                                                           │   │
│   │         ▼                                                           │   │
│   │  ┌──────────────┐                                                  │   │
│   │  │     HPA      │ → "CPU > 70%? Add more pods!"                   │   │
│   │  │              │                                                  │   │
│   │  └──────┬───────┘                                                  │   │
│   │         │                                                           │   │
│   │         ▼                                                           │   │
│   │  [Pod][Pod][Pod] → [Pod][Pod][Pod][Pod][Pod]                      │   │
│   │                                                                     │   │
│   └────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   LAYER 3: Cluster Autoscaler                                              │
│   ───────────────────────────                                              │
│   Scales nodes when pods can't be scheduled                                │
│                                                                              │
│   ┌────────────────────────────────────────────────────────────────────┐   │
│   │                                                                     │   │
│   │  Pending Pods: 15 (no room on existing nodes)                      │   │
│   │         │                                                           │   │
│   │         ▼                                                           │   │
│   │  ┌──────────────┐                                                  │   │
│   │  │   Cluster    │ → "Pending pods? Add nodes!"                    │   │
│   │  │  Autoscaler  │                                                  │   │
│   │  └──────┬───────┘                                                  │   │
│   │         │                                                           │   │
│   │         ▼                                                           │   │
│   │  [Node 1][Node 2][Node 3] → [Node 1][Node 2][Node 3][Node 4][5]   │   │
│   │                                                                     │   │
│   └────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   TIMING CONSIDERATIONS:                                                    │
│   ├── KEDA: Seconds (reacts to events immediately)                         │
│   ├── HPA: 15-30 seconds (metric collection interval)                      │
│   └── Cluster Autoscaler: 2-5 minutes (VM provisioning time)              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```yaml
# HPA with stabilization windows to prevent thrashing
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-api
  minReplicas: 3
  maxReplicas: 100
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0  # Scale up immediately
      policies:
        - type: Percent
          value: 100  # Can double pod count
          periodSeconds: 15
        - type: Pods
          value: 4
          periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
        - type: Percent
          value: 10  # Scale down gradually (10% at a time)
          periodSeconds: 60
---
# KEDA ScaledObject for queue-based scaling
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor-scaler
spec:
  scaleTargetRef:
    name: order-processor
  pollingInterval: 15
  cooldownPeriod: 300
  minReplicaCount: 1
  maxReplicaCount: 50
  triggers:
    - type: azure-servicebus
      metadata:
        queueName: orders
        namespace: orders-ns
        messageCount: "10"  # Scale when > 10 messages per replica
      authenticationRef:
        name: azure-servicebus-auth
---
# Cluster autoscaler profile (in AKS cluster config)
# az aks update with autoscaler profile:
# --cluster-autoscaler-profile \
#   scan-interval=10s \
#   scale-down-delay-after-add=10m \
#   scale-down-delay-after-delete=10s \
#   scale-down-delay-after-failure=3m \
#   scale-down-unneeded-time=10m \
#   scale-down-unready-time=20m \
#   scale-down-utilization-threshold=0.5 \
#   max-graceful-termination-sec=600 \
#   balance-similar-node-groups=true \
#   expander=least-waste
```

---

## Pattern 4: Network Security (Zero Trust)

### ❌ Anti-Pattern: Flat Network, No Policies

```
THE "TRUST EVERYONE" NETWORK:
─────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │                     FLAT NETWORK (No Policies)                        ││
│   │                                                                        ││
│   │   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐       ││
│   │   │ Frontend │◄──►│ Backend  │◄──►│ Database │◄──►│ Payments │       ││
│   │   └──────────┘    └──────────┘    └──────────┘    └──────────┘       ││
│   │        ▲               ▲               ▲               ▲              ││
│   │        │               │               │               │              ││
│   │        └───────────────┼───────────────┼───────────────┘              ││
│   │                        │               │                              ││
│   │                   ALL PODS CAN TALK TO ALL PODS                       ││
│   │                                                                        ││
│   │   ATTACK SCENARIO:                                                    ││
│   │   1. Attacker compromises Frontend pod (XSS vulnerability)           ││
│   │   2. From Frontend, attacker can reach Database directly!            ││
│   │   3. From Frontend, attacker can reach Payments service!             ││
│   │   4. Lateral movement is trivial                                     ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   REAL INCIDENT:                                                            │
│   "Attacker exploited a logging library vulnerability in one pod,          │
│    then used the flat network to enumerate and attack every other          │
│    service. Complete data breach in under 2 hours."                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ✅ Good Pattern: Zero Trust with Network Policies

```
ZERO TRUST NETWORK ARCHITECTURE:
────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   DEFAULT: DENY ALL                                                         │
│   ─────────────────                                                         │
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │  NAMESPACE: frontend                                                   ││
│   │  ┌──────────────────────────────────────────────────────────────────┐ ││
│   │  │ Network Policy: Allow ingress from Ingress Controller only       │ ││
│   │  │                 Allow egress to backend namespace only           │ ││
│   │  │                                                                   │ ││
│   │  │   ┌──────────┐                                                   │ ││
│   │  │   │ Frontend │──────────────────────────────────┐                │ ││
│   │  │   └──────────┘                                  │                │ ││
│   │  └─────────────────────────────────────────────────┼────────────────┘ ││
│   └────────────────────────────────────────────────────┼──────────────────┘│
│                                                        │                    │
│                                                        ▼                    │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │  NAMESPACE: backend                                                    ││
│   │  ┌──────────────────────────────────────────────────────────────────┐ ││
│   │  │ Network Policy: Allow ingress from frontend namespace only       │ ││
│   │  │                 Allow egress to database namespace only          │ ││
│   │  │                                                                   │ ││
│   │  │   ┌──────────┐                                                   │ ││
│   │  │   │ Backend  │──────────────────────────────────┐                │ ││
│   │  │   └──────────┘                                  │                │ ││
│   │  └─────────────────────────────────────────────────┼────────────────┘ ││
│   └────────────────────────────────────────────────────┼──────────────────┘│
│                                                        │                    │
│                                                        ▼                    │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │  NAMESPACE: database                                                   ││
│   │  ┌──────────────────────────────────────────────────────────────────┐ ││
│   │  │ Network Policy: Allow ingress from backend namespace only        │ ││
│   │  │                 Deny ALL egress (database shouldn't initiate)    │ ││
│   │  │                                                                   │ ││
│   │  │   ┌──────────┐                                                   │ ││
│   │  │   │ Database │ ✗ Cannot reach Frontend                          │ ││
│   │  │   │ Pod      │ ✗ Cannot reach Payments                          │ ││
│   │  │   └──────────┘ ✗ Cannot reach Internet                          │ ││
│   │  └──────────────────────────────────────────────────────────────────┘ ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   ATTACK SCENARIO (MITIGATED):                                              │
│   1. Attacker compromises Frontend pod                                      │
│   2. ✗ Cannot reach Database (blocked by network policy)                   │
│   3. ✗ Cannot reach Payments (blocked by network policy)                   │
│   4. Can only reach Backend (intended behavior)                            │
│   5. Blast radius contained to Frontend namespace                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```yaml
# Default deny all ingress and egress in namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: backend
spec:
  podSelector: {}  # Applies to all pods
  policyTypes:
    - Ingress
    - Egress
---
# Allow backend to receive traffic from frontend only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend-api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: frontend
          podSelector:
            matchLabels:
              app: frontend-web
      ports:
        - protocol: TCP
          port: 8080
---
# Allow backend to talk to database only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend-api
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              name: database
          podSelector:
            matchLabels:
              app: postgresql
      ports:
        - protocol: TCP
          port: 5432
    # Allow DNS resolution
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
---
# Database: Only accept connections from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress
  namespace: database
spec:
  podSelector:
    matchLabels:
      app: postgresql
  policyTypes:
    - Ingress
    - Egress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: backend
      ports:
        - protocol: TCP
          port: 5432
  egress: []  # Deny all egress - database shouldn't initiate connections
```

---

## Pattern 5: Private Cluster Architecture

### ❌ Anti-Pattern: Public API Server

```
PUBLIC API SERVER EXPOSURE:
───────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   INTERNET                                                                   │
│       │                                                                      │
│       │  kubectl from anywhere                                              │
│       │  (with valid kubeconfig)                                            │
│       │                                                                      │
│       ▼                                                                      │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │                     PUBLIC API SERVER                                  ││
│   │                     (api.prod-aks.eastus.azmk8s.io)                   ││
│   │                                                                        ││
│   │   RISKS:                                                              ││
│   │   ├── API server exposed to entire internet                           ││
│   │   ├── Brute force attacks on authentication                           ││
│   │   ├── Zero-day exploits against API server                           ││
│   │   ├── Credential theft = cluster compromise from anywhere            ││
│   │   └── DDoS attacks on control plane                                  ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   COMPLIANCE ISSUE:                                                         │
│   Many regulations (PCI-DSS, HIPAA, SOC2) require that management          │
│   interfaces NOT be accessible from the public internet.                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ✅ Good Pattern: Private Cluster with Secure Access

```
PRIVATE CLUSTER ARCHITECTURE:
─────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   INTERNET                                                                   │
│       │                                                                      │
│       │  ✗ Cannot reach API server                                         │
│       │                                                                      │
│       └────────── X ──────────────────────────────┐                        │
│                                                    │                        │
│   ┌────────────────────────────────────────────────┼────────────────────┐  │
│   │                          AZURE                 │                     │  │
│   │                                                │                     │  │
│   │   OPTION 1: Developer VPN Access              │                     │  │
│   │   ─────────────────────────────               │                     │  │
│   │   ┌────────────┐    ┌─────────────┐          │                     │  │
│   │   │ Developer  │───►│ VPN Gateway │          │                     │  │
│   │   │ Laptop     │    │ (P2S VPN)   │          │                     │  │
│   │   └────────────┘    └──────┬──────┘          │                     │  │
│   │                            │                  │                     │  │
│   │   OPTION 2: Bastion Host   │                  │                     │  │
│   │   ──────────────────────   │                  │                     │  │
│   │   ┌────────────┐    ┌──────┴──────┐          │                     │  │
│   │   │ Azure      │───►│ Jump Box    │          │                     │  │
│   │   │ Bastion    │    │ VM in VNet  │          │                     │  │
│   │   └────────────┘    └──────┬──────┘          │                     │  │
│   │                            │                  │                     │  │
│   │   OPTION 3: GitHub Actions │                  │                     │  │
│   │   ─────────────────────────│                  │                     │  │
│   │   ┌────────────┐    ┌──────┴──────┐          │                     │  │
│   │   │ GitHub     │───►│Self-Hosted  │          │                     │  │
│   │   │ Actions    │    │Runner in    │          │                     │  │
│   │   │            │    │VNet         │          │                     │  │
│   │   └────────────┘    └──────┬──────┘          │                     │  │
│   │                            │                  │                     │  │
│   │                            ▼                  │                     │  │
│   │   ┌────────────────────────────────────────────────────────────┐  │  │
│   │   │                 AKS CLUSTER (PRIVATE)                       │  │  │
│   │   │                                                             │  │  │
│   │   │   ┌─────────────────────────────────────────────────────┐  │  │  │
│   │   │   │          PRIVATE API SERVER                          │  │  │  │
│   │   │   │          (Private IP: 10.0.0.100)                   │  │  │  │
│   │   │   │                                                      │  │  │  │
│   │   │   │  Private DNS Zone: privatelink.eastus.azmk8s.io     │  │  │  │
│   │   │   │  Resolved only within VNet                          │  │  │  │
│   │   │   │                                                      │  │  │  │
│   │   │   │  ✓ kubectl works from VNet                          │  │  │  │
│   │   │   │  ✓ CI/CD works from self-hosted runners             │  │  │  │
│   │   │   │  ✗ Cannot access from public internet               │  │  │  │
│   │   │   └─────────────────────────────────────────────────────┘  │  │  │
│   │   │                                                             │  │  │
│   │   │   ┌───────────────────────────────────────────────────┐    │  │  │
│   │   │   │              NODE POOLS                            │    │  │  │
│   │   │   │  (Also in private subnets, no public IPs)         │    │  │  │
│   │   │   └───────────────────────────────────────────────────┘    │  │  │
│   │   │                                                             │  │  │
│   │   └─────────────────────────────────────────────────────────────┘  │  │
│   │                                                                     │  │
│   └─────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```bash
# Create private AKS cluster
az aks create \
  --resource-group aks-rg \
  --name private-prod-aks \
  --enable-private-cluster \
  --private-dns-zone system \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --enable-managed-identity \
  --disable-public-fqdn \
  --api-server-authorized-ip-ranges ""

# Access from jump box
az network bastion ssh \
  --name myBastion \
  --resource-group bastion-rg \
  --target-resource-id /subscriptions/.../vms/jumpbox \
  --auth-type password

# Or use AKS command invoke (no jump box needed)
az aks command invoke \
  --resource-group aks-rg \
  --name private-prod-aks \
  --command "kubectl get nodes"

# For CI/CD: Self-hosted runner in VNet
# GitHub Actions workflow:
# runs-on: self-hosted  # Runner in VNet with kubectl access
```

---

## Pattern 6: Resource Quotas and Limits

### ❌ Anti-Pattern: No Resource Limits

```
THE "UNLIMITED RESOURCES" DISASTER:
───────────────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   NODE: 16 CPU, 64GB Memory                                                 │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │                                                                        ││
│   │   ┌──────────────────────────────────────────────────────────────┐   ││
│   │   │  Pod: memory-leak-app (No limits set)                        │   ││
│   │   │                                                               │   ││
│   │   │  Memory Usage: 500MB... 2GB... 8GB... 32GB... 60GB...       │   ││
│   │   │                                                               │   ││
│   │   │  "I'll just keep allocating until someone stops me!"         │   ││
│   │   └──────────────────────────────────────────────────────────────┘   ││
│   │                                                                        ││
│   │   ┌──────────────────────────────────────────────────────────────┐   ││
│   │   │  Pod: critical-api (Also no limits)                          │   ││
│   │   │                                                               │   ││
│   │   │  Memory: OOM KILLED! Node ran out of memory!                 │   ││
│   │   │                                                               │   ││
│   │   └──────────────────────────────────────────────────────────────┘   ││
│   │                                                                        ││
│   │   ┌──────────────────────────────────────────────────────────────┐   ││
│   │   │  Pod: CoreDNS (System critical)                              │   ││
│   │   │                                                               │   ││
│   │   │  OOM KILLED! Cluster DNS is down!                            │   ││
│   │   │                                                               │   ││
│   │   └──────────────────────────────────────────────────────────────┘   ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   RESULT: One bad actor brought down the entire node                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ✅ Good Pattern: Layered Resource Governance

```
RESOURCE GOVERNANCE LAYERS:
───────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   LAYER 1: Namespace Resource Quotas (Team/Project Level)                  │
│   ─────────────────────────────────────────────────────                    │
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │  NAMESPACE: team-payments                                              ││
│   │                                                                        ││
│   │  ResourceQuota:                                                       ││
│   │  ├── requests.cpu: 20 cores (total for namespace)                    ││
│   │  ├── requests.memory: 40Gi                                           ││
│   │  ├── limits.cpu: 40 cores                                            ││
│   │  ├── limits.memory: 80Gi                                             ││
│   │  ├── pods: 100 (max pods)                                            ││
│   │  ├── services.loadbalancers: 2                                       ││
│   │  └── persistentvolumeclaims: 20                                      ││
│   │                                                                        ││
│   │  Current Usage:                                                       ││
│   │  ├── CPU: 12/20 cores (60%)                                          ││
│   │  ├── Memory: 28/40Gi (70%)                                           ││
│   │  └── Pods: 45/100 (45%)                                              ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   LAYER 2: LimitRange (Default Limits for Pods)                            │
│   ─────────────────────────────────────────────                            │
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │  LimitRange: default-limits                                           ││
│   │                                                                        ││
│   │  Container Defaults:                                                  ││
│   │  ├── Default request: 100m CPU, 128Mi memory                         ││
│   │  ├── Default limit: 500m CPU, 512Mi memory                           ││
│   │  ├── Max limit: 4 CPU, 8Gi memory                                    ││
│   │  └── Min request: 50m CPU, 64Mi memory                               ││
│   │                                                                        ││
│   │  Effect: Pods without limits get sensible defaults                   ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   LAYER 3: Pod Resource Requests & Limits                                  │
│   ───────────────────────────────────────                                  │
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │  Pod: payment-api                                                      ││
│   │                                                                        ││
│   │  resources:                                                           ││
│   │    requests:           # Guaranteed resources                         ││
│   │      cpu: 500m         # 0.5 CPU cores reserved                      ││
│   │      memory: 512Mi     # 512MB reserved                              ││
│   │    limits:             # Maximum allowed                              ││
│   │      cpu: 2000m        # Can burst to 2 cores                        ││
│   │      memory: 2Gi       # Killed if exceeds 2GB                       ││
│   │                                                                        ││
│   │  QoS Class: Burstable (requests < limits)                            ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
│   LAYER 4: Priority Classes (Eviction Priority)                            │
│   ─────────────────────────────────────────────                            │
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐│
│   │                                                                        ││
│   │  system-cluster-critical (1000000000) - CoreDNS, kube-proxy          ││
│   │           │                                                           ││
│   │           ▼                                                           ││
│   │  system-node-critical (900000000) - CNI, CSI drivers                 ││
│   │           │                                                           ││
│   │           ▼                                                           ││
│   │  high-priority (100000) - Payment processing                         ││
│   │           │                                                           ││
│   │           ▼                                                           ││
│   │  normal (0) - Standard workloads                                     ││
│   │           │                                                           ││
│   │           ▼                                                           ││
│   │  low-priority (-100) - Batch jobs, can be evicted                    ││
│   │                                                                        ││
│   │  Under memory pressure: Low priority evicted first                   ││
│   │                                                                        ││
│   └───────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```yaml
# Namespace Resource Quota
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-payments
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    pods: "100"
    services.loadbalancers: "2"
    persistentvolumeclaims: "20"
    count/deployments.apps: "20"
    count/services: "30"
---
# LimitRange - Default limits for pods
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: team-payments
spec:
  limits:
    - type: Container
      default:          # Default limits if not specified
        cpu: 500m
        memory: 512Mi
      defaultRequest:   # Default requests if not specified
        cpu: 100m
        memory: 128Mi
      max:             # Maximum allowed
        cpu: "4"
        memory: 8Gi
      min:             # Minimum required
        cpu: 50m
        memory: 64Mi
    - type: PersistentVolumeClaim
      max:
        storage: 100Gi
      min:
        storage: 1Gi
---
# Priority Class for critical workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "High priority for payment processing workloads"
preemptionPolicy: PreemptLowerPriority
---
# Priority Class for batch jobs
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: -100
globalDefault: false
description: "Low priority for batch jobs - can be evicted"
preemptionPolicy: Never
---
# Well-configured deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-api
  namespace: team-payments
spec:
  replicas: 3
  template:
    spec:
      priorityClassName: high-priority
      containers:
        - name: api
          image: myacr.azurecr.io/payment-api:v1
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
          # Liveness/readiness probes ensure unhealthy pods are replaced
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /ready
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
```

---

## Pattern 7: Upgrade Strategy

### ❌ Anti-Pattern: Ignore Updates or Big Bang Upgrades

```
UPGRADE ANTI-PATTERNS:
──────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   ANTI-PATTERN 1: "Never Upgrade"                                          │
│   ───────────────────────────────                                          │
│                                                                              │
│   Current Version: 1.24 (Released 2 years ago)                             │
│   Latest Version: 1.29                                                      │
│   Supported: NO (1.24 is out of support!)                                  │
│                                                                              │
│   CONSEQUENCES:                                                             │
│   ├── No security patches                                                  │
│   ├── No bug fixes                                                         │
│   ├── Azure support won't help                                             │
│   ├── Can't use new features                                               │
│   └── Eventually forced to upgrade 5+ versions at once                     │
│                                                                              │
│   ─────────────────────────────────────────────────────────────────────    │
│                                                                              │
│   ANTI-PATTERN 2: "Big Bang Upgrade"                                       │
│   ───────────────────────────────────                                      │
│                                                                              │
│   Friday 5pm: "Let's upgrade all clusters to 1.29!"                        │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐ │
│   │ PROD CLUSTER                                                         │ │
│   │                                                                      │ │
│   │ 1.26 ─────────────────────────────────────────────────────► 1.29   │ │
│   │                                                                      │ │
│   │ Saturday 2am: "Why are all pods crashing?!"                         │ │
│   │ Deprecated APIs removed in 1.28, app wasn't tested!                 │ │
│   │                                                                      │ │
│   └─────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### ✅ Good Pattern: Blue-Green with Staged Rollout

```
STAGED UPGRADE STRATEGY:
────────────────────────

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                              │
│   KUBERNETES SUPPORT POLICY:                                                │
│   ─────────────────────────                                                │
│   Azure supports: Current version (N) + two previous (N-1, N-2)            │
│                                                                              │
│   Current: 1.29 │ Supported: 1.29, 1.28, 1.27 │ Unsupported: 1.26, ...    │
│                                                                              │
│   RULE: Stay within N-2 and upgrade every 3-4 months                       │
│                                                                              │
│   ───────────────────────────────────────────────────────────────────────  │
│                                                                              │
│   STAGED UPGRADE APPROACH:                                                  │
│                                                                              │
│   Week 1: DEV CLUSTER                                                       │
│   ────────────────────                                                      │
│   ┌─────────────────────────────────────────────────────────────────────┐ │
│   │ DEV: 1.27 → 1.28                                                     │ │
│   │                                                                      │ │
│   │ ✓ Test API compatibility                                             │ │
│   │ ✓ Run integration tests                                              │ │
│   │ ✓ Check for deprecated APIs (kubectl deprecations)                   │ │
│   │ ✓ Verify monitoring works                                            │ │
│   │ ✓ 1 week soak time                                                   │ │
│   └─────────────────────────────────────────────────────────────────────┘ │
│              │                                                              │
│              ▼                                                              │
│   Week 2-3: STAGING CLUSTER                                                │
│   ─────────────────────────                                                │
│   ┌─────────────────────────────────────────────────────────────────────┐ │
│   │ STAGING: 1.27 → 1.28                                                 │ │
│   │                                                                      │ │
│   │ ✓ Production-like workloads                                          │ │
│   │ ✓ Performance testing                                                │ │
│   │ ✓ Load testing                                                       │ │
│   │ ✓ 1-2 week soak time                                                 │ │
│   └─────────────────────────────────────────────────────────────────────┘ │
│              │                                                              │
│              ▼                                                              │
│   Week 4: PRODUCTION (Blue-Green Node Pools)                               │
│   ──────────────────────────────────────────                               │
│   ┌─────────────────────────────────────────────────────────────────────┐ │
│   │ PROD: Blue-Green Node Pool Upgrade                                   │ │
│   │                                                                      │ │
│   │   STEP 1: Upgrade Control Plane                                      │ │
│   │   ┌──────────────────────────────────────────────────────────────┐  │ │
│   │   │ Control Plane: 1.27 → 1.28 (managed, zero downtime)          │  │ │
│   │   └──────────────────────────────────────────────────────────────┘  │ │
│   │                                                                      │ │
│   │   STEP 2: Create new node pool (Green) with 1.28                    │ │
│   │   ┌──────────────────────────────────────────────────────────────┐  │ │
│   │   │ Blue Pool (1.27)    │    Green Pool (1.28)                   │  │ │
│   │   │ ┌────┐┌────┐┌────┐ │    ┌────┐┌────┐┌────┐                  │  │ │
│   │   │ │Node││Node││Node│ │    │Node││Node││Node│ ← New nodes      │  │ │
│   │   │ └────┘└────┘└────┘ │    └────┘└────┘└────┘                  │  │ │
│   │   │       (pods here)   │    (ready, no pods yet)                │  │ │
│   │   └──────────────────────────────────────────────────────────────┘  │ │
│   │                                                                      │ │
│   │   STEP 3: Cordon blue pool, drain pods to green                     │ │
│   │   ┌──────────────────────────────────────────────────────────────┐  │ │
│   │   │ Blue Pool (CORDONED) │    Green Pool (1.28)                  │  │ │
│   │   │ ┌────┐┌────┐┌────┐  │    ┌────┐┌────┐┌────┐                 │  │ │
│   │   │ │    ││    ││    │  │    │Pod ││Pod ││Pod │ ← Pods migrated │  │ │
│   │   │ └────┘└────┘└────┘  │    └────┘└────┘└────┘                 │  │ │
│   │   │   (draining...)     │    (serving traffic)                   │  │ │
│   │   └──────────────────────────────────────────────────────────────┘  │ │
│   │                                                                      │ │
│   │   STEP 4: Delete blue pool (after validation)                       │ │
│   │   ┌──────────────────────────────────────────────────────────────┐  │ │
│   │   │                      Green Pool (1.28)                       │  │ │
│   │   │                      ┌────┐┌────┐┌────┐                     │  │ │
│   │   │                      │Pod ││Pod ││Pod │ ✓ Upgrade complete  │  │ │
│   │   │                      └────┘└────┘└────┘                     │  │ │
│   │   └──────────────────────────────────────────────────────────────┘  │ │
│   │                                                                      │ │
│   │   ROLLBACK: If issues, uncordon blue, cordon green, drain back      │ │
│   │                                                                      │ │
│   └─────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```bash
#!/bin/bash
# Blue-Green Node Pool Upgrade Script

RESOURCE_GROUP="aks-rg"
CLUSTER_NAME="prod-aks"
OLD_POOL="nodepool1"
NEW_POOL="nodepool2"
NEW_VERSION="1.28.0"

# Step 1: Upgrade control plane first
echo "Upgrading control plane to $NEW_VERSION..."
az aks upgrade \
  --resource-group $RESOURCE_GROUP \
  --name $CLUSTER_NAME \
  --kubernetes-version $NEW_VERSION \
  --control-plane-only \
  --yes

# Step 2: Create new node pool with new version
echo "Creating green node pool..."
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER_NAME \
  --name $NEW_POOL \
  --kubernetes-version $NEW_VERSION \
  --node-count 3 \
  --node-vm-size Standard_D8s_v5 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 50

# Wait for nodes to be ready
echo "Waiting for green nodes to be ready..."
kubectl wait --for=condition=Ready nodes -l agentpool=$NEW_POOL --timeout=600s

# Step 3: Cordon old nodes (prevent new pods)
echo "Cordoning blue nodes..."
kubectl cordon -l agentpool=$OLD_POOL

# Step 4: Drain old nodes (evict pods to new pool)
echo "Draining blue nodes..."
for node in $(kubectl get nodes -l agentpool=$OLD_POOL -o name); do
  kubectl drain $node \
    --ignore-daemonsets \
    --delete-emptydir-data \
    --force \
    --grace-period=300
done

# Step 5: Validate (manual check point)
echo "=== VALIDATION CHECKPOINT ==="
echo "Verify application health before deleting old pool:"
echo "- Check pod status: kubectl get pods -A"
echo "- Check endpoints: curl https://api.example.com/health"
echo "- Check metrics in Grafana/Azure Monitor"
read -p "Ready to delete old pool? (yes/no) " confirm

if [ "$confirm" = "yes" ]; then
  # Step 6: Delete old node pool
  echo "Deleting blue node pool..."
  az aks nodepool delete \
    --resource-group $RESOURCE_GROUP \
    --cluster-name $CLUSTER_NAME \
    --name $OLD_POOL \
    --yes

  echo "Upgrade complete!"
else
  echo "Rolling back..."
  kubectl uncordon -l agentpool=$OLD_POOL
  echo "Rollback complete. Blue pool restored."
fi
```

---

## Architecture Decision Checklist

```
PRODUCTION AKS CHECKLIST:
─────────────────────────

NODE POOLS:
☐ Separate system pool (tainted, 3+ nodes)
☐ User pools sized for workload types
☐ Spot pools for batch/interruptible
☐ GPU pools if needed (tainted)
☐ All pools span 3 availability zones

NETWORKING:
☐ Azure CNI (not kubenet for production)
☐ Private cluster for sensitive workloads
☐ Network policies enabled (Azure or Calico)
☐ Default deny policies in place
☐ Service mesh if needed (Istio/Linkerd)

IDENTITY & SECURITY:
☐ Entra ID integration enabled
☐ Azure RBAC for Kubernetes
☐ Workload Identity (not pod identity v1)
☐ Defender for Containers enabled
☐ Azure Policy add-on enabled
☐ Image scanning in ACR

AUTOSCALING:
☐ Cluster autoscaler enabled
☐ HPA configured with appropriate metrics
☐ KEDA for event-driven workloads
☐ Stabilization windows configured
☐ Pod Disruption Budgets defined

RESOURCE GOVERNANCE:
☐ Resource quotas per namespace
☐ LimitRange defaults set
☐ All pods have requests AND limits
☐ Priority classes defined

OBSERVABILITY:
☐ Container Insights enabled
☐ Prometheus metrics enabled
☐ Log Analytics workspace configured
☐ Alerts for key metrics
☐ Distributed tracing (if microservices)

DISASTER RECOVERY:
☐ Multi-zone deployment
☐ ZRS storage for stateful workloads
☐ Backup strategy (Velero or native)
☐ DR cluster in another region (if required)
☐ Documented recovery procedures

UPGRADES:
☐ Stay within N-2 support window
☐ Automated upgrade testing in dev
☐ Blue-green upgrade strategy
☐ Rollback procedures documented
```

---

*Navigation: [Container Apps](02-container-apps.md) | [README](README.md) | [Case Studies](case-studies.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
