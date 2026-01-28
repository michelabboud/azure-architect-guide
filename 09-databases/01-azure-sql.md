# Azure SQL Comprehensive Guide

## Overview

Azure SQL is Microsoft's family of relational database services built on SQL Server. Unlike AWS RDS which offers multiple engines under one umbrella, Azure SQL provides three distinct deployment options optimized for different scenarios: Single Database, Elastic Pool, and Managed Instance.

## AWS Comparison

| Feature | AWS RDS SQL Server | Azure SQL Database | Azure SQL MI |
|---------|-------------------|-------------------|--------------|
| Deployment model | Instance-based | Database-as-a-service | Instance-based PaaS |
| SQL Agent | Yes | No | Yes |
| CLR integration | Yes | No | Yes |
| Cross-DB queries | Yes | No (Elastic Query) | Yes |
| Linked servers | Yes | No | Yes |
| Max DB size | 16TB | 100TB (Hyperscale) | 16TB |
| Serverless | No | Yes | No |
| Multi-AZ | Yes | Zone redundant | Zone redundant |
| Read replicas | Up to 5 | Up to 4 (Hyperscale 30) | Failover groups |

---

## Deployment Options

### Architecture Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Azure SQL Deployment Options                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SINGLE DATABASE                                                             │
│  ───────────────                                                             │
│  ┌──────────────────┐                                                       │
│  │   SQL Database   │  • Isolated database with dedicated resources         │
│  │   ┌──────────┐   │  • Predictable performance                            │
│  │   │   DB 1   │   │  • Ideal for single applications                      │
│  │   └──────────┘   │  • Supports serverless compute tier                   │
│  └──────────────────┘                                                       │
│                                                                              │
│  ELASTIC POOL                                                                │
│  ────────────                                                                │
│  ┌──────────────────────────────────┐                                       │
│  │         Shared Resources          │  • Multiple DBs share resources      │
│  │   ┌─────┐ ┌─────┐ ┌─────┐       │  • Cost-effective for SaaS            │
│  │   │ DB1 │ │ DB2 │ │ DB3 │ ...   │  • Variable usage patterns           │
│  │   └─────┘ └─────┘ └─────┘       │  • Up to 500 DBs per pool             │
│  └──────────────────────────────────┘                                       │
│                                                                              │
│  MANAGED INSTANCE                                                            │
│  ────────────────                                                            │
│  ┌──────────────────────────────────┐                                       │
│  │    SQL Server Instance (PaaS)     │  • Near 100% SQL Server compatible  │
│  │   ┌─────┐ ┌─────┐ ┌─────┐       │  • SQL Agent, CLR, linked servers    │
│  │   │ DB1 │ │ DB2 │ │ DB3 │       │  • Native VNet integration            │
│  │   └─────┘ └─────┘ └─────┘       │  • Ideal for lift-and-shift          │
│  │   SQL Agent │ Service Broker     │                                       │
│  └──────────────────────────────────┘                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### When to Use Each Option

| Scenario | Recommended Option | Reasoning |
|----------|-------------------|-----------|
| New cloud-native app | Single Database | Simplest, fully managed |
| SaaS multi-tenant | Elastic Pool | Cost sharing across tenants |
| Lift-and-shift from on-prem | Managed Instance | SQL Server compatibility |
| Bursty workloads | Serverless Single DB | Auto-pause saves costs |
| Massive scale (100TB+) | Hyperscale | Distributed storage |
| Cross-database queries | Managed Instance | Native T-SQL support |

---

## High Availability

