# Cosmos DB Comprehensive Guide

## Overview

Azure Cosmos DB is a globally distributed, multi-model database service designed for mission-critical applications. Unlike AWS DynamoDB which offers only key-value and document models with two consistency levels, Cosmos DB supports five data models through different APIs and provides five tunable consistency levels.

## AWS Comparison

| Feature | AWS DynamoDB | Azure Cosmos DB |
|---------|--------------|-----------------|
| Data models | Key-value, Document | Document, Key-value, Graph, Column-family, Table |
| Consistency levels | 2 (eventual, strong) | 5 (tunable per request) |
| Global distribution | Global Tables | Native multi-region, multi-write |
| Partitioning | Automatic (partition key) | Automatic (partition key) |
| Capacity model | On-demand or Provisioned | Provisioned RU/s or Serverless |
| Secondary indexes | GSI, LSI (limited) | Automatic indexing (all properties) |
| Change feed | DynamoDB Streams | Native change feed |
| Transactions | TransactWriteItems (25 items) | Stored procedures, batch (100 items) |
| Latency SLA | No SLA | <10ms read, <10ms write at p99 |
| Availability SLA | 99.999% (global) | 99.999% (multi-region) |

---

## APIs Comparison

### Supported APIs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Cosmos DB APIs                                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  NoSQL (Core) API                                                           │
│  ───────────────                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Native Cosmos DB API (recommended for new apps)                   │   │
│  │ • JSON documents with SQL-like query syntax                         │   │
│  │ • Best performance and feature support                              │   │
│  │ • Full SDK support (.NET, Java, Python, Node.js)                   │   │
│  │                                                                      │   │
│  │ Query: SELECT * FROM c WHERE c.category = "electronics"            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  MongoDB API                                                                │
│  ───────────                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Wire protocol compatible with MongoDB 4.2+                        │   │
│  │ • Use existing MongoDB drivers and tools                            │   │
│  │ • Supports aggregation pipeline                                     │   │
│  │ • Ideal for MongoDB migrations                                      │   │
│  │                                                                      │   │
│  │ Query: db.products.find({ category: "electronics" })               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Cassandra API                                                              │
│  ─────────────                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • CQL (Cassandra Query Language) compatible                         │   │
│  │ • Wide-column store model                                           │   │
│  │ • Use existing Cassandra drivers                                    │   │
│  │ • Ideal for Cassandra migrations                                    │   │
│  │                                                                      │   │
│  │ Query: SELECT * FROM products WHERE category = 'electronics'       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Gremlin API (Graph)                                                        │
│  ───────────────────                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Apache TinkerPop Gremlin traversal language                       │   │
│  │ • Vertices and edges for graph data                                 │   │
│  │ • Social networks, recommendation engines                           │   │
│  │ • Fraud detection, knowledge graphs                                 │   │
│  │                                                                      │   │
│  │ Query: g.V().has('person','name','john').out('knows')              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Table API                                                                  │
│  ─────────                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Compatible with Azure Table Storage API                           │   │
│  │ • Key-value pairs with schema flexibility                           │   │
│  │ • Migration path from Table Storage                                 │   │
│  │ • Premium features (global distribution, indexing)                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  PostgreSQL API (Distributed)                                               │
│  ────────────────────────────                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • PostgreSQL wire protocol (uses Citus extension)                   │   │
│  │ • Distributed relational database                                   │   │
│  │ • Horizontal scaling with sharding                                  │   │
│  │ • Ideal for PostgreSQL apps needing scale-out                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### API Selection Guide

| Scenario | Recommended API | Reasoning |
|----------|-----------------|-----------|
| New cloud-native app | NoSQL (Core) | Best performance, all features |
| MongoDB migration | MongoDB | Wire protocol compatibility |
| Cassandra migration | Cassandra | CQL compatibility |
| Social graph, recommendations | Gremlin | Native graph traversal |
| Azure Table Storage upgrade | Table | API compatibility |
| Distributed PostgreSQL | PostgreSQL | Horizontal scaling |

---

## Consistency Levels

