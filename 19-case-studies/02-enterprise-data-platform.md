# Case Study 2: Enterprise Data Platform

## Executive Summary

**Company**: GlobalBank Financial Services
**Industry**: Banking & Financial Services
**Challenge**: Modernize legacy data warehouse and enable real-time analytics
**Outcome**: 80% faster reporting, unified data governance, $4.5M annual savings

### The Business Context

GlobalBank had siloed data systems across retail banking, wealth management, and credit card divisions. Each division maintained separate data warehouses, making enterprise-wide analytics impossible. The CEO mandated a unified data platform to:

1. Enable real-time fraud detection across all product lines
2. Provide 360-degree customer views for cross-selling
3. Meet regulatory requirements (Basel III, GDPR, SOX)
4. Reduce total cost of data infrastructure by 40%

## Architecture Overview

```
                                    ┌─────────────────────────────────────┐
                                    │         Data Sources Layer          │
                    ┌───────────────┼───────────────┬───────────────┐     │
                    │               │               │               │     │
              ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐   ┌─────▼─────┐
              │  Core     │   │  Trading  │   │  Mobile   │   │  External │
              │  Banking  │   │  Systems  │   │  Banking  │   │  Data     │
              │  (Oracle) │   │  (SQL)    │   │  (Events) │   │  (APIs)   │
              └─────┬─────┘   └─────┬─────┘   └─────┬─────┘   └─────┬─────┘
                    │               │               │               │
                    └───────────────┼───────────────┼───────────────┘
                                    │               │
        ┌───────────────────────────▼───────────────▼───────────────────────────┐
        │                        Ingestion Layer                                 │
        │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
        │  │  Data Factory   │  │   Event Hubs    │  │  Azure          │        │
        │  │  (Batch ETL)    │  │  (Streaming)    │  │  Functions      │        │
        │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘        │
        └───────────┼────────────────────┼────────────────────┼─────────────────┘
                    │                    │                    │
        ┌───────────▼────────────────────▼────────────────────▼─────────────────┐
        │                    Storage Layer (Data Lake)                          │
        │  ┌─────────────────────────────────────────────────────────────┐      │
        │  │              Azure Data Lake Storage Gen2                    │      │
        │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐    │      │
        │  │  │  Bronze  │  │  Silver  │  │  Gold    │  │ Platinum │    │      │
        │  │  │  (Raw)   │──▶│(Cleansed)│──▶│(Curated)│──▶│(Semantic)│    │      │
        │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘    │      │
        │  └─────────────────────────────────────────────────────────────┘      │
        └───────────────────────────────┬───────────────────────────────────────┘
                                        │
        ┌───────────────────────────────▼───────────────────────────────────────┐
        │                    Processing Layer                                    │
        │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
        │  │ Azure Synapse   │  │  Azure          │  │   Databricks    │        │
        │  │ (Analytics)     │  │  ML             │  │   (Data Eng)    │        │
        │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │
        └───────────────────────────────┬───────────────────────────────────────┘
                                        │
        ┌───────────────────────────────▼───────────────────────────────────────┐
        │                    Serving Layer                                       │
        │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
        │  │  Synapse        │  │  Analysis       │  │  Power BI       │        │
        │  │  Serverless     │  │  Services       │  │  (Semantic)     │        │
        │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │
        └───────────────────────────────┬───────────────────────────────────────┘
                                        │
        ┌───────────────────────────────▼───────────────────────────────────────┐
        │                    Governance Layer                                    │
        │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐        │
        │  │  Microsoft      │  │  Azure          │  │  Azure          │        │
        │  │  Purview        │  │  Key Vault      │  │  Monitor        │        │
        │  └─────────────────┘  └─────────────────┘  └─────────────────┘        │
        └───────────────────────────────────────────────────────────────────────┘
```

## AWS to Azure Service Mapping