### Zone Redundancy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Zone-Redundant Configuration                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                         Azure Region (e.g., East US)                         │
│  ┌────────────────────────────────────────────────────────────────────┐     │
│  │                                                                     │     │
│  │   Zone 1              Zone 2              Zone 3                   │     │
│  │  ┌────────────┐      ┌────────────┐      ┌────────────┐           │     │
│  │  │            │      │            │      │            │           │     │
│  │  │  Primary   │─────▶│ Secondary  │─────▶│ Secondary  │           │     │
│  │  │  Replica   │ sync │  Replica   │ sync │  Replica   │           │     │
│  │  │            │      │            │      │            │           │     │
│  │  └────────────┘      └────────────┘      └────────────┘           │     │
│  │       │                   │                   │                    │     │
│  │       ▼                   ▼                   ▼                    │     │
│  │  ┌────────────┐      ┌────────────┐      ┌────────────┐           │     │
│  │  │  Storage   │◀────▶│  Storage   │◀────▶│  Storage   │           │     │
│  │  └────────────┘      └────────────┘      └────────────┘           │     │
│  │                                                                     │     │
│  └────────────────────────────────────────────────────────────────────┘     │
│                                                                              │
│  • Automatic failover within region (seconds)                               │
│  • No data loss (synchronous replication)                                   │
│  • 99.995% SLA (vs 99.99% without zone redundancy)                         │
│  • Available in Business Critical and Premium tiers                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Auto-Failover Groups

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Auto-Failover Groups (Geo-DR)                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Primary Region                      Secondary Region                       │
│   (East US)                          (West US)                              │
│  ┌─────────────────────┐            ┌─────────────────────┐                │
│  │                     │            │                     │                │
│  │  ┌───────────────┐  │   Async    │  ┌───────────────┐  │                │
│  │  │ SQL Database  │  │──────────▶│  │ SQL Database  │  │                │
│  │  │   (Primary)   │  │  Geo-Rep  │  │  (Secondary)  │  │                │
│  │  │               │  │            │  │  (Read-Only)  │  │                │
│  │  └───────────────┘  │            │  └───────────────┘  │                │
│  │                     │            │                     │                │
│  └─────────────────────┘            └─────────────────────┘                │
│           │                                   │                             │
│           ▼                                   ▼                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              Failover Group DNS Endpoints                            │   │
│  │                                                                      │   │
│  │  Read-Write:  <group-name>.database.windows.net                     │   │
│  │  Read-Only:   <group-name>.secondary.database.windows.net           │   │
│  │                                                                      │   │
│  │  Automatic failover based on policy (30 sec - 1 hour grace period)  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bicep: Failover Group Configuration

```bicep
// Primary SQL Server
resource primaryServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: 'sql-primary-${uniqueString(resourceGroup().id)}'
  location: 'eastus'
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlAdminPassword
  }
}

// Secondary SQL Server (different region)
resource secondaryServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: 'sql-secondary-${uniqueString(resourceGroup().id)}'
  location: 'westus'
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlAdminPassword
  }
}

// Primary Database
resource primaryDatabase 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: primaryServer
  name: 'mydb'
  location: 'eastus'
  sku: {
    name: 'GP_Gen5'
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 2
  }
  properties: {
    zoneRedundant: true
  }
}

// Failover Group
resource failoverGroup 'Microsoft.Sql/servers/failoverGroups@2023-05-01-preview' = {
  parent: primaryServer
  name: 'myapp-failover-group'
  properties: {
    readWriteEndpoint: {
      failoverPolicy: 'Automatic'
      failoverWithDataLossGracePeriodMinutes: 60
    }
    readOnlyEndpoint: {
      failoverPolicy: 'Enabled'
    }
    partnerServers: [
      {
        id: secondaryServer.id
      }
    ]
    databases: [
      primaryDatabase.id
    ]
  }
}

output failoverGroupReadWriteEndpoint string = '${failoverGroup.name}.database.windows.net'
output failoverGroupReadOnlyEndpoint string = '${failoverGroup.name}.secondary.database.windows.net'
```

### CLI: Failover Group Commands

```bash
# Create failover group
az sql failover-group create \
  --name myapp-failover-group \
  --partner-server sql-secondary \
  --resource-group myRG \
  --server sql-primary \
  --partner-resource-group myRG \
  --failover-policy Automatic \
  --grace-period 1

# Add database to failover group
az sql failover-group update \
  --name myapp-failover-group \
  --add-db mydb \
  --resource-group myRG \
  --server sql-primary

# Manual failover (planned)
az sql failover-group set-primary \
  --name myapp-failover-group \
  --resource-group myRG \
  --server sql-secondary

# Forced failover (with potential data loss)
az sql failover-group set-primary \
  --name myapp-failover-group \
  --resource-group myRG \
  --server sql-secondary \
  --allow-data-loss
```

---

## Performance Tiers

