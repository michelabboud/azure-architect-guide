# Chapter 15: Databases

## Overview

This chapter covers Azure database services for relational and non-relational workloads. As an AWS architect, you'll find familiar concepts with Azure-specific features and deployment options.

## AWS Architect Quick Reference

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| RDS (SQL Server) | Azure SQL Database | Hyperscale tier, serverless |
| RDS (MySQL) | Azure Database for MySQL | Flexible Server architecture |
| RDS (PostgreSQL) | Azure Database for PostgreSQL | Citus for scale-out |
| Aurora | Azure SQL Hyperscale | Different scaling model |
| DynamoDB | Cosmos DB | Multiple APIs, 5 consistency levels |
| DocumentDB | Cosmos DB (MongoDB API) | Native MongoDB wire protocol |
| ElastiCache (Redis) | Azure Cache for Redis | Enterprise tier with modules |
| Neptune | Cosmos DB (Gremlin API) | Graph API support |
| Redshift | Azure Synapse Analytics | Integrated analytics platform |

## Chapter Contents

### [Quick Reference](quick-reference.md)
- CLI commands
- Connection strings
- Performance tuning

### [01 - Azure SQL](01-azure-sql.md)
- Deployment options (Single, Elastic Pool, Managed Instance)
- High availability configurations
- Performance tiers
- Security features

### [02 - Cosmos DB](02-cosmos-db.md)
- APIs and data models
- Consistency levels
- Partitioning strategies
- Global distribution

### [03 - PostgreSQL & MySQL](03-postgresql-mysql.md)
- Flexible Server architecture
- High availability
- Migration strategies

### [04 - Redis Cache](04-redis-cache.md)
- Tiers and features
- Clustering
- Geo-replication

## Database Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Azure Database Selection Guide                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Relational Data?                                                            │
│  │                                                                           │
│  ├── Yes ──► SQL Server features needed?                                    │
│  │           │                                                               │
│  │           ├── Full SQL Server ──► Azure SQL MI                           │
│  │           │   (Agent, CLR, etc.)                                         │
│  │           │                                                               │
│  │           ├── Standard features ──► Azure SQL DB                         │
│  │           │                                                               │
│  │           └── Open source preferred ──► PostgreSQL/MySQL Flexible        │
│  │                                                                           │
│  └── No ──► What data model?                                                │
│             │                                                                │
│             ├── Key-Value/Document ──► Cosmos DB                            │
│             │                                                                │
│             ├── Wide-column ──► Cosmos DB (Cassandra API)                   │
│             │                                                                │
│             ├── Graph ──► Cosmos DB (Gremlin API)                           │
│             │                                                                │
│             └── In-memory cache ──► Azure Cache for Redis                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Service Comparison

### Azure SQL Deployment Options

| Feature | SQL Database | Elastic Pool | Managed Instance |
|---------|-------------|--------------|------------------|
| Use Case | Single workload | Multi-tenant | Lift & shift |
| SQL Agent | No | No | Yes |
| CLR | No | No | Yes |
| Cross-DB queries | No | Yes | Yes |
| Linked servers | No | No | Yes |
| Max DB size | 100TB (Hyperscale) | 4TB | 16TB |
| Pricing | Per DB | Shared pool | Per instance |

### Cosmos DB APIs

| API | Data Model | Wire Protocol | Use Case |
|-----|------------|---------------|----------|
| NoSQL (Core) | Document | Native | New apps, flexible schema |
| MongoDB | Document | MongoDB | MongoDB migrations |
| Cassandra | Wide-column | CQL | Cassandra migrations |
| Gremlin | Graph | TinkerPop | Graph relationships |
| Table | Key-Value | Azure Table | Table Storage upgrade |
| PostgreSQL | Relational | PostgreSQL | Distributed PostgreSQL |

