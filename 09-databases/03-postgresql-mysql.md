# Azure Database for PostgreSQL and MySQL

## Overview

Azure Database for PostgreSQL and MySQL are fully managed database services that provide enterprise-grade security, high availability, and intelligent performance optimization. The Flexible Server deployment option offers more control and cost optimization compared to the legacy Single Server option.

## AWS Comparison

| Feature | AWS RDS PostgreSQL/MySQL | Azure Flexible Server |
|---------|--------------------------|----------------------|
| Deployment model | Multi-AZ, Single-AZ | Zone-redundant HA, Same-zone HA |
| Maintenance window | Required, configurable | Configurable, can defer |
| Stop/start | Yes (7 days max) | Yes (30 days max) |
| Burstable tiers | Yes (T3) | Yes (B-series) |
| Reserved capacity | Reserved Instances | Reserved Capacity |
| Read replicas | Up to 5 | Up to 10 |
| Automated backups | 35 days max | 35 days (zone-redundant) |
| Major version upgrade | In-place | In-place |
| Private connectivity | VPC peering, PrivateLink | VNet integration (native) |
| Parameter groups | Parameter groups | Server parameters |

---

## Flexible Server Architecture

### Deployment Options

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Flexible Server Architecture                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  NON-HA DEPLOYMENT (Development/Test)                                       │
│  ─────────────────────────────────────                                       │
│                                                                              │
│       Availability Zone 1                                                    │
│      ┌─────────────────────────────────────────┐                            │
│      │                                         │                            │
│      │   ┌─────────────────┐                   │                            │
│      │   │  PostgreSQL/    │                   │                            │
│      │   │  MySQL Server   │                   │                            │
│      │   │  (Primary)      │                   │                            │
│      │   └────────┬────────┘                   │                            │
│      │            │                            │                            │
│      │   ┌────────▼────────┐                   │                            │
│      │   │  Locally        │                   │                            │
│      │   │  Redundant      │                   │                            │
│      │   │  Storage        │                   │                            │
│      │   └─────────────────┘                   │                            │
│      │                                         │                            │
│      └─────────────────────────────────────────┘                            │
│                                                                              │
│  SLA: 99.9% (single zone)                                                   │
│                                                                              │
│  SAME-ZONE HA                                                               │
│  ────────────                                                               │
│                                                                              │
│       Availability Zone 1                                                    │
│      ┌─────────────────────────────────────────┐                            │
│      │                                         │                            │
│      │   ┌─────────────────┐                   │                            │
│      │   │     Primary     │                   │                            │
│      │   │     Server      │                   │                            │
│      │   └────────┬────────┘                   │                            │
│      │            │ Sync Replication           │                            │
│      │   ┌────────▼────────┐                   │                            │
│      │   │    Standby      │                   │                            │
│      │   │    Server       │                   │                            │
│      │   └─────────────────┘                   │                            │
│      │                                         │                            │
│      └─────────────────────────────────────────┘                            │
│                                                                              │
│  SLA: 99.95%                                                                │
│                                                                              │
│  ZONE-REDUNDANT HA (Production)                                             │
│  ──────────────────────────────                                             │
│                                                                              │
│       Zone 1                        Zone 2                                  │
│      ┌──────────────────────┐      ┌──────────────────────┐                │
│      │                      │      │                      │                │
│      │   ┌──────────────┐   │      │   ┌──────────────┐   │                │
│      │   │   Primary    │   │ Sync │   │   Standby    │   │                │
│      │   │   Server     │───┼──────┼──▶│   Server     │   │                │
│      │   └──────┬───────┘   │      │   └──────┬───────┘   │                │
│      │          │           │      │          │           │                │
│      │   ┌──────▼───────┐   │      │   ┌──────▼───────┐   │                │
│      │   │  ZRS Storage │◀──┼──────┼──▶│  ZRS Storage │   │                │
│      │   └──────────────┘   │      │   └──────────────┘   │                │
│      │                      │      │                      │                │
│      └──────────────────────┘      └──────────────────────┘                │
│                                                                              │
│  SLA: 99.99%                                                                │
│  Automatic failover: 60-120 seconds                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Compute Tiers

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Compute Tiers                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  BURSTABLE (B-series) - Development, low-traffic                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • 1-20 vCores                                                       │   │
│  │ • CPU credits for bursting                                          │   │
│  │ • Most cost-effective                                               │   │
│  │ • Not suitable for sustained high CPU                               │   │
│  │                                                                      │   │
│  │ SKUs: Standard_B1s, Standard_B1ms, Standard_B2s, Standard_B2ms...  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  GENERAL PURPOSE (D-series) - Production workloads                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • 2-96 vCores                                                       │   │
│  │ • Balanced compute/memory ratio                                     │   │
│  │ • Predictable performance                                           │   │
│  │ • Most common choice                                                │   │
│  │                                                                      │   │
│  │ SKUs: Standard_D2ds_v4, Standard_D4ds_v4, Standard_D8ds_v4...      │   │
│  │ vCore:Memory = 1:4 (e.g., 4 vCores = 16 GB RAM)                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  MEMORY OPTIMIZED (E-series) - Heavy analytics, large working sets        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • 2-96 vCores                                                       │   │
│  │ • Higher memory per vCore                                           │   │
│  │ • Large datasets, complex queries                                   │   │
│  │ • In-memory processing                                              │   │
│  │                                                                      │   │
│  │ SKUs: Standard_E2ds_v4, Standard_E4ds_v4, Standard_E8ds_v4...      │   │
│  │ vCore:Memory = 1:8 (e.g., 4 vCores = 32 GB RAM)                    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## High Availability Options

