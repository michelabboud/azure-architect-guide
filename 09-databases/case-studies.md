# Database Case Studies

> *"Data is the new oil. And like oil, it's valuable, but if unrefined, it cannot really be used."* — Clive Humby
>
> *Also like oil, it can cause massive explosions if handled incorrectly.*

---

## Case Study 1: The Great DynamoDB-to-Cosmos Migration 🚀

### The Setup

**Company:** StreamFlix (fictional)
**Challenge:** Migrate from DynamoDB to Cosmos DB without users noticing
**Scale:** 50M users, 10B requests/day, 50TB data
**Constraint:** Zero downtime (streaming service, remember?)

### Why They Switched

```
The Conversation That Started It All:
─────────────────────────────────────

CTO: "We're going all-in on Azure for our new AI features."
DBA: "Cool. What about our DynamoDB tables?"
CTO: "Move them to Cosmos DB."
DBA: "All 47 of them?"
CTO: "Yes."
DBA: "With zero downtime?"
CTO: "Obviously."
DBA: *internal screaming* "Challenge accepted."
```

### The Migration Strategy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DUAL-WRITE MIGRATION PATTERN                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  PHASE 1: Setup (Week 1-2)                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │  • Provision Cosmos DB with same capacity planning                    │   │
│  │  • Create matching containers (partition key strategy!)               │   │
│  │  • Set up Azure Data Factory for initial sync                        │   │
│  │  • Deploy dual-write application layer                               │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  PHASE 2: Historical Migration (Week 2-3)                                   │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │   DynamoDB ──────────────► Azure Data Factory                        │   │
│  │   (50TB)                         │                                    │   │
│  │                                  │ Parallel pipelines                │   │
│  │                                  │ 1M docs/minute                    │   │
│  │                                  ▼                                    │   │
│  │                            Cosmos DB                                  │   │
│  │                                                                       │   │
│  │   Time: 72 hours                                                     │   │
│  │   Cost: $12,000 in throughput (worth it)                            │   │
│  │                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  PHASE 3: Dual-Write (Week 3-4)                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │   Application                                                         │   │
│  │       │                                                               │   │
│  │       ├──────► DynamoDB (Primary reads/writes)                       │   │
│  │       │                                                               │   │
│  │       └──────► Cosmos DB (Shadow writes, validation reads)           │   │
│  │                     │                                                 │   │
│  │                     ▼                                                 │   │
│  │               Compare results                                         │   │
│  │               Log discrepancies                                       │   │
│  │               Alert if mismatch > 0.01%                              │   │
│  │                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  PHASE 4: Cutover (Week 5)                                                  │
│  ┌──────────────────────────────────────────────────────────────────────┐   │
│  │                                                                       │   │
│  │   Feature flag: COSMOS_PRIMARY = true                                 │   │
│  │                                                                       │   │
│  │   Gradual rollout:                                                    │   │
│  │   • 1% traffic  → Monitor for 4 hours                                │   │
│  │   • 10% traffic → Monitor for 8 hours                                │   │
│  │   • 50% traffic → Monitor for 24 hours                               │   │
│  │   • 100% traffic → 🎉                                                 │   │
│  │                                                                       │   │
│  │   Rollback plan: Flip flag, < 30 seconds                             │   │
│  │                                                                       │   │
│  └──────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Partition Key Puzzle

Their biggest challenge? Partition key design.

```
DynamoDB Design:                    Cosmos DB Design:
─────────────────                   ─────────────────

Table: user_watch_history           Container: watchHistory
PK: userId                          Partition Key: /userId
SK: timestamp
                                    Problem: Hot partition!
                                    User "binge_watcher_9000" has
                                    500K documents, causing throttling

Solution:
─────────

// Before: Simple userId partition
{
  "userId": "user123",
  "movieId": "movie456",
  "timestamp": "2024-01-15T..."
}

// After: Synthetic partition key
{
  "userId": "user123",
  "partitionKey": "user123_2024-01",  // userId + month
  "movieId": "movie456",
  "timestamp": "2024-01-15T..."
}

// Query pattern adjusted:
// Instead of: WHERE userId = 'user123'
// Now: WHERE partitionKey IN ('user123_2024-01', 'user123_2023-12', ...)
```

### Results

| Metric | DynamoDB | Cosmos DB | Verdict |
|--------|----------|-----------|---------|
| P99 Latency | 15ms | 8ms | 🏆 Cosmos |
| Cost (comparable workload) | $45K/month | $38K/month | 🏆 Cosmos |
| Global distribution | 3 regions | 5 regions | 🏆 Cosmos |
| Migration downtime | — | 0 seconds | 🎉 |

---

## Case Study 2: When SQL Server Met Azure (A Love Story) 💙

