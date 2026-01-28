# Architecture Styles Quick Reference

## Architecture Styles at a Glance

```
ARCHITECTURE STYLES OVERVIEW:
════════════════════════════

┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                               │
│  STYLE           USE CASE                COMPLEXITY    SCALING              │
│  ─────────────────────────────────────────────────────────────────────────  │
│                                                                               │
│  N-Tier          Traditional apps        Low           Tier-based            │
│                  Web/business apps                                           │
│                                                                               │
│  Microservices   Independent services    High          Service-based         │
│                  Polyglot systems                      (horizontal)          │
│                                                                               │
│  Event-Driven    Real-time processing    Medium        Event topic-based     │
│                  Reactive systems                      (publish-subscribe)   │
│                                                                               │
│  Big Data        Analytics & insights    High          Distributed parallel  │
│                  Data pipelines                        processing            │
│                                                                               │
│  Big Compute     Simulations, rendering  Very High     Node-based pooling    │
│                  Scientific computing                  (HPC)                 │
│                                                                               │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Quick Decision Matrix

```
CHOOSING YOUR ARCHITECTURE STYLE:
═════════════════════════════════

                    ┌─────────────────────────────────────────┐
                    │ Application Architecture?               │
                    └──────────────────┬──────────────────────┘
                                       │
                    ┌──────────────────┼──────────────────────┐
                    │                  │                      │
                    ▼                  ▼                      ▼
            ┌───────────────┐   ┌────────────────┐   ┌─────────────────┐
            │   Monolithic  │   │  Distributed   │   │   Data Heavy    │
            │   (Single)    │   │   (Multiple)   │   │  (Petabyte+)    │
            └───────┬───────┘   └────────┬───────┘   └────────┬────────┘
                    │                    │                     │
           ┌────────┴────────┐           │                     │
           │                 │           │                     │
           ▼                 ▼           ▼                     ▼
    ┌────────────────┐  ┌────────────┐  ┌──────────┐  ┌──────────────┐
    │ Single-Tier?   │  │ Real-time? │  │ Compute  │  │ Analytics?   │
    │                │  │            │  │intensive?│  │              │
    │ Yes → N-Tier   │  │ Yes → Ev-  │  │ Yes →    │  │ Yes → Big    │
    │ No  → Questions│  │    Driven  │  │ Big      │  │   Data       │
    │ scale?         │  │ No  → µ-Svc│  │Compute   │  │              │
    └────────┬───────┘  └────────────┘  └──────────┘  └──────────────┘
             │
             ▼
    ┌───────────────────┐
    │ Scale per tier?   │
    │                   │
    │ Yes → N-Tier      │
    │ No  → Microservices
    └───────────────────┘
```

---

## Architecture Style Comparison

| Aspect | N-Tier | Microservices | Event-Driven | Big Data | Big Compute |
|--------|--------|---------------|--------------|----------|------------|
| **Complexity** | Low | High | Medium | High | Very High |
| **Team Size** | Small (1-2) | Large (per service) | Medium | Large | Medium |
| **Deployment** | Monolithic | Independent | Event-based | Batch/Stream | Job-based |
| **Scaling** | Tier-level | Service-level | Event rate | Data volume | Compute nodes |
| **State Management** | Centralized DB | Per-service DB | Event store | Data lake | Distributed |
| **Failure Impact** | Entire app | Single service | Event loss | Processing delay | Job retry |
| **Communication** | Synchronous | Async + Sync | Async (fire-forget) | Batch/Stream | Job queues |
| **Data Consistency** | ACID | Eventual | Eventual | Eventual | Eventual |
| **AWS Equivalent** | EC2 + RDS | ECS/EKS + DynamoDB | EventBridge + Lambda | EMR + S3 | Batch + EC2 |

---

## N-Tier Architecture

### Quick Overview

```
┌────────────────────────────────────────────────────────┐
│  PRESENTATION (Web)                                    │
│  ├─ IIS / App Service                                 │
│  └─ Response → Requests                               │
├────────────────────────────────────────────────────────┤
│  BUSINESS (Application)                                │
│  ├─ Business logic                                     │
│  ├─ Validation                                         │
│  └─ Service orchestration                              │
├────────────────────────────────────────────────────────┤
│  DATA (Database)                                       │
│  ├─ SQL Server / Azure SQL                            │
│  ├─ Reads/Writes                                      │
│  └─ Transactions                                       │
└────────────────────────────────────────────────────────┘
```

### When to Use
- Migrating on-premises applications to cloud
- Traditional business applications (CRM, ERP)
- Clear separation of concerns needed
- Team is unfamiliar with distributed systems
- Monolithic deployment acceptable

### Azure Services
| Component | Service | Notes |
|-----------|---------|-------|
| Load Balancer | Application Gateway | L7 routing, WAF |
| Web Tier | App Service / VMs | Managed or IaaS |
| App Tier | App Service / VMs | Internal LB |
| Database | Azure SQL / SQL Server | Zone-redundant |
| Cache | Azure Cache for Redis | Session/data caching |

### Key Commands

```bash
# Create N-Tier infrastructure
az group create --name myRG --location eastus