### Failover Behavior

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    HA Failover Process                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Automatic Failover Triggers:                                               │
│  • Primary server unresponsive                                              │
│  • Zone failure                                                             │
│  • Compute host failure                                                     │
│                                                                              │
│  Failover Timeline:                                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  0s         30s        60s        90s        120s                   │   │
│  │  │          │          │          │          │                      │   │
│  │  ▼          ▼          ▼          ▼          ▼                      │   │
│  │  ┌──────────┬──────────┬──────────┬──────────┐                     │   │
│  │  │ Detect   │ Promote  │ Recovery │ Ready    │                     │   │
│  │  │ Failure  │ Standby  │ (if WAL) │          │                     │   │
│  │  └──────────┴──────────┴──────────┴──────────┘                     │   │
│  │                                                                      │   │
│  │  Detection: 30-40 seconds                                          │   │
│  │  Failover:  30-60 seconds                                          │   │
│  │  Total:     60-120 seconds typical                                 │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  DNS Propagation:                                                           │
│  • Server FQDN remains the same                                            │
│  • TTL: 30 seconds (automatic DNS update)                                  │
│  • Applications reconnect to same endpoint                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Read Replicas

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Read Replica Architecture                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                          Primary Server                                      │
│                         ┌─────────────┐                                     │
│                         │  Read/Write │                                     │
│                         │   Server    │                                     │
│                         └──────┬──────┘                                     │
│                                │                                             │
│           ┌────────────────────┼────────────────────┐                       │
│           │                    │                    │                       │
│           ▼                    ▼                    ▼                       │
│   ┌───────────────┐    ┌───────────────┐    ┌───────────────┐             │
│   │ Read Replica  │    │ Read Replica  │    │ Read Replica  │             │
│   │  (Same Zone)  │    │ (Diff Zone)   │    │ (Diff Region) │             │
│   │               │    │               │    │               │             │
│   │   Async       │    │   Async       │    │   Async       │             │
│   │   Replication │    │   Replication │    │   Replication │             │
│   └───────────────┘    └───────────────┘    └───────────────┘             │
│                                                                              │
│   Configuration:                                                            │
│   • Up to 10 read replicas per server                                      │
│   • Same or different region                                               │
│   • Asynchronous replication (eventual consistency)                        │
│   • Can be promoted to standalone server                                   │
│   • Independent compute tier (can be smaller/larger)                       │
│                                                                              │
│   Use Cases:                                                                │
│   • Read scale-out for reporting                                           │
│   • Geographic read distribution                                           │
│   • Disaster recovery (cross-region)                                       │
│   • Analytics workloads                                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bicep: High Availability Configuration

