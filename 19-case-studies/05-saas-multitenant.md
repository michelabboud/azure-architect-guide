# Case Study 5: SaaS Multi-Tenant Architecture

## Executive Summary

**Company**: CloudHR Solutions
**Industry**: HR Technology / SaaS
**Challenge**: Build scalable B2B SaaS platform serving enterprises of varying sizes
**Outcome**: 2,000+ tenants, 5M users, 99.95% uptime, $50M ARR

### The Business Context

CloudHR provides HR management software to businesses ranging from 50-employee startups to 100,000-employee enterprises. The platform needed to:

1. Support multi-tenant isolation with enterprise security requirements
2. Scale from small tenants to enterprise customers (1000x data variation)
3. Enable tenant-specific customizations and integrations
4. Provide predictable costs that align with pricing tiers

## Architecture Overview

```
                                   ┌──────────────────────────────────────┐
                                   │            Tenants                    │
                    ┌──────────────┤  Small (1.5K)  │  Medium (400)       │
                    │              │  Large (90)    │  Enterprise (10)    │
                    │              └──────────────────────────────────────┘
                    │
                    ▼
    ┌───────────────────────────────────────────────────────────────────────────┐
    │                         Access Layer                                       │
    │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐            │
    │  │ Azure Front Door │  │ Azure AD B2C    │  │ API Management  │            │
    │  │ (WAF + Routing)  │  │ (Tenant IdP)    │  │ (Rate Limiting) │            │
    │  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘            │
    └───────────┼────────────────────┼────────────────────┼─────────────────────┘
                │                    │                    │
                └────────────────────┼────────────────────┘
                                     │
    ┌────────────────────────────────▼──────────────────────────────────────────┐
    │                        Application Layer                                   │
    │                                                                            │
    │  ┌──────────────────────────────────────────────────────────────────┐     │
    │  │                    Azure Kubernetes Service                       │     │
    │  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐ │     │
    │  │  │  Web App   │  │  Worker    │  │  API       │  │ Background │ │     │
    │  │  │  (React)   │  │  Service   │  │  Gateway   │  │  Jobs      │ │     │
    │  │  └────────────┘  └────────────┘  └────────────┘  └────────────┘ │     │
    │  └──────────────────────────────────────────────────────────────────┘     │
    │                                                                            │
    │  Tenant Context Middleware ──────────────────────────────────────────────▶│
    │                                                                            │
    └────────────────────────────────┬──────────────────────────────────────────┘
                                     │
    ┌────────────────────────────────▼──────────────────────────────────────────┐
    │                         Data Layer                                         │
    │                                                                            │
    │  ┌──────────────────────┐  ┌──────────────────────┐  ┌─────────────────┐  │
    │  │    Pool Model        │  │   Database per       │  │   Dedicated     │  │
    │  │    (Small Tenants)   │  │   Tenant (Large)     │  │   (Enterprise)  │  │
    │  │                      │  │                      │  │                 │  │
    │  │  ┌────────────────┐  │  │  ┌────────────────┐  │  │  ┌───────────┐  │  │
    │  │  │  SQL Elastic   │  │  │  │  SQL Database  │  │  │  │  SQL MI   │  │  │
    │  │  │  Pool (S3)     │  │  │  │  (Standard)    │  │  │  │  (BC)     │  │  │
    │  │  │  1500 tenants  │  │  │  │  per tenant    │  │  │  │           │  │  │
    │  │  └────────────────┘  │  │  └────────────────┘  │  │  └───────────┘  │  │
    │  └──────────────────────┘  └──────────────────────┘  └─────────────────┘  │
    │                                                                            │
    └────────────────────────────────────────────────────────────────────────────┘
```

## AWS to Azure Service Mapping

| Component | AWS Equivalent | Azure Implementation | Multi-Tenant Feature |
|-----------|---------------|---------------------|---------------------|
| Identity | Cognito User Pools | Azure AD B2C | Custom policies per tenant |
| API Gateway | API Gateway + WAF | APIM + Front Door | Tenant-based rate limiting |
| Compute | EKS | AKS | Namespace isolation |
| Database | Aurora + RDS | SQL Elastic Pools | Pool per tier |
| Cache | ElastiCache | Azure Cache for Redis | Tenant-prefixed keys |
| Storage | S3 | Blob Storage | Container per tenant |
| Search | OpenSearch | Cognitive Search | Index per tenant |
| Background Jobs | SQS + Lambda | Service Bus + Functions | Tenant priority queues |

