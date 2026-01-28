# Azure Technology Choices Guide

## Overview

Selecting the right technology is one of the most consequential decisions an architect makes. A poor choice early on can lead to costly migrations, performance bottlenecks, or operational nightmares. This guide provides frameworks for making informed technology decisions across compute, data, and messaging services in Azure.

---

## 1. Compute Selection Framework

### The Compute Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        COMPUTE SELECTION DECISION TREE                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  What level of control do you need?                                              │
│  │                                                                               │
│  ├── Full OS control / Custom drivers / Lift-and-shift                          │
│  │   │                                                                           │
│  │   └── VIRTUAL MACHINES                                                        │
│  │       • VM Scale Sets for scaling                                             │
│  │       • Availability Zones for HA                                             │
│  │       • Spot VMs for cost savings                                             │
│  │                                                                               │
│  ├── Container workloads                                                         │
│  │   │                                                                           │
│  │   ├── Need Kubernetes features / ecosystem?                                   │
│  │   │   │                                                                       │
│  │   │   ├── Yes, full K8s control ────────► AZURE KUBERNETES SERVICE (AKS)     │
│  │   │   │                                                                       │
│  │   │   └── No, just containers ──────────► AZURE CONTAINER APPS               │
│  │   │                                        (Serverless containers)            │
│  │   │                                                                           │
│  │   └── Simple container hosting ─────────► CONTAINER INSTANCES (ACI)          │
│  │       (Burst workloads, CI/CD)                                                │
│  │                                                                               │
│  ├── Web applications / APIs                                                     │
│  │   │                                                                           │
│  │   ├── Need custom containers? ──────────► APP SERVICE (Container)            │
│  │   │                                                                           │
│  │   └── Standard web app ─────────────────► APP SERVICE (Code)                 │
│  │                                                                               │
│  └── Event-driven / Short-lived tasks                                            │
│      │                                                                           │
│      ├── Execution < 10 min? ──────────────► AZURE FUNCTIONS (Consumption)      │
│      │                                                                           │
│      └── Long-running? ────────────────────► DURABLE FUNCTIONS                  │
│                                              or CONTAINER APPS JOBS              │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Compute Options Comparison

| Feature | VMs | AKS | Container Apps | App Service | Functions |
|---------|-----|-----|----------------|-------------|-----------|
| **Abstraction** | IaaS | CaaS | Serverless Containers | PaaS | FaaS |
| **Scaling** | Manual/VMSS | HPA/KEDA | KEDA (auto) | Auto-scale rules | Auto (Consumption) |
| **Min instances** | 1 | 1 (+ system pods) | 0 | 1 (or 0 for Premium) | 0 |
| **Max scale** | VMSS limits | 5000 nodes | 300 replicas | 30 instances | 200 instances |
| **Cold start** | Minutes | Seconds | Seconds | None (always on) | Seconds |
| **Networking** | Full VNet | VNet integrated | VNet integrated | VNet integrated | VNet (Premium) |
| **State** | Stateful | Both | Stateless preferred | Stateless | Stateless |
| **Cost model** | Per VM hour | Per node hour | Per vCPU-second | Per App Service Plan | Per execution |

### AWS to Azure Compute Mapping

| AWS Service | Azure Equivalent | Notes |
|-------------|-----------------|-------|
| EC2 | Virtual Machines | Direct equivalent |
| EC2 Auto Scaling | VM Scale Sets | Similar functionality |
| ECS | Container Instances | Simple container hosting |
| EKS | Azure Kubernetes Service | Managed Kubernetes |
| Fargate | Container Apps / ACI | Serverless containers |
| Elastic Beanstalk | App Service | PaaS web hosting |
| Lambda | Azure Functions | Serverless compute |
| Step Functions | Durable Functions | Workflow orchestration |
| AWS Batch | Azure Batch | Batch computing |

### When to Use Each Compute Option

#### Azure Virtual Machines

```
BEST FOR:
├── Lift-and-shift migrations from on-premises
├── Applications requiring specific OS configurations
├── Legacy applications with OS-level dependencies
├── HPC workloads requiring specialized hardware
└── Applications requiring persistent local disk

AVOID WHEN:
├── Application is stateless and horizontally scalable
├── Team lacks infrastructure management expertise
└── Rapid scaling is critical
```

#### Azure Kubernetes Service (AKS)

```
BEST FOR:
├── Microservices architectures at scale
├── Teams with Kubernetes expertise
├── Multi-cloud portability requirements
├── Complex networking requirements (service mesh)
├── Workloads requiring advanced scheduling
└── GitOps deployment models

AVOID WHEN:
├── Simple applications (overkill)
├── Team lacks container orchestration experience
├── Cost sensitivity (control plane overhead)
└── Single-tenant simple workloads
```

#### Azure Container Apps

```
BEST FOR:
├── Microservices without Kubernetes complexity
├── Event-driven container workloads
├── HTTP APIs and web applications
├── Background processing jobs
├── Teams wanting managed infrastructure
└── Scale-to-zero requirements

AVOID WHEN:
├── Need full Kubernetes API access
├── Complex networking (service mesh)
├── Stateful workloads with persistent storage
└── Windows containers
```

