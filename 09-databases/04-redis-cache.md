# Azure Cache for Redis

## Overview

Azure Cache for Redis is a fully managed, in-memory data store based on Redis. It provides microsecond latency for caching, session management, real-time analytics, and message brokering scenarios. Azure offers multiple tiers from basic development caches to enterprise-grade deployments with Redis modules.

## AWS Comparison

| Feature | AWS ElastiCache for Redis | Azure Cache for Redis |
|---------|---------------------------|----------------------|
| Deployment model | Cluster mode enabled/disabled | Basic, Standard, Premium, Enterprise |
| Clustering | 1-500 shards | Up to 10 shards (Premium), 30+ (Enterprise) |
| Persistence | RDB, AOF | RDB (Premium+), AOF (Enterprise) |
| Geo-replication | Global Datastore | Active geo-replication (Premium+) |
| Redis modules | No | Enterprise tier (RediSearch, RedisBloom, etc.) |
| VPC/VNet | Yes | VNet injection (Premium+) |
| Encryption | In-transit, at-rest | In-transit, at-rest |
| Max size | 6.1 TB (cluster) | 1.2 TB (Premium), 12 TB (Enterprise) |
| Max connections | Varies by node | 10K-40K (varies by tier) |
| SLA | 99.99% (multi-AZ) | 99.9% (Standard), 99.99% (Premium) |

---

## Tiers and Features

### Tier Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Azure Cache for Redis Tiers                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  BASIC                                                                       │
│  ─────                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Single node, no replication                                       │   │
│  │ • No SLA                                                            │   │
│  │ • Sizes: 250 MB - 53 GB                                            │   │
│  │ • Use: Development/testing only                                     │   │
│  │                                                                      │   │
│  │   ┌─────────────┐                                                   │   │
│  │   │   Redis     │                                                   │   │
│  │   │   (Single)  │                                                   │   │
│  │   └─────────────┘                                                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  STANDARD                                                                    │
│  ────────                                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • Primary + Replica (auto-failover)                                 │   │
│  │ • 99.9% SLA                                                         │   │
│  │ • Sizes: 250 MB - 53 GB                                            │   │
│  │ • Use: Production workloads                                         │   │
│  │                                                                      │   │
│  │   ┌─────────────┐     ┌─────────────┐                              │   │
│  │   │   Primary   │────▶│   Replica   │                              │   │
│  │   │             │     │             │                              │   │
│  │   └─────────────┘     └─────────────┘                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  PREMIUM                                                                     │
│  ───────                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • All Standard features plus:                                       │   │
│  │ • Clustering (up to 10 shards)                                      │   │
│  │ • VNet integration                                                  │   │
│  │ • Zone redundancy                                                   │   │
│  │ • Geo-replication (active-passive)                                  │   │
│  │ • Data persistence (RDB snapshots)                                  │   │
│  │ • Import/Export                                                     │   │
│  │ • Sizes: 6 GB - 120 GB per shard                                   │   │
│  │ • 99.95% SLA (zone redundant)                                      │   │
│  │                                                                      │   │
│  │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                 │   │
│  │   │ Shard 0 │ │ Shard 1 │ │ Shard 2 │ │   ...   │                 │   │
│  │   │ P + R   │ │ P + R   │ │ P + R   │ │         │                 │   │
│  │   └─────────┘ └─────────┘ └─────────┘ └─────────┘                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  ENTERPRISE / ENTERPRISE FLASH                                              │
│  ─────────────────────────────                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • All Premium features plus:                                        │   │
│  │ • Redis modules (RediSearch, RedisJSON, RedisBloom, RedisTimeSeries)│   │
│  │ • Active-active geo-replication                                     │   │
│  │ • AOF persistence                                                   │   │
│  │ • 99.999% SLA (with geo-replication)                               │   │
│  │ • Enterprise: All in-memory (up to 12 TB)                          │   │
│  │ • Enterprise Flash: Tiered storage (up to 1.5 TB per shard)       │   │
│  │                                                                      │   │
│  │   Region A                    Region B                              │   │
│  │   ┌─────────────┐            ┌─────────────┐                       │   │
│  │   │  Enterprise │◀──────────▶│  Enterprise │  Active-Active        │   │
│  │   │    Cache    │   Sync     │    Cache    │                       │   │
│  │   └─────────────┘            └─────────────┘                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Feature Matrix

