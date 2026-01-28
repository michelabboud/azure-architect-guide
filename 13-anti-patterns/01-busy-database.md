# Anti-Pattern: Busy Database

## Overview

The Busy Database anti-pattern occurs when too much processing is offloaded to the database server instead of the application tier. This pattern seems efficient on paper—let the database handle complex operations close to the data—but it creates a critical bottleneck that limits scalability.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         BUSY DATABASE ANTI-PATTERN                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Application Tier (scalable)                                               │
│   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐                          │
│   │ App 1   │ │ App 2   │ │ App 3   │ │ App N   │                          │
│   │ (idle)  │ │ (idle)  │ │ (idle)  │ │ (idle)  │                          │
│   └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘                          │
│        │           │           │           │                                 │
│        └───────────┴─────┬─────┴───────────┘                                │
│                          │                                                   │
│                          ▼                                                   │
│   ┌──────────────────────────────────────────────────────────┐              │
│   │                    DATABASE                               │              │
│   │  ┌─────────────────────────────────────────────────────┐ │              │
│   │  │  Stored Procedures doing:                            │ │              │
│   │  │  • Complex calculations                              │ │              │
│   │  │  • Data transformation                               │ │              │
│   │  │  • Business logic                                    │ │              │
│   │  │  • Report generation                                 │ │              │
│   │  │  • XML/JSON processing                               │ │              │
│   │  └─────────────────────────────────────────────────────┘ │              │
│   │                                                          │              │
│   │  CPU: 95%  │  Memory: 90%  │  I/O: SATURATED            │              │
│   └──────────────────────────────────────────────────────────┘              │
│                                                                              │
│   ❌ Database becomes the bottleneck                                        │
│   ❌ Cannot scale independently                                             │
│   ❌ All apps affected when DB is overloaded                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. Complex Stored Procedures

```sql
-- ANTI-PATTERN: Business logic in stored procedure
CREATE PROCEDURE CalculateMonthlyReport
    @StartDate DATE,
    @EndDate DATE
AS
BEGIN
    -- Complex aggregations
    SELECT
        c.CustomerName,
        SUM(o.TotalAmount) as Revenue,
        COUNT(DISTINCT o.OrderId) as OrderCount,
        AVG(o.TotalAmount) as AvgOrderValue,
        -- Complex business rules
        CASE
            WHEN SUM(o.TotalAmount) > 100000 THEN 'Platinum'
            WHEN SUM(o.TotalAmount) > 50000 THEN 'Gold'
            WHEN SUM(o.TotalAmount) > 10000 THEN 'Silver'
            ELSE 'Bronze'
        END as CustomerTier,
        -- String manipulation
        STUFF((
            SELECT ', ' + p.ProductName
            FROM OrderItems oi
            JOIN Products p ON oi.ProductId = p.ProductId
            JOIN Orders o2 ON oi.OrderId = o2.OrderId
            WHERE o2.CustomerId = c.CustomerId
            FOR XML PATH('')
        ), 1, 2, '') as ProductsPurchased
    FROM Customers c
    JOIN Orders o ON c.CustomerId = o.CustomerId
    WHERE o.OrderDate BETWEEN @StartDate AND @EndDate
    GROUP BY c.CustomerId, c.CustomerName
    ORDER BY Revenue DESC;
END
```

### 2. Views with Heavy Processing

```sql
-- ANTI-PATTERN: View with complex joins and calculations
CREATE VIEW vw_CustomerAnalytics AS
SELECT
    c.CustomerId,
    c.CustomerName,
    c.Email,
    -- Heavy subqueries
    (SELECT COUNT(*) FROM Orders WHERE CustomerId = c.CustomerId) as TotalOrders,
    (SELECT SUM(TotalAmount) FROM Orders WHERE CustomerId = c.CustomerId) as LifetimeValue,
    (SELECT MAX(OrderDate) FROM Orders WHERE CustomerId = c.CustomerId) as LastOrderDate,
    (SELECT AVG(TotalAmount) FROM Orders WHERE CustomerId = c.CustomerId) as AvgOrderValue,
    -- More subqueries
    DATEDIFF(DAY,
        (SELECT MAX(OrderDate) FROM Orders WHERE CustomerId = c.CustomerId),
        GETDATE()) as DaysSinceLastOrder
FROM Customers c;
```

### 3. CLR Functions for Business Logic