### The Five Levels Explained

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Cosmos DB Consistency Levels                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   STRONG                                                                     │
│   ──────                                                                     │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ • Linearizable reads (always see latest committed write)           │   │
│   │ • Global ordering of operations                                     │   │
│   │ • Highest latency, highest RU cost (~2x)                           │   │
│   │ • Not available with multi-region writes                           │   │
│   │                                                                      │   │
│   │ Use case: Financial transactions, inventory management             │   │
│   │ AWS equivalent: DynamoDB strongly consistent read                  │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   BOUNDED STALENESS                                                         │
│   ─────────────────                                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ • Reads lag behind writes by at most K versions or T time          │   │
│   │ • Configurable staleness window                                    │   │
│   │ • Good for global apps needing strong-ish consistency              │   │
│   │                                                                      │   │
│   │ Configuration: maxStalenessPrefix: 100, maxIntervalInSeconds: 5   │   │
│   │ Use case: Leaderboards, analytics with freshness requirements      │   │
│   │ AWS equivalent: No direct equivalent                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   SESSION (Default)                                                         │
│   ─────────────────                                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ • Read-your-writes within a session                                │   │
│   │ • Monotonic reads and writes per session                           │   │
│   │ • Best balance of consistency and performance                      │   │
│   │ • Most popular choice (default)                                    │   │
│   │                                                                      │   │
│   │ Use case: User profiles, shopping carts, user sessions            │   │
│   │ AWS equivalent: No direct equivalent (closest to eventual + cache)│   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   CONSISTENT PREFIX                                                         │
│   ─────────────────                                                         │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ • Reads never see out-of-order writes                              │   │
│   │ • If writes are A, B, C, reads see A, AB, or ABC (never AC)       │   │
│   │ • No read-your-writes guarantee                                    │   │
│   │                                                                      │   │
│   │ Use case: Social media feeds, activity logs                        │   │
│   │ AWS equivalent: No direct equivalent                               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   EVENTUAL                                                                  │
│   ────────                                                                  │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ • Lowest latency, lowest RU cost                                   │   │
│   │ • No ordering guarantees                                           │   │
│   │ • Eventually converges to final state                              │   │
│   │                                                                      │   │
│   │ Use case: Likes/views counters, non-critical reads                 │   │
│   │ AWS equivalent: DynamoDB eventually consistent read               │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   Consistency Spectrum:                                                      │
│                                                                              │
│   Strong ◄─────────────────────────────────────────────────────► Eventual   │
│     │         │              │              │              │                │
│   100%      Bounded       Session      Consistent     Eventual              │
│  guarantee  Staleness    (default)      Prefix                              │
│     │         │              │              │              │                │
│   ◄──── Higher Latency/Cost                Lower Latency/Cost ────►        │
│   ◄──── Lower Availability                 Higher Availability ────►       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Consistency Level Comparison

| Level | Latency | Cost (RU) | Availability | Multi-Write |
|-------|---------|-----------|--------------|-------------|
| Strong | Highest | 2x | Lower | No |
| Bounded Staleness | High | 2x | Medium | Yes* |
| Session | Medium | 1x | High | Yes |
| Consistent Prefix | Low | 1x | Higher | Yes |
| Eventual | Lowest | 1x | Highest | Yes |

*Bounded Staleness with multi-write uses Strong within region

### Setting Consistency Per Request

```python
from azure.cosmos import CosmosClient, ConsistencyLevel

# Account-level default (set at creation)
client = CosmosClient(
    url=endpoint,
    credential=key,
    consistency_level=ConsistencyLevel.Session  # Default
)

# Override per request (can only weaken, not strengthen)
container.read_item(
    item="doc1",
    partition_key="pk1",
    session_token=session_token,  # For session consistency
    consistency_level=ConsistencyLevel.Eventual  # Weaker than account default
)
```

---

## Partitioning Strategies