#### Azure App Service

```
BEST FOR:
├── Traditional web applications
├── RESTful APIs
├── Deployment slots for blue-green deployments
├── Built-in authentication requirements
├── Teams wanting zero infrastructure management
└── .NET, Java, Node.js, Python, PHP applications

AVOID WHEN:
├── Event-driven processing
├── Long-running background tasks
├── Need for fine-grained scaling control
└── Extreme cost optimization (scale-to-zero)
```

#### Azure Functions

```
BEST FOR:
├── Event-driven processing
├── Webhook handlers
├── Scheduled tasks (CRON)
├── Data transformation pipelines
├── IoT backend processing
├── API backends with variable load
└── Pay-per-execution cost model

AVOID WHEN:
├── Long-running processes (> 10 min)
├── Complex stateful workflows (use Durable)
├── Consistent high-throughput workloads
└── WebSocket connections
```

### GPU and HPC Compute Options

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          GPU & HPC COMPUTE OPTIONS                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   GPU WORKLOADS                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                                                                          │  │
│   │   TRAINING (Large models)           INFERENCE (Real-time)                │  │
│   │   │                                 │                                    │  │
│   │   ├── NC-series (NVIDIA T4)         ├── NC-series (NVIDIA T4)           │  │
│   │   ├── ND-series (NVIDIA A100)       ├── NV-series (NVIDIA Tesla M60)    │  │
│   │   └── NDm v4 (NVIDIA A100 80GB)     └── Container Apps (GPU preview)    │  │
│   │                                                                          │  │
│   │   Use: Azure Machine Learning       Use: AKS with GPU node pools        │  │
│   │        managed compute                   Azure ML managed endpoints     │  │
│   │                                                                          │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   HPC WORKLOADS                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                                                                          │  │
│   │   VM Series        Use Case                  InfiniBand    Cores         │  │
│   │   ─────────────────────────────────────────────────────────────────────  │  │
│   │   HBv4            Memory-bound HPC           Yes (NDR)     176           │  │
│   │   HBv3            General HPC                Yes (HDR)     120           │  │
│   │   HC              Compute-intensive          Yes (EDR)     44            │  │
│   │   HX              Large memory HPC           Yes (HDR)     176           │  │
│   │                                                                          │  │
│   │   Orchestration Options:                                                 │  │
│   │   • Azure Batch - Job scheduling at scale                               │  │
│   │   • Azure CycleCloud - HPC cluster management                           │  │
│   │   • SLURM / PBS on VMs - Traditional schedulers                         │  │
│   │                                                                          │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Compute Cost Optimization Strategies

| Strategy | Service | Potential Savings |
|----------|---------|------------------|
| **Spot VMs** | VMs, VMSS, AKS | Up to 90% |
| **Reserved Instances** | VMs, App Service | Up to 72% (3-year) |
| **Savings Plans** | Compute | Up to 65% (3-year) |
| **Scale-to-zero** | Container Apps, Functions | Pay only when used |
| **Right-sizing** | All | 20-50% typical |
| **Dev/Test pricing** | All | 40-50% for non-prod |

---

## 2. Data Store Decision Tree

### The Data Store Selection Framework

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        DATA STORE DECISION TREE                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  What type of data and access pattern?                                           │
│  │                                                                               │
│  ├── RELATIONAL DATA (Structured, ACID transactions)                             │
│  │   │                                                                           │
│  │   ├── Microsoft SQL ecosystem? ─────────────► AZURE SQL DATABASE             │
│  │   │   • Intelligent performance tuning                                        │
│  │   │   • Built-in HA (99.995% SLA)                                            │
│  │   │                                                                           │
│  │   ├── Open source / PostgreSQL expertise? ──► AZURE DATABASE FOR POSTGRESQL  │
│  │   │   • Flexible Server (recommended)                                         │
│  │   │   • Citus extension for scale-out                                        │
│  │   │                                                                           │
│  │   └── MySQL workloads? ─────────────────────► AZURE DATABASE FOR MYSQL       │
│  │                                                                               │
│  ├── NoSQL / DOCUMENT DATA                                                       │
│  │   │                                                                           │
│  │   ├── Global distribution needed? ──────────► COSMOS DB                      │
│  │   │   │                                                                       │
│  │   │   ├── Document model ──► Core (NoSQL) API                                │
│  │   │   ├── MongoDB compat ──► MongoDB API                                     │
│  │   │   ├── Cassandra ──────► Cassandra API                                    │
│  │   │   ├── Graph data ─────► Gremlin API                                      │
│  │   │   └── Key-value ──────► Table API                                        │
│  │   │                                                                           │
│  │   └── Single region, cost-sensitive? ───────► COSMOS DB (Single region)     │
│  │       or TABLE STORAGE (simple key-value)                                     │
│  │                                                                               │
│  ├── CACHING                                                                     │
│  │   │                                                                           │
│  │   ├── Session state / Object caching ───────► AZURE CACHE FOR REDIS         │
│  │   │                                                                           │
│  │   └── Read-through caching of SQL? ─────────► Redis + SQL Database           │
│  │                                                                               │
│  ├── SEARCH                                                                      │
│  │   │                                                                           │
│  │   ├── Full-text search / Faceted nav ───────► AZURE AI SEARCH               │
│  │   │   • Vector search for AI/RAG                                             │
│  │   │                                                                           │
│  │   └── Log analytics / Observability ────────► LOG ANALYTICS                  │
│  │                                                                               │
│  ├── TIME SERIES                                                                 │
│  │   │                                                                           │
│  │   └── IoT / Metrics / Telemetry ────────────► AZURE DATA EXPLORER (ADX)     │
│  │                                               or TIME SERIES INSIGHTS        │
│  │                                                                               │
│  └── FILE / BLOB STORAGE                                                         │
│      │                                                                           │
│      ├── Unstructured data (images, videos) ───► BLOB STORAGE                   │
│      │                                                                           │
│      ├── Shared file access (SMB/NFS) ─────────► AZURE FILES                    │
│      │                                                                           │
│      └── Data lake / Analytics ────────────────► ADLS GEN2                      │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Data Store Comparison Matrix