### Cosmos DB Consistency Levels

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   Cosmos DB Consistency Spectrum                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Strong ◄───────────────────────────────────────────────────────► Eventual  │
│                                                                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │  Strong  │  │ Bounded  │  │ Session  │  │Consistent│  │ Eventual │      │
│  │          │  │ Staleness│  │          │  │  Prefix  │  │          │      │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘      │
│                                                                              │
│  Latency:      Highest ◄─────────────────────────────────────► Lowest       │
│  Availability: Lower ◄───────────────────────────────────────► Higher       │
│  Throughput:   Lower ◄───────────────────────────────────────► Higher       │
│  Cost:         Higher ◄──────────────────────────────────────► Lower        │
│                                                                              │
│  AWS DynamoDB Equivalent: Eventual consistency / Strong consistency         │
│  (Azure offers 5 levels vs AWS's 2)                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Architecture Patterns

### Multi-Region Database

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Multi-Region Database Pattern                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Azure SQL with Auto-Failover Groups                                        │
│  ────────────────────────────────────                                        │
│                                                                              │
│   Primary Region (East US)              Secondary Region (West US)          │
│  ┌─────────────────────────┐           ┌─────────────────────────┐         │
│  │                         │           │                         │         │
│  │   ┌─────────────────┐   │   Geo-    │   ┌─────────────────┐   │         │
│  │   │   SQL Server    │   │  Replica  │   │   SQL Server    │   │         │
│  │   │   (Primary)     │──────────────────▶│   (Secondary)   │   │         │
│  │   │                 │   │  Async    │   │   (Read-only)   │   │         │
│  │   └─────────────────┘   │           │   └─────────────────┘   │         │
│  │                         │           │                         │         │
│  └─────────────────────────┘           └─────────────────────────┘         │
│                                                                              │
│  Failover Group Listener:                                                    │
│  ├── Read-Write: <failover-group>.database.windows.net                      │
│  └── Read-Only:  <failover-group>.secondary.database.windows.net            │
│                                                                              │
│  Cosmos DB Multi-Region                                                      │
│  ──────────────────────                                                      │
│                                                                              │
│   East US              West Europe           Southeast Asia                 │
│  ┌─────────────┐      ┌─────────────┐      ┌─────────────┐                 │
│  │   Cosmos    │◄────▶│   Cosmos    │◄────▶│   Cosmos    │                 │
│  │   Replica   │      │   Replica   │      │   Replica   │                 │
│  │   (Write)   │      │   (Write)   │      │   (Write)   │                 │
│  └─────────────┘      └─────────────┘      └─────────────┘                 │
│         │                   │                    │                          │
│         └───────────────────┼────────────────────┘                          │
│                             │                                                │
│                    Multi-Region Write                                        │
│                    (Automatic conflict resolution)                           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Caching Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Cache-Aside Pattern                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Application                                                                │
│       │                                                                      │
│       │ 1. Request data                                                      │
│       ▼                                                                      │
│   ┌───────────────────┐                                                     │
│   │   Check Cache     │                                                     │
│   └─────────┬─────────┘                                                     │
│             │                                                                │
│       ┌─────┴─────┐                                                         │
│       │           │                                                         │
│   Cache Hit   Cache Miss                                                    │
│       │           │                                                         │
│       ▼           ▼                                                         │
│   Return      ┌───────────────────┐                                         │
│   Data        │   Query Database  │                                         │
│               └─────────┬─────────┘                                         │
│                         │                                                    │
│                         ▼                                                    │
│               ┌───────────────────┐                                         │
│               │   Store in Cache  │                                         │
│               │   (with TTL)      │                                         │
│               └─────────┬─────────┘                                         │
│                         │                                                    │
│                         ▼                                                    │
│                    Return Data                                               │
│                                                                              │
│   Azure Implementation:                                                      │
│   ├── Azure Cache for Redis (cache)                                         │
│   ├── Azure SQL / Cosmos DB (database)                                      │
│   └── TTL: Set based on data volatility                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Cost Comparison

### Azure SQL Pricing Tiers

| Tier | vCores | Storage | Monthly Cost* |
|------|--------|---------|---------------|
| Basic | N/A | 2GB | $5 |
| Standard S0 | N/A | 250GB | $15 |
| General Purpose | 2-80 | Up to 4TB | $200-8,000 |
| Business Critical | 2-128 | Up to 4TB | $500-25,000 |
| Hyperscale | 2-128 | Up to 100TB | $300-12,000 |

### Cosmos DB Pricing (Request Units)

| Provisioned RU/s | Storage (GB) | Monthly Cost* |
|------------------|--------------|---------------|
| 400 | 5 | $25 |
| 1,000 | 50 | $60 |
| 10,000 | 500 | $600 |
| 100,000 | 5,000 | $6,000 |
| Serverless | Pay-per-use | Variable |

*Approximate costs, vary by region

## Learning Objectives

After completing this chapter, you will:

1. **Select** the appropriate Azure database service for your workload
2. **Design** high-availability database architectures
3. **Configure** Cosmos DB consistency and partitioning
4. **Implement** Azure SQL security features
5. **Optimize** database performance and cost
6. **Migrate** from AWS database services to Azure

---

*Continue to [Quick Reference](quick-reference.md)*

*Back to [Main Guide](../README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