### Partition Key Fundamentals

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Cosmos DB Partitioning                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Physical vs Logical Partitions                                             │
│  ──────────────────────────────                                             │
│                                                                              │
│  Container                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  Logical Partitions (defined by partition key)                      │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │Customer │ │Customer │ │Customer │ │Customer │ │Customer │       │   │
│  │  │   A     │ │   B     │ │   C     │ │   D     │ │   E     │       │   │
│  │  │ 5 docs  │ │ 12 docs │ │ 3 docs  │ │ 20 docs │ │ 8 docs  │       │   │
│  │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│  │       │           │           │           │           │             │   │
│  │       └───────────┼───────────┘           └───────────┘             │   │
│  │                   │                             │                    │   │
│  │                   ▼                             ▼                    │   │
│  │  Physical Partitions (managed by Cosmos DB)                         │   │
│  │  ┌─────────────────────────┐  ┌─────────────────────────┐          │   │
│  │  │    Physical Partition 1  │  │    Physical Partition 2  │          │   │
│  │  │    (Customers A, B, C)   │  │    (Customers D, E)      │          │   │
│  │  │    Max: 50 GB, 10K RU/s │  │    Max: 50 GB, 10K RU/s │          │   │
│  │  └─────────────────────────┘  └─────────────────────────┘          │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Key Constraints:                                                           │
│  • Logical partition max size: 20 GB                                       │
│  • Physical partition max: 50 GB storage, 10,000 RU/s                     │
│  • Partition key cannot be changed after container creation               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Partition Key Best Practices

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Partition Key Selection                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  GOOD Partition Keys:                                                       │
│  ───────────────────                                                        │
│                                                                              │
│  ✅ High cardinality (many unique values)                                   │
│     • userId, tenantId, deviceId, orderId                                  │
│                                                                              │
│  ✅ Evenly distributed (no hot partitions)                                  │
│     • Spread requests across partitions                                     │
│                                                                              │
│  ✅ Frequently used in WHERE clauses                                        │
│     • Enables point reads (most efficient)                                  │
│                                                                              │
│  ✅ Matches query patterns                                                  │
│     • Same partition = single-partition query (fast)                       │
│                                                                              │
│  BAD Partition Keys:                                                        │
│  ──────────────────                                                         │
│                                                                              │
│  ❌ Low cardinality                                                         │
│     • status ("active"/"inactive"), country                                │
│     • Creates few large partitions                                         │
│                                                                              │
│  ❌ Timestamp alone                                                         │
│     • All current writes go to same partition (hot spot)                   │
│                                                                              │
│  ❌ Monotonically increasing values                                         │
│     • Sequential IDs create hot partitions                                 │
│                                                                              │
│  ❌ Rarely used in queries                                                  │
│     • Forces cross-partition queries                                        │
│                                                                              │
│  COMPOSITE Partition Keys:                                                  │
│  ─────────────────────────                                                  │
│                                                                              │
│  For time-series or hot partition scenarios:                               │
│                                                                              │
│  tenantId + yyyy-MM → "tenant123_2024-01"                                  │
│  • Spreads load across time-based partitions                               │
│  • Enables efficient queries within time range                             │
│                                                                              │
│  Synthetic partition key:                                                   │
│  • Combine multiple fields: `${tenantId}_${region}_${category}`           │
│  • Store computed value in document                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hierarchical Partition Keys (Preview)

```python
# Create container with hierarchical partition key
container_definition = {
    "id": "orders",
    "partitionKey": {
        "paths": ["/tenantId", "/userId", "/orderDate"],
        "kind": "MultiHash",
        "version": 2
    }
}

# Enables efficient queries at any level:
# - All orders for a tenant (tenantId)
# - All orders for a user within tenant (tenantId + userId)
# - Specific orders for user on date (tenantId + userId + orderDate)
```

---

## Global Distribution

