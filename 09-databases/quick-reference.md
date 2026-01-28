# Quick Reference: Databases

## Azure SQL CLI Commands

### Create SQL Server and Database

```bash
# Create logical server
az sql server create \
  --resource-group myRG \
  --name my-sql-server \
  --location eastus \
  --admin-user sqladmin \
  --admin-password 'SecureP@ss123!'

# Create database
az sql db create \
  --resource-group myRG \
  --server my-sql-server \
  --name mydb \
  --service-objective S0 \
  --backup-storage-redundancy Local

# Create with vCore model
az sql db create \
  --resource-group myRG \
  --server my-sql-server \
  --name mydb-vcore \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2 \
  --compute-model Provisioned
```

### Elastic Pool

```bash
# Create elastic pool
az sql elastic-pool create \
  --resource-group myRG \
  --server my-sql-server \
  --name mypool \
  --edition Standard \
  --dtu 100 \
  --db-max-dtu 20 \
  --db-min-dtu 0

# Add database to pool
az sql db update \
  --resource-group myRG \
  --server my-sql-server \
  --name mydb \
  --elastic-pool mypool
```

### Failover Group

```bash
# Create failover group
az sql failover-group create \
  --resource-group myRG \
  --server my-sql-server \
  --name my-failover-group \
  --partner-server my-sql-server-secondary \
  --partner-resource-group myRG-secondary \
  --failover-policy Automatic \
  --grace-period 1

# Add database
az sql failover-group update \
  --resource-group myRG \
  --server my-sql-server \
  --name my-failover-group \
  --add-db mydb
```

## Cosmos DB CLI Commands

### Create Account and Database

```bash
# Create Cosmos DB account
az cosmosdb create \
  --resource-group myRG \
  --name my-cosmos-account \
  --kind GlobalDocumentDB \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1 \
  --default-consistency-level Session \
  --enable-automatic-failover true

# Create database
az cosmosdb sql database create \
  --resource-group myRG \
  --account-name my-cosmos-account \
  --name mydb \
  --throughput 400

# Create container
az cosmosdb sql container create \
  --resource-group myRG \
  --account-name my-cosmos-account \
  --database-name mydb \
  --name mycontainer \
  --partition-key-path "/customerId" \
  --throughput 400
```

### Serverless

```bash
# Create serverless account
az cosmosdb create \
  --resource-group myRG \
  --name my-cosmos-serverless \
  --capabilities EnableServerless \
  --locations regionName=eastus
```

## PostgreSQL CLI Commands

```bash
# Create Flexible Server
az postgres flexible-server create \
  --resource-group myRG \
  --name my-postgres \
  --location eastus \
  --admin-user pgadmin \
  --admin-password 'SecureP@ss123!' \
  --sku-name Standard_D2s_v3 \
  --storage-size 128 \
  --version 15

# Create database
az postgres flexible-server db create \
  --resource-group myRG \
  --server-name my-postgres \
  --database-name mydb

# Configure firewall
az postgres flexible-server firewall-rule create \
  --resource-group myRG \
  --name my-postgres \
  --rule-name AllowMyIP \
  --start-ip-address 203.0.113.1 \
  --end-ip-address 203.0.113.1
```

## Redis Cache CLI Commands

```bash
# Create Redis Cache
az redis create \
  --resource-group myRG \
  --name my-redis \
  --location eastus \
  --sku Premium \
  --vm-size P1 \
  --enable-non-ssl-port false

# Get connection string
az redis list-keys \
  --resource-group myRG \
  --name my-redis

# Create with clustering
az redis create \
  --resource-group myRG \
  --name my-redis-cluster \
  --location eastus \
  --sku Premium \
  --vm-size P1 \
  --shard-count 3
```

## Connection Strings

### Azure SQL

```
# SQL Authentication
Server=tcp:{server}.database.windows.net,1433;Initial Catalog={database};Persist Security Info=False;User ID={username};Password={password};MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;

# Azure AD Authentication
Server=tcp:{server}.database.windows.net,1433;Initial Catalog={database};Authentication=Active Directory Default;Encrypt=True;
```

### Cosmos DB

```
# Connection string
AccountEndpoint=https://{account}.documents.azure.com:443/;AccountKey={key};

# With specific database
AccountEndpoint=https://{account}.documents.azure.com:443/;AccountKey={key};Database={database};
```

### PostgreSQL

```
# Standard connection
Host={server}.postgres.database.azure.com;Database={database};Port=5432;User Id={username}@{server};Password={password};SSL Mode=Require;
```

### Redis