| Feature | Azure SQL | Cosmos DB | PostgreSQL | Redis | Blob Storage |
|---------|-----------|-----------|------------|-------|--------------|
| **Model** | Relational | Multi-model | Relational | Key-value | Object |
| **Consistency** | Strong | Tunable | Strong | Eventual | Strong |
| **Scale** | Vertical + Read replicas | Horizontal (global) | Vertical + Citus | Clustered | Unlimited |
| **Max size** | 100 TB | Unlimited | 16 TB | 120 GB (P5) | 5 PB |
| **Latency** | ms | < 10ms global | ms | < 1ms | ms |
| **Global dist** | Geo-replicas | Native multi-region | Read replicas | Geo-replication | RA-GRS/GZRS |
| **Pricing** | DTU/vCore | RU/s + storage | vCore | Cache size | GB stored + ops |

### AWS to Azure Data Store Mapping

| AWS Service | Azure Equivalent | Migration Path |
|-------------|-----------------|----------------|
| RDS (SQL Server) | Azure SQL Database | DMS, Azure Migrate |
| RDS (PostgreSQL) | Azure Database for PostgreSQL | DMS, pg_dump |
| RDS (MySQL) | Azure Database for MySQL | DMS, mysqldump |
| DynamoDB | Cosmos DB (Table API or NoSQL) | DMS, custom migration |
| DocumentDB | Cosmos DB (MongoDB API) | Native compatibility |
| ElastiCache (Redis) | Azure Cache for Redis | RDB export/import |
| S3 | Blob Storage / ADLS Gen2 | AzCopy, Data Factory |
| Redshift | Synapse Analytics | Data Factory, Polybase |
| Neptune | Cosmos DB (Gremlin API) | Custom migration |
| OpenSearch | Azure AI Search | Custom migration |

### When to Use Azure SQL vs Cosmos DB vs PostgreSQL

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    AZURE SQL vs COSMOS DB vs POSTGRESQL                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   CHOOSE AZURE SQL WHEN:                                                         │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │  ✓ Complex transactions with ACID guarantees                             │  │
│   │  ✓ Complex joins and reporting queries                                   │  │
│   │  ✓ Existing SQL Server expertise/applications                            │  │
│   │  ✓ Need for stored procedures, triggers, views                          │  │
│   │  ✓ BI tool integration (SSRS, Power BI DirectQuery)                      │  │
│   │  ✓ Regulatory compliance requiring specific SQL features                 │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   CHOOSE COSMOS DB WHEN:                                                         │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │  ✓ Global distribution with multi-region writes                          │  │
│   │  ✓ Guaranteed < 10ms latency at any scale                               │  │
│   │  ✓ Flexible/schema-less data models                                      │  │
│   │  ✓ Horizontal scaling with automatic partitioning                        │  │
│   │  ✓ Multi-model requirements (document, graph, key-value)                │  │
│   │  ✓ Tunable consistency levels                                            │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   CHOOSE POSTGRESQL WHEN:                                                        │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │  ✓ Open source preference / avoiding vendor lock-in                      │  │
│   │  ✓ Advanced data types (JSON, arrays, custom types)                     │  │
│   │  ✓ PostGIS for geospatial workloads                                     │  │
│   │  ✓ Extensions ecosystem (pg_vector, TimescaleDB)                        │  │
│   │  ✓ Citus for horizontal scaling with SQL                                │  │
│   │  ✓ Team has PostgreSQL expertise                                        │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Redis Cache Patterns

| Pattern | Use Case | Implementation |
|---------|----------|----------------|
| **Cache-Aside** | Read-heavy workloads | App checks cache first, loads from DB on miss |
| **Write-Through** | Write consistency | Write to cache and DB simultaneously |
| **Write-Behind** | Write performance | Write to cache, async persist to DB |
| **Session Store** | Stateless web apps | Store session in Redis, share across instances |
| **Rate Limiting** | API throttling | Use Redis counters with TTL |
| **Pub/Sub** | Real-time notifications | Redis channels for message broadcasting |
| **Distributed Lock** | Coordination | SETNX for mutex implementation |