```bicep
@description('Location for resources')
param location string = resourceGroup().location

@description('Administrator username')
param adminUsername string

@secure()
@description('Administrator password')
param adminPassword string

// PostgreSQL Flexible Server with Zone-Redundant HA
resource postgresServer 'Microsoft.DBforPostgreSQL/flexibleServers@2023-06-01-preview' = {
  name: 'postgres-${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Standard_D4ds_v4'
    tier: 'GeneralPurpose'
  }
  properties: {
    version: '15'
    administratorLogin: adminUsername
    administratorLoginPassword: adminPassword
    storage: {
      storageSizeGB: 128
      autoGrow: 'Enabled'
    }
    backup: {
      backupRetentionDays: 35
      geoRedundantBackup: 'Enabled'
    }
    highAvailability: {
      mode: 'ZoneRedundant'
      standbyAvailabilityZone: '2'
    }
    availabilityZone: '1'
    network: {
      delegatedSubnetResourceId: postgresSubnet.id
      privateDnsZoneArmResourceId: privateDnsZone.id
    }
  }
}

// MySQL Flexible Server with Same-Zone HA
resource mysqlServer 'Microsoft.DBforMySQL/flexibleServers@2023-06-30' = {
  name: 'mysql-${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Standard_D4ds_v4'
    tier: 'GeneralPurpose'
  }
  properties: {
    version: '8.0.21'
    administratorLogin: adminUsername
    administratorLoginPassword: adminPassword
    storage: {
      storageSizeGB: 128
      autoGrow: 'Enabled'
      autoIoScaling: 'Enabled'  // MySQL-specific
    }
    backup: {
      backupRetentionDays: 35
      geoRedundantBackup: 'Enabled'
    }
    highAvailability: {
      mode: 'SameZone'
    }
    network: {
      delegatedSubnetResourceId: mysqlSubnet.id
      privateDnsZoneArmResourceId: privateDnsZoneMySQL.id
    }
  }
}

// Read Replica (PostgreSQL)
resource readReplica 'Microsoft.DBforPostgreSQL/flexibleServers@2023-06-01-preview' = {
  name: 'postgres-replica-${uniqueString(resourceGroup().id)}'
  location: 'westus'  // Different region
  sku: {
    name: 'Standard_D2ds_v4'  // Can be different size
    tier: 'GeneralPurpose'
  }
  properties: {
    createMode: 'Replica'
    sourceServerResourceId: postgresServer.id
  }
}
```

---

## Migration Strategies from AWS RDS

### Migration Options

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Migration from AWS RDS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Option 1: AZURE DATABASE MIGRATION SERVICE (Online)                        │
│  ───────────────────────────────────────────────────                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │   AWS RDS                  DMS                      Azure Flexible  │   │
│  │  ┌─────────┐           ┌─────────┐               ┌─────────┐       │   │
│  │  │PostgreSQL│──────────▶│  Azure  │──────────────▶│PostgreSQL│       │   │
│  │  │ /MySQL  │  Ongoing  │   DMS   │  Continuous   │ /MySQL  │       │   │
│  │  └─────────┘  Sync     └─────────┘  Sync         └─────────┘       │   │
│  │                                                                      │   │
│  │  • Minimal downtime (minutes)                                       │   │
│  │  • Continuous replication until cutover                            │   │
│  │  • Requires network connectivity (VPN/ExpressRoute)                │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Option 2: NATIVE LOGICAL REPLICATION (PostgreSQL)                          │
│  ────────────────────────────────────────────────────                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │   AWS RDS                                       Azure Flexible       │   │
│  │  ┌─────────┐    Logical Replication            ┌─────────┐          │   │
│  │  │PostgreSQL│─────────────────────────────────▶│PostgreSQL│          │   │
│  │  │(Publisher)│                                  │(Subscriber)│         │   │
│  │  └─────────┘                                   └─────────┘          │   │
│  │                                                                      │   │
│  │  • PostgreSQL 10+ required                                          │   │
│  │  • Fine-grained table selection                                     │   │
│  │  • Minimal downtime                                                 │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Option 3: DUMP AND RESTORE (Offline)                                       │
│  ────────────────────────────────────                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │   1. pg_dump / mysqldump                                            │   │
│  │   2. Transfer to Azure Blob                                         │   │
│  │   3. pg_restore / mysql import                                      │   │
│  │                                                                      │   │
│  │  • Simple for small databases                                       │   │
│  │  • Downtime = dump time + transfer + restore                       │   │
│  │  • Best for < 100 GB                                                │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Migration Steps Using DMS