### Multi-Region Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Cosmos DB Global Distribution                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Single-Write (Default)                                                     │
│  ──────────────────────                                                     │
│                                                                              │
│    East US (Write)          West Europe           Southeast Asia            │
│   ┌─────────────┐          ┌─────────────┐       ┌─────────────┐           │
│   │   Primary   │────────▶│   Replica   │──────▶│   Replica   │           │
│   │   (R/W)     │  Async   │   (Read)    │ Async │   (Read)    │           │
│   └─────────────┘          └─────────────┘       └─────────────┘           │
│         │                                                                    │
│         │ All writes go here                                                │
│         │ Automatic failover if unavailable                                 │
│                                                                              │
│  Multi-Region Write                                                         │
│  ─────────────────                                                          │
│                                                                              │
│    East US (R/W)           West Europe (R/W)     Southeast Asia (R/W)       │
│   ┌─────────────┐          ┌─────────────┐       ┌─────────────┐           │
│   │    R/W      │◀────────▶│    R/W      │◀─────▶│    R/W      │           │
│   │   Region    │  Sync*   │   Region    │ Sync* │   Region    │           │
│   └─────────────┘          └─────────────┘       └─────────────┘           │
│                                                                              │
│   * Conflict resolution: Last-Writer-Wins (LWW) or Custom (stored proc)   │
│                                                                              │
│  Benefits:                                                                   │
│  • Low-latency writes from any region                                      │
│  • 99.999% availability SLA                                                │
│  • Automatic conflict resolution                                           │
│                                                                              │
│  Considerations:                                                            │
│  • Higher cost (RU charge in each write region)                           │
│  • Strong consistency not available                                        │
│  • Must handle conflicts                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Conflict Resolution

```python
# Custom conflict resolution using stored procedure
conflict_resolution_policy = {
    "mode": "Custom",
    "conflictResolutionProcedure": "dbo.resolveConflict"
}

# Stored procedure for custom resolution
resolve_conflict_sproc = """
function resolveConflict(incomingItem, existingItem, isTombstone, conflictingItems) {
    // Example: Keep higher version or merge fields
    if (!existingItem) {
        // No existing item, accept incoming
        return true; // Accept incoming
    }

    if (incomingItem.version > existingItem.version) {
        return true; // Accept incoming (higher version)
    }

    if (incomingItem.version === existingItem.version) {
        // Same version - merge strategy
        existingItem.lastModified = Math.max(
            incomingItem.lastModified,
            existingItem.lastModified
        );
        return existingItem; // Return merged
    }

    return false; // Reject incoming (keep existing)
}
"""

# Last-Writer-Wins (default) - uses _ts or custom path
conflict_resolution_policy = {
    "mode": "LastWriterWins",
    "conflictResolutionPath": "/lastModified"  # Custom timestamp field
}
```

### Bicep: Global Distribution

```bicep
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2023-11-15' = {
  name: 'cosmos-${uniqueString(resourceGroup().id)}'
  location: 'eastus'
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
    enableMultipleWriteLocations: true  // Multi-region writes
    enableAutomaticFailover: true
    locations: [
      {
        locationName: 'eastus'
        failoverPriority: 0
        isZoneRedundant: true
      }
      {
        locationName: 'westeurope'
        failoverPriority: 1
        isZoneRedundant: true
      }
      {
        locationName: 'southeastasia'
        failoverPriority: 2
        isZoneRedundant: true
      }
    ]
  }
}

resource database 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-11-15' = {
  parent: cosmosAccount
  name: 'mydb'
  properties: {
    resource: {
      id: 'mydb'
    }
  }
}

resource container 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-11-15' = {
  parent: database
  name: 'orders'
  properties: {
    resource: {
      id: 'orders'
      partitionKey: {
        paths: ['/customerId']
        kind: 'Hash'
      }
      indexingPolicy: {
        indexingMode: 'consistent'
        includedPaths: [
          { path: '/*' }
        ]
        excludedPaths: [
          { path: '/"_etag"/?' }
        ]
      }
      conflictResolutionPolicy: {
        mode: 'LastWriterWins'
        conflictResolutionPath: '/_ts'
      }
    }
    options: {
      autoscaleSettings: {
        maxThroughput: 10000
      }
    }
  }
}
```

---

## Request Units (RU) Optimization