| Feature | Basic | Standard | Premium | Enterprise |
|---------|-------|----------|---------|------------|
| SLA | None | 99.9% | 99.9% | 99.99% |
| Replication | No | Yes | Yes | Yes |
| Clustering | No | No | 10 shards | 30+ shards |
| VNet | No | No | Yes | Yes |
| Zone redundancy | No | No | Yes | Yes |
| Geo-replication | No | No | Active-passive | Active-active |
| Persistence | No | No | RDB | RDB + AOF |
| Redis modules | No | No | No | Yes |
| Max memory | 53 GB | 53 GB | 1.2 TB | 12 TB |

---

## Clustering and Sharding

### Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Redis Cluster Architecture                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Data Distribution (Hash Slots)                                             │
│  ──────────────────────────────                                             │
│  • 16,384 hash slots distributed across shards                             │
│  • Key → CRC16(key) mod 16384 → slot → shard                              │
│  • Multi-key operations require keys in same slot                          │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │   Shard 0              Shard 1              Shard 2                 │   │
│  │   Slots: 0-5460        Slots: 5461-10922    Slots: 10923-16383     │   │
│  │  ┌───────────────┐    ┌───────────────┐    ┌───────────────┐       │   │
│  │  │    Primary    │    │    Primary    │    │    Primary    │       │   │
│  │  │  (6 GB RAM)   │    │  (6 GB RAM)   │    │  (6 GB RAM)   │       │   │
│  │  └───────┬───────┘    └───────┬───────┘    └───────┬───────┘       │   │
│  │          │                    │                    │                │   │
│  │  ┌───────▼───────┐    ┌───────▼───────┐    ┌───────▼───────┐       │   │
│  │  │    Replica    │    │    Replica    │    │    Replica    │       │   │
│  │  │   (Standby)   │    │   (Standby)   │    │   (Standby)   │       │   │
│  │  └───────────────┘    └───────────────┘    └───────────────┘       │   │
│  │                                                                      │   │
│  │  Total Capacity: 18 GB (3 shards × 6 GB)                           │   │
│  │  Total Throughput: 3× single shard                                  │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  Hash Tag for Co-location:                                                  │
│  • Use {hashtag} to ensure keys are on same shard                         │
│  • Example: user:{123}:profile, user:{123}:settings → same shard         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Scaling Operations

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Scaling Strategies                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  VERTICAL SCALING (Scale Up/Down)                                           │
│  ─────────────────────────────────                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │   Before: C1 (1 GB)    →    After: C3 (13 GB)                       │   │
│  │  ┌───────────────┐         ┌───────────────────────────┐            │   │
│  │  │    1 GB       │    →    │         13 GB             │            │   │
│  │  └───────────────┘         └───────────────────────────┘            │   │
│  │                                                                      │   │
│  │  • Minimal downtime (< 30 seconds)                                  │   │
│  │  • Data preserved during scale                                      │   │
│  │  • Can scale within same tier                                       │   │
│  │  • Cross-tier requires migration                                    │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  HORIZONTAL SCALING (Shards)                                                │
│  ───────────────────────────                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │   Before: 3 shards    →    After: 5 shards                          │   │
│  │  ┌───┐┌───┐┌───┐         ┌───┐┌───┐┌───┐┌───┐┌───┐                │   │
│  │  │ 0 ││ 1 ││ 2 │    →    │ 0 ││ 1 ││ 2 ││ 3 ││ 4 │                │   │
│  │  └───┘└───┘└───┘         └───┘└───┘└───┘└───┘└───┘                │   │
│  │                                                                      │   │
│  │  • Premium tier: 1-10 shards                                        │   │
│  │  • Hash slots redistributed                                         │   │
│  │  • Online operation (some latency)                                  │   │
│  │  • Scale out for throughput                                         │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bicep: Clustered Cache

```bicep
@description('Location for resources')
param location string = resourceGroup().location

@description('Number of shards for clustering')
@minValue(1)
@maxValue(10)
param shardCount int = 3

// Premium cache with clustering
resource redisCache 'Microsoft.Cache/redis@2023-08-01' = {
  name: 'redis-cluster-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    sku: {
      name: 'Premium'
      family: 'P'
      capacity: 1  // P1 = 6 GB per shard
    }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
    redisConfiguration: {
      'maxmemory-policy': 'volatile-lru'
      'maxfragmentationmemory-reserved': '125'
      'maxmemory-reserved': '125'
    }
    shardCount: shardCount  // Enable clustering
    replicasPerMaster: 1
  }
  zones: ['1', '2', '3']  // Zone redundancy
}

// Firewall rules
resource firewallRule 'Microsoft.Cache/redis/firewallRules@2023-08-01' = {
  parent: redisCache
  name: 'AllowAppSubnet'
  properties: {
    startIP: '10.0.1.0'
    endIP: '10.0.1.255'
  }
}

output hostName string = redisCache.properties.hostName
output sslPort int = redisCache.properties.sslPort
output connectionString string = '${redisCache.properties.hostName}:${redisCache.properties.sslPort},password=${redisCache.listKeys().primaryKey},ssl=True,abortConnect=False'
```