## Key Architectural Decisions

### Decision 1: Tenant Isolation Model

**The Debate:**

| Model | Isolation | Cost Efficiency | Complexity | Use Case |
|-------|-----------|----------------|------------|----------|
| **Shared Everything** | Low | High | Low | Consumer apps |
| **Shared App, Separate DB** | High | Medium | Medium | B2B SaaS |
| **Separate App + DB** | Very High | Low | High | Enterprise only |
| **Hybrid** | Variable | Optimized | High | Mixed customer base |

**Decision: Hybrid model with tiered isolation**

**Rationale:**
1. Small tenants: Shared elastic pool (cost-effective)
2. Large tenants: Dedicated databases (performance isolation)
3. Enterprise tenants: Dedicated SQL MI (compliance, network isolation)
4. All tiers share compute (AKS) with namespace isolation

**Tenant Configuration:**
```csharp
public enum TenantTier
{
    Starter,    // Shared elastic pool, shared Redis prefix
    Growth,     // Shared elastic pool, dedicated Redis database
    Business,   // Dedicated SQL database, dedicated Redis database
    Enterprise  // Dedicated SQL MI, dedicated Redis instance, VNet integration
}

public class TenantConfiguration
{
    public string TenantId { get; set; }
    public TenantTier Tier { get; set; }
    public string DatabaseConnectionString { get; set; }
    public string RedisCacheConfiguration { get; set; }
    public string BlobContainerName { get; set; }
    public bool VNetIntegrated { get; set; }
    public Dictionary<string, string> CustomSettings { get; set; }
}

public class TenantResolver : ITenantResolver
{
    private readonly IMemoryCache _cache;
    private readonly CosmosContainer _tenantCatalog;

    public async Task<TenantConfiguration> ResolveAsync(string tenantId)
    {
        // Check cache first (TTL: 5 minutes)
        if (_cache.TryGetValue($"tenant:{tenantId}", out TenantConfiguration config))
            return config;

        // Fetch from catalog
        config = await _tenantCatalog.ReadItemAsync<TenantConfiguration>(
            tenantId,
            new PartitionKey(tenantId));

        _cache.Set($"tenant:{tenantId}", config, TimeSpan.FromMinutes(5));
        return config;
    }
}
```

### Decision 2: Database Scaling Strategy

**The Debate:**

| Strategy | Pros | Cons |
|----------|------|------|
| **Single DB, Row-Level Security** | Simple, cost-effective | Noisy neighbor, limited scale |
| **Schema per Tenant** | Good isolation, shared resources | Complex migrations |
| **Database per Tenant** | Full isolation, independent scaling | Expensive, connection limits |
| **Elastic Pools** | Balanced isolation/cost | Pool sizing complexity |

**Decision: Elastic Pools for small/medium tenants, dedicated databases for large**

**Rationale:**
1. Elastic pools share resources efficiently for variable workloads
2. Automatic tenant routing based on tier
3. Easy promotion: move tenant from pool to dedicated
4. Row-level security within shared pools for additional isolation

**Elastic Pool Configuration:**
```bicep
// Elastic pool for small tenants
resource starterPool 'Microsoft.Sql/servers/elasticPools@2023-05-01-preview' = {
  parent: sqlServer
  name: 'starter-pool'
  location: location
  sku: {
    name: 'StandardPool'
    tier: 'Standard'
    capacity: 200  // 200 eDTUs
  }
  properties: {
    perDatabaseSettings: {
      minCapacity: 0
      maxCapacity: 20
    }
    maxSizeBytes: 500 * 1024 * 1024 * 1024  // 500 GB
  }
}

// Template for tenant database in pool
resource tenantDatabase 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'tenant-${tenantId}'
  location: location
  sku: {
    name: 'ElasticPool'
  }
  properties: {
    elasticPoolId: starterPool.id
    collation: 'SQL_Latin1_General_CP1_CI_AS'
  }
}
```