```
# Redis connection string
{name}.redis.cache.windows.net:6380,password={key},ssl=True,abortConnect=False
```

## Cosmos DB Code Snippets

### C# SDK

```csharp
using Microsoft.Azure.Cosmos;

// Initialize client
var client = new CosmosClient(connectionString, new CosmosClientOptions
{
    ApplicationRegion = Regions.EastUS,
    ConsistencyLevel = ConsistencyLevel.Session
});

// Get container
var container = client.GetContainer("mydb", "mycontainer");

// Create item
var item = new { id = Guid.NewGuid().ToString(), customerId = "C123", name = "Product A" };
await container.CreateItemAsync(item, new PartitionKey(item.customerId));

// Query items
var query = container.GetItemQueryIterator<dynamic>(
    "SELECT * FROM c WHERE c.customerId = 'C123'");

while (query.HasMoreResults)
{
    var response = await query.ReadNextAsync();
    foreach (var item in response)
    {
        Console.WriteLine(item);
    }
}

// Point read (most efficient)
var response = await container.ReadItemAsync<dynamic>("item-id", new PartitionKey("C123"));
```

### Azure SQL with Entity Framework

```csharp
// DbContext configuration
public class AppDbContext : DbContext
{
    protected override void OnConfiguring(DbContextOptionsBuilder options)
    {
        options.UseSqlServer(connectionString, sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 5,
                maxRetryDelay: TimeSpan.FromSeconds(30),
                errorNumbersToAdd: null);
        });
    }
}

// Using Azure AD authentication
services.AddDbContext<AppDbContext>(options =>
{
    var connection = new SqlConnection(connectionString);
    connection.AccessToken = await new DefaultAzureCredential()
        .GetTokenAsync(new TokenRequestContext(new[] { "https://database.windows.net/.default" }));
    options.UseSqlServer(connection);
});
```

## Bicep Snippets

### Azure SQL

```bicep
resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: 'sql-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlPassword
    minimalTlsVersion: '1.2'
    publicNetworkAccess: 'Disabled'
  }
}

resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'mydb'
  location: location
  sku: {
    name: 'GP_Gen5'
    tier: 'GeneralPurpose'
    family: 'Gen5'
    capacity: 2
  }
  properties: {
    zoneRedundant: true
    readScale: 'Enabled'
    requestedBackupStorageRedundancy: 'Geo'
  }
}
```

### Cosmos DB

```bicep
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' = {
  name: 'cosmos-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    databaseAccountOfferType: 'Standard'
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
    locations: [
      {
        locationName: location
        failoverPriority: 0
        isZoneRedundant: true
      }
      {
        locationName: secondaryLocation
        failoverPriority: 1
        isZoneRedundant: true
      }
    ]
    enableAutomaticFailover: true
    enableMultipleWriteLocations: false
  }
}

resource cosmosDatabase 'Microsoft.DocumentDB/databaseAccounts/sqlDatabases@2023-04-15' = {
  parent: cosmosAccount
  name: 'mydb'
  properties: {
    resource: {
      id: 'mydb'
    }
    options: {
      autoscaleSettings: {
        maxThroughput: 4000
      }
    }
  }
}
```

## Performance Tuning

### Azure SQL Query Hints

```sql
-- Force index
SELECT * FROM Orders WITH (INDEX(IX_CustomerID))
WHERE CustomerID = 'C123';

-- Optimize for unknown
SELECT * FROM Orders
WHERE CustomerID = @CustomerID
OPTION (OPTIMIZE FOR UNKNOWN);

-- Enable Query Store
ALTER DATABASE mydb SET QUERY_STORE = ON;

-- View top resource-consuming queries
SELECT TOP 10
    qs.query_id,
    qt.query_sql_text,
    rs.avg_duration,
    rs.avg_cpu_time,
    rs.avg_logical_io_reads
FROM sys.query_store_query qs
JOIN sys.query_store_query_text qt ON qs.query_text_id = qt.query_text_id
JOIN sys.query_store_runtime_stats rs ON qs.query_id = rs.query_id
ORDER BY rs.avg_cpu_time DESC;
```

### Cosmos DB Indexing

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    {
      "path": "/customerId/*"
    },
    {
      "path": "/orderDate/*"
    }
  ],
  "excludedPaths": [
    {
      "path": "/largePayload/*"
    },
    {
      "path": "/*"
    }
  ],
  "compositeIndexes": [
    [
      { "path": "/customerId", "order": "ascending" },
      { "path": "/orderDate", "order": "descending" }
    ]
  ]
}
```

---

*Back to [Databases Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