# App Service for web tier
az appservice plan create --name myPlan --resource-group myRG --sku Standard
az webapp create --name myWebApp --resource-group myRG --plan myPlan

# Internal Load Balancer for app tier
az network lb create \
  --resource-group myRG \
  --name internal-lb \
  --sku Standard \
  --public-ip-address "" \
  --frontend-ip-name internal-frontend \
  --backend-pool-name app-pool

# Azure SQL for data tier
az sql server create \
  --resource-group myRG \
  --name mysqlserver \
  --admin-user sqladmin \
  --admin-password P@ssw0rd1234

az sql db create \
  --resource-group myRG \
  --server mysqlserver \
  --name myDatabase \
  --service-objective S2
```

---

## Microservices Architecture

### Quick Overview

```
┌─────────────────────────────────────────┐
│         API GATEWAY (APIM)              │
│  • Auth • Rate limit • Routing           │
└────────────────┬────────────────────────┘
                 │
    ┌────────────┼────────────┐
    ▼            ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Orders  │  │Products │  │Customers│
│ Service │  │ Service │  │ Service │
├─────────┤  ├─────────┤  ├─────────┤
│   DB    │  │   DB    │  │   DB    │
└─────────┘  └─────────┘  └─────────┘
    │            │            │
    └────────────┼────────────┘
                 ▼
        ┌──────────────────┐
        │  Service Bus /   │
        │  Event Grid      │
        │  (Async Comms)   │
        └──────────────────┘
```

### When to Use
- Large, complex applications
- Independent scaling requirements per service
- Multiple teams (one team per service)
- Technology diversity needed
- Continuous deployment required

### Azure Services
| Component | Service | Notes |
|-----------|---------|-------|
| Orchestration | AKS (Kubernetes) | Container orchestration |
| Service Communication | Service Mesh (Istio/Dapr) | Advanced routing, security |
| API Management | APIM | Gateway, throttling, auth |
| Messaging | Service Bus | Async service-to-service |
| Databases | Azure SQL, Cosmos DB | Per-service databases |

### Key Commands

```bash
# Create AKS cluster
az aks create \
  --resource-group myRG \
  --name myAKS \
  --node-count 3 \
  --vm-set-type VirtualMachineScaleSets \
  --load-balancer-sku standard

# Deploy microservice (example: Order Service)
kubectl create deployment order-service \
  --image=myregistry.azurecr.io/order-service:1.0

# Expose service
kubectl expose deployment order-service \
  --port=80 \
  --target-port=8080 \
  --type=LoadBalancer

# Setup APIM for gateway
az apim create \
  --name myAPIM \
  --resource-group myRG \
  --publisher-name "My Company" \
  --publisher-email admin@mycompany.com
```

---

## Event-Driven Architecture

### Quick Overview

```
EVENT PRODUCERS           EVENT GRID/HUBS        EVENT CONSUMERS
──────────────           ───────────────        ───────────────