```bash
# 1. Create DMS Service
az dms create \
  --resource-group myRG \
  --name myDMS \
  --location eastus \
  --sku-name Premium_4vCores \
  --subnet /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}

# 2. Create migration project
az dms project create \
  --resource-group myRG \
  --service-name myDMS \
  --name rds-to-azure-pg \
  --source-platform PostgreSQL \
  --target-platform AzureDbForPostgreSql

# 3. Configure source (AWS RDS) - via Portal or REST API
# Requires:
# - RDS endpoint
# - Database credentials
# - SSL certificate (if enabled)

# 4. Configure target (Azure Flexible Server)
az dms project task create \
  --resource-group myRG \
  --service-name myDMS \
  --project-name rds-to-azure-pg \
  --name migrate-all-tables \
  --source-connection-json '{
    "serverName": "rds-instance.xxx.us-east-1.rds.amazonaws.com",
    "port": 5432,
    "userName": "postgres",
    "password": "***",
    "databaseName": "mydb"
  }' \
  --target-connection-json '{
    "serverName": "postgres-xxx.postgres.database.azure.com",
    "port": 5432,
    "userName": "postgres",
    "password": "***",
    "databaseName": "mydb"
  }' \
  --database-options-json '{"selectedTables": ["public.*"]}'

# 5. Start migration
az dms project task start \
  --resource-group myRG \
  --service-name myDMS \
  --project-name rds-to-azure-pg \
  --name migrate-all-tables

# 6. Monitor progress
az dms project task show \
  --resource-group myRG \
  --service-name myDMS \
  --project-name rds-to-azure-pg \
  --name migrate-all-tables \
  --expand output

# 7. Cutover (when ready)
az dms project task cutover \
  --resource-group myRG \
  --service-name myDMS \
  --project-name rds-to-azure-pg \
  --name migrate-all-tables \
  --object-name mydb
```

### Native Logical Replication (PostgreSQL)

```sql
-- ON AWS RDS (Source)
-- 1. Ensure rds.logical_replication = 1 in parameter group
-- 2. Create publication
CREATE PUBLICATION azure_migration FOR ALL TABLES;

-- ON AZURE (Target)
-- 3. Create subscription
CREATE SUBSCRIPTION azure_sub
CONNECTION 'host=rds-instance.xxx.rds.amazonaws.com port=5432 dbname=mydb user=postgres password=xxx sslmode=require'
PUBLICATION azure_migration;

-- 4. Monitor replication
SELECT * FROM pg_stat_subscription;

-- 5. Check replication lag
SELECT
    pid,
    state,
    sent_lsn,
    write_lsn,
    flush_lsn,
    replay_lsn,
    (sent_lsn - replay_lsn) as lag_bytes
FROM pg_stat_replication;

-- 6. Cutover - stop writes to source, wait for sync
-- 7. Drop subscription on Azure
DROP SUBSCRIPTION azure_sub;
```

---

## Performance Optimization

### PostgreSQL Performance Tuning

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Performance Configuration                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Key Server Parameters:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Parameter              │ Description         │ Recommended Value    │   │
│  ├────────────────────────┼─────────────────────┼──────────────────────┤   │
│  │ shared_buffers         │ Memory for caching  │ 25% of RAM           │   │
│  │ effective_cache_size   │ Planner estimate    │ 75% of RAM           │   │
│  │ maintenance_work_mem   │ Maintenance ops     │ 512MB-2GB            │   │
│  │ work_mem               │ Per-operation mem   │ 16-64MB              │   │
│  │ max_connections        │ Max connections     │ Based on vCores      │   │
│  │ checkpoint_timeout     │ Checkpoint interval │ 15-30min             │   │
│  │ max_wal_size           │ WAL size before chk │ 4-8GB                │   │
│  │ random_page_cost       │ SSD optimized       │ 1.1 (default 4.0)    │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Azure-Specific Parameters (Server Parameters in Portal):                  │
│  • intelligent_tuning: Automatically adjusts parameters                    │
│  • pg_qs.query_capture_mode: Query Store capture mode                     │
│  • pgms_wait_sampling.query_capture_mode: Wait statistics               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### MySQL Performance Tuning

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MySQL Performance Configuration                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Key Server Parameters:                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Parameter                  │ Description        │ Recommended       │   │
│  ├────────────────────────────┼────────────────────┼───────────────────┤   │
│  │ innodb_buffer_pool_size    │ Buffer pool        │ 70-80% of RAM     │   │
│  │ innodb_log_file_size       │ Redo log size      │ 1-2GB             │   │
│  │ innodb_flush_log_at_trx_commit │ Durability    │ 1 (safe) or 2     │   │
│  │ innodb_io_capacity         │ I/O throughput     │ 1000-2000         │   │
│  │ max_connections            │ Max connections    │ Based on workload │   │
│  │ tmp_table_size             │ Temp table size    │ 64-256MB          │   │
│  │ table_open_cache           │ Table cache        │ 2000-4000         │   │
│  │ slow_query_log             │ Slow query log     │ ON                │   │
│  │ long_query_time            │ Slow query threshold│ 2 seconds        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Azure-Specific Features:                                                   │
│  • Auto-grow IOPS: Automatically scales IOPS with storage                 │
│  • Intelligent Performance: Auto-index recommendations                    │
│  • Query Store: Track query performance over time                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### CLI: Performance Configuration