---

## Geo-Replication

### Active Geo-Replication (Premium)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Active Geo-Replication (Premium)                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Active-Passive Replication:                                                │
│  • One primary (read/write), one or more secondaries (read-only)           │
│  • Asynchronous replication                                                │
│  • Manual failover required                                                │
│                                                                              │
│   Primary Region                     Secondary Region                       │
│   (East US)                         (West US)                              │
│  ┌──────────────────────┐          ┌──────────────────────┐               │
│  │                      │          │                      │               │
│  │   ┌──────────────┐   │   Async  │   ┌──────────────┐   │               │
│  │   │  Premium P1  │───┼──────────┼──▶│  Premium P1  │   │               │
│  │   │  (Primary)   │   │   Geo-   │   │ (Secondary)  │   │               │
│  │   │   R/W        │   │   Rep    │   │   Read-only  │   │               │
│  │   └──────────────┘   │          │   └──────────────┘   │               │
│  │                      │          │                      │               │
│  └──────────────────────┘          └──────────────────────┘               │
│                                                                              │
│  Failover Process:                                                          │
│  1. Unlink secondary from primary                                          │
│  2. Secondary becomes standalone primary                                   │
│  3. Update application connection strings                                  │
│  4. (Optional) Create new geo-replica link                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Active-Active Geo-Replication (Enterprise)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Active-Active Geo-Replication (Enterprise)                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  • All instances accept reads AND writes                                   │
│  • Conflict resolution (last-write-wins by default)                       │
│  • Near-real-time sync (< 1 second typical)                               │
│  • 99.999% SLA                                                             │
│                                                                              │
│   Region A                Region B                Region C                  │
│   (East US)              (West EU)              (SE Asia)                  │
│  ┌───────────────┐      ┌───────────────┐      ┌───────────────┐          │
│  │               │      │               │      │               │          │
│  │  Enterprise   │◀────▶│  Enterprise   │◀────▶│  Enterprise   │          │
│  │    Cache      │      │    Cache      │      │    Cache      │          │
│  │    (R/W)      │      │    (R/W)      │      │    (R/W)      │          │
│  │               │      │               │      │               │          │
│  └───────────────┘      └───────────────┘      └───────────────┘          │
│         ▲                      ▲                      ▲                    │
│         │                      │                      │                    │
│         └──────────────────────┴──────────────────────┘                    │
│                        CRDT-based sync                                      │
│                   (Conflict-free Replicated Data Types)                    │
│                                                                              │
│  Use Cases:                                                                 │
│  • Global session store                                                    │
│  • Multi-region applications                                               │
│  • Low-latency reads/writes worldwide                                      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### CLI: Geo-Replication Setup

```bash
# Create primary cache (Premium)
az redis create \
  --resource-group myRG \
  --name redis-primary \
  --location eastus \
  --sku Premium \
  --vm-size P1 \
  --shard-count 3

# Create secondary cache (same SKU required)
az redis create \
  --resource-group myRG \
  --name redis-secondary \
  --location westus \
  --sku Premium \
  --vm-size P1 \
  --shard-count 3

# Link secondary to primary (geo-replication)
az redis server-link create \
  --name redis-primary \
  --resource-group myRG \
  --replication-role Secondary \
  --server-to-link /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Cache/Redis/redis-secondary

# Check replication status
az redis server-link list \
  --name redis-primary \
  --resource-group myRG

# Force unlink (failover)
az redis server-link delete \
  --name redis-primary \
  --resource-group myRG \
  --linked-server-name redis-secondary

# Enterprise: Create with active geo-replication
az redisenterprise create \
  --name redis-enterprise \
  --resource-group myRG \
  --location eastus \
  --sku Enterprise_E10 \
  --zones 1 2 3

# Create database with geo-replication
az redisenterprise database create \
  --cluster-name redis-enterprise \
  --resource-group myRG \
  --database-name default \
  --grouping-policy "OnRedisEnterpriseClusterUpdate" \
  --linked-databases '[
    {"id": "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Cache/redisEnterprise/redis-enterprise-west/databases/default"},
    {"id": "/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Cache/redisEnterprise/redis-enterprise-asia/databases/default"}
  ]'
```