┌──────────┐             ┌──────────────┐       ┌──────────┐
│IoT Hub   │─Events─►    │              │       │Function  │
└──────────┘             │ Event Grid / │       │  Azure   │
                         │ Event Hubs   │──►    │ Service  │
┌──────────┐             │              │       │  Bus     │
│App       │─Events─►    │ • Filter     │       │ Logic    │
│Service   │             │ • Route      │       │  App     │
└──────────┘             │ • Transform  │       └──────────┘
                         │              │       └──────────┐
┌──────────┐             └──────────────┘       └──────────┘
│Storage   │─Events─►
│Account   │
└──────────┘
```

### When to Use
- Real-time data processing
- Decoupled components (fire-and-forget)
- Multiple consumers of same event
- Reactive applications
- Telemetry and monitoring

### Azure Services
| Component | Service | Notes |
|-----------|---------|-------|
| Event Ingestion | Azure Event Hubs | High-throughput streaming |
| Event Routing | Azure Event Grid | Low-latency event distribution |
| Event Processing | Azure Functions | Serverless consumers |
| Queuing | Service Bus | Guaranteed delivery |
| Stream Processing | Stream Analytics | Real-time analytics |

### Key Commands

```bash
# Create Event Grid Topic
az eventgrid topic create \
  --name myEventTopic \
  --location eastus \
  --resource-group myRG

# Create Event Subscription
az eventgrid event-subscription create \
  --name mySubscription \
  --source-resource-id /subscriptions/.../myEventTopic \
  --endpoint /subscriptions/.../myFunction \
  --endpoint-type azurefunction \
  --included-event-types "All"

# Publish event
az eventgrid topic event-subscription list \
  --topic-name myEventTopic \
  --resource-group myRG

# Create Event Hubs namespace (high-throughput)
az eventhubs namespace create \
  --name myEventHubsNS \
  --resource-group myRG

az eventhubs eventhub create \
  --name myEventHub \
  --namespace-name myEventHubsNS \
  --resource-group myRG
```

---

## Big Data Architecture

### Quick Overview

```
DATA SOURCES              PROCESSING              CONSUMPTION
────────────              ──────────              ────────────

┌─────────┐              ┌────────────────┐      ┌──────────┐
│  IoT    │              │   BATCH LAYER  │      │Power BI  │
│ Devices │──Ingest─►    │  Synapse Spark │──►   │Dashboard │
└─────────┘              │  (Historical)  │      └──────────┘
                         └────────────────┘
┌─────────┐              ┌────────────────┐      ┌──────────┐
│  Apps   │──Ingest──►   │  SPEED LAYER   │──►   │Grafana   │
└─────────┘              │ Stream Analytics│      │Dashboard │
                         │  (Real-time)   │      └──────────┘
┌─────────┐              └────────────────┘
│  Logs   │
└─────────┘              ┌────────────────┐
                         │  DATA LAKE     │
                         │  (ADLS Gen2)   │
                         │  (Archive)     │
                         └────────────────┘
```

### When to Use
- Analytics and business intelligence
- Large-scale data processing (TB+)
- Batch + real-time processing needed
- Data warehousing
- Machine learning pipelines

### Azure Services
| Component | Service | Notes |
|-----------|---------|-------|
| Data Lake | ADLS Gen2 | Hierarchical storage, analytics |
| Batch Processing | Synapse Spark Pools | Distributed batch processing |
| Stream Processing | Stream Analytics | Real-time event processing |
| Data Warehouse | Synapse SQL Pool | Dedicated SQL analytics |
| Orchestration | Data Factory | Data pipeline automation |

### Key Commands

```bash
# Create ADLS Gen2 (Data Lake)
az storage account create \
  --name mydatalake \
  --resource-group myRG \
  --kind StorageV2 \
  --enable-hierarchical-namespace true

# Create Synapse Workspace
az synapse workspace create \
  --name mysynapse \
  --resource-group myRG \
  --storage-account mydatalake \
  --file-system mycontainer \
  --sql-admin-login-user sqladmin \
  --sql-admin-login-password P@ssw0rd1234

# Create Spark Pool
az synapse spark pool create \
  --name mySpark \
  --workspace-name mysynapse \
  --resource-group myRG \
  --spark-version 3.2 \
  --node-count 5