### Understanding RUs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Request Unit (RU) Fundamentals                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1 RU = Cost to read a 1 KB document by ID (point read)                    │
│                                                                              │
│  Operation Costs (approximate):                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Operation                              │ RU Cost                    │   │
│  ├────────────────────────────────────────┼────────────────────────────┤   │
│  │ Point read (1 KB by ID + partition key)│ 1 RU                       │   │
│  │ Point read (10 KB document)            │ ~3 RU                      │   │
│  │ Insert (1 KB document)                 │ ~5 RU                      │   │
│  │ Replace (1 KB document)                │ ~10 RU                     │   │
│  │ Delete (1 KB document)                 │ ~5 RU                      │   │
│  │ Query (single partition, 10 results)   │ ~10-50 RU                  │   │
│  │ Query (cross-partition, 10 results)    │ ~50-200 RU                 │   │
│  │ Stored procedure execution             │ Variable                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Factors Affecting RU Cost:                                                 │
│  • Document size (larger = more RUs)                                       │
│  • Indexing (more indexes = more write RUs)                                │
│  • Consistency level (Strong = 2x read RUs)                                │
│  • Query complexity (filters, aggregations)                                │
│  • Cross-partition queries (fan-out cost)                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Provisioning Models

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Throughput Provisioning Options                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  MANUAL PROVISIONED                                                         │
│  ─────────────────                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Fixed RU/s (400 - 1,000,000)                                      │   │
│  │ • Pay for provisioned capacity                                      │   │
│  │ • Manual scaling                                                     │   │
│  │ • Best for predictable workloads                                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  AUTOSCALE                                                                  │
│  ─────────                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  RU/s                                                                │   │
│  │    ▲                                                                 │   │
│  │ Max├─────────────────────────┐                                      │   │
│  │    │                 ┌───────┴───┐                                  │   │
│  │    │        ┌────────┘           └────┐                             │   │
│  │    │   ┌────┘                         └────┐                        │   │
│  │10% ├───┘                                   └─── (min = 10% of max)  │   │
│  │    └──────────────────────────────────────────────▶ Time            │   │
│  │                                                                      │   │
│  │ • Scales 10% to 100% of max                                         │   │
│  │ • Pay for what you use (per second billing)                        │   │
│  │ • Best for variable workloads                                       │   │
│  │ • Max: 1,000 - 1,000,000 RU/s                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  SERVERLESS                                                                 │
│  ──────────                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Pay per RU consumed (no provisioned capacity)                     │   │
│  │ • Burst up to 5,000 RU/s per partition                             │   │
│  │ • Max 50 GB storage                                                 │   │
│  │ • No SLA                                                            │   │
│  │ • Best for dev/test, low-traffic apps                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  SHARED THROUGHPUT (Database-level)                                        │
│  ──────────────────────────────────                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • RU/s shared across containers in database                        │   │
│  │ • Containers can also have dedicated throughput                    │   │
│  │ • Best for multi-tenant with uneven usage                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### RU Optimization Strategies

```python
from azure.cosmos import CosmosClient, PartitionKey
import json

# 1. Use point reads (most efficient - 1 RU for 1KB)
def get_order_efficient(container, order_id, customer_id):
    # Point read: O(1), ~1 RU
    return container.read_item(
        item=order_id,
        partition_key=customer_id
    )

# Avoid: Query when you know the ID
def get_order_inefficient(container, order_id):
    # Query: Scans partition, ~3-10 RU
    query = f"SELECT * FROM c WHERE c.id = '{order_id}'"
    return list(container.query_items(query, enable_cross_partition_query=True))

# 2. Project only needed fields
def get_order_summary(container, customer_id):
    # Return only needed fields (reduces RU and bandwidth)
    query = """
        SELECT c.id, c.total, c.status
        FROM c
        WHERE c.customerId = @customerId
    """
    return list(container.query_items(
        query=query,
        parameters=[{"name": "@customerId", "value": customer_id}],
        partition_key=customer_id  # Single partition query
    ))

# 3. Use continuation tokens for pagination
def get_orders_paginated(container, customer_id, page_size=10):
    query = "SELECT * FROM c WHERE c.customerId = @customerId"
    results = container.query_items(
        query=query,
        parameters=[{"name": "@customerId", "value": customer_id}],
        partition_key=customer_id,
        max_item_count=page_size  # Limit per page
    )

    page = results.fetch_next_block()
    continuation = results.continuation_token
    return page, continuation

# 4. Check RU consumption
def check_ru_charge(container, order_id, customer_id):
    response = container.read_item(
        item=order_id,
        partition_key=customer_id
    )
    # Access RU charge from response headers
    ru_charge = container.client_connection.last_response_headers['x-ms-request-charge']
    print(f"RU charge: {ru_charge}")
    return response

# 5. Batch operations (transactional batch)
def create_order_with_items(container, order, items):
    # All operations in same partition, atomic
    batch = [
        ("create", (order,)),
    ]
    for item in items:
        batch.append(("create", (item,)))

    container.execute_item_batch(
        batch_operations=batch,
        partition_key=order['customerId']
    )
```