### DTU vs vCore Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DTU vs vCore Pricing Models                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DTU (Database Transaction Unit)                                            │
│  ───────────────────────────────                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  DTU = Bundled measure of CPU + Memory + I/O                        │   │
│  │                                                                      │   │
│  │  Tiers:                                                              │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐  ┌────────┐        │   │
│  │  │ Basic  │  │  S0    │  │  S3    │  │  P1    │  │  P15   │        │   │
│  │  │ 5 DTU  │  │ 10 DTU │  │100 DTU │  │125 DTU │  │4000 DTU│        │   │
│  │  └────────┘  └────────┘  └────────┘  └────────┘  └────────┘        │   │
│  │  Basic      Standard               Premium                          │   │
│  │                                                                      │   │
│  │  Best for: Simple workloads, predictable performance needs         │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  vCore (Virtual Core)                                                       │
│  ────────────────────                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  Independent control over compute and storage                       │   │
│  │                                                                      │   │
│  │  Service Tiers:                                                      │   │
│  │  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐        │   │
│  │  │General Purpose │  │Business Critical│  │   Hyperscale   │        │   │
│  │  │  2-128 vCores  │  │   2-128 vCores  │  │  2-128 vCores  │        │   │
│  │  │  Remote storage│  │  Local SSD      │  │  Tiered storage│        │   │
│  │  │  5-7ms latency │  │  1-2ms latency  │  │  Scale-out     │        │   │
│  │  └────────────────┘  └────────────────┘  └────────────────┘        │   │
│  │                                                                      │   │
│  │  Compute:  Gen5 (Intel), Fsv2 (compute-optimized), DC (confidential)│   │
│  │                                                                      │   │
│  │  Best for: Migrating from SQL Server, need specific vCores/RAM     │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Hyperscale Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Hyperscale Architecture                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              ┌───────────────┐                              │
│                              │   Compute     │                              │
│                              │   (Primary)   │                              │
│                              └───────┬───────┘                              │
│                                      │                                       │
│                      ┌───────────────┼───────────────┐                      │
│                      ▼               ▼               ▼                      │
│              ┌───────────┐   ┌───────────┐   ┌───────────┐                 │
│              │ HA Replica│   │ HA Replica│   │ HA Replica│                 │
│              │     1     │   │     2     │   │     3     │                 │
│              └───────────┘   └───────────┘   └───────────┘                 │
│                                      │                                       │
│                      ┌───────────────┼───────────────┐                      │
│                      ▼               ▼               ▼                      │
│              ┌───────────┐   ┌───────────┐   ┌───────────┐                 │
│              │Named Read │   │Named Read │   │Geo-Second-│                 │
│              │ Replica 1 │   │ Replica 2 │   │   ary     │                 │
│              └───────────┘   └───────────┘   └───────────┘                 │
│                                                                              │
│                              ┌───────────────┐                              │
│                              │  Log Service  │                              │
│                              │   (Landing)   │                              │
│                              └───────┬───────┘                              │
│                                      │                                       │
│              ┌───────────────────────┼───────────────────────┐              │
│              ▼                       ▼                       ▼              │
│       ┌───────────┐           ┌───────────┐           ┌───────────┐        │
│       │Page Server│           │Page Server│           │Page Server│        │
│       │  (0-100GB)│           │(100-200GB)│           │(200-300GB)│        │
│       └───────────┘           └───────────┘           └───────────┘        │
│              │                       │                       │              │
│              ▼                       ▼                       ▼              │
│       ┌─────────────────────────────────────────────────────────────┐      │
│       │                    Azure Blob Storage                        │      │
│       │                    (Up to 100TB)                             │      │
│       └─────────────────────────────────────────────────────────────┘      │
│                                                                              │
│  Key Benefits:                                                               │
│  • Near-instant backups (storage snapshots)                                 │
│  • Fast restore (minutes, not hours)                                        │
│  • Up to 30 named read replicas                                             │
│  • Storage auto-scales (no pre-provisioning)                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Serverless Compute

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Serverless Compute Tier                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Auto-scaling and Auto-pause                                                │
│                                                                              │
│         vCores                                                               │
│           ▲                                                                  │
│      Max ─┤         ┌──────┐                    ┌──────┐                    │
│           │         │      │                    │      │                    │
│           │    ┌────┘      └────┐          ┌────┘      │                    │
│           │    │                │          │           │                    │
│      Min ─┤────┘                └──────────┘           └────                │
│           │                                                                  │
│        0 ─┤─ ─ ─ ─ ─ ─PAUSED─ ─ ─ ─ ─ ─ ─ ─ ─PAUSED─ ─ ─ ─ ─               │
│           └──────────────────────────────────────────────────▶ Time         │
│                                                                              │
│  Configuration:                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Min vCores:        0.5 - 40 vCores (can set to 0.5 for cost saving) │   │
│  │ Max vCores:        0.5 - 40 vCores                                   │   │
│  │ Auto-pause delay:  1 hour - 7 days (or disabled)                    │   │
│  │ Resume latency:    ~1 minute (first connection triggers resume)     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Billing:                                                                    │
│  • Compute: Per second based on vCores used                                 │
│  • Storage: Per GB per month (always charged)                               │
│  • Paused:  Only storage costs (no compute)                                 │
│                                                                              │
│  Best for:                                                                   │
│  • Development/test environments                                            │
│  • Infrequently used databases                                              │
│  • Unpredictable workloads                                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bicep: Serverless Database