```bash
# PostgreSQL - Update server parameters
az postgres flexible-server parameter set \
  --resource-group myRG \
  --server-name mypostgres \
  --name shared_buffers \
  --value "4GB"

az postgres flexible-server parameter set \
  --resource-group myRG \
  --server-name mypostgres \
  --name work_mem \
  --value "32MB"

# Enable Query Store
az postgres flexible-server parameter set \
  --resource-group myRG \
  --server-name mypostgres \
  --name pg_qs.query_capture_mode \
  --value "ALL"

# MySQL - Update server parameters
az mysql flexible-server parameter set \
  --resource-group myRG \
  --server-name mymysql \
  --name innodb_buffer_pool_size \
  --value "8589934592"  # 8GB in bytes

# Enable slow query log
az mysql flexible-server parameter set \
  --resource-group myRG \
  --server-name mymysql \
  --name slow_query_log \
  --value "ON"

# List current parameters
az postgres flexible-server parameter list \
  --resource-group myRG \
  --server-name mypostgres \
  --query "[?source=='user-override']" \
  --output table
```

### Performance Monitoring

```sql
-- PostgreSQL: Top queries by execution time
SELECT
    substring(query, 1, 100) as query,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round(mean_exec_time::numeric, 2) as mean_time_ms,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- PostgreSQL: Index usage
SELECT
    schemaname,
    tablename,
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- PostgreSQL: Missing indexes (table scans)
SELECT
    schemaname,
    relname,
    seq_scan,
    seq_tup_read,
    idx_scan,
    n_live_tup
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC;

-- MySQL: Slow queries
SELECT * FROM mysql.slow_log
ORDER BY start_time DESC
LIMIT 10;

-- MySQL: Query execution statistics (if Performance Schema enabled)
SELECT
    DIGEST_TEXT,
    COUNT_STAR as executions,
    SUM_TIMER_WAIT/1000000000000 as total_time_sec,
    AVG_TIMER_WAIT/1000000000000 as avg_time_sec
FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

---

## Networking and Security

### Private Access Configuration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Private Access (VNet Integration)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Unlike AWS RDS (requires VPC peering or PrivateLink),                      │
│  Azure Flexible Server has native VNet integration                          │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                         Virtual Network                              │   │
│  │                                                                      │   │
│  │   App Subnet                    DB Subnet (Delegated)               │   │
│  │  ┌─────────────────┐          ┌─────────────────────────┐          │   │
│  │  │                 │          │                         │          │   │
│  │  │   ┌─────────┐   │          │   ┌─────────────────┐   │          │   │
│  │  │   │   App   │───┼──────────┼──▶│  PostgreSQL/    │   │          │   │
│  │  │   │  Server │   │  Private │   │  MySQL Server   │   │          │   │
│  │  │   └─────────┘   │   IP     │   │                 │   │          │   │
│  │  │                 │          │   └─────────────────┘   │          │   │
│  │  └─────────────────┘          └─────────────────────────┘          │   │
│  │                                                                      │   │
│  │  Private DNS Zone: privatelink.postgres.database.azure.com          │   │
│  │  Server FQDN resolves to private IP                                 │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Requirements:                                                              │
│  • Delegated subnet (Microsoft.DBforPostgreSQL/flexibleServers)           │
│  • Private DNS zone linked to VNet                                         │
│  • Subnet must have at least /28 (16 IPs)                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bicep: Network Configuration

```bicep
// Virtual Network
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'vnet-databases'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: 'app-subnet'
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
      {
        name: 'postgres-subnet'
        properties: {
          addressPrefix: '10.0.2.0/24'
          delegations: [
            {
              name: 'postgres-delegation'
              properties: {
                serviceName: 'Microsoft.DBforPostgreSQL/flexibleServers'
              }
            }
          ]
        }
      }
      {
        name: 'mysql-subnet'
        properties: {
          addressPrefix: '10.0.3.0/24'
          delegations: [
            {
              name: 'mysql-delegation'
              properties: {
                serviceName: 'Microsoft.DBforMySQL/flexibleServers'
              }
            }
          ]
        }
      }
    ]
  }
}