---

## Best Practices

### Connection Management

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Connection Best Practices                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CONNECTION POOLING                                                         │
│  ──────────────────                                                         │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │   BAD: New connection per request                                   │   │
│  │   ┌───────┐  ┌───────┐  ┌───────┐                                  │   │
│  │   │ Req 1 │──│ Conn  │──│ Redis │                                  │   │
│  │   └───────┘  └───────┘  └───────┘                                  │   │
│  │   ┌───────┐  ┌───────┐  ┌───────┐                                  │   │
│  │   │ Req 2 │──│ Conn  │──│ Redis │  (High overhead)                 │   │
│  │   └───────┘  └───────┘  └───────┘                                  │   │
│  │                                                                      │   │
│  │   GOOD: Connection multiplexing (StackExchange.Redis)              │   │
│  │   ┌───────┐  ┌─────────────────┐  ┌───────┐                        │   │
│  │   │ Req 1 │──│                 │  │       │                        │   │
│  │   └───────┘  │  Single Shared  │──│ Redis │                        │   │
│  │   ┌───────┐  │   Connection    │  │       │                        │   │
│  │   │ Req 2 │──│  (Multiplexed)  │  │       │                        │   │
│  │   └───────┘  └─────────────────┘  └───────┘                        │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  CONFIGURATION RECOMMENDATIONS                                              │
│  ─────────────────────────────                                              │
│                                                                              │
│  • Use SSL/TLS (port 6380)                                                 │
│  • Set abortConnect=false (retry on failure)                               │
│  • Configure connectTimeout (5000ms recommended)                           │
│  • Enable connection resilience (auto-reconnect)                           │
│  • Use singleton pattern for connection                                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### C# Connection Example

```csharp
using StackExchange.Redis;

public class RedisConnectionFactory
{
    private static readonly Lazy<ConnectionMultiplexer> LazyConnection =
        new Lazy<ConnectionMultiplexer>(() =>
        {
            var configOptions = new ConfigurationOptions
            {
                EndPoints = { "your-cache.redis.cache.windows.net:6380" },
                Password = Environment.GetEnvironmentVariable("REDIS_PASSWORD"),
                Ssl = true,
                AbortOnConnectFail = false,  // Don't throw on first failure
                ConnectTimeout = 5000,
                SyncTimeout = 5000,
                AsyncTimeout = 5000,
                ConnectRetry = 3,
                ReconnectRetryPolicy = new ExponentialRetry(5000),
                DefaultDatabase = 0
            };

            return ConnectionMultiplexer.Connect(configOptions);
        });

    public static ConnectionMultiplexer Connection => LazyConnection.Value;

    public static IDatabase GetDatabase() => Connection.GetDatabase();
}

// Usage
public class CacheService
{
    private readonly IDatabase _cache;

    public CacheService()
    {
        _cache = RedisConnectionFactory.GetDatabase();
    }

    public async Task<string?> GetAsync(string key)
    {
        return await _cache.StringGetAsync(key);
    }

    public async Task SetAsync(string key, string value, TimeSpan? expiry = null)
    {
        await _cache.StringSetAsync(key, value, expiry ?? TimeSpan.FromMinutes(10));
    }

    public async Task<T?> GetObjectAsync<T>(string key) where T : class
    {
        var value = await _cache.StringGetAsync(key);
        if (value.IsNullOrEmpty) return null;
        return JsonSerializer.Deserialize<T>(value!);
    }

    public async Task SetObjectAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var json = JsonSerializer.Serialize(value);
        await _cache.StringSetAsync(key, json, expiry ?? TimeSpan.FromMinutes(10));
    }
}
```