```bicep
resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: 'sql-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlAdminPassword
    minimalTlsVersion: '1.2'
  }
}

// Serverless database
resource serverlessDb 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'dev-database'
  location: location
  sku: {
    name: 'GP_S_Gen5'  // S = Serverless
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 2  // Max vCores
  }
  properties: {
    autoPauseDelay: 60  // Minutes until auto-pause (-1 to disable)
    minCapacity: json('0.5')  // Min vCores
    zoneRedundant: false
  }
}

// Hyperscale database
resource hyperscaleDb 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'large-database'
  location: location
  sku: {
    name: 'HS_Gen5'  // HS = Hyperscale
    tier: 'Hyperscale'
    family: 'Gen5'
    capacity: 4
  }
  properties: {
    highAvailabilityReplicaCount: 2  // HA replicas
    readScale: 'Enabled'  // Read scale-out
  }
}
```

---

## Security Features

### Transparent Data Encryption (TDE)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Transparent Data Encryption                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  TDE encrypts data at rest (database files, backups, transaction logs)      │
│                                                                              │
│  Encryption Hierarchy:                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │       Option 1: Service-Managed Key                                 │   │
│  │       ────────────────────────────                                  │   │
│  │       ┌─────────────────┐                                           │   │
│  │       │   Azure-managed │  • Default, no configuration needed       │   │
│  │       │   TDE protector │  • Microsoft manages key rotation        │   │
│  │       └────────┬────────┘                                           │   │
│  │                │                                                     │   │
│  │                ▼                                                     │   │
│  │       ┌─────────────────┐                                           │   │
│  │       │  Database       │                                           │   │
│  │       │  Encryption Key │                                           │   │
│  │       └────────┬────────┘                                           │   │
│  │                │                                                     │   │
│  │                ▼                                                     │   │
│  │       ┌─────────────────┐                                           │   │
│  │       │  Data Files     │                                           │   │
│  │       │  (Encrypted)    │                                           │   │
│  │       └─────────────────┘                                           │   │
│  │                                                                      │   │
│  │       Option 2: Customer-Managed Key (BYOK)                         │   │
│  │       ─────────────────────────────────────                         │   │
│  │       ┌─────────────────┐                                           │   │
│  │       │   Azure Key     │  • Customer controls the key              │   │
│  │       │   Vault         │  • Key rotation controlled by customer    │   │
│  │       │   (Your Key)    │  • Required for regulatory compliance     │   │
│  │       └────────┬────────┘                                           │   │
│  │                │                                                     │   │
│  │                ▼                                                     │   │
│  │       ┌─────────────────┐                                           │   │
│  │       │  TDE Protector  │                                           │   │
│  │       └────────┬────────┘                                           │   │
│  │                │                                                     │   │
│  │                ▼                                                     │   │
│  │       ┌─────────────────┐                                           │   │
│  │       │  DEK → Data     │                                           │   │
│  │       └─────────────────┘                                           │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Always Encrypted

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Always Encrypted (Client-Side Encryption)                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Data is encrypted BEFORE reaching the database                             │
│  Azure never sees the plaintext data                                        │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │    Application                                Database                │  │
│  │   ┌───────────┐                           ┌───────────────┐          │  │
│  │   │  Plaintext│    Driver encrypts        │   Encrypted   │          │  │
│  │   │  SSN:     │───────────────────────────▶│   SSN:        │          │  │
│  │   │123-45-6789│    using CMK              │  0x7F4A2B...  │          │  │
│  │   └───────────┘                           └───────────────┘          │  │
│  │        │                                          │                   │  │
│  │        │                                          │                   │  │
│  │        │     Column Master Key (CMK)              │                   │  │
│  │        │     stored in Key Vault                  │                   │  │
│  │        │     ┌─────────────────┐                  │                   │  │
│  │        └────▶│   Azure Key     │◀─────────────────┘                   │  │
│  │              │   Vault         │   CEK wrapped by CMK                │  │
│  │              └─────────────────┘                                      │  │
│  │                                                                       │  │
│  │   Encryption Types:                                                   │  │
│  │   • Deterministic: Same plaintext → same ciphertext (allows = )     │  │
│  │   • Randomized: Same plaintext → different ciphertext (more secure) │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Row-Level Security

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Row-Level Security (RLS)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Filter rows based on user context - ideal for multi-tenant apps           │
│                                                                              │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                                                                       │  │
│  │   Sales Table (without RLS):                                         │  │
│  │   ┌──────────┬─────────────┬────────────┐                            │  │
│  │   │ SalesRep │ Customer    │ Amount     │                            │  │
│  │   ├──────────┼─────────────┼────────────┤                            │  │
│  │   │ Alice    │ Contoso     │ $10,000    │                            │  │
│  │   │ Bob      │ Fabrikam    │ $15,000    │                            │  │
│  │   │ Alice    │ Northwind   │ $8,000     │                            │  │
│  │   │ Charlie  │ AdventureW  │ $12,000    │                            │  │
│  │   └──────────┴─────────────┴────────────┘                            │  │
│  │                                                                       │  │
│  │   With RLS (Alice queries):                                          │  │
│  │   ┌──────────┬─────────────┬────────────┐                            │  │
│  │   │ SalesRep │ Customer    │ Amount     │                            │  │
│  │   ├──────────┼─────────────┼────────────┤                            │  │
│  │   │ Alice    │ Contoso     │ $10,000    │  ◀── Only Alice's rows    │  │
│  │   │ Alice    │ Northwind   │ $8,000     │                            │  │
│  │   └──────────┴─────────────┴────────────┘                            │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### T-SQL: Row-Level Security