# Create SQL Analytics Pool
az synapse sql pool create \
  --name myDW \
  --workspace-name mysynapse \
  --resource-group myRG \
  --performance-level DW100c
```

---

## Big Compute Architecture

### Quick Overview

```
JOB SUBMISSION         JOB SCHEDULER        COMPUTE RESOURCES
──────────────         ─────────────        ────────────────

┌──────────┐          ┌──────────────┐     ┌─────────────────┐
│ Portal   │          │              │     │  Compute Pool   │
│ API      │──Queue──►│ Azure Batch  │     │  (Scalable)     │
│ CLI      │          │              │     │                 │
└──────────┘          │ • Scheduling │     │ ┌─────┐┌─────┐ │
                      │ • Queueing   │     │ │ VM  ││ VM  │ │
                      │ • Scaling    │──► │ │ HPC ││ HPC │ │
                      │ • Monitoring │     │ └─────┘└─────┘ │
                      │ • Retry      │     │  (up to 10K+)  │
                      └──────────────┘     └─────────────────┘
                                                    │
                                                    ▼
                                          ┌─────────────────┐
                                          │  Data Storage   │
                                          │ • Blob Storage  │
                                          │ • Azure Files   │
                                          │ • NetApp Files  │
                                          └─────────────────┘
```

### When to Use
- Computationally intensive workloads
- Simulations (Monte Carlo, CFD)
- Rendering and media processing
- Scientific computing
- Genomic analysis

### Azure Services
| Component | Service | Notes |
|-----------|---------|-------|
| Job Scheduling | Azure Batch | Task scheduling, auto-scaling |
| Compute Resources | VMs (HPC series) | HBv3, HCv3, N-series (GPU) |
| Spot VMs | Batch + Spot | 90% cost savings |
| Storage | Blob/Azure Files | Shared data access |
| High-Speed Network | InfiniBand | RDMA for HPC |

### Key Commands

```bash
# Create Batch Account
az batch account create \
  --name mybatch \
  --resource-group myRG

# Create Batch Pool (HPC-optimized)
az batch pool create \
  --account-name mybatch \
  --id mypool \
  --vm-size Standard_HB120rs_v3 \
  --target-node-count 10 \
  --node-agent-sku-id batch.node.centos 7

# Submit Batch Job
az batch job create \
  --account-name mybatch \
  --id myjob \
  --pool-id mypool

# Add Task to Job
az batch task create \
  --account-name mybatch \
  --job-id myjob \
  --task-id mytask \
  --command-line "python simulate.py"
```

---

## AWS to Azure Architecture Pattern Mapping

| AWS Pattern | Azure Equivalent | AWS Service | Azure Service | Notes |
|------------|-----------------|-------------|----------------|-------|
| N-Tier (EC2 + RDS) | N-Tier | EC2 + ALB + RDS | VMs + App Gw + SQL | VM Scale Sets for auto-scaling |
| Microservices (ECS/EKS) | Microservices | ECS/EKS | AKS | Kubernetes native, add Service Fabric |
| Event-Driven (EventBridge) | Event-Driven | EventBridge + Lambda | Event Grid + Functions | Event Grid has lower latency |
| Streaming (Kinesis) | Stream Processing | Kinesis Streams | Event Hubs | Event Hubs higher throughput |
| Batch (AWS Batch) | Big Compute | Batch + EC2 | Azure Batch | Similar architecture |
| Analytics (EMR) | Big Data | EMR + S3 | Synapse + ADLS | HDInsight alternative |
| Async (SQS) | Queuing | SQS | Service Bus | Service Bus supports multiple patterns |
| Serverless (Lambda) | Functions | Lambda | Azure Functions | KEDA for Kubernetes |

---

## Decision Flow: Which Architecture Style?

```
START
  │
  ├─► Is data < 100GB and needs real-time responses?
  │   └─► YES ──► N-Tier (traditional app)
  │
  ├─► Do you have 3+ independent business domains?
  │   └─► YES ──► Microservices
  │
  ├─► Do you need fire-and-forget event processing?
  │   └─► YES ──► Event-Driven
  │
  ├─► Is data > 1TB and needs analytics?
  │   └─► YES ──► Big Data (Lambda architecture)
  │
  ├─► Are calculations CPU-intensive (simulations, etc.)?
  │   └─► YES ──► Big Compute
  │
  └─► Hybrid: Combine multiple patterns
      • Microservices + Event-Driven
      • Big Data + Event-Driven (Kappa)
      • N-Tier + Event-Driven