```sql
-- ANTI-PATTERN: Using CLR for complex processing
CREATE FUNCTION dbo.CalculateComplexPricing(@ProductId INT, @Quantity INT)
RETURNS DECIMAL(18,2)
AS EXTERNAL NAME PricingAssembly.PricingCalculator.Calculate;

-- Usage in queries
SELECT ProductName, dbo.CalculateComplexPricing(ProductId, 100) as Price
FROM Products;
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| High CPU | Database CPU constantly above 80% |
| Slow queries | Simple queries take unexpectedly long |
| Blocking | Long-running procedures block other queries |
| Resource contention | `sys.dm_exec_requests` shows many waiting queries |
| Can't scale | Adding app servers doesn't improve performance |

### Detection Queries

```sql
-- Find CPU-intensive queries
SELECT TOP 10
    qs.total_worker_time / qs.execution_count AS avg_cpu_time,
    qs.execution_count,
    SUBSTRING(st.text, (qs.statement_start_offset/2) + 1,
        ((CASE qs.statement_end_offset
            WHEN -1 THEN DATALENGTH(st.text)
            ELSE qs.statement_end_offset
        END - qs.statement_start_offset)/2) + 1) AS query_text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
ORDER BY avg_cpu_time DESC;

-- Find stored procedures with high CPU
SELECT TOP 10
    OBJECT_NAME(ps.object_id) AS procedure_name,
    ps.total_worker_time / ps.execution_count AS avg_cpu_time,
    ps.execution_count,
    ps.total_worker_time
FROM sys.dm_exec_procedure_stats ps
ORDER BY ps.total_worker_time DESC;

-- Check for blocking
SELECT
    r.blocking_session_id,
    r.session_id,
    r.wait_type,
    r.wait_time,
    t.text AS query_text
FROM sys.dm_exec_requests r
CROSS APPLY sys.dm_exec_sql_text(r.sql_handle) t
WHERE r.blocking_session_id > 0;
```

### Azure Monitor Metrics

```kusto
// High CPU on Azure SQL
AzureDiagnostics
| where ResourceType == "SERVERS/DATABASES"
| where MetricName == "cpu_percent"
| summarize AvgCPU = avg(Average), MaxCPU = max(Maximum) by bin(TimeGenerated, 5m)
| where AvgCPU > 80
| order by TimeGenerated desc

// Long-running queries
AzureDiagnostics
| where Category == "QueryStoreRuntimeStatistics"
| where duration_s > 30
| project TimeGenerated, query_hash_s, duration_s, cpu_time_s
| order by duration_s desc
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Application Tier (scalable)                                               │
│   ┌─────────────────────────────────────────────────────────────────┐       │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐              │       │
│   │  │ App 1   │ │ App 2   │ │ App 3   │ │ App N   │              │       │
│   │  │ • Logic │ │ • Logic │ │ • Logic │ │ • Logic │              │       │
│   │  │ • Calc  │ │ • Calc  │ │ • Calc  │ │ • Calc  │              │       │
│   │  │ • Trans │ │ • Trans │ │ • Trans │ │ • Trans │              │       │
│   │  └────┬────┘ └────┬────┘ └────┬────┘ └────┬────┘              │       │
│   └───────┼───────────┼───────────┼───────────┼────────────────────┘       │
│           │           │           │           │                             │
│           └───────────┴─────┬─────┴───────────┘                            │
│                             │ Simple CRUD                                   │
│                             ▼                                               │
│   ┌──────────────────────────────────────────────────────────┐              │
│   │                    DATABASE                               │              │
│   │                                                          │              │
│   │  Only responsible for:                                   │              │
│   │  • Data storage                                          │              │
│   │  • Data retrieval                                        │              │
│   │  • Referential integrity                                 │              │
│   │  • Indexing                                              │              │
│   │                                                          │              │
│   │  CPU: 30%  │  Memory: 50%  │  I/O: Normal               │              │
│   └──────────────────────────────────────────────────────────┘              │
│                                                                              │
│   ✅ Scale compute independently                                            │
│   ✅ Database remains responsive                                            │
│   ✅ Easier to optimize each layer                                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Refactored Code

```csharp
// SOLUTION: Move logic to application tier

public class CustomerAnalyticsService
{
    private readonly IDbConnection _db;
    private readonly ICacheService _cache;

    public async Task<CustomerReport> GenerateMonthlyReportAsync(
        DateTime startDate,
        DateTime endDate)
    {
        // Simple query - just get the data
        var orders = await _db.QueryAsync<OrderSummary>(
            @"SELECT CustomerId, OrderId, TotalAmount, OrderDate
              FROM Orders
              WHERE OrderDate BETWEEN @StartDate AND @EndDate",
            new { StartDate = startDate, EndDate = endDate });

        var customers = await _db.QueryAsync<Customer>(
            @"SELECT CustomerId, CustomerName, Email
              FROM Customers
              WHERE CustomerId IN @CustomerIds",
            new { CustomerIds = orders.Select(o => o.CustomerId).Distinct() });

        // Process in application tier
        var report = customers.Select(c => new CustomerReportLine
        {
            CustomerId = c.CustomerId,
            CustomerName = c.CustomerName,
            Revenue = orders
                .Where(o => o.CustomerId == c.CustomerId)
                .Sum(o => o.TotalAmount),
            OrderCount = orders
                .Count(o => o.CustomerId == c.CustomerId),
            CustomerTier = CalculateTier(orders
                .Where(o => o.CustomerId == c.CustomerId)
                .Sum(o => o.TotalAmount))
        }).ToList();

        return new CustomerReport { Lines = report };
    }

    private string CalculateTier(decimal revenue) => revenue switch
    {
        > 100000 => "Platinum",
        > 50000 => "Gold",
        > 10000 => "Silver",
        _ => "Bronze"
    };
}
```