| Component | AWS (Legacy Reference) | Azure (Implementation) | Key Difference |
|-----------|----------------------|------------------------|----------------|
| Data Lake | S3 + Lake Formation | ADLS Gen2 + Purview | Native hierarchical namespace, Unity Catalog integration |
| ETL | Glue + Step Functions | Data Factory + Synapse Pipelines | Visual designer, 90+ connectors |
| Streaming | Kinesis | Event Hubs + Stream Analytics | Native Kafka compatibility |
| Data Warehouse | Redshift | Synapse Dedicated Pools | Serverless query option |
| ML Platform | SageMaker | Azure ML + Databricks | MLflow integration, responsible AI |
| Data Catalog | Glue Data Catalog | Microsoft Purview | Cross-cloud scanning, lineage |
| BI | QuickSight | Power BI | Better enterprise integration |

## Key Architectural Decisions

### Decision 1: Medallion Architecture Implementation

**The Debate:**

| Approach | Pros | Cons |
|----------|------|------|
| **Medallion (Bronze/Silver/Gold)** | Clear data quality layers, reprocessing flexibility | Storage overhead, complexity |
| **Single Zone** | Simple, lower storage cost | No separation of concerns, reprocessing is painful |
| **Lambda Architecture** | Real-time + batch | Complexity, code duplication |
| **Kappa Architecture** | Single processing path | Not suitable for heavy batch workloads |

**Decision: Medallion Architecture with Platinum layer for semantic models**

**Rationale:**
1. Banking regulations require full data lineage (Bronze preserves raw data)
2. Different teams consume data at different quality levels
3. Reprocessing specific layers without affecting downstream is critical
4. Added Platinum layer for business-ready semantic models

**Folder Structure:**
```
datalake/
├── bronze/
│   ├── core_banking/
│   │   ├── transactions/
│   │   │   ├── year=2024/month=01/day=15/
│   │   │   └── _metadata/
│   │   └── accounts/
│   ├── trading/
│   └── mobile/
├── silver/
│   ├── cleansed/
│   │   ├── transactions_cleansed/
│   │   └── accounts_cleansed/
│   └── conformed/
│       └── customer_360/
├── gold/
│   ├── aggregated/
│   │   ├── daily_transaction_summary/
│   │   └── customer_metrics/
│   └── features/
│       └── fraud_features/
└── platinum/
    ├── semantic_models/
    │   ├── finance_model/
    │   └── risk_model/
    └── ml_datasets/
```

### Decision 2: Synapse vs Databricks for Data Processing

**The Debate:**

| Criteria | Synapse | Databricks |
|----------|---------|------------|
| SQL Analytics | Native, excellent | Good with Spark SQL |
| Data Engineering | Good | Excellent |
| ML Integration | Azure ML | MLflow native |
| Cost | Pay-per-query option | Always-on clusters |
| Skill Availability | SQL abundant | Spark skills needed |
| Unity Catalog | No | Yes (governance) |

**Decision: Hybrid - Synapse for SQL workloads, Databricks for Data Engineering & ML**

**Rationale:**
1. Existing SQL expertise leveraged for BI workloads
2. Databricks Unity Catalog provides superior data governance
3. Databricks notebooks better for data science collaboration
4. Synapse Serverless perfect for ad-hoc exploration
5. Cost optimization: SQL workloads on serverless, complex transforms on Databricks