```sql
-- Create security policy function
CREATE FUNCTION dbo.fn_SecurityPredicate(@SalesRep AS nvarchar(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS fn_SecurityResult
WHERE @SalesRep = USER_NAME()
   OR USER_NAME() = 'dbo'  -- Admin sees all
   OR IS_MEMBER('SalesManagers') = 1;  -- Managers see all

-- Create security policy
CREATE SECURITY POLICY SalesSecurityPolicy
ADD FILTER PREDICATE dbo.fn_SecurityPredicate(SalesRep) ON dbo.Sales
WITH (STATE = ON);

-- Test as different users
EXECUTE AS USER = 'Alice';
SELECT * FROM dbo.Sales;  -- Only sees Alice's rows
REVERT;

EXECUTE AS USER = 'Bob';
SELECT * FROM dbo.Sales;  -- Only sees Bob's rows
REVERT;
```

### Bicep: Security Configuration

```bicep
resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: 'sql-${uniqueString(resourceGroup().id)}'
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlAdminPassword
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'  // Force private endpoint
  }
}

// Azure AD admin
resource aadAdmin 'Microsoft.Sql/servers/administrators@2023-05-01-preview' = {
  parent: sqlServer
  name: 'ActiveDirectory'
  properties: {
    administratorType: 'ActiveDirectory'
    login: 'SQL Admins'
    sid: aadGroupObjectId
    tenantId: subscription().tenantId
  }
}

// Auditing
resource auditSettings 'Microsoft.Sql/servers/auditingSettings@2023-05-01-preview' = {
  parent: sqlServer
  name: 'default'
  properties: {
    state: 'Enabled'
    storageEndpoint: storageAccount.properties.primaryEndpoints.blob
    storageAccountAccessKey: storageAccount.listKeys().keys[0].value
    retentionDays: 90
    auditActionsAndGroups: [
      'SUCCESSFUL_DATABASE_AUTHENTICATION_GROUP'
      'FAILED_DATABASE_AUTHENTICATION_GROUP'
      'BATCH_COMPLETED_GROUP'
    ]
  }
}

// Advanced Threat Protection
resource threatProtection 'Microsoft.Sql/servers/securityAlertPolicies@2023-05-01-preview' = {
  parent: sqlServer
  name: 'Default'
  properties: {
    state: 'Enabled'
    emailAddresses: ['security@contoso.com']
    emailAccountAdmins: true
    retentionDays: 30
  }
}

// Vulnerability Assessment
resource vulnerabilityAssessment 'Microsoft.Sql/servers/vulnerabilityAssessments@2023-05-01-preview' = {
  parent: sqlServer
  name: 'default'
  properties: {
    storageContainerPath: '${storageAccount.properties.primaryEndpoints.blob}vulnerability-assessment'
    recurringScans: {
      isEnabled: true
      emailSubscriptionAdmins: true
      emails: ['security@contoso.com']
    }
  }
}

// TDE with Customer-Managed Key
resource tdeProtector 'Microsoft.Sql/servers/encryptionProtector@2023-05-01-preview' = {
  parent: sqlServer
  name: 'current'
  properties: {
    serverKeyType: 'AzureKeyVault'
    serverKeyName: 'myKeyVault_myTdeKey_${keyVersion}'
    autoRotationEnabled: true
  }
}
```