### Analytics Platform Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      ANALYTICS PLATFORM DECISION TREE                            │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  What is your primary analytics need?                                            │
│  │                                                                               │
│  ├── ENTERPRISE DATA WAREHOUSE                                                   │
│  │   │                                                                           │
│  │   ├── Microsoft ecosystem / T-SQL expertise                                   │
│  │   │   └──────────────────────────────────► SYNAPSE ANALYTICS                 │
│  │   │       • Dedicated SQL pools                                               │
│  │   │       • Serverless SQL for ad-hoc                                        │
│  │   │       • Integrated with Power BI                                         │
│  │   │                                                                           │
│  │   └── Unified analytics + AI ──────────────► MICROSOFT FABRIC                │
│  │           • OneLake (unified storage)                                         │
│  │           • Real-Time Analytics                                               │
│  │           • Power BI native                                                   │
│  │                                                                               │
│  ├── DATA SCIENCE / ML WORKLOADS                                                 │
│  │   │                                                                           │
│  │   ├── Spark + ML expertise ─────────────────► DATABRICKS                     │
│  │   │   • Unity Catalog for governance                                          │
│  │   │   • MLflow integration                                                    │
│  │   │   • Delta Lake                                                            │
│  │   │                                                                           │
│  │   └── Azure-native ML ──────────────────────► AZURE ML + SYNAPSE             │
│  │                                                                               │
│  ├── REAL-TIME ANALYTICS                                                         │
│  │   │                                                                           │
│  │   ├── Log/telemetry analytics ──────────────► AZURE DATA EXPLORER (ADX)     │
│  │   │   • KQL (Kusto Query Language)                                           │
│  │   │   • Sub-second query on billions of rows                                 │
│  │   │                                                                           │
│  │   └── Streaming analytics ──────────────────► STREAM ANALYTICS               │
│  │       or FABRIC Real-Time Analytics                                           │
│  │                                                                               │
│  └── DATA LAKE / LAKEHOUSE                                                       │
│      │                                                                           │
│      ├── Delta Lake format ────────────────────► DATABRICKS                     │
│      │                                                                           │
│      └── Open formats + Microsoft ─────────────► FABRIC / SYNAPSE + ADLS        │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Synapse vs Fabric vs Databricks

| Aspect | Synapse Analytics | Microsoft Fabric | Databricks |
|--------|------------------|------------------|------------|
| **Best for** | Enterprise DW, T-SQL | Unified analytics, BI | Data science, ML |
| **Query Language** | T-SQL | T-SQL, Spark, KQL | Spark SQL, Python |
| **Storage** | Dedicated pools, ADLS | OneLake | Delta Lake (ADLS) |
| **Real-time** | Limited | Native (ADX-based) | Structured Streaming |
| **ML Integration** | Azure ML | Built-in | MLflow native |
| **Governance** | Purview | Built-in Purview | Unity Catalog |
| **Pricing Model** | Compute + storage | Capacity units | DBU + storage |
| **BI Tool** | Power BI (separate) | Power BI (integrated) | Power BI, Tableau |

---

## 3. Messaging Technology Comparison

### Messaging Decision Tree

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      MESSAGING TECHNOLOGY DECISION TREE                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  What is your messaging requirement?                                             │
│  │                                                                               │
│  ├── COMMANDS (Do something)                                                     │
│  │   │                                                                           │
│  │   ├── Simple FIFO queue ────────────────────► STORAGE QUEUES                 │
│  │   │   • 500 TB max queue size                                                 │
│  │   │   • At-least-once delivery                                                │
│  │   │   • Cost effective                                                        │
│  │   │                                                                           │
│  │   └── Enterprise messaging ─────────────────► SERVICE BUS QUEUES             │
│  │       • FIFO guaranteed                                                       │
│  │       • Sessions, dead-letter                                                 │
│  │       • Transactions, duplicate detection                                     │
│  │                                                                               │
│  ├── EVENTS (Something happened)                                                 │
│  │   │                                                                           │
│  │   ├── React to Azure resource events ───────► EVENT GRID                     │
│  │   │   • Blob created, VM started, etc.                                       │
│  │   │   • Push-based (webhooks)                                                 │
│  │   │   • Filtering and routing                                                 │
│  │   │                                                                           │
│  │   ├── Pub/Sub with filtering ───────────────► SERVICE BUS TOPICS             │
│  │   │   • Multiple subscribers                                                  │
│  │   │   • SQL-like filters                                                      │
│  │   │   • Sessions per subscription                                             │
│  │   │                                                                           │
│  │   └── Event broadcasting ───────────────────► EVENT GRID (Custom topics)    │
│  │       • 1M events/sec per topic                                               │
│  │       • 24hr retention                                                        │
│  │                                                                               │
│  └── STREAMING (Continuous data flow)                                            │
│      │                                                                           │
│      ├── High-throughput ingestion ────────────► EVENT HUBS                     │
│      │   • Millions of events/sec                                                │
│      │   • Kafka protocol compatible                                             │
│      │   • Capture to storage                                                    │
│      │                                                                           │
│      └── IoT device telemetry ─────────────────► IOT HUB                        │
│          • Device management                                                     │
│          • Bi-directional communication                                          │
│          • Device twins                                                          │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Messaging Services Comparison