**Workload Distribution:**
```
┌─────────────────────────────────────────────────────────────────┐
│                     Workload Distribution                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Databricks                          Synapse                     │
│  ───────────                         ───────                     │
│  • Bronze → Silver ETL               • Serverless SQL queries    │
│  • Silver → Gold transforms          • Dedicated pool for BI     │
│  • ML feature engineering            • Power BI DirectQuery      │
│  • Real-time streaming (Spark)       • Ad-hoc exploration        │
│  • Data quality frameworks           • SQL-based reporting       │
│                                                                  │
│  Cost: ~60% of processing            Cost: ~40% of processing    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Decision 3: Real-time Fraud Detection Architecture

**The Debate:**

| Option | Latency | Throughput | Cost | Complexity |
|--------|---------|------------|------|------------|
| Databricks Structured Streaming | Seconds | Very High | $$$ | High |
| Stream Analytics | Sub-second | High | $$ | Medium |
| Azure Functions + Event Hubs | Milliseconds | Medium | $ | Low |
| Synapse Real-time Analytics | Seconds | High | $$ | Medium |

**Decision: Event Hubs + Stream Analytics + Azure ML for real-time scoring**

**Rationale:**
1. Sub-second latency required for transaction approval
2. Stream Analytics handles 99% of rules-based fraud checks
3. Azure ML endpoint for complex ML model scoring
4. Event Hubs Capture for audit trail

**Implementation:**
```sql
-- Stream Analytics Query for Fraud Detection
WITH TransactionFeatures AS (
    SELECT
        transactionId,
        customerId,
        amount,
        merchantCategory,
        location,
        timestamp,
        -- Calculate velocity features
        COUNT(*) OVER (
            PARTITION BY customerId
            LIMIT DURATION(minute, 5)
        ) as txn_count_5min,
        SUM(amount) OVER (
            PARTITION BY customerId
            LIMIT DURATION(hour, 1)
        ) as total_amount_1hr,
        -- Distance from last transaction
        LAG(location) OVER (
            PARTITION BY customerId
            LIMIT DURATION(minute, 30)
        ) as last_location
    FROM transactions TIMESTAMP BY timestamp
),
RuleBasedFraud AS (
    SELECT
        *,
        CASE
            WHEN txn_count_5min > 10 THEN 'HIGH_VELOCITY'
            WHEN total_amount_1hr > 10000 THEN 'HIGH_AMOUNT'
            WHEN UDF.CalculateDistance(location, last_location) > 500
                 AND DATEDIFF(minute, last_timestamp, timestamp) < 30
                 THEN 'IMPOSSIBLE_TRAVEL'
            ELSE 'NORMAL'
        END as rule_flag
    FROM TransactionFeatures
)

-- Output suspicious transactions to ML scoring
SELECT * INTO mlScoringOutput
FROM RuleBasedFraud
WHERE rule_flag != 'NORMAL'

-- Output normal transactions directly to approval
SELECT * INTO approvedOutput
FROM RuleBasedFraud
WHERE rule_flag = 'NORMAL'
```

## Technical Deep Dive

### Data Factory Pipeline for Core Banking

```json
{
  "name": "CoreBanking_Bronze_Pipeline",
  "properties": {
    "activities": [
      {
        "name": "GetWatermark",
        "type": "Lookup",
        "linkedServiceName": {
          "referenceName": "WatermarkStore",
          "type": "LinkedServiceReference"
        },
        "typeProperties": {
          "source": {
            "type": "AzureSqlSource",
            "sqlReaderQuery": "SELECT MAX(LastModified) FROM watermarks WHERE TableName = 'transactions'"
          }
        }
      },
      {
        "name": "CopyTransactions",
        "type": "Copy",
        "dependsOn": ["GetWatermark"],
        "typeProperties": {
          "source": {
            "type": "OracleSource",
            "oracleReaderQuery": {
              "value": "SELECT * FROM transactions WHERE modified_date > '@{activity('GetWatermark').output.firstRow.Watermark}'",
              "type": "Expression"
            }
          },
          "sink": {
            "type": "ParquetSink",
            "storeSettings": {
              "type": "AzureBlobFSWriteSettings"
            },
            "formatSettings": {
              "type": "ParquetWriteSettings"
            }
          }
        },
        "inputs": [{
          "referenceName": "OracleTransactions",
          "type": "DatasetReference"
        }],
        "outputs": [{
          "referenceName": "BronzeTransactions",
          "type": "DatasetReference",
          "parameters": {
            "folderPath": "@concat('bronze/core_banking/transactions/year=', formatDateTime(utcnow(),'yyyy'), '/month=', formatDateTime(utcnow(),'MM'), '/day=', formatDateTime(utcnow(),'dd'))"
          }
        }]
      },
      {
        "name": "UpdateWatermark",
        "type": "SqlServerStoredProcedure",
        "dependsOn": ["CopyTransactions"],
        "typeProperties": {
          "storedProcedureName": "usp_UpdateWatermark",
          "storedProcedureParameters": {
            "TableName": { "value": "transactions" },
            "Watermark": { "value": "@utcnow()" }
          }
        }
      },
      {
        "name": "TriggerDatabricksNotebook",
        "type": "DatabricksNotebook",
        "dependsOn": ["UpdateWatermark"],
        "typeProperties": {
          "notebookPath": "/Shared/ETL/bronze_to_silver_transactions",
          "baseParameters": {
            "processing_date": "@formatDateTime(utcnow(),'yyyy-MM-dd')"
          }
        }
      }
    ]
  }
}
```

### Databricks Bronze to Silver Transformation

```python
# bronze_to_silver_transactions.py
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
from delta import DeltaTable
import great_expectations as gx

