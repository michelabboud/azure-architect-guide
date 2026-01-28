# Architecture Styles Case Studies

*Real-world architecture decisions with the debates that shaped them*

---

## Case Study 1: Monolith to Microservices - The "Right Way"

### The Setup

**Company:** TravelBuddy
**Industry:** Online Travel Booking
**Team Size:** 85 engineers

### The Problem: Monolith Growing Pains

```
THE BEAST THEY'D CREATED:
─────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   TravelBuddy.exe (The Monolith)                                    │
│   ──────────────────────────────                                    │
│                                                                      │
│   ┌────────────────────────────────────────────────────────────┐   │
│   │                                                             │   │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐        │   │
│   │   │ Booking │ │ Payment │ │ Hotels  │ │ Flights │        │   │
│   │   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘        │   │
│   │        │           │           │           │              │   │
│   │        └───────────┴───────────┴───────────┘              │   │
│   │                         │                                  │   │
│   │                    ┌────┴────┐                             │   │
│   │                    │ Shared  │  1.2M lines of code        │   │
│   │                    │  DB     │  472 database tables        │   │
│   │                    └─────────┘  8-hour build times         │   │
│   │                                 3-week release cycles      │   │
│   │                                                             │   │
│   └────────────────────────────────────────────────────────────┘   │
│                                                                      │
│   "Touching the booking code? That's a 3-week project."            │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Architecture Decision

The team debated three approaches:

**Option A: "Big Bang" Microservices**
- Rewrite everything from scratch
- 18-24 months estimated
- High risk, high reward
- *CTO's nightmare scenario*

**Option B: "Strangler Fig" Pattern**
- Gradually extract services
- Keep monolith running
- 12-18 months
- *Selected approach*

**Option C: "Modular Monolith" First**
- Refactor internal modules
- Add clear boundaries
- Then extract if needed
- *Considered but rejected* (too late, boundaries already violated)

### The Strangler Fig Journey

```
PHASE 1: IDENTIFY THE "EASY WINS" (Month 1-2)
─────────────────────────────────────────────

Analysis of code coupling:

┌────────────────────┬───────────────┬──────────────┬─────────────────┐
│ Module             │ Coupling Score│ Change Freq  │ Extract First?  │
├────────────────────┼───────────────┼──────────────┼─────────────────┤
│ Email Service      │ Low (12 deps) │ Low          │ ✓ Perfect       │
│ Currency Converter │ Low (8 deps)  │ Low          │ ✓ Perfect       │
│ Notifications      │ Medium (23)   │ Medium       │ ✓ Good          │
│ Hotel Search       │ High (67 deps)│ High         │ ✗ Later         │
│ Booking Engine     │ Extreme (143) │ Very High    │ ✗ Much Later    │
│ Payment            │ High (89 deps)│ Medium       │ ✗ Later         │
└────────────────────┴───────────────┴──────────────┴─────────────────┘

Winner: Email Service (low risk, high learning value)
```

```
PHASE 2: EXTRACT EMAIL SERVICE (Month 2-3)
──────────────────────────────────────────

Before:
┌─────────────────────────────────────────────────────────────────────┐
│   Monolith                                                          │
│   ┌───────────────────────────────────────────────────────────┐    │
│   │                                                            │    │
│   │   Booking ────► EmailHelper.SendConfirmation() ─────────► SMTP │
│   │                                                            │    │
│   │   (Direct method call, 47 places in code)                  │    │
│   └───────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────┘

After:
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Monolith                          Email Service                    │
│   ┌────────────────────────┐        ┌────────────────────────┐     │
│   │                        │        │                        │     │
│   │   Booking ───────────────► Service Bus ───► Email API    │     │
│   │                        │        │             │          │     │
│   │   (Send message,       │        │             ▼          │     │
│   │    forget about it)    │        │           SendGrid     │     │
│   └────────────────────────┘        └────────────────────────┘     │
│                                                                      │
│   Benefits achieved:                                                 │
│   ├── Email service deploys independently                           │
│   ├── Can switch email providers without touching monolith         │
│   ├── Async processing (no more timeout on bulk emails)            │
│   └── Team gained microservice experience                          │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