| Feature | Storage Queue | Service Bus | Event Grid | Event Hubs |
|---------|--------------|-------------|------------|------------|
| **Pattern** | Queue | Queue/Pub-Sub | Pub-Sub | Stream |
| **Delivery** | At-least-once | At-least-once/once | At-least-once | At-least-once |
| **Ordering** | No | FIFO (sessions) | No | Per partition |
| **Max msg size** | 64 KB | 256 KB-100 MB | 1 MB | 1 MB |
| **Max throughput** | 2K msg/sec | 4K msg/sec/queue | 10M events/sec | Millions/sec |
| **Retention** | 7 days | 14 days (Premium) | 24 hours | 7 days (90 max) |
| **Dead-letter** | No | Yes | Yes | No (use capture) |
| **Transactions** | No | Yes | No | No |
| **Protocol** | REST | AMQP, REST | HTTP, AMQP | AMQP, Kafka |

### AWS to Azure Messaging Mapping

| AWS Service | Azure Equivalent | Key Differences |
|-------------|-----------------|-----------------|
| SQS | Storage Queues / Service Bus Queues | Service Bus offers enterprise features |
| SNS | Event Grid / Service Bus Topics | Event Grid for push, Service Bus for pull |
| EventBridge | Event Grid | Similar event routing capabilities |
| Kinesis Data Streams | Event Hubs | Event Hubs has Kafka compatibility |
| Kinesis Firehose | Event Hubs Capture | Capture to ADLS or Blob |
| IoT Core | IoT Hub | IoT Hub includes device management |
| Amazon MQ | Service Bus / RabbitMQ on AKS | No direct ActiveMQ equivalent |

### Service Bus vs Event Grid vs Event Hubs

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                SERVICE BUS vs EVENT GRID vs EVENT HUBS                           │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   SERVICE BUS                                                                    │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │  Purpose: Enterprise messaging with reliability guarantees               │  │
│   │                                                                          │  │
│   │  Sender ────► Queue ────► Single Consumer                               │  │
│   │                                                                          │  │
│   │  Sender ────► Topic ─┬──► Subscription 1 ────► Consumer A               │  │
│   │                      ├──► Subscription 2 ────► Consumer B               │  │
│   │                      └──► Subscription 3 ────► Consumer C               │  │
│   │                                                                          │  │
│   │  Features: FIFO, Sessions, Transactions, Dead-letter, Deduplication     │  │
│   │  Use: Order processing, financial transactions, workflow                │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   EVENT GRID                                                                     │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │  Purpose: Reactive event-driven architectures (push model)               │  │
│   │                                                                          │  │
│   │  Azure Services ─┐                                                       │  │
│   │  Custom Apps ────┼──► Event Grid ─┬──► Azure Function                   │  │
│   │  SaaS Apps ──────┘                ├──► Logic App                        │  │
│   │                                   ├──► Webhook                          │  │
│   │                                   └──► Event Hubs (bridge)              │  │
│   │                                                                          │  │
│   │  Features: Filtering, retry with dead-letter, fan-out                   │  │
│   │  Use: React to Azure events, serverless triggers, notifications         │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   EVENT HUBS                                                                     │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │  Purpose: Big data streaming and ingestion                               │  │
│   │                                                                          │  │
│   │  Producers ────► [ Partition 0 ]────┐                                   │  │
│   │  (millions/sec)  [ Partition 1 ]────┼──► Consumer Groups ──► Processing │  │
│   │                  [ Partition 2 ]────┤                                   │  │
│   │                  [ Partition N ]────┘                                   │  │
│   │                                                                          │  │
│   │  Features: Partitioned log, Kafka protocol, Capture to storage          │  │
│   │  Use: Telemetry, logging, stream processing, analytics                  │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Messaging Pattern Selection Guide

| Pattern | Recommended Service | Scenario |
|---------|-------------------|----------|
| **Command** | Service Bus Queue | Process orders, execute tasks |
| **Event notification** | Event Grid | Blob uploaded, resource created |
| **Pub/Sub with filtering** | Service Bus Topics | Route messages to specific subscribers |
| **Event streaming** | Event Hubs | IoT telemetry, click streams |
| **Request/Reply** | Service Bus (correlation) | Synchronous-style over async |
| **Competing consumers** | Service Bus Queue | Work distribution across workers |
| **Fan-out** | Event Grid | Notify multiple systems of an event |
| **Event sourcing** | Event Hubs | Append-only event log |

---

## 4. Case Studies with Technology Decisions