### CLI: Throughput Management

```bash
# Check current throughput
az cosmosdb sql container throughput show \
  --account-name mycosmosdb \
  --database-name mydb \
  --name orders \
  --resource-group myRG

# Update manual throughput
az cosmosdb sql container throughput update \
  --account-name mycosmosdb \
  --database-name mydb \
  --name orders \
  --resource-group myRG \
  --throughput 10000

# Enable autoscale
az cosmosdb sql container throughput migrate \
  --account-name mycosmosdb \
  --database-name mydb \
  --name orders \
  --resource-group myRG \
  --throughput-type autoscale

# Set autoscale max
az cosmosdb sql container throughput update \
  --account-name mycosmosdb \
  --database-name mydb \
  --name orders \
  --resource-group myRG \
  --max-throughput 10000

# Check RU consumption metrics
az monitor metrics list \
  --resource $(az cosmosdb show -n mycosmosdb -g myRG --query id -o tsv) \
  --metric "TotalRequestUnits" \
  --interval PT1H \
  --output table
```

---

## Indexing

### Indexing Strategies

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Cosmos DB Indexing                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Default: Everything is indexed automatically                               │
│                                                                              │
│  Indexing Policy Options:                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  includedPaths: Paths to index                                      │   │
│  │  excludedPaths: Paths to skip                                       │   │
│  │                                                                      │   │
│  │  Example - Index only queried fields:                               │   │
│  │  {                                                                   │   │
│  │    "indexingMode": "consistent",                                    │   │
│  │    "automatic": true,                                               │   │
│  │    "includedPaths": [                                               │   │
│  │      { "path": "/customerId/?" },                                   │   │
│  │      { "path": "/orderDate/?" },                                    │   │
│  │      { "path": "/status/?" }                                        │   │
│  │    ],                                                                │   │
│  │    "excludedPaths": [                                               │   │
│  │      { "path": "/*" }                                               │   │
│  │    ]                                                                 │   │
│  │  }                                                                   │   │
│  │                                                                      │   │
│  │  Benefits of selective indexing:                                    │   │
│  │  • Reduced write RU costs                                           │   │
│  │  • Lower storage costs                                              │   │
│  │  • Faster write performance                                         │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Composite Indexes (for ORDER BY on multiple fields):                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  "compositeIndexes": [                                              │   │
│  │    [                                                                 │   │
│  │      { "path": "/status", "order": "ascending" },                   │   │
│  │      { "path": "/orderDate", "order": "descending" }                │   │
│  │    ]                                                                 │   │
│  │  ]                                                                   │   │
│  │                                                                      │   │
│  │  Enables: SELECT * FROM c ORDER BY c.status ASC, c.orderDate DESC  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Spatial Indexes (for geospatial queries):                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  "spatialIndexes": [                                                │   │
│  │    {                                                                 │   │
│  │      "path": "/location/*",                                         │   │
│  │      "types": ["Point", "Polygon"]                                  │   │
│  │    }                                                                 │   │
│  │  ]                                                                   │   │
│  │                                                                      │   │
│  │  Enables: ST_DISTANCE, ST_WITHIN, ST_INTERSECTS                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Change Feed