// Private DNS Zone for PostgreSQL
resource privateDnsZonePostgres 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.postgres.database.azure.com'
  location: 'global'
}

resource dnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZonePostgres
  name: 'postgres-dns-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}

// PostgreSQL with Private Access
resource postgresServer 'Microsoft.DBforPostgreSQL/flexibleServers@2023-06-01-preview' = {
  name: 'postgres-private-${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Standard_D4ds_v4'
    tier: 'GeneralPurpose'
  }
  properties: {
    version: '15'
    administratorLogin: 'pgadmin'
    administratorLoginPassword: adminPassword
    storage: {
      storageSizeGB: 128
    }
    network: {
      delegatedSubnetResourceId: vnet.properties.subnets[1].id
      privateDnsZoneArmResourceId: privateDnsZonePostgres.id
    }
  }
}

// Azure AD Authentication
resource aadAdmin 'Microsoft.DBforPostgreSQL/flexibleServers/administrators@2023-06-01-preview' = {
  parent: postgresServer
  name: aadAdminObjectId  // Object ID of AAD user/group
  properties: {
    tenantId: subscription().tenantId
    principalType: 'User'  // or 'Group', 'ServicePrincipal'
    principalName: 'db-admin@contoso.com'
  }
}

// Firewall rule (only for public access)
resource firewallRule 'Microsoft.DBforPostgreSQL/flexibleServers/firewallRules@2023-06-01-preview' = if (publicAccess) {
  parent: postgresServer
  name: 'AllowAzureServices'
  properties: {
    startIpAddress: '0.0.0.0'
    endIpAddress: '0.0.0.0'
  }
}
```

### CLI: Security Configuration

```bash
# Enable Azure AD authentication
az postgres flexible-server ad-admin create \
  --resource-group myRG \
  --server-name mypostgres \
  --display-name "DB Admins" \
  --object-id <aad-group-object-id> \
  --type Group

# Configure SSL
az postgres flexible-server parameter set \
  --resource-group myRG \
  --server-name mypostgres \
  --name require_secure_transport \
  --value "ON"

# Enable data encryption with CMK
az postgres flexible-server update \
  --resource-group myRG \
  --name mypostgres \
  --key https://mykeyvault.vault.azure.net/keys/mykey/version \
  --identity myUserAssignedIdentity

# Configure backup
az postgres flexible-server update \
  --resource-group myRG \
  --name mypostgres \
  --backup-retention 35 \
  --geo-redundant-backup Enabled

# Stop server (cost saving)
az postgres flexible-server stop \
  --resource-group myRG \
  --name mypostgres

# Start server
az postgres flexible-server start \
  --resource-group myRG \
  --name mypostgres
```

---

## Extensions and Features

### PostgreSQL Extensions

```sql
-- List available extensions
SELECT name, default_version, comment
FROM pg_available_extensions
ORDER BY name;

-- Popular extensions available on Azure
CREATE EXTENSION IF NOT EXISTS postgis;          -- Geospatial
CREATE EXTENSION IF NOT EXISTS pg_stat_statements; -- Query statistics
CREATE EXTENSION IF NOT EXISTS pgcrypto;         -- Cryptographic functions
CREATE EXTENSION IF NOT EXISTS uuid-ossp;        -- UUID generation
CREATE EXTENSION IF NOT EXISTS pg_trgm;          -- Trigram similarity
CREATE EXTENSION IF NOT EXISTS btree_gin;        -- GIN index for scalars
CREATE EXTENSION IF NOT EXISTS hstore;           -- Key-value pairs
CREATE EXTENSION IF NOT EXISTS citext;           -- Case-insensitive text

-- Azure-specific extensions
CREATE EXTENSION IF NOT EXISTS azure;            -- Azure integration
CREATE EXTENSION IF NOT EXISTS pg_cron;          -- Scheduled jobs
CREATE EXTENSION IF NOT EXISTS pgaudit;          -- Audit logging
```

### MySQL Features

```sql
-- Check MySQL version and features
SELECT VERSION();
SHOW VARIABLES LIKE 'innodb%';

-- Azure-specific features
-- 1. Query Store
SHOW VARIABLES LIKE 'query_store%';

-- 2. Slow query log to Azure Monitor
SHOW VARIABLES LIKE 'slow_query_log%';

-- 3. Intelligent Performance recommendations
-- Available in Azure Portal under "Intelligent Performance"
```

---

*Next: [Redis Cache](04-redis-cache.md)* | *Previous: [Cosmos DB](02-cosmos-db.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