**Row-Level Security for Shared Pools:**
```sql
-- Row-Level Security Policy
CREATE SCHEMA Security;
GO

CREATE FUNCTION Security.fn_tenantAccessPredicate(@TenantId NVARCHAR(50))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS fn_accessResult
WHERE @TenantId = SESSION_CONTEXT(N'TenantId');
GO

-- Apply to all tenant tables
CREATE SECURITY POLICY TenantSecurityPolicy
ADD FILTER PREDICATE Security.fn_tenantAccessPredicate(TenantId) ON dbo.Employees,
ADD FILTER PREDICATE Security.fn_tenantAccessPredicate(TenantId) ON dbo.TimeSheets,
ADD FILTER PREDICATE Security.fn_tenantAccessPredicate(TenantId) ON dbo.PayrollRecords,
ADD BLOCK PREDICATE Security.fn_tenantAccessPredicate(TenantId) ON dbo.Employees,
ADD BLOCK PREDICATE Security.fn_tenantAccessPredicate(TenantId) ON dbo.TimeSheets,
ADD BLOCK PREDICATE Security.fn_tenantAccessPredicate(TenantId) ON dbo.PayrollRecords
WITH (STATE = ON);
GO
```

### Decision 3: API Rate Limiting and Quotas

**The Debate:**

| Approach | Granularity | Flexibility | Complexity |
|----------|-------------|-------------|------------|
| **Fixed limits per tier** | Low | Low | Low |
| **Dynamic based on usage** | High | High | High |
| **Token bucket with bursting** | Medium | Medium | Medium |
| **APIM policies** | High | High | Low |

**Decision: APIM policies with tier-based quotas and bursting**

**Rationale:**
1. APIM provides built-in rate limiting without custom code
2. Different quotas per tier align with pricing model
3. Burst allowance handles legitimate traffic spikes
4. Per-endpoint granularity for resource-intensive operations

**APIM Policy:**
```xml
<policies>
    <inbound>
        <!-- Extract tenant ID from JWT -->
        <set-variable name="tenantId"
            value="@(context.Request.Headers.GetValueOrDefault("X-Tenant-ID",
                context.User?.GetClaimValue("tenant_id")))" />

        <!-- Get tenant tier from cache or backend -->
        <cache-lookup-value key="@("tier:" + context.Variables["tenantId"])"
                           variable-name="tenantTier" />

        <choose>
            <when condition="@(context.Variables["tenantTier"] == null)">
                <send-request mode="new" response-variable-name="tierResponse">
                    <set-url>@("https://tenant-service/api/tenants/" +
                              context.Variables["tenantId"] + "/tier")</set-url>
                </send-request>
                <set-variable name="tenantTier"
                    value="@(((IResponse)context.Variables["tierResponse"]).Body.As<string>())" />
                <cache-store-value key="@("tier:" + context.Variables["tenantId"])"
                                   value="@(context.Variables["tenantTier"])"
                                   duration="300" />
            </when>
        </choose>

        <!-- Apply tier-based rate limits -->
        <choose>
            <when condition="@(context.Variables["tenantTier"] == "Starter")">
                <rate-limit-by-key calls="100" renewal-period="60"
                    counter-key="@(context.Variables["tenantId"])" />
                <quota-by-key calls="10000" renewal-period="86400"
                    counter-key="@(context.Variables["tenantId"])" />
            </when>
            <when condition="@(context.Variables["tenantTier"] == "Growth")">
                <rate-limit-by-key calls="500" renewal-period="60"
                    counter-key="@(context.Variables["tenantId"])" />
                <quota-by-key calls="100000" renewal-period="86400"
                    counter-key="@(context.Variables["tenantId"])" />
            </when>
            <when condition="@(context.Variables["tenantTier"] == "Business")">
                <rate-limit-by-key calls="2000" renewal-period="60"
                    counter-key="@(context.Variables["tenantId"])" />
                <quota-by-key calls="1000000" renewal-period="86400"
                    counter-key="@(context.Variables["tenantId"])" />
            </when>
            <when condition="@(context.Variables["tenantTier"] == "Enterprise")">
                <!-- Enterprise: Higher limits, separate tracking -->
                <rate-limit-by-key calls="10000" renewal-period="60"
                    counter-key="@(context.Variables["tenantId"])" />
                <!-- No daily quota for enterprise -->
            </when>
        </choose>

        <!-- Set tenant context for backend -->
        <set-header name="X-Tenant-ID" exists-action="override">
            <value>@(context.Variables["tenantId"].ToString())</value>
        </set-header>
    </inbound>
</policies>
```

## Technical Deep Dive