### Case Study 1: E-Commerce Platform Technology Stack

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    E-COMMERCE PLATFORM ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   REQUIREMENTS:                                                                  │
│   • 50,000 concurrent users during peak sales                                   │
│   • Global presence (NA, EU, APAC)                                              │
│   • Sub-second product search                                                    │
│   • Real-time inventory updates                                                  │
│   • Payment processing reliability                                               │
│                                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────────┐   │
│   │                         FRONT END                                        │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                     │   │
│   │  │ Azure CDN   │  │ Front Door  │  │ Static Web  │                     │   │
│   │  │(Assets)     │  │(Global LB)  │  │ Apps (SPA)  │                     │   │
│   │  └─────────────┘  └──────┬──────┘  └─────────────┘                     │   │
│   └──────────────────────────┼──────────────────────────────────────────────┘   │
│                              │                                                   │
│   ┌──────────────────────────▼──────────────────────────────────────────────┐   │
│   │                         API LAYER                                        │   │
│   │  ┌─────────────────────────────────────────────────────────────────┐    │   │
│   │  │              Azure API Management (APIM)                         │    │   │
│   │  │  • Rate limiting  • OAuth 2.0  • Request routing                │    │   │
│   │  └───────────────────────────┬─────────────────────────────────────┘    │   │
│   └──────────────────────────────┼──────────────────────────────────────────┘   │
│                                  │                                               │
│   ┌──────────────────────────────▼──────────────────────────────────────────┐   │
│   │                         MICROSERVICES (AKS)                              │   │
│   │                                                                          │   │
│   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │   │
│   │  │ Catalog  │ │  Cart    │ │  Order   │ │ Payment  │ │ Shipping │      │   │
│   │  │ Service  │ │ Service  │ │ Service  │ │ Service  │ │ Service  │      │   │
│   │  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘      │   │
│   │       │            │            │            │            │             │   │
│   └───────┼────────────┼────────────┼────────────┼────────────┼─────────────┘   │
│           │            │            │            │            │                  │
│   ┌───────▼────────────▼────────────▼────────────▼────────────▼─────────────┐   │
│   │                         DATA STORES                                      │   │
│   │                                                                          │   │
│   │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐   │   │
│   │  │  Cosmos DB   │ │ Azure Redis  │ │  Azure SQL   │ │  AI Search   │   │   │
│   │  │  (Catalog,   │ │  (Cart,      │ │  (Orders,    │ │  (Product    │   │   │
│   │  │   Inventory) │ │   Sessions)  │ │   Payments)  │ │   Search)    │   │   │
│   │  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘   │   │
│   │                                                                          │   │
│   └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐   │
│   │                         MESSAGING                                         │   │
│   │                                                                          │   │
│   │  ┌──────────────────────────┐  ┌──────────────────────────┐             │   │
│   │  │ Service Bus (Orders)     │  │ Event Grid (Inventory)   │             │   │
│   │  │ • FIFO processing       │  │ • Stock updates          │             │   │
│   │  │ • Dead-letter for retry │  │ • Cache invalidation     │             │   │
│   │  └──────────────────────────┘  └──────────────────────────┘             │   │
│   │                                                                          │   │
│   └──────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Technology Decisions Explained:**

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Compute** | AKS | Microservices at scale, team has K8s expertise |
| **Product Catalog** | Cosmos DB | Global distribution, flexible schema for products |
| **Orders/Payments** | Azure SQL | ACID transactions critical for financial data |
| **Cart/Sessions** | Redis | Sub-millisecond latency, ephemeral data |
| **Search** | AI Search | Full-text search, faceted navigation, vector search for recommendations |
| **Order Processing** | Service Bus | Reliable command processing with FIFO |
| **Inventory Events** | Event Grid | React to stock changes, cache invalidation |

---