### Memory Management

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Memory Management Policies                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  EVICTION POLICIES (maxmemory-policy)                                       │
│  ────────────────────────────────────                                       │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Policy              │ Description                                   │   │
│  ├─────────────────────┼───────────────────────────────────────────────┤   │
│  │ volatile-lru        │ Evict LRU keys with TTL (recommended)        │   │
│  │ allkeys-lru         │ Evict LRU keys (any key)                     │   │
│  │ volatile-lfu        │ Evict LFU keys with TTL                      │   │
│  │ allkeys-lfu         │ Evict LFU keys (any key)                     │   │
│  │ volatile-ttl        │ Evict keys with shortest TTL                 │   │
│  │ volatile-random     │ Random eviction (keys with TTL)              │   │
│  │ allkeys-random      │ Random eviction (any key)                    │   │
│  │ noeviction          │ Return error on write (default)              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  MEMORY RESERVATION                                                         │
│  ──────────────────                                                         │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  Total Memory: 6 GB                                                 │   │
│  │  ┌────────────────────────────────────────────────────────────────┐ │   │
│  │  │                                                                │ │   │
│  │  │  maxmemory-reserved (125 MB)        ─┬─ Non-cache ops         │ │   │
│  │  │  maxfragmentationmemory-reserved     │  (replication,         │ │   │
│  │  │  (125 MB)                            │   failover, etc.)      │ │   │
│  │  │                                                                │ │   │
│  │  │  ─────────────────────────────────────────────────────────────│ │   │
│  │  │                                                                │ │   │
│  │  │  Available for data: ~5.75 GB                                 │ │   │
│  │  │                                                                │ │   │
│  │  └────────────────────────────────────────────────────────────────┘ │   │
│  │                                                                      │   │
│  │  Recommendation:                                                    │   │
│  │  • Standard/Premium: 10% for reserved memory                       │   │
│  │  • Heavy write load: 25% for fragmentation                        │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Data Patterns

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Common Redis Patterns                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CACHE-ASIDE                                                                │
│  ───────────                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  1. Check cache                                                     │   │
│  │  2. If miss, query database                                        │   │
│  │  3. Store in cache with TTL                                        │   │
│  │  4. Return data                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  SESSION STORE                                                              │
│  ─────────────                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Key: session:{sessionId}                                          │   │
│  │  Value: { userId, cart, preferences }                              │   │
│  │  TTL: Sliding expiration (e.g., 30 minutes)                       │   │
│  │                                                                      │   │
│  │  Commands: SETEX, EXPIRE, GET, DEL                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  RATE LIMITING                                                              │
│  ─────────────                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Key: ratelimit:{userId}:{minute}                                  │   │
│  │                                                                      │   │
│  │  INCR ratelimit:user123:202401151430                               │   │
│  │  EXPIRE ratelimit:user123:202401151430 60                          │   │
│  │  IF count > limit THEN reject                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  LEADERBOARD                                                                │
│  ───────────                                                                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Key: leaderboard:game1                                            │   │
│  │  Type: Sorted Set                                                   │   │
│  │                                                                      │   │
│  │  ZADD leaderboard:game1 1500 "player1"                             │   │
│  │  ZINCRBY leaderboard:game1 10 "player1"                            │   │
│  │  ZREVRANGE leaderboard:game1 0 9 WITHSCORES  (top 10)             │   │
│  │  ZRANK leaderboard:game1 "player1"  (player's rank)               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  PUB/SUB MESSAGING                                                          │
│  ─────────────────                                                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Channel: notifications:user123                                    │   │
│  │                                                                      │   │
│  │  Publisher: PUBLISH notifications:user123 "New message!"          │   │
│  │  Subscriber: SUBSCRIBE notifications:user123                       │   │
│  │                                                                      │   │
│  │  Use case: Real-time notifications, live updates                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Bicep Examples

### Standard Cache with Private Endpoint

```bicep
@description('Location for resources')
param location string = resourceGroup().location

// VNet for private connectivity
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'vnet-redis'
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
        name: 'private-endpoints'
        properties: {
          addressPrefix: '10.0.2.0/24'
        }
      }
    ]
  }
}

// Redis Cache (Standard)
resource redisCache 'Microsoft.Cache/redis@2023-08-01' = {
  name: 'redis-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    sku: {
      name: 'Standard'
      family: 'C'
      capacity: 1  // C1 = 1 GB
    }
    enableNonSslPort: false
    minimumTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'
    redisConfiguration: {
      'maxmemory-policy': 'volatile-lru'
    }
  }
}

// Private Endpoint
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'redis-pe'
  location: location
  properties: {
    subnet: {
      id: vnet.properties.subnets[1].id
    }
    privateLinkServiceConnections: [
      {
        name: 'redis-connection'
        properties: {
          privateLinkServiceId: redisCache.id
          groupIds: ['redisCache']
        }
      }
    ]
  }
}

// Private DNS Zone
resource privateDnsZone 'Microsoft.Network/privateDnsZones@2020-06-01' = {
  name: 'privatelink.redis.cache.windows.net'
  location: 'global'
}

resource dnsZoneLink 'Microsoft.Network/privateDnsZones/virtualNetworkLinks@2020-06-01' = {
  parent: privateDnsZone
  name: 'redis-dns-link'
  location: 'global'
  properties: {
    registrationEnabled: false
    virtualNetwork: {
      id: vnet.id
    }
  }
}

resource dnsRecord 'Microsoft.Network/privateDnsZones/A@2020-06-01' = {
  parent: privateDnsZone
  name: redisCache.name
  properties: {
    ttl: 300
    aRecords: [
      {
        ipv4Address: privateEndpoint.properties.customDnsConfigs[0].ipAddresses[0]
      }
    ]
  }
}

output redisHostName string = redisCache.properties.hostName
output redisPort int = redisCache.properties.sslPort
```

### Enterprise Cache with Modules

```bicep
@description('Location for resources')
param location string = resourceGroup().location

// Enterprise Redis cluster
resource enterpriseCluster 'Microsoft.Cache/redisEnterprise@2023-11-01' = {
  name: 'redis-enterprise-${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Enterprise_E10'
    capacity: 2  // Number of nodes
  }
  zones: ['1', '2', '3']
  properties: {
    minimumTlsVersion: '1.2'
  }
}

// Database with modules
resource enterpriseDatabase 'Microsoft.Cache/redisEnterprise/databases@2023-11-01' = {
  parent: enterpriseCluster
  name: 'default'
  properties: {
    clientProtocol: 'Encrypted'
    port: 10000
    clusteringPolicy: 'EnterpriseCluster'
    evictionPolicy: 'VolatileLRU'
    persistence: {
      aofEnabled: true
      aofFrequency: 'Always'
      rdbEnabled: true
      rdbFrequency: '12h'
    }
    modules: [
      {
        name: 'RediSearch'  // Full-text search
      }
      {
        name: 'RedisJSON'   // JSON document support
      }
      {
        name: 'RedisBloom'  // Probabilistic data structures
      }
    ]
  }
}

output endpoint string = enterpriseDatabase.properties.hostName
output port int = enterpriseDatabase.properties.port
```

---

## Monitoring and Troubleshooting

### Key Metrics

```bash
# View cache metrics
az monitor metrics list \
  --resource $(az redis show -n myredis -g myRG --query id -o tsv) \
  --metric "usedmemory,connectedclients,cacheHits,cacheMisses,serverLoad" \
  --interval PT5M \
  --output table

# Cache hit ratio
az monitor metrics list \
  --resource $(az redis show -n myredis -g myRG --query id -o tsv) \
  --metric "cacheHits,cacheMisses" \
  --interval PT1H \
  --output table

# Check for connection issues
az redis list-keys \
  --resource-group myRG \
  --name myredis

# Test connectivity
redis-cli -h myredis.redis.cache.windows.net -p 6380 \
  -a "<access-key>" --tls PING
```

### Common Issues

| Issue | Symptoms | Solution |
|-------|----------|----------|
| High server load | Slow responses, timeouts | Scale up or add shards |
| Memory pressure | Evictions, OOM errors | Increase cache size, review TTL |
| Connection issues | Timeouts, connection refused | Check firewall, use connection pooling |
| High latency | Slow operations | Use pipelining, check network |
| Cache misses | Low hit ratio | Review key expiration, cache strategy |

### Diagnostic Commands

```bash
# Redis INFO (via redis-cli)
redis-cli -h myredis.redis.cache.windows.net -p 6380 \
  -a "<key>" --tls INFO

# Memory usage
redis-cli -h myredis.redis.cache.windows.net -p 6380 \
  -a "<key>" --tls MEMORY STATS

# Slow log
redis-cli -h myredis.redis.cache.windows.net -p 6380 \
  -a "<key>" --tls SLOWLOG GET 10

# Client list
redis-cli -h myredis.redis.cache.windows.net -p 6380 \
  -a "<key>" --tls CLIENT LIST

# Key statistics
redis-cli -h myredis.redis.cache.windows.net -p 6380 \
  -a "<key>" --tls DBSIZE
```

---

*Previous: [PostgreSQL & MySQL](03-postgresql-mysql.md)* | *Back to [Databases Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