### Tenant Onboarding Workflow

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Tenant Onboarding Flow                                │
└─────────────────────────────────────────────────────────────────────────────┘

  Customer Signs Up                 Provisioning                    Ready
       │                                 │                            │
       ▼                                 ▼                            ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 1. Create    │───▶│ 2. Create    │───▶│ 3. Provision │───▶│ 4. Configure │
│   Azure AD   │    │   Tenant     │    │   Resources  │    │   & Notify   │
│   B2C Tenant │    │   Record     │    │              │    │              │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
       │                   │                   │                    │
       ▼                   ▼                   ▼                    ▼
  • Custom domain     • Cosmos DB        • Database           • Welcome email
  • Branding          • Feature flags    • Blob container     • Admin setup
  • User flows        • Tier config      • Redis namespace    • Sample data
```

**Durable Function for Provisioning:**
```csharp
[FunctionName("TenantProvisioningOrchestrator")]
public static async Task<TenantProvisioningResult> RunOrchestrator(
    [OrchestrationTrigger] IDurableOrchestrationContext context)
{
    var request = context.GetInput<TenantProvisioningRequest>();
    var result = new TenantProvisioningResult { TenantId = request.TenantId };

    try
    {
        // Step 1: Create tenant record in catalog
        var tenantRecord = await context.CallActivityAsync<TenantRecord>(
            "CreateTenantRecord",
            request);
        result.CatalogCreated = true;

        // Step 2: Provision Azure AD B2C (parallel with database)
        var b2cTask = context.CallActivityAsync<B2CResult>(
            "ProvisionB2CTenant",
            new B2CRequest
            {
                TenantId = request.TenantId,
                DisplayName = request.CompanyName,
                CustomDomain = request.CustomDomain
            });

        // Step 3: Provision database based on tier
        var dbTask = context.CallActivityAsync<DatabaseResult>(
            "ProvisionDatabase",
            new DatabaseRequest
            {
                TenantId = request.TenantId,
                Tier = request.Tier,
                Region = request.Region
            });

        // Step 4: Provision blob storage
        var storageTask = context.CallActivityAsync<StorageResult>(
            "ProvisionBlobStorage",
            new StorageRequest
            {
                TenantId = request.TenantId,
                Tier = request.Tier
            });

        // Wait for all parallel tasks
        await Task.WhenAll(b2cTask, dbTask, storageTask);

        result.B2CProvisioned = b2cTask.Result.Success;
        result.DatabaseProvisioned = dbTask.Result.Success;
        result.StorageProvisioned = storageTask.Result.Success;

        // Step 5: Run database migrations
        await context.CallActivityAsync(
            "RunDatabaseMigrations",
            new MigrationRequest
            {
                TenantId = request.TenantId,
                ConnectionString = dbTask.Result.ConnectionString
            });
        result.MigrationsComplete = true;

        // Step 6: Seed initial data
        if (request.SeedSampleData)
        {
            await context.CallActivityAsync(
                "SeedSampleData",
                new SeedRequest { TenantId = request.TenantId });
        }

        // Step 7: Update tenant status to Active
        await context.CallActivityAsync(
            "ActivateTenant",
            request.TenantId);
        result.Status = "Active";

        // Step 8: Send welcome notification
        await context.CallActivityAsync(
            "SendWelcomeEmail",
            new EmailRequest
            {
                To = request.AdminEmail,
                TenantId = request.TenantId,
                LoginUrl = b2cTask.Result.LoginUrl
            });

        return result;
    }
    catch (Exception ex)
    {
        // Compensation: rollback partially created resources
        await context.CallActivityAsync(
            "RollbackProvisioning",
            new RollbackRequest
            {
                TenantId = request.TenantId,
                Result = result
            });

        throw;
    }
}
```

### Multi-Tenant Middleware

```csharp
public class TenantMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ITenantResolver _tenantResolver;
    private readonly ILogger<TenantMiddleware> _logger;

    public TenantMiddleware(
        RequestDelegate next,
        ITenantResolver tenantResolver,
        ILogger<TenantMiddleware> logger)
    {
        _next = next;
        _tenantResolver = tenantResolver;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // Extract tenant identifier
        var tenantId = ExtractTenantId(context);

        if (string.IsNullOrEmpty(tenantId))
        {
            context.Response.StatusCode = 400;
            await context.Response.WriteAsJsonAsync(new
            {
                error = "Tenant identifier required"
            });
            return;
        }

        try
        {
            // Resolve tenant configuration
            var tenantConfig = await _tenantResolver.ResolveAsync(tenantId);

            if (tenantConfig == null)
            {
                context.Response.StatusCode = 404;
                await context.Response.WriteAsJsonAsync(new
                {
                    error = "Tenant not found"
                });
                return;
            }

            // Check tenant status
            if (tenantConfig.Status != TenantStatus.Active)
            {
                context.Response.StatusCode = 403;
                await context.Response.WriteAsJsonAsync(new
                {
                    error = $"Tenant is {tenantConfig.Status}"
                });
                return;
            }

            // Set tenant context for the request
            context.Items["TenantId"] = tenantId;
            context.Items["TenantConfig"] = tenantConfig;

            // Set database connection for this request
            var dbContext = context.RequestServices
                .GetRequiredService<ApplicationDbContext>();
            dbContext.SetTenantConnection(tenantConfig.DatabaseConnectionString);

            // Set session context for row-level security
            await dbContext.Database.ExecuteSqlRawAsync(
                "EXEC sp_set_session_context @key=N'TenantId', @value=@tenantId",
                new SqlParameter("@tenantId", tenantId));

            // Add tenant to logging scope
            using (_logger.BeginScope(new Dictionary<string, object>
            {
                ["TenantId"] = tenantId,
                ["TenantTier"] = tenantConfig.Tier
            }))
            {
                await _next(context);
            }
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error processing request for tenant {TenantId}", tenantId);
            throw;
        }
    }

    private string ExtractTenantId(HttpContext context)
    {
        // Priority 1: Header (set by APIM or internal services)
        if (context.Request.Headers.TryGetValue("X-Tenant-ID", out var headerValue))
            return headerValue.ToString();

        // Priority 2: JWT claim
        var tenantClaim = context.User?.FindFirst("tenant_id");
        if (tenantClaim != null)
            return tenantClaim.Value;

        // Priority 3: Subdomain (e.g., acme.cloudhr.com)
        var host = context.Request.Host.Host;
        var subdomain = host.Split('.').FirstOrDefault();
        if (subdomain != null && subdomain != "www" && subdomain != "app")
            return subdomain;

        return null;
    }
}
```

## Cost Analysis

### Monthly Cost by Tenant Tier

| Component | Starter (1.5K tenants) | Growth (400 tenants) | Business (90 tenants) | Enterprise (10 tenants) |
|-----------|----------------------|---------------------|----------------------|------------------------|
| Database | $2,000 (elastic pool) | $4,000 (larger pool) | $18,000 (dedicated) | $25,000 (SQL MI) |
| Storage | $500 (shared) | $1,000 (dedicated containers) | $2,000 | $3,000 |
| Compute (AKS share) | $5,000 | $8,000 | $10,000 | $15,000 |
| Redis | $500 (shared) | $1,000 (db/tenant) | $3,000 (dedicated) | $5,000 (instance) |
| **Total** | **$8,000** | **$14,000** | **$33,000** | **$48,000** |
| **Per tenant** | **$5.33** | **$35** | **$367** | **$4,800** |

### Revenue vs Cost Analysis

| Tier | Price/Month | Tenants | Revenue | Cost | Margin |
|------|-------------|---------|---------|------|--------|
| Starter | $49 | 1,500 | $73,500 | $8,000 | 89% |
| Growth | $199 | 400 | $79,600 | $14,000 | 82% |
| Business | $999 | 90 | $89,910 | $33,000 | 63% |
| Enterprise | $4,999 | 10 | $49,990 | $48,000 | 4% |
| **Total** | | **2,000** | **$293,000** | **$103,000** | **65%** |

*Note: Enterprise tier includes dedicated support and custom development*

## Lessons Learned

### What Worked Well

1. **Elastic Pools**: Efficient resource sharing for small tenants
2. **Durable Functions**: Reliable tenant provisioning with compensation
3. **APIM Rate Limiting**: No custom code for quota management

### Challenges Encountered

1. **Database Migrations**: Coordinating schema changes across 2,000 databases
2. **Noisy Neighbor Detection**: Needed custom monitoring for shared resources
3. **Feature Flag Management**: Complex with per-tenant overrides

### Recommendations

1. **Start with shared resources**: Easier to isolate later than consolidate
2. **Automate everything**: Provisioning, migrations, scaling decisions
3. **Meter everything**: Usage data drives pricing and optimization

---

*Back to [Chapter 08: Case Studies](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