### The Setup

**Company:** FinanceFirst Bank (fictional)
**Situation:** 15-year-old SQL Server estate, 200+ databases
**Goal:** Modernize without breaking anything (banks hate broken things)

### The Database Landscape

```
DISCOVERY PHASE RESULTS:
────────────────────────

Total Databases: 247

By Size:
├── < 100 GB: 180 databases
├── 100 GB - 1 TB: 52 databases
├── 1 TB - 5 TB: 12 databases
└── > 5 TB: 3 databases (the scary ones)

By Criticality:
├── Tier 1 (Core Banking): 15 databases
├── Tier 2 (Customer Facing): 45 databases
├── Tier 3 (Internal Apps): 87 databases
└── Tier 4 (Nobody knows): 100 databases 🤷

By SQL Version:
├── SQL Server 2019: 50
├── SQL Server 2017: 80
├── SQL Server 2016: 70
├── SQL Server 2012: 40 (uh oh)
└── SQL Server 2008: 7 (WHY?!)
```

### The Decision Framework

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DATABASE MODERNIZATION DECISION TREE                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Start Here                                                                 │
│       │                                                                      │
│       ▼                                                                      │
│   ┌─────────────────────────────────────────┐                               │
│   │ Does it use SQL Server-specific features?│                               │
│   │ (Linked servers, CLR, Service Broker)   │                               │
│   └────────────────┬────────────────────────┘                               │
│                    │                                                         │
│          ┌────────┴────────┐                                                │
│          │                 │                                                │
│         YES               NO                                                │
│          │                 │                                                │
│          ▼                 ▼                                                │
│   ┌─────────────┐   ┌─────────────────────────────┐                        │
│   │ SQL Managed │   │ Need instance-level control?│                        │
│   │ Instance    │   └──────────────┬──────────────┘                        │
│   └─────────────┘                  │                                        │
│                          ┌─────────┴─────────┐                              │
│                          │                   │                              │
│                         YES                 NO                              │
│                          │                   │                              │
│                          ▼                   ▼                              │
│                   ┌─────────────┐     ┌─────────────────────┐              │
│                   │ SQL Managed │     │ Size > 4TB or need  │              │
│                   │ Instance    │     │ 100+ DTU equivalent?│              │
│                   └─────────────┘     └──────────┬──────────┘              │
│                                                  │                          │
│                                        ┌─────────┴─────────┐                │
│                                        │                   │                │
│                                       YES                 NO                │
│                                        │                   │                │
│                                        ▼                   ▼                │
│                                 ┌─────────────┐     ┌─────────────┐        │
│                                 │ SQL DB      │     │ SQL DB      │        │
│                                 │ Hyperscale  │     │ Standard    │        │
│                                 └─────────────┘     └─────────────┘        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Migration Waves

**Wave 1: Low-Risk Wins (Month 1-2)**
```
Target: Tier 4 databases (the "nobody knows" ones)
Method: Azure Database Migration Service
Result: 87 databases migrated
Surprises: 12 databases actually were important
Learning: Always validate with app owners
```

**Wave 2: Internal Apps (Month 3-4)**
```
Target: Tier 3 databases
Method: Online migration with DMS
Downtime: < 5 minutes per database
Fun fact: Found 15 databases with no connections in 2 years
         (Deleted them, nobody noticed)
```

**Wave 3: Customer Facing (Month 5-6)**
```
Target: Tier 2 databases
Method: Transactional replication + cutover
Downtime: < 30 seconds (maintenance window)
Stress level: Medium-High
Coffee consumed: Excessive
```

**Wave 4: Core Banking (Month 7-8)**
```
Target: Tier 1 databases (15 most critical)
Method: SQL Managed Instance with Link
Downtime: Zero (real-time sync, instant failover)
Regulatory approval: Required and obtained
Backup plans: Had backup plans for our backup plans
```

### Cost Optimization Post-Migration

```
BEFORE: On-Premises
────────────────────
• 12 physical servers: $180K/year (hardware refresh)
• SQL licenses: $400K/year
• Data center costs: $150K/year
• DBA overtime: Priceless (actually $80K)
• Total: ~$810K/year

AFTER: Azure
─────────────
• SQL Managed Instance (Tier 1): $15K/month
• SQL Database (Tier 2-3): $20K/month
• Hyperscale (Big DBs): $8K/month
• Reserved capacity (3-year): -40%
• Total: ~$310K/year

SAVINGS: $500K/year 💰
ROI: < 1 year
DBA sleep quality: Significantly improved
```

---

## Case Study 3: The Real-Time Analytics Adventure 📊

### The Setup