### CLI: Security Commands

```bash
# Enable Microsoft Defender for SQL
az sql db threat-policy update \
  --resource-group myRG \
  --server sqlserver \
  --database mydb \
  --state Enabled \
  --email-addresses security@contoso.com \
  --email-account-admins true

# Configure auditing
az sql server audit-policy update \
  --resource-group myRG \
  --name sqlserver \
  --state Enabled \
  --storage-account mystorageaccount \
  --retention-days 90

# Enable Azure AD authentication
az sql server ad-admin create \
  --resource-group myRG \
  --server sqlserver \
  --display-name "SQL Admins" \
  --object-id <aad-group-object-id>

# Configure TDE with customer-managed key
az sql server tde-key set \
  --resource-group myRG \
  --server sqlserver \
  --server-key-type AzureKeyVault \
  --kid "https://myvault.vault.azure.net/keys/mykey/version"

# Create private endpoint
az network private-endpoint create \
  --name sql-pe \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet private-endpoints \
  --private-connection-resource-id $(az sql server show -g myRG -n sqlserver --query id -o tsv) \
  --group-id sqlServer \
  --connection-name sql-connection
```

---

## Performance Optimization

### Query Performance Insight

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Query Performance Insight                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Built-in query performance monitoring:                                     │
│                                                                              │
│  1. Query Store (enabled by default)                                        │
│     ├── Captures query plans                                                │
│     ├── Tracks execution statistics                                         │
│     └── Enables plan regression detection                                   │
│                                                                              │
│  2. Automatic Tuning                                                        │
│     ├── CREATE INDEX: Creates missing indexes                              │
│     ├── DROP INDEX: Removes unused indexes                                 │
│     └── FORCE PLAN: Fixes plan regressions                                 │
│                                                                              │
│  3. Intelligent Insights                                                    │
│     ├── AI-based anomaly detection                                         │
│     ├── Root cause analysis                                                │
│     └── Recommendations                                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### CLI: Performance Commands

```bash
# Check DTU usage
az monitor metrics list \
  --resource $(az sql db show -g myRG -s sqlserver -n mydb --query id -o tsv) \
  --metric dtu_consumption_percent \
  --interval PT5M \
  --output table

# Get automatic tuning recommendations
az sql db tde-recommendation list \
  --resource-group myRG \
  --server sqlserver \
  --database mydb

# Enable automatic tuning
az sql db update \
  --resource-group myRG \
  --server sqlserver \
  --name mydb \
  --auto-pause-delay 60

# Query Query Store for expensive queries
az sql db execute \
  --resource-group myRG \
  --server sqlserver \
  --name mydb \
  --query "
    SELECT TOP 10
      qt.query_sql_text,
      rs.avg_duration,
      rs.count_executions
    FROM sys.query_store_query_text qt
    JOIN sys.query_store_query q ON qt.query_text_id = q.query_text_id
    JOIN sys.query_store_plan p ON q.query_id = p.query_id
    JOIN sys.query_store_runtime_stats rs ON p.plan_id = rs.plan_id
    ORDER BY rs.avg_duration DESC
  "
```

---

## Migration from AWS RDS

### Migration Options

| Method | Downtime | Use Case |
|--------|----------|----------|
| Azure Database Migration Service | Minimal | Large databases, continuous sync |
| BACPAC export/import | Hours | Smaller databases, schema + data |
| Transactional replication | Minimal | Complex scenarios, SQL Server |
| Always On availability groups | Minimal | SQL Server to Managed Instance |

### Azure DMS Configuration

```bash
# Create DMS service
az dms create \
  --resource-group myRG \
  --name myDMS \
  --location eastus \
  --sku-name Premium_4vCores \
  --subnet /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/virtualNetworks/{vnet}/subnets/{subnet}

# Create migration project
az dms project create \
  --resource-group myRG \
  --service-name myDMS \
  --name rds-to-azure-sql \
  --source-platform SQL \
  --target-platform SQLDB

# The actual migration is configured through the Azure Portal
# or using the DMS REST API for detailed task configuration
```

---

*Next: [Cosmos DB](02-cosmos-db.md)* | *Back to [Databases Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