```

---

## Key Azure Services Reference

### Messaging & Events
| Service | Purpose | Throughput | Latency | Use Case |
|---------|---------|-----------|---------|----------|
| Service Bus | Reliable messaging | 1,000s msg/s | 10-50ms | Enterprise queues |
| Event Grid | Event distribution | 100K events/s | <100ms | Event routing |
| Event Hubs | Stream ingestion | 1M+ events/s | <1s | IoT/telemetry |

### Compute Platforms
| Service | Type | Scaling | Auto-scale | Best For |
|---------|------|---------|-----------|----------|
| App Service | PaaS | Horizontal | Yes | Web/API apps |
| AKS | K8s | Pod-level | KEDA | Microservices |
| Azure Functions | Serverless | Auto | Yes | Event-driven |
| Azure Batch | HPC | Node pools | Yes | Compute jobs |

### Data Services
| Service | Type | Scale | Consistency | Best For |
|---------|------|-------|-------------|----------|
| Azure SQL | RDBMS | TB | ACID | Transactional |
| Cosmos DB | NoSQL | PB | Eventual | Distributed |
| ADLS Gen2 | Data Lake | EB | Eventual | Analytics |
| Synapse | DW | PB | Eventual | Data warehousing |

---

## Combining Architecture Styles

### Microservices + Event-Driven
```
Scenario: E-commerce platform with inventory updates

┌─────────────┐  ┌──────────────┐  ┌─────────────┐
│Order Service│  │Inventory Svc │  │Payment Svc  │
└──────┬──────┘  └──────┬───────┘  └──────┬──────┘
       │                │                 │
       └────────────────┼─────────────────┘
                        │
                   [Event Grid]
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ▼               ▼               ▼
    [Function]    [Logic App]    [Webhook]
    (Analytics)   (Notification) (External)
```

### N-Tier + Event-Driven
```
Scenario: Traditional app with audit logging

┌────────────────────────────────┐
│   PRESENTATION (Web)           │
└────────────────┬───────────────┘
                 │
┌────────────────▼───────────────┐
│  BUSINESS TIER                 │
│  • Publish events              │
└────────────────┬───────────────┘
                 │
┌────────────────▼───────────────┐
│  DATA TIER                     │
└────────────────┬───────────────┘
                 │
           [Event Grid]
                 │
        ┌────────┴────────┐
        │                 │
        ▼                 ▼
    [Audit Log]    [Archive]
```

---

## Common Pitfalls & Solutions

| Architecture | Pitfall | Solution |
|-------------|---------|----------|
| **Microservices** | "Distributed Monolith" (tightly coupled services) | Define clear domain boundaries, use async communication |
| **Microservices** | Cascading failures | Implement circuit breakers, retry logic, timeouts |
| **Event-Driven** | Duplicate event processing | Idempotent handlers, deduplication logic |
| **Event-Driven** | Lost events | Implement dead-letter queues, event sourcing |
| **Big Data** | Slow batch processing | Partition data, use appropriate pool sizes |
| **Big Compute** | Job timeouts | Implement checkpointing, resume capability |
| **N-Tier** | Bottlenecks in business tier | Scale independently, use internal load balancer |

---

## Next Steps & References

- **Deep Dive**: [Architecture Styles Details](01-architecture-styles.md)
- **Design Principles**: [Design Principles & Patterns](02-design-principles.md)
- **Patterns**: [Design Patterns Chapter](../12-design-patterns/README.md)
- **Implementation**: [Infrastructure as Code](../17-infrastructure-as-code/README.md)

---

*Deep Dive: [Architecture Styles](01-architecture-styles.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