**Company:** GameStream (fictional)
**Challenge:** Real-time analytics for 100M gaming events/second
**Requirements:**
- Sub-second query latency
- 90-day hot data retention
- 7-year cold data retention
- Multi-region for global players

### The Architecture Evolution

```
VERSION 1 (The Naïve Approach):
───────────────────────────────

Everything in Cosmos DB!

Result:
• Cost: $150K/month 💸
• Query latency: 200ms-2s (too slow)
• Aggregations: Painful
• Finance team: Unhappy

VERSION 2 (The Overengineered Approach):
────────────────────────────────────────

Lambda architecture with everything!
• Kafka → Cosmos DB (hot)
• Kafka → Spark → Azure SQL (warm)
• Spark → Data Lake (cold)
• Synapse for analytics

Result:
• Cost: $80K/month
• Complexity: 🤯
• Operational overhead: 3 FTEs
• Time to debug issues: Days

VERSION 3 (The "Why Didn't We Do This First" Approach):
───────────────────────────────────────────────────────

Azure Data Explorer (ADX) for the win!
```

### The Winning Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REAL-TIME GAMING ANALYTICS                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Game Servers (Global)                                                      │
│   100M events/second                                                         │
│         │                                                                    │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                      Event Hubs (Partitioned)                        │   │
│   │                      100 partitions, 20 TUs                          │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         │ Streaming ingestion                                                │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Azure Data Explorer (ADX)                         │   │
│   │                                                                      │   │
│   │   ┌─────────────────────────────────────────────────────────────┐   │   │
│   │   │  HOT CACHE (SSD)           │  COLD STORAGE (Blob)           │   │   │
│   │   │  Last 7 days               │  8-90 days                     │   │   │
│   │   │  Sub-second queries        │  Seconds to query              │   │   │
│   │   │  High cost per GB          │  Low cost per GB               │   │   │
│   │   └─────────────────────────────────────────────────────────────┘   │   │
│   │                                                                      │   │
│   │   Cluster: 16 nodes, D14_v2                                         │   │
│   │   Ingestion: 100M events/sec sustained                              │   │
│   │   Query latency: P95 < 500ms                                        │   │
│   │                                                                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│         │                                                                    │
│         │ Continuous export (90+ days)                                      │
│         ▼                                                                    │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                    Data Lake Gen2 (Archive)                          │   │
│   │                    Parquet format, 7-year retention                  │   │
│   │                    Query via Synapse Serverless                      │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   DASHBOARDS                                                                 │
│   ┌───────────────┐  ┌───────────────┐  ┌───────────────┐                  │
│   │ Grafana       │  │ Power BI      │  │ Custom React  │                  │
│   │ (Operations)  │  │ (Business)    │  │ (Players)     │                  │
│   └───────────────┘  └───────────────┘  └───────────────┘                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Sample Queries

```kusto
// Real-time player count by region (sub-second)
GameEvents
| where Timestamp > ago(5m)
| where EventType == "PlayerActive"
| summarize ActivePlayers = dcount(PlayerId) by Region
| order by ActivePlayers desc

// Revenue by game mode (last 24 hours)
Transactions
| where Timestamp > ago(24h)
| summarize Revenue = sum(Amount) by GameMode
| render piechart

// Anomaly detection (cheating?)
GameEvents
| where Timestamp > ago(1h)
| where EventType == "Kill"
| summarize Kills = count() by PlayerId
| where Kills > 100  // Probably cheating
| project PlayerId, Kills, Suspicion = "HIGH"
```

### Results

| Metric | Before (Lambda) | After (ADX) |
|--------|-----------------|-------------|
| Monthly cost | $80K | $35K |
| Query latency (P95) | 2-5 seconds | 300ms |
| Operational FTEs | 3 | 0.5 |
| Time to new dashboard | 1-2 weeks | 1-2 hours |
| Developer happiness | 😫 | 😊 |

---

## Case Study 4: The Multi-Model Database Dilemma 🤔

### The Setup

**Company:** HealthTech (fictional)
**Challenge:** Different data types, different access patterns, one platform

### The Data Diversity

```
DATA TYPES IN THE SYSTEM:
─────────────────────────

1. Patient Records (Relational)
   • HIPAA compliance required
   • Complex joins across tables
   • Audit trail mandatory
   • Volume: 50M patients

2. Medical Images (Binary/Blob)
   • X-rays, MRIs, CT scans
   • Large files (50MB - 2GB)
   • Metadata queries needed
   • Volume: 500TB

3. IoT Sensor Data (Time-series)
   • Heart monitors, glucose sensors
   • High-frequency (1000 samples/sec/device)
   • 30-day hot, 7-year archive
   • Volume: 10B samples/day

4. Doctor Notes (Document)
   • Unstructured text
   • Full-text search needed
   • Version history
   • Volume: 100M documents

5. Patient Relationships (Graph)
   • Family history
   • Treatment pathways
   • Doctor-patient relationships
   • Complex traversals
```