# Initialize
spark = SparkSession.builder.getOrCreate()
processing_date = dbutils.widgets.get("processing_date")

# Read Bronze Data
bronze_df = spark.read.parquet(
    f"abfss://datalake@storageaccount.dfs.core.windows.net/bronze/core_banking/transactions/year={processing_date[:4]}/month={processing_date[5:7]}/day={processing_date[8:10]}"
)

# Data Quality Checks
def validate_data(df):
    """Apply data quality rules"""
    expectations = {
        "transaction_id_not_null": df.filter(col("transaction_id").isNull()).count() == 0,
        "amount_positive": df.filter(col("amount") <= 0).count() == 0,
        "valid_currency": df.filter(~col("currency").isin(["USD", "EUR", "GBP"])).count() < df.count() * 0.01,
        "customer_id_format": df.filter(~col("customer_id").rlike("^[A-Z]{2}[0-9]{8}$")).count() == 0
    }

    failed_checks = [k for k, v in expectations.items() if not v]
    if failed_checks:
        # Log to monitoring but don't fail - quarantine bad records
        print(f"Data quality issues: {failed_checks}")
        return False
    return True

# Cleanse and Transform
silver_df = bronze_df \
    .withColumn("amount", col("amount").cast(DecimalType(18, 2))) \
    .withColumn("transaction_date", to_date(col("timestamp"))) \
    .withColumn("transaction_hour", hour(col("timestamp"))) \
    .withColumn("is_weekend", dayofweek(col("transaction_date")).isin([1, 7])) \
    .withColumn("amount_bucket",
        when(col("amount") < 100, "small")
        .when(col("amount") < 1000, "medium")
        .when(col("amount") < 10000, "large")
        .otherwise("very_large")
    ) \
    .withColumn("_ingestion_timestamp", current_timestamp()) \
    .withColumn("_source_system", lit("core_banking")) \
    .dropDuplicates(["transaction_id"])

# Validate
if not validate_data(silver_df):
    # Quarantine bad records
    bad_records = silver_df.filter(
        col("transaction_id").isNull() |
        (col("amount") <= 0)
    )
    bad_records.write.mode("append").parquet(
        f"abfss://datalake@storageaccount.dfs.core.windows.net/quarantine/transactions/{processing_date}"
    )

    # Continue with good records
    silver_df = silver_df.filter(
        col("transaction_id").isNotNull() &
        (col("amount") > 0)
    )

# Write to Silver (Delta Lake)
silver_df.write \
    .format("delta") \
    .mode("merge") \
    .option("mergeSchema", "true") \
    .partitionBy("transaction_date") \
    .save("abfss://datalake@storageaccount.dfs.core.windows.net/silver/cleansed/transactions")

# Update Delta table statistics
spark.sql("OPTIMIZE delta.`abfss://datalake@storageaccount.dfs.core.windows.net/silver/cleansed/transactions` ZORDER BY (customer_id)")
```

### Microsoft Purview Data Governance

```python
# Register data assets and lineage in Purview
from azure.purview.catalog import PurviewCatalogClient
from azure.identity import DefaultAzureCredential
from pyapacheatlas.core import AtlasProcess, AtlasEntity