### Case Study 2: IoT Solution Technology Stack

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    IOT MONITORING PLATFORM ARCHITECTURE                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   REQUIREMENTS:                                                                  │
│   • 100,000 devices sending telemetry every 10 seconds                          │
│   • Real-time anomaly detection                                                  │
│   • Historical data retention for 2 years                                        │
│   • Device management and OTA updates                                            │
│   • Dashboard with 5-second refresh                                              │
│                                                                                  │
│   DEVICES (100K+)                                                                │
│   ┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐                                          │
│   │ Dev ││ Dev ││ Dev ││ Dev ││ Dev │  ─── MQTT/AMQP ───┐                       │
│   └─────┘└─────┘└─────┘└─────┘└─────┘                   │                       │
│                                                          │                       │
│   ┌──────────────────────────────────────────────────────▼───────────────────┐  │
│   │                           IOT HUB                                         │  │
│   │  • Device provisioning (DPS)                                             │  │
│   │  • Device twins for configuration                                         │  │
│   │  • Cloud-to-device messaging                                              │  │
│   │  • 10M messages/day capacity                                             │  │
│   └────────────────────────────────┬─────────────────────────────────────────┘  │
│                                    │                                             │
│          ┌─────────────────────────┼─────────────────────────┐                  │
│          │                         │                         │                  │
│          ▼                         ▼                         ▼                  │
│   ┌─────────────┐          ┌─────────────┐          ┌─────────────┐            │
│   │   Event     │          │   Stream    │          │   Azure     │            │
│   │   Hubs      │          │  Analytics  │          │  Functions  │            │
│   │(Capture)    │          │ (Real-time) │          │ (Routing)   │            │
│   └──────┬──────┘          └──────┬──────┘          └─────────────┘            │
│          │                        │                                             │
│          ▼                        ▼                                             │
│   ┌─────────────┐          ┌─────────────┐                                     │
│   │   ADLS      │          │   Azure     │                                     │
│   │   Gen2      │          │   Data      │                                     │
│   │ (Raw Data)  │          │  Explorer   │                                     │
│   └──────┬──────┘          │ (Real-time  │                                     │
│          │                 │  Analytics) │                                     │
│          │                 └──────┬──────┘                                     │
│          │                        │                                             │
│          ▼                        ▼                                             │
│   ┌─────────────┐          ┌─────────────┐                                     │
│   │  Databricks │          │  Grafana /  │                                     │
│   │  (ML/       │          │  Power BI   │                                     │
│   │  Historical)│          │ (Dashboards)│                                     │
│   └─────────────┘          └─────────────┘                                     │
│                                                                                  │
│   ANOMALY DETECTION PIPELINE                                                    │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                                                                          │  │
│   │  Stream Analytics ──► ML Model (Azure ML) ──► Alert (Logic App)         │  │
│   │  • Tumbling windows                           • Email / SMS             │  │
│   │  • Anomaly detection function                 • Teams notification      │  │
│   │                                                                          │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Technology Decisions Explained:**

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Device Ingestion** | IoT Hub | Device management, provisioning, bi-directional communication |
| **Streaming Buffer** | Event Hubs | High-throughput telemetry ingestion, Kafka compatibility |
| **Real-time Processing** | Stream Analytics | SQL-based stream processing, built-in anomaly detection |
| **Hot Store** | Azure Data Explorer | Sub-second queries on time-series at scale |
| **Cold Store** | ADLS Gen2 | Cost-effective long-term storage, analytics-ready |
| **ML/Historical** | Databricks | Advanced analytics, ML model training |
| **Dashboards** | Grafana + Power BI | Real-time operational + business dashboards |

---

### Case Study 3: Multi-Tenant SaaS Platform

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    MULTI-TENANT SAAS PLATFORM ARCHITECTURE                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   REQUIREMENTS:                                                                  │
│   • 500 tenants, ranging from 10 to 10,000 users each                           │
│   • Tenant isolation for data and performance                                    │
│   • Self-service onboarding                                                      │
│   • Per-tenant billing based on usage                                            │
│   • Customizable branding per tenant                                             │
│                                                                                  │
│   TENANTS                                                                        │
│   ┌────────┐ ┌────────┐ ┌────────┐                                             │
│   │Tenant A│ │Tenant B│ │Tenant C│                                             │
│   │(Small) │ │(Medium)│ │(Large) │                                             │
│   └───┬────┘ └───┬────┘ └───┬────┘                                             │
│       │          │          │                                                    │
│       └──────────┴──────────┴───────────────────────────────────────┐           │
│                                                                      │           │
│   ┌──────────────────────────────────────────────────────────────────▼────────┐ │
│   │                    IDENTITY & ROUTING                                      │ │
│   │  ┌───────────────────┐  ┌───────────────────┐  ┌───────────────────┐     │ │
│   │  │    Entra ID       │  │   Front Door      │  │  Tenant Router    │     │ │
│   │  │  (B2B / B2C)      │  │  (Custom domains) │  │  (Tenant context) │     │ │
│   │  └───────────────────┘  └───────────────────┘  └───────────────────┘     │ │
│   └──────────────────────────────────────────────────────────────────┬────────┘ │
│                                                                      │           │
│   ┌──────────────────────────────────────────────────────────────────▼────────┐ │
│   │                    APPLICATION TIER                                        │ │
│   │                                                                            │ │
│   │   SHARED COMPUTE (Container Apps)                                          │ │
│   │   ┌────────────────────────────────────────────────────────────────────┐  │ │
│   │   │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │  │ │
│   │   │  │  Web API │ │ Worker   │ │ Scheduler│ │ Webhooks │              │  │ │
│   │   │  │ (Shared) │ │ (Shared) │ │ (Shared) │ │ (Shared) │              │  │ │
│   │   │  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │  │ │
│   │   │                                                                    │  │ │
│   │   │  Per-request tenant context injection                              │  │ │
│   │   │  Resource governance via middleware                                │  │ │
│   │   └────────────────────────────────────────────────────────────────────┘  │ │
│   │                                                                            │ │
│   │   DEDICATED COMPUTE (Premium tenants - AKS)                               │ │
│   │   ┌────────────────────────────────────────────────────────────────────┐  │ │
│   │   │  ┌──────────────────┐                                              │  │ │
│   │   │  │   Tenant C       │  Dedicated namespace / node pool            │  │ │
│   │   │  │   (Isolated)     │  Custom scaling policies                    │  │ │
│   │   │  └──────────────────┘                                              │  │ │
│   │   └────────────────────────────────────────────────────────────────────┘  │ │
│   └────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                    DATA TIER (Isolation Models)                           │  │
│   │                                                                          │  │
│   │   SHARED DATABASE (Pool model - Small tenants)                           │  │
│   │   ┌────────────────────────────────────────────────────────────────────┐ │  │
│   │   │   Azure SQL Elastic Pool                                            │ │  │
│   │   │   ┌────────────┐ ┌────────────┐ ┌────────────┐                    │ │  │
│   │   │   │ TenantA_DB │ │ TenantB_DB │ │  Shared    │                    │ │  │
│   │   │   │            │ │            │ │  Schema    │                    │ │  │
│   │   │   └────────────┘ └────────────┘ │(TenantId)  │                    │ │  │
│   │   │                                  └────────────┘                    │ │  │
│   │   └────────────────────────────────────────────────────────────────────┘ │  │
│   │                                                                          │  │
│   │   DEDICATED DATABASE (Premium tenants)                                   │  │
│   │   ┌────────────────────────────────────────────────────────────────────┐ │  │
│   │   │   ┌─────────────────────────┐                                      │ │  │
│   │   │   │  TenantC_DB (Dedicated) │  Own Azure SQL instance            │ │  │
│   │   │   │  Business Critical tier │  Customer-managed keys              │ │  │
│   │   │   └─────────────────────────┘                                      │ │  │
│   │   └────────────────────────────────────────────────────────────────────┘ │  │
│   │                                                                          │  │
│   │   SHARED SERVICES                                                        │  │
│   │   ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                   │  │
│   │   │    Redis     │ │   Cosmos DB  │ │ Blob Storage │                   │  │
│   │   │ (per-tenant  │ │  (Tenant     │ │ (per-tenant  │                   │  │
│   │   │  key prefix) │ │  partition)  │ │  container)  │                   │  │
│   │   └──────────────┘ └──────────────┘ └──────────────┘                   │  │
│   │                                                                          │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│   ┌──────────────────────────────────────────────────────────────────────────┐  │
│   │                    BILLING & METERING                                     │  │
│   │                                                                          │  │
│   │   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐              │  │
│   │   │  Event Hubs  │───►│  Functions   │───►│  Cosmos DB   │              │  │
│   │   │  (Usage      │    │  (Aggregate) │    │  (Usage      │              │  │
│   │   │   events)    │    │              │    │   store)     │              │  │
│   │   └──────────────┘    └──────────────┘    └──────────────┘              │  │
│   │                                                                          │  │
│   └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Technology Decisions Explained:**