### The Multi-Database Solution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    POLYGLOT PERSISTENCE ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌───────────────────────────────────────────────────────────────────────┐ │
│   │                        API LAYER (FHIR Compliant)                      │ │
│   └───────────────────────────────────────────────────────────────────────┘ │
│                                    │                                         │
│         ┌──────────────┬───────────┼───────────┬──────────────┐            │
│         │              │           │           │              │            │
│         ▼              ▼           ▼           ▼              ▼            │
│   ┌───────────┐  ┌───────────┐ ┌───────┐ ┌───────────┐ ┌───────────┐      │
│   │ Azure SQL │  │ Blob +    │ │  ADX  │ │ Cosmos DB │ │ Cosmos DB │      │
│   │ (HIPAA)   │  │ Cognitive │ │       │ │ (Document)│ │ (Gremlin) │      │
│   │           │  │ Search    │ │       │ │           │ │           │      │
│   │ Patient   │  │           │ │ IoT   │ │ Notes     │ │ Graph     │      │
│   │ Records   │  │ Medical   │ │ Data  │ │           │ │           │      │
│   │           │  │ Images    │ │       │ │           │ │           │      │
│   └───────────┘  └───────────┘ └───────┘ └───────────┘ └───────────┘      │
│                                                                              │
│   WHY EACH CHOICE:                                                          │
│   ─────────────────                                                         │
│   • SQL: ACID transactions, complex queries, mature compliance tooling     │
│   • Blob + Search: Cost-effective large file storage + metadata queries    │
│   • ADX: Purpose-built for time-series, compression, fast aggregations    │
│   • Cosmos (Doc): Flexible schema, global distribution for mobile apps    │
│   • Cosmos (Gremlin): Native graph traversal for relationship queries     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Integration Pattern

```csharp
// Unified Patient Context Service

public class PatientService
{
    private readonly SqlConnection _sqlDb;
    private readonly CosmosClient _cosmosClient;
    private readonly DataExplorerClient _adxClient;

    public async Task<PatientContext> GetFullPatientContext(string patientId)
    {
        // Parallel fetch from all databases
        var tasks = new[]
        {
            GetDemographicsAsync(patientId),      // SQL
            GetRecentNotesAsync(patientId),       // Cosmos Document
            GetVitalHistoryAsync(patientId),      // ADX
            GetFamilyHistoryAsync(patientId),     // Cosmos Gremlin
            GetImageMetadataAsync(patientId)      // Cognitive Search
        };

        await Task.WhenAll(tasks);

        return new PatientContext
        {
            Demographics = await tasks[0],
            RecentNotes = await tasks[1],
            VitalHistory = await tasks[2],
            FamilyHistory = await tasks[3],
            Images = await tasks[4]
        };
    }
}
```

### Lessons Learned

| Lesson | Details |
|--------|---------|
| **Use the right tool** | Don't force SQL for everything |
| **Plan for integration** | API layer abstracts complexity |
| **Consider operations** | More databases = more to manage |
| **Think about consistency** | Cross-database transactions are hard |
| **Security uniformly** | One identity, one audit trail |

---

## Summary: Database Selection Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    WHEN TO USE WHAT                                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Azure SQL Database                                                          │
│  ✓ Complex transactions                                                     │
│  ✓ Existing SQL Server apps                                                 │
│  ✓ Regulatory compliance                                                    │
│  ✓ Complex reporting                                                        │
│                                                                              │
│  Cosmos DB                                                                   │
│  ✓ Global distribution needed                                               │
│  ✓ Variable/flexible schema                                                 │
│  ✓ High write throughput                                                    │
│  ✓ Low latency at scale                                                     │
│                                                                              │
│  Azure Data Explorer (ADX)                                                   │
│  ✓ Time-series / telemetry                                                  │
│  ✓ Log analytics                                                            │
│  ✓ High-volume ingestion                                                    │
│  ✓ Fast ad-hoc queries                                                      │
│                                                                              │
│  Azure Database for PostgreSQL                                               │
│  ✓ Open source preference                                                   │
│  ✓ PostGIS (geospatial)                                                     │
│  ✓ Existing PostgreSQL apps                                                 │
│                                                                              │
│  Azure Cache for Redis                                                       │
│  ✓ Session state                                                            │
│  ✓ Caching layer                                                            │
│  ✓ Pub/sub messaging                                                        │
│  ✓ Rate limiting                                                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

*Back to [Chapter Overview](README.md)* | *Next: [Extra Resources](extra-resources.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