### Change Feed Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Cosmos DB Change Feed                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Cosmos DB Container                          │   │
│  │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│  │  │Partition│ │Partition│ │Partition│ │Partition│ │Partition│       │   │
│  │  │    1    │ │    2    │ │    3    │ │    4    │ │    5    │       │   │
│  │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘       │   │
│  │       │           │           │           │           │             │   │
│  └───────┼───────────┼───────────┼───────────┼───────────┼─────────────┘   │
│          │           │           │           │           │                  │
│          ▼           ▼           ▼           ▼           ▼                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Change Feed                                  │   │
│  │  • Ordered within partition                                         │   │
│  │  • Persistent (replay from any point)                              │   │
│  │  • Push or pull model                                               │   │
│  └───────────────────────────────────────────────────────────────────────┘   │
│          │                                                                   │
│          ├──────────────┬──────────────┬──────────────┐                     │
│          ▼              ▼              ▼              ▼                     │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐               │
│  │  Azure    │  │  Event    │  │  Search   │  │  Custom   │               │
│  │ Functions │  │   Hub     │  │  Indexer  │  │ Processor │               │
│  └───────────┘  └───────────┘  └───────────┘  └───────────┘               │
│                                                                              │
│  Use Cases:                                                                 │
│  • Event-driven architectures                                              │
│  • Materialized views / read models                                        │
│  • Real-time analytics                                                     │
│  • Data synchronization                                                    │
│  • Search index updates                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Azure Functions Change Feed Trigger

```csharp
// Azure Functions with Cosmos DB Change Feed trigger
[FunctionName("ProcessOrderChanges")]
public static async Task Run(
    [CosmosDBTrigger(
        databaseName: "mydb",
        containerName: "orders",
        Connection = "CosmosDBConnection",
        LeaseContainerName = "leases",
        CreateLeaseContainerIfNotExists = true)]
    IReadOnlyList<Order> changes,
    ILogger log)
{
    if (changes == null || changes.Count == 0) return;

    foreach (var order in changes)
    {
        log.LogInformation($"Order {order.Id} changed: {order.Status}");

        // Update search index
        await UpdateSearchIndex(order);

        // Send notification
        if (order.Status == "Shipped")
        {
            await SendShipmentNotification(order);
        }

        // Update analytics
        await PublishToEventHub(order);
    }
}
```

---

## Complete Bicep Example

```bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Cosmos DB account name')
param cosmosAccountName string = 'cosmos-${uniqueString(resourceGroup().id)}'

resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2023-11-15' = {
  name: cosmosAccountName
  location: location
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
    enableAutomaticFailover: true
    enableMultipleWriteLocations: false
    locations: [
      {
        locationName: location
        failoverPriority: 0
        isZoneRedundant: true
      }
      {
        locationName: 'westus'
        failoverPriority: 1
        isZoneRedundant: true
      }
    ]
    backupPolicy: {
      type: 'Continuous'
      continuousModeProperties: {
        tier: 'Continuous7Days'
      }
    }
    capabilities: [
      {
        name: 'EnableServerless'  // For serverless, remove throughput settings
      }
    ]
    ipRules: []
    virtualNetworkRules: []
    enableFreeTier: false
  }
}

resource database 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-11-15' = {
  parent: cosmosAccount
  name: 'ecommerce'
  properties: {
    resource: {
      id: 'ecommerce'
    }
  }
}

resource ordersContainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-11-15' = {
  parent: database
  name: 'orders'
  properties: {
    resource: {
      id: 'orders'
      partitionKey: {
        paths: ['/customerId']
        kind: 'Hash'
      }
      indexingPolicy: {
        indexingMode: 'consistent'
        automatic: true
        includedPaths: [
          {
            path: '/customerId/?'
          }
          {
            path: '/orderDate/?'
          }
          {
            path: '/status/?'
          }
        ]
        excludedPaths: [
          {
            path: '/*'
          }
          {
            path: '/"_etag"/?'
          }
        ]
        compositeIndexes: [
          [
            {
              path: '/status'
              order: 'ascending'
            }
            {
              path: '/orderDate'
              order: 'descending'
            }
          ]
        ]
      }
      defaultTtl: -1  // TTL enabled but requires per-document ttl
      conflictResolutionPolicy: {
        mode: 'LastWriterWins'
        conflictResolutionPath: '/_ts'
      }
    }
  }
}

resource leaseContainer 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers@2023-11-15' = {
  parent: database
  name: 'leases'
  properties: {
    resource: {
      id: 'leases'
      partitionKey: {
        paths: ['/id']
        kind: 'Hash'
      }
    }
  }
}

output cosmosDbEndpoint string = cosmosAccount.properties.documentEndpoint
output cosmosDbName string = cosmosAccount.name
```

---

*Next: [PostgreSQL & MySQL](03-postgresql-mysql.md)* | *Previous: [Azure SQL](01-azure-sql.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