| Component | Choice | Rationale |
|-----------|--------|-----------|
| **Shared Compute** | Container Apps | Scale-to-zero, cost-effective for variable load |
| **Premium Compute** | AKS | Dedicated resources, advanced isolation |
| **Identity** | Entra ID B2B/B2C | Enterprise SSO and consumer identity |
| **Shared Data** | SQL Elastic Pool | Cost-effective multi-tenant database |
| **Premium Data** | Dedicated Azure SQL | Performance isolation, compliance |
| **Document Store** | Cosmos DB | Partition by tenantId for isolation |
| **Caching** | Redis | Per-tenant key prefix for isolation |
| **Metering** | Event Hubs + Functions | High-throughput usage tracking |

### Multi-Tenancy Data Isolation Patterns

| Pattern | Isolation Level | Cost | Complexity | Use When |
|---------|----------------|------|------------|----------|
| **Shared schema** | Low | Lowest | Low | Cost-sensitive, trusted tenants |
| **Schema per tenant** | Medium | Low | Medium | Moderate isolation needs |
| **Database per tenant** | High | Medium | Medium | Compliance, noisy neighbors |
| **Dedicated instance** | Highest | High | High | Enterprise, regulated industries |

---

## Technology Selection Checklist

Before finalizing technology choices, validate against these criteria:

### Compute Selection Checklist

- [ ] Scaling requirements defined (min/max instances, scaling triggers)
- [ ] Cold start tolerance assessed
- [ ] State management strategy determined
- [ ] Networking requirements documented (VNet, private endpoints)
- [ ] Cost model aligned with usage patterns
- [ ] Team expertise evaluated
- [ ] Deployment complexity acceptable

### Data Store Selection Checklist

- [ ] Data model requirements clear (relational, document, key-value)
- [ ] Consistency requirements defined (strong, eventual, tunable)
- [ ] Performance SLAs documented (latency, throughput)
- [ ] Scaling strategy determined (vertical, horizontal, global)
- [ ] Data residency and compliance requirements met
- [ ] Backup and DR requirements addressed
- [ ] Cost projections completed

### Messaging Selection Checklist

- [ ] Message pattern identified (command, event, stream)
- [ ] Ordering requirements defined
- [ ] Delivery guarantees specified
- [ ] Message size and throughput estimated
- [ ] Dead-letter handling planned
- [ ] Integration with compute tier confirmed
- [ ] Monitoring and alerting strategy defined

---

*Continue to [Case Studies](case-studies.md)*

*Back to [Design Principles](02-design-principles.md)*

*Back to [Architecture Styles Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