### Azure-Specific Solutions

#### Use Azure Functions for Processing

```csharp
// Azure Function for heavy processing
public class ReportGenerator
{
    [FunctionName("GenerateReport")]
    public async Task<IActionResult> Run(
        [HttpTrigger(AuthorizationLevel.Function, "post")] HttpRequest req,
        [Sql("SELECT * FROM Orders WHERE OrderDate >= @StartDate",
            CommandType = System.Data.CommandType.Text,
            Parameters = "@StartDate={Query.startDate}",
            ConnectionStringSetting = "SqlConnectionString")]
        IEnumerable<Order> orders)
    {
        // Process in Function compute
        var report = ProcessOrders(orders);
        return new OkObjectResult(report);
    }
}
```

#### Use Azure Synapse for Analytics

```sql
-- Move analytics workload to Synapse
-- Keep OLTP database simple

-- In Synapse Analytics (for reporting)
SELECT
    c.CustomerName,
    SUM(o.TotalAmount) as Revenue,
    COUNT(*) as OrderCount
FROM Orders o
JOIN Customers c ON o.CustomerId = c.CustomerId
GROUP BY c.CustomerName

-- In Azure SQL (simple OLTP)
-- Just CRUD operations
INSERT INTO Orders (CustomerId, TotalAmount, OrderDate)
VALUES (@CustomerId, @TotalAmount, @OrderDate);
```

## Test Cases

### Unit Test: Verify Logic Moved to App Tier

```csharp
[Fact]
public void CalculateTier_ReturnsCorrectTier()
{
    var service = new CustomerAnalyticsService(null, null);

    Assert.Equal("Platinum", service.CalculateTier(150000));
    Assert.Equal("Gold", service.CalculateTier(75000));
    Assert.Equal("Silver", service.CalculateTier(25000));
    Assert.Equal("Bronze", service.CalculateTier(5000));
}

[Fact]
public async Task GenerateReport_UsesSimpleQueries()
{
    // Arrange
    var mockDb = new Mock<IDbConnection>();
    mockDb.Setup(x => x.QueryAsync<OrderSummary>(It.IsAny<string>(), It.IsAny<object>()))
        .Callback<string, object>((sql, _) =>
        {
            // Verify query is simple (no JOINs, no aggregations)
            Assert.DoesNotContain("SUM", sql.ToUpper());
            Assert.DoesNotContain("AVG", sql.ToUpper());
            Assert.DoesNotContain("CASE", sql.ToUpper());
        })
        .ReturnsAsync(new List<OrderSummary>());

    var service = new CustomerAnalyticsService(mockDb.Object, null);

    // Act
    await service.GenerateMonthlyReportAsync(DateTime.Now.AddMonths(-1), DateTime.Now);

    // Assert - verification happens in callback
}
```

### Performance Test: Database CPU

```csharp
[Fact]
public async Task DatabaseCPU_StaysBelowThreshold_UnderLoad()
{
    // Arrange
    var tasks = Enumerable.Range(0, 100)
        .Select(_ => _service.GenerateReportAsync());

    // Act
    var startCpu = await GetDatabaseCpuAsync();
    await Task.WhenAll(tasks);
    var endCpu = await GetDatabaseCpuAsync();

    // Assert
    Assert.True(endCpu < 60, $"Database CPU too high: {endCpu}%");
}

private async Task<double> GetDatabaseCpuAsync()
{
    var result = await _db.QueryFirstAsync<double>(
        @"SELECT AVG(end_time - start_time)
          FROM sys.dm_exec_requests
          WHERE status = 'running'");
    return result;
}
```

## Best Practices Checklist

- [ ] Database only handles CRUD operations
- [ ] Business logic resides in application tier
- [ ] Complex calculations done in application code
- [ ] String manipulation done in application code
- [ ] No CLR assemblies for business logic
- [ ] Views are simple (no subqueries, minimal JOINs)
- [ ] Stored procedures are simple (no loops, no cursors)
- [ ] Reports generated in dedicated analytics service

## AWS to Azure Migration Note

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| RDS with heavy procedures | Azure SQL + Azure Functions |
| Redshift for OLTP | Azure SQL (OLTP) + Synapse (Analytics) |
| Lambda + RDS | Azure Functions + Azure SQL |

---

*Back to [Anti-Patterns Overview](README.md)*

*Next: [Busy Front End](02-busy-frontend.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