credential = DefaultAzureCredential()
client = PurviewCatalogClient(
    endpoint="https://globalbank-purview.purview.azure.com",
    credential=credential
)

# Define data lineage
bronze_entity = AtlasEntity(
    name="bronze_transactions",
    typeName="azure_datalake_gen2_path",
    qualified_name="https://storageaccount.dfs.core.windows.net/bronze/core_banking/transactions",
    attributes={
        "description": "Raw transaction data from core banking Oracle DB",
        "owner": "data-engineering-team",
        "classification": "PII-Financial"
    }
)

silver_entity = AtlasEntity(
    name="silver_transactions",
    typeName="azure_datalake_gen2_path",
    qualified_name="https://storageaccount.dfs.core.windows.net/silver/cleansed/transactions",
    attributes={
        "description": "Cleansed and standardized transaction data",
        "owner": "data-engineering-team"
    }
)

# Create lineage relationship
etl_process = AtlasProcess(
    name="bronze_to_silver_transactions",
    typeName="databricks_notebook",
    qualified_name="databricks://workspace/Shared/ETL/bronze_to_silver_transactions",
    inputs=[bronze_entity],
    outputs=[silver_entity],
    attributes={
        "description": "Daily ETL job that cleanses transaction data",
        "schedule": "Daily 2:00 AM UTC"
    }
)
```

## Cost Analysis

### Monthly Cost Breakdown

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| ADLS Gen2 | 100TB hot, 500TB cool | $15,000 |
| Databricks | 50 DBU standard, 200 DBU jobs | $35,000 |
| Synapse Dedicated Pool | DW1000c | $12,000 |
| Synapse Serverless | 50TB scanned/month | $2,500 |
| Data Factory | 1M activities, 500 IR hours | $3,500 |
| Event Hubs | Dedicated, 8 TU | $6,500 |
| Stream Analytics | 36 SU | $4,000 |
| Azure ML | Compute clusters | $8,000 |
| Microsoft Purview | Standard | $1,500 |
| Power BI Premium | P1 capacity | $5,000 |
| **Total** | | **$93,000** |

### Legacy vs Modern Platform Cost

| Category | Legacy (3 Data Warehouses) | Modern Platform | Savings |
|----------|---------------------------|-----------------|---------|
| Infrastructure | $180,000 | $93,000 | 48% |
| Licensing | $85,000 | $25,000 | 71% |
| Operations (FTEs) | $120,000 | $60,000 | 50% |
| **Total** | **$385,000** | **$178,000** | **54%** |

## Lessons Learned

### What Worked Well

1. **Medallion Architecture**: Clear separation enabled different teams to work independently
2. **Purview Integration**: Automated lineage tracking satisfied audit requirements
3. **Hybrid Synapse + Databricks**: Leveraged strengths of each platform

### Challenges Encountered

1. **Change Data Capture Complexity**: Oracle CDC required significant configuration
2. **Data Quality at Scale**: Needed Great Expectations framework for automated validation
3. **Cost Management**: Databricks cluster auto-termination critical for cost control

### Recommendations

1. **Start with governance**: Set up Purview classifications before loading data
2. **Implement data contracts**: Define schemas between Bronze/Silver/Gold upfront
3. **Use managed tables**: Delta Lake managed tables simplify governance

## Discussion Questions

1. **Why was the Medallion architecture chosen over Lambda architecture?**

2. **How would you handle schema evolution in the Silver layer?**

3. **What's the trade-off between Synapse Serverless and Dedicated Pools?**

4. **How do you ensure GDPR compliance with this architecture?**

5. **What would change if real-time requirements increased from seconds to milliseconds?**

---

*Continue to [Case Study 3: Financial Services Migration](03-financial-services-migration.md)*

*Back to [Chapter 08: Case Studies](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