```
PHASE 3: THE HARD PART - HOTEL SEARCH (Month 4-8)
─────────────────────────────────────────────────

The Challenge:
├── 67 internal dependencies
├── 5 different teams touching this code
├── Real-time availability required
├── 2M searches/day

Strategy: "Branch by Abstraction"

Step 1: Create interface
┌─────────────────────────────────────────────────────────────────────┐
│   interface IHotelSearchService {                                    │
│       Task<SearchResults> SearchAsync(SearchQuery query);            │
│       Task<HotelDetails> GetDetailsAsync(string hotelId);           │
│   }                                                                  │
│                                                                      │
│   // Old implementation still works                                  │
│   class LegacyHotelSearch : IHotelSearchService { ... }             │
│                                                                      │
│   // New implementation talks to microservice                       │
│   class NewHotelSearchClient : IHotelSearchService { ... }          │
└─────────────────────────────────────────────────────────────────────┘

Step 2: Feature flag rollout
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   if (FeatureFlags.UseNewHotelSearch)                               │
│   {                                                                  │
│       // 1% → 5% → 25% → 50% → 100%                                │
│       return await _newHotelSearch.SearchAsync(query);              │
│   }                                                                  │
│   return await _legacyHotelSearch.SearchAsync(query);               │
│                                                                      │
│   Week 1:  1% traffic to new service (find obvious bugs)           │
│   Week 2:  5% traffic (validate performance)                        │
│   Week 3:  25% traffic (stress test)                                │
│   Week 4:  50% traffic (build confidence)                           │
│   Week 5:  100% traffic (🎉)                                        │
│   Week 8:  Delete legacy code (the best feeling)                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Azure Architecture for Extracted Services

```bicep
// Hotel Search Microservice on AKS
resource hotelSearchDeployment 'apps/Deployment@v1' = {
  metadata: {
    name: 'hotel-search'
    labels: {
      app: 'hotel-search'
    }
  }
  spec: {
    replicas: 6  // High availability for critical service
    selector: {
      matchLabels: {
        app: 'hotel-search'
      }
    }
    template: {
      spec: {
        containers: [
          {
            name: 'hotel-search'
            image: 'travelbuddy.azurecr.io/hotel-search:v2.1.0'
            resources: {
              requests: {
                cpu: '500m'
                memory: '1Gi'
              }
              limits: {
                cpu: '2'
                memory: '4Gi'
              }
            }
            ports: [
              { containerPort: 8080 }
            ]
            livenessProbe: {
              httpGet: {
                path: '/health'
                port: 8080
              }
              initialDelaySeconds: 15
              periodSeconds: 10
            }
          }
        ]
      }
    }
  }
}
```

### Final Architecture (18 months later)

```
FROM MONOLITH TO MICROSERVICES:
───────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Azure Front Door (Global Load Balancing)                          │
│              │                                                       │
│              ▼                                                       │
│   ┌──────────────────────┐                                          │
│   │   API Management     │                                          │
│   │   (Rate limiting,    │                                          │
│   │    Auth, Routing)    │                                          │
│   └──────────┬───────────┘                                          │
│              │                                                       │
│   ┌──────────┴──────────────────────────────────────────────┐      │
│   │                                                          │      │
│   │                  AKS Cluster                             │      │
│   │                                                          │      │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐      │      │
│   │   │ Search  │ │ Booking │ │ Payment │ │Inventory│      │      │
│   │   │ Service │ │ Service │ │ Service │ │ Service │      │      │
│   │   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘      │      │
│   │        │           │           │           │            │      │
│   └────────┼───────────┼───────────┼───────────┼────────────┘      │
│            │           │           │           │                    │
│            ▼           ▼           ▼           ▼                    │
│   ┌─────────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────┐         │
│   │ Cosmos DB   │ │Azure SQL│ │ Stripe  │ │ Cosmos DB   │         │
│   │ (Search)    │ │(Booking)│ │(Payment)│ │ (Hotels)    │         │
│   └─────────────┘ └─────────┘ └─────────┘ └─────────────┘         │
│                                                                      │
│   The "Tiny Monolith" (what's left):                                │
│   ┌──────────────────────────────────────────────────────────────┐ │
│   │  Legacy code for edge cases we haven't migrated yet          │ │
│   │  ~50K lines (down from 1.2M)                                 │ │
│   │  Will decompose over next 6 months                           │ │
│   └──────────────────────────────────────────────────────────────┘ │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Results

| Metric | Monolith | Microservices | Change |
|--------|----------|---------------|--------|
| Deploy Frequency | 1/month | 20/day | 600x faster |
| Build Time | 8 hours | 5 minutes | 96x faster |
| Mean Time to Recovery | 4 hours | 15 minutes | 16x faster |
| Team Velocity | Declining | 40% increase | Teams unblocked |
| Code Ownership | "Shared" (nobody) | Clear ownership | Accountability |

---

## Case Study 2: Event-Driven Architecture for IoT

### The Setup

**Company:** SmartFarm Solutions
**Industry:** Agricultural IoT
**Challenge:** Process data from 50,000 sensors across 2,000 farms

### The Problem

```
INITIAL N-TIER DESIGN (THE MISTAKE):
────────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   50,000 Sensors ──► HTTP POST ──► API Servers ──► SQL Database     │
│                                                                      │
│   What could go wrong?                                              │
│                                                                      │
│   ├── 50K sensors × 1 reading/minute = 50K requests/minute          │
│   ├── Each reading: HTTP connection setup, auth, response wait      │
│   ├── SQL database: 50K inserts/minute = overwhelmed                │
│   ├── Peak harvest: 3x normal load = complete failure               │
│   │                                                                  │
│   └── Result: Lost sensor data, unhappy farmers, dead crops 🥀      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### The Event-Driven Solution

```
┌─────────────────────────────────────────────────────────────────────┐
│                EVENT-DRIVEN IoT ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   INGESTION TIER                                                     │
│   ──────────────                                                     │
│                                                                      │
│   ┌─────────────┐                                                   │
│   │   Sensors   │     Uses MQTT (persistent connection)             │
│   │   (50,000)  │     Low bandwidth, battery efficient              │
│   └──────┬──────┘                                                   │
│          │                                                           │
│          ▼                                                           │
│   ┌─────────────────────────────────────────────────────────┐      │
│   │                     IoT Hub                              │      │
│   │                                                          │      │
│   │   • Device authentication (X.509 per device)            │      │
│   │   • Protocol translation (MQTT → AMQP)                  │      │
│   │   • Device twin for configuration                        │      │
│   │   • Built-in routing to multiple endpoints              │      │
│   │                                                          │      │
│   │   Capacity: 6M messages/minute (we use 3M at peak)      │      │
│   └─────────────────────────┬───────────────────────────────┘      │
│                             │                                        │
│   ┌─────────────────────────┴───────────────────────────────┐      │
│   │                                                          │      │
│   │   PROCESSING TIER (Event-Driven)                        │      │
│   │   ──────────────────────────────                        │      │
│   │                                                          │      │
│   │   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │      │
│   │   │   Route 1   │  │   Route 2   │  │   Route 3   │    │      │
│   │   │             │  │             │  │             │    │      │
│   │   │ All Data    │  │ Alerts Only │  │ Maintenance │    │      │
│   │   │     ↓       │  │     ↓       │  │     ↓       │    │      │
│   │   │ Event Hubs  │  │ Service Bus │  │ Storage Blob│    │      │
│   │   │     ↓       │  │     ↓       │  │     ↓       │    │      │
│   │   │ Stream      │  │ Functions   │  │ Batch       │    │      │
│   │   │ Analytics   │  │ (Notify)    │  │ Processing  │    │      │
│   │   └─────────────┘  └─────────────┘  └─────────────┘    │      │
│   │                                                          │      │
│   └──────────────────────────────────────────────────────────┘      │
│                                                                      │
│   STORAGE TIER (Hot/Warm/Cold)                                      │
│   ─────────────────────────────                                     │
│                                                                      │
│   ┌─────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
│   │ Cosmos DB   │  │ Azure SQL       │  │ ADLS Gen2       │        │
│   │ (Real-time) │  │ (Aggregations)  │  │ (Historical)    │        │
│   │             │  │                 │  │                 │        │
│   │ Last 24 hrs │  │ Daily summaries │  │ Years of data   │        │
│   │ Hot queries │  │ Reports         │  │ ML training     │        │
│   └─────────────┘  └─────────────────┘  └─────────────────┘        │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Stream Analytics for Real-Time Alerts

```sql
-- Soil moisture alert (prevents crop damage)
SELECT
    deviceId,
    farm.name as farmName,
    AVG(moisture) as avgMoisture,
    MIN(moisture) as minMoisture,
    'LOW_MOISTURE_ALERT' as alertType,
    System.Timestamp() as alertTime
INTO [alerts-queue]
FROM [iot-hub-input] TIMESTAMP BY eventTime
JOIN [farm-reference] farm ON deviceId = farm.sensorId
WHERE sensorType = 'soil-moisture'
GROUP BY deviceId, farm.name, TumblingWindow(minute, 5)
HAVING AVG(moisture) < farm.minMoistureThreshold

-- Temperature anomaly detection (pest/disease early warning)
SELECT
    deviceId,
    temperature,
    AnomalyDetection_SpikeAndDip(temperature, 80, 120, 'spikesanddips')
        OVER(PARTITION BY deviceId LIMIT DURATION(hour, 1)) as anomaly
INTO [anomaly-alerts]
FROM [iot-hub-input]
WHERE anomaly.IsAnomaly = 1
```

### Azure Functions for Alert Processing

```csharp
[FunctionName("ProcessMoistureAlert")]
public static async Task ProcessAlert(
    [ServiceBusTrigger("alerts-queue")] SoilMoistureAlert alert,
    [CosmosDB(
        databaseName: "smartfarm",
        containerName: "alerts",
        Connection = "CosmosConnection")] IAsyncCollector<AlertDocument> alertStore,
    ILogger log)
{
    // Store alert
    await alertStore.AddAsync(new AlertDocument
    {
        id = Guid.NewGuid().ToString(),
        deviceId = alert.DeviceId,
        farmName = alert.FarmName,
        alertType = alert.AlertType,
        timestamp = alert.AlertTime,
        resolved = false
    });

    // Send notification via preferred channel
    var farmer = await GetFarmerPreferences(alert.FarmName);

    switch (farmer.PreferredNotification)
    {
        case "sms":
            await _twilioService.SendSms(farmer.Phone,
                $"⚠️ Low soil moisture detected at {alert.FarmName}. " +
                $"Current: {alert.AvgMoisture}%. Check irrigation system.");
            break;
        case "push":
            await _pushService.Send(farmer.DeviceToken, alert);
            break;
        case "email":
            await _emailService.SendAlert(farmer.Email, alert);
            break;
    }

    log.LogInformation("Alert processed: {AlertType} for {FarmName}",
        alert.AlertType, alert.FarmName);
}
```

### Results

| Metric | N-Tier | Event-Driven | Improvement |
|--------|--------|--------------|-------------|
| Max Throughput | 50K msg/min | 6M msg/min | 120x |
| Alert Latency | 5-15 minutes | <30 seconds | 95% faster |
| Data Loss at Peak | 12% | 0% | Critical for farming |
| Monthly Cost | $8,500 | $3,200 | 62% savings |
| Crop Loss (irrigation alerts) | $120K/year | $15K/year | $105K saved |

---

## Case Study 3: The Big Data Architecture Decision

### The Setup

**Company:** RetailInsight Analytics
**Industry:** Retail Analytics
**Challenge:** Analyze 2TB of daily transaction data for 50 retail clients

### The Architecture Options Debate

```
THE MEETING THAT SHAPED EVERYTHING:
───────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Data Engineer: "We should use Spark on Databricks!"               │
│   DBA: "No! Azure Synapse can handle this!"                         │
│   Architect: "What about a Lambda architecture?"                     │
│   Junior Dev: "I heard Kafka is web scale..."                       │
│   CTO: "Just make it work. We have 3 months."                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘

OPTION ANALYSIS:
────────────────

Option A: Synapse SQL Pool (MPP Database)
├── Pros: Familiar SQL, integrated analytics
├── Cons: Expensive at scale, less flexible for ML
└── Best for: BI-heavy, SQL-centric teams

Option B: Databricks + Delta Lake
├── Pros: Flexible, great for ML, unified batch/stream
├── Cons: Learning curve, requires Spark skills
└── Best for: ML workloads, data engineering focus

Option C: Lambda Architecture (Batch + Speed layers)
├── Pros: Real-time + historical queries
├── Cons: Two codebases, complex operations
└── Best for: When you truly need both real-time AND historical

DECISION: Option B with Synapse for serving layer
Reasoning: ML is strategic, Databricks handles both batch and stream
```

### The Medallion Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                 MEDALLION ARCHITECTURE (Bronze/Silver/Gold)          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   DATA SOURCES                                                       │
│   ────────────                                                       │
│   ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐                      │
│   │ POS    │ │ Web    │ │ Mobile │ │ ERP    │                      │
│   │ Systems│ │ Events │ │  App   │ │ Data   │                      │
│   └───┬────┘ └───┬────┘ └───┬────┘ └───┬────┘                      │
│       └──────────┴──────────┴──────────┘                            │
│                       │                                              │
│                       ▼                                              │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                     BRONZE LAYER                             │  │
│   │                   (Raw Data Lake)                            │  │
│   │                                                              │  │
│   │   ADLS Gen2 + Delta Lake Format                             │  │
│   │                                                              │  │
│   │   • Raw data exactly as received                            │  │
│   │   • Schema-on-read                                          │  │
│   │   • Full history (GDPR: 7 years)                           │  │
│   │   • Partitioned by date and source                         │  │
│   │                                                              │  │
│   │   Example: /bronze/pos/2024/01/15/store_001/*.parquet      │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                       │                                              │
│                       ▼  Databricks notebooks (scheduled)            │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                     SILVER LAYER                             │  │
│   │                (Cleansed & Enriched)                        │  │
│   │                                                              │  │
│   │   • Deduplicated records                                    │  │
│   │   • Data quality validated                                  │  │
│   │   • Standardized schemas                                    │  │
│   │   • Enriched with reference data                           │  │
│   │                                                              │  │
│   │   Tables: transactions_cleaned, products_master,            │  │
│   │           customers_unified, stores_dimension               │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                       │                                              │
│                       ▼  Aggregation jobs                            │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                      GOLD LAYER                              │  │
│   │              (Business-Ready Datasets)                       │  │
│   │                                                              │  │
│   │   Aggregated for specific use cases:                        │  │
│   │                                                              │  │
│   │   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐    │  │
│   │   │ Sales         │ │ Customer 360  │ │ Inventory     │    │  │
│   │   │ Analytics     │ │ View          │ │ Optimization  │    │  │
│   │   │               │ │               │ │               │    │  │
│   │   │ - Daily sales │ │ - RFM scores  │ │ - Stock levels│    │  │
│   │   │ - Trends      │ │ - Segments    │ │ - Reorder pts │    │  │
│   │   │ - Forecasts   │ │ - LTV         │ │ - Predictions │    │  │
│   │   └───────────────┘ └───────────────┘ └───────────────┘    │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                       │                                              │
│                       ▼                                              │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                   SERVING LAYER                              │  │
│   │                                                              │  │
│   │   ┌───────────────┐ ┌───────────────┐ ┌───────────────┐    │  │
│   │   │ Synapse SQL   │ │ Power BI      │ │ ML Models     │    │  │
│   │   │ (Ad-hoc)      │ │ (Dashboards)  │ │ (Predictions) │    │  │
│   │   └───────────────┘ └───────────────┘ └───────────────┘    │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Databricks Notebook: Bronze to Silver

```python
# Bronze to Silver transformation for POS data
from pyspark.sql import functions as F
from pyspark.sql.window import Window
from delta.tables import DeltaTable

# Read bronze data (raw POS transactions)
bronze_df = spark.read.format("delta").load("/bronze/pos/")

# Data quality checks
quality_checks = bronze_df.select(
    F.count("*").alias("total_records"),
    F.sum(F.when(F.col("transaction_id").isNull(), 1).otherwise(0)).alias("null_trans_id"),
    F.sum(F.when(F.col("total_amount") < 0, 1).otherwise(0)).alias("negative_amounts"),
    F.countDistinct("store_id").alias("unique_stores")
)
quality_checks.display()

# Cleansing transformations
silver_df = bronze_df.filter(
    (F.col("transaction_id").isNotNull()) &
    (F.col("total_amount") >= 0) &
    (F.col("transaction_date").between("2020-01-01", F.current_date()))
).withColumn(
    # Standardize timestamps
    "transaction_timestamp",
    F.to_timestamp(F.col("transaction_date"))
).withColumn(
    # Deduplicate within 1-hour window
    "row_num",
    F.row_number().over(
        Window.partitionBy("transaction_id")
        .orderBy(F.col("_metadata.file_modification_time").desc())
    )
).filter(F.col("row_num") == 1).drop("row_num")

# Enrich with store reference data
stores_df = spark.read.format("delta").load("/reference/stores/")
enriched_df = silver_df.join(
    stores_df,
    silver_df.store_id == stores_df.store_id,
    "left"
).select(
    silver_df["*"],
    stores_df.region,
    stores_df.store_type,
    stores_df.timezone
)

# Write to Silver layer with merge (upsert)
silver_table = DeltaTable.forPath(spark, "/silver/transactions/")
silver_table.alias("target").merge(
    enriched_df.alias("source"),
    "target.transaction_id = source.transaction_id"
).whenMatchedUpdateAll().whenNotMatchedInsertAll().execute()

print(f"Processed {enriched_df.count()} records to Silver layer")
```

### Results

| Metric | Before (Ad-hoc) | After (Medallion) | Improvement |
|--------|-----------------|-------------------|-------------|
| Data Processing Time | 18 hours | 2 hours | 89% faster |
| Query Performance | Minutes | Seconds | 95% faster |
| Data Quality Issues | Weekly incidents | Zero in 6 months | Reliable |
| New Client Onboarding | 2 weeks | 2 days | 85% faster |
| Monthly Analytics Cost | $45,000 | $18,000 | 60% savings |

---

## Summary: Architecture Selection Framework

```
CHOOSING YOUR ARCHITECTURE STYLE:
─────────────────────────────────

┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│   Start here: What's your PRIMARY challenge?                        │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                                                              │  │
│   │   "We're migrating from on-premises"                        │  │
│   │   └──► N-TIER (lift-and-shift, then modernize)              │  │
│   │                                                              │  │
│   │   "We need to decouple background work"                     │  │
│   │   └──► WEB-QUEUE-WORKER (simple, effective)                 │  │
│   │                                                              │  │
│   │   "Our monolith is too big, teams blocked"                  │  │
│   │   └──► MICROSERVICES (via Strangler Fig)                    │  │
│   │                                                              │  │
│   │   "We need real-time processing, IoT, streaming"            │  │
│   │   └──► EVENT-DRIVEN (push-based, reactive)                  │  │
│   │                                                              │  │
│   │   "We have massive data, need analytics & ML"               │  │
│   │   └──► BIG DATA (lakehouse, Medallion)                      │  │
│   │                                                              │  │
│   │   "We need HPC, simulations, rendering"                     │  │
│   │   └──► BIG COMPUTE (Azure Batch, Spot VMs)                  │  │
│   │                                                              │  │
│   └─────────────────────────────────────────────────────────────┘  │
│                                                                      │
│   REMEMBER:                                                         │
│   ├── Most systems use MULTIPLE styles                              │
│   ├── Start simple, add complexity only when needed                 │
│   ├── The "best" architecture is one your team can operate          │
│   └── Architecture is not a one-time decision                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

*Navigation: [Quick Reference](quick-reference.md) | [README](README.md) | [Main Guide](../README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
