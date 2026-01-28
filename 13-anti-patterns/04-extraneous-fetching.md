# Anti-Pattern: Extraneous Fetching

## Overview

The Extraneous Fetching anti-pattern occurs when an application retrieves more data than it needs. This includes fetching entire entities when only a few fields are required, retrieving all records when only a subset is needed, or loading related data that won't be used.

## The Problem

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     EXTRANEOUS FETCHING ANTI-PATTERN                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Client Request: "Show customer names"                                     │
│                                                                              │
│   ┌─────────┐                        ┌─────────────────────┐                │
│   │         │ ──── GET /customers ──►│                     │                │
│   │ Client  │                        │     Database        │                │
│   │         │ ◄───────────────────── │                     │                │
│   └─────────┘                        └─────────────────────┘                │
│                                                                              │
│   Response: 50MB JSON (10,000 customers × 50 fields each)                   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ {                                                                    │   │
│   │   "id": 1,                           ← Used                         │   │
│   │   "name": "John Doe",                ← Used                         │   │
│   │   "email": "john@example.com",       ← NOT USED                     │   │
│   │   "phone": "555-1234",               ← NOT USED                     │   │
│   │   "address": {...},                  ← NOT USED (5KB)               │   │
│   │   "orderHistory": [...],             ← NOT USED (100KB)             │   │
│   │   "preferences": {...},              ← NOT USED                     │   │
│   │   "paymentMethods": [...],           ← NOT USED (sensitive!)        │   │
│   │   "profileImage": "base64...",       ← NOT USED (500KB)             │   │
│   │   ...                                                                │   │
│   │ }                                                                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ❌ Wasted bandwidth (50MB vs 100KB needed)                                │
│   ❌ Wasted memory (deserializing unused data)                              │
│   ❌ Wasted database I/O (reading unnecessary columns)                      │
│   ❌ Security risk (exposing sensitive fields)                              │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Common Manifestations

### 1. SELECT * Queries

```csharp
// ANTI-PATTERN: Selecting all columns
public async Task<List<Customer>> GetCustomerNamesAsync()
{
    // Fetches ALL columns including blobs, large text fields, etc.
    return await _db.Customers.ToListAsync();
}

// Usage only needs name
foreach (var customer in customers)
{
    Console.WriteLine(customer.Name); // Only using ONE field!
}
```

### 2. No Pagination

```csharp
// ANTI-PATTERN: Loading all records at once
public async Task<List<Order>> GetOrdersAsync(int customerId)
{
    // Could return millions of orders!
    return await _db.Orders
        .Where(o => o.CustomerId == customerId)
        .ToListAsync();
}
```

### 3. Eager Loading Everything

```csharp
// ANTI-PATTERN: Loading all related data "just in case"
public async Task<Customer> GetCustomerAsync(int id)
{
    return await _db.Customers
        .Include(c => c.Orders)
            .ThenInclude(o => o.LineItems)
                .ThenInclude(li => li.Product)
                    .ThenInclude(p => p.Reviews)
        .Include(c => c.Addresses)
        .Include(c => c.PaymentMethods)
        .Include(c => c.Preferences)
        .Include(c => c.SupportTickets)
        .FirstOrDefaultAsync(c => c.Id == id);
}
// Might only need customer name and email!
```

### 4. REST API Over-fetching

```javascript
// ANTI-PATTERN: API returns entire entity
// GET /api/products/123
{
    "id": 123,
    "name": "Widget",
    "description": "A long description...",  // 10KB
    "specifications": {...},                   // 50KB
    "images": [...],                          // 500KB of base64
    "reviews": [...],                         // 200KB
    "relatedProducts": [...],                 // 100KB
    "inventory": {...},
    "pricing": {...}
}

// Client only needed: { "id": 123, "name": "Widget", "price": 9.99 }
```

## Detection

### Symptoms

| Symptom | Indicator |
|---------|-----------|
| Large response sizes | Response bodies >> actual UI data |
| Slow API responses | Time spent serializing unused data |
| High memory usage | Large objects in memory |
| Expensive queries | Query plans show unnecessary table scans |
| High data transfer costs | Egress charges disproportionate to usage |

### Detection Queries

```sql
-- Find queries returning many columns
SELECT TOP 20
    qs.total_logical_reads / qs.execution_count AS avg_reads,
    qs.execution_count,
    st.text
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) st
WHERE st.text LIKE '%SELECT *%'
ORDER BY avg_reads DESC;
```

```kusto
// Application Insights: Large responses
requests
| where timestamp > ago(1h)
| where responseSize > 1000000  // > 1MB
| summarize Count = count(), AvgSize = avg(responseSize) by name
| order by AvgSize desc
```

```kusto
// Cosmos DB: High RU queries
AzureDiagnostics
| where ResourceType == "DOCUMENTDB"
| where requestCharge_s > 100
| project TimeGenerated, requestCharge_s, querytext_s
| order by requestCharge_s desc
```

## Solution

### Correct Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         CORRECT ARCHITECTURE                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Client Request: "Show customer names"                                     │
│                                                                              │
│   ┌─────────┐                        ┌─────────────────────┐                │
│   │         │ ── GET /customers      │                     │                │
│   │ Client  │    ?fields=id,name ───►│     Database        │                │
│   │         │ ◄───────────────────── │                     │                │
│   └─────────┘                        └─────────────────────┘                │
│                                                                              │
│   Response: 100KB JSON (10,000 customers × 2 fields each)                   │
│                                                                              │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │ [                                                                    │   │
│   │   { "id": 1, "name": "John Doe" },                                  │   │
│   │   { "id": 2, "name": "Jane Smith" },                                │   │
│   │   ...                                                                │   │
│   │ ]                                                                    │   │
│   └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│   ✅ Minimal bandwidth (100KB vs 50MB)                                      │
│   ✅ Fast serialization                                                     │
│   ✅ Low memory footprint                                                   │
│   ✅ Sensitive data not exposed                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Solution 1: Projection (Select Specific Columns)

```csharp
// SOLUTION: Project only needed fields
public async Task<List<CustomerNameDto>> GetCustomerNamesAsync()
{
    return await _db.Customers
        .Select(c => new CustomerNameDto
        {
            Id = c.Id,
            Name = c.Name
        })
        .ToListAsync();
}

// DTO for specific use case
public record CustomerNameDto(int Id, string Name);
```

### Solution 2: Pagination

```csharp
// SOLUTION: Paginate results
public async Task<PagedResult<Order>> GetOrdersAsync(
    int customerId,
    int page = 1,
    int pageSize = 20)
{
    var query = _db.Orders.Where(o => o.CustomerId == customerId);

    var totalCount = await query.CountAsync();

    var items = await query
        .OrderByDescending(o => o.OrderDate)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(o => new OrderSummary
        {
            Id = o.Id,
            OrderDate = o.OrderDate,
            Total = o.Total
        })
        .ToListAsync();

    return new PagedResult<Order>
    {
        Items = items,
        TotalCount = totalCount,
        Page = page,
        PageSize = pageSize
    };
}
```

### Solution 3: GraphQL for Flexible Queries

```graphql
# Client specifies exactly what they need
query GetCustomerNames {
  customers {
    id
    name
  }
}

query GetCustomerWithOrders($id: ID!) {
  customer(id: $id) {
    id
    name
    email
    orders(first: 10) {
      id
      total
      orderDate
    }
  }
}
```

```csharp
// Hot Chocolate GraphQL implementation
public class Query
{
    [UsePaging]
    [UseProjection]
    [UseFiltering]
    [UseSorting]
    public IQueryable<Customer> GetCustomers([Service] AppDbContext db)
        => db.Customers;
}
```

### Solution 4: OData for REST APIs

```csharp
// Enable OData endpoints
[EnableQuery]
public IQueryable<Customer> Get()
{
    return _db.Customers;
}

// Client queries
// GET /api/customers?$select=id,name
// GET /api/customers?$select=id,name&$top=10&$skip=20
// GET /api/customers?$filter=country eq 'USA'&$select=id,name
```

### Solution 5: Purpose-Built Endpoints

```csharp
// SOLUTION: Create specific endpoints for specific needs

// For dropdown lists
[HttpGet("lookup")]
public async Task<ActionResult<List<CustomerLookup>>> GetLookup()
{
    return await _db.Customers
        .Select(c => new CustomerLookup(c.Id, c.Name))
        .ToListAsync();
}

// For dashboard cards
[HttpGet("summary")]
public async Task<ActionResult<CustomerSummary>> GetSummary(int id)
{
    return await _db.Customers
        .Where(c => c.Id == id)
        .Select(c => new CustomerSummary
        {
            Name = c.Name,
            OrderCount = c.Orders.Count,
            TotalSpent = c.Orders.Sum(o => o.Total)
        })
        .FirstOrDefaultAsync();
}

// For full details (when actually needed)
[HttpGet("{id}/details")]
public async Task<ActionResult<CustomerDetails>> GetDetails(int id)
{
    return await _db.Customers
        .Where(c => c.Id == id)
        .Select(c => new CustomerDetails { /* all fields */ })
        .FirstOrDefaultAsync();
}
```

### Azure-Specific Solutions

#### Cosmos DB: Projection

```csharp
// Cosmos DB query with projection
var query = container.GetItemQueryIterator<CustomerName>(
    new QueryDefinition("SELECT c.id, c.name FROM c WHERE c.type = 'customer'"),
    requestOptions: new QueryRequestOptions
    {
        MaxItemCount = 100
    });

// Uses less RU because it returns less data
```

#### Azure SQL: Column-Level Security

```sql
-- Prevent over-fetching sensitive data
CREATE FUNCTION dbo.fn_securitypredicate(@SensitiveColumn AS nvarchar(100))
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN SELECT 1 AS fn_securitypredicate_result
WHERE IS_MEMBER('SensitiveDataReaders') = 1;

-- Only authorized users can see sensitive columns
```

#### API Management: Response Transformation

```xml
<!-- Azure API Management policy to filter response -->
<outbound>
    <set-body>
        @{
            var response = context.Response.Body.As<JObject>();
            var filtered = new JObject
            {
                ["id"] = response["id"],
                ["name"] = response["name"],
                ["email"] = response["email"]
            };
            return filtered.ToString();
        }
    </set-body>
</outbound>
```

## Test Cases

### Unit Test: Verify Projection

```csharp
[Fact]
public async Task GetCustomerNames_ShouldOnlyQueryRequiredColumns()
{
    // Arrange
    var queryLog = new List<string>();
    var options = new DbContextOptionsBuilder<AppDbContext>()
        .UseSqlServer(_connectionString)
        .LogTo(sql => queryLog.Add(sql))
        .Options;

    using var context = new AppDbContext(options);
    var service = new CustomerService(context);

    // Act
    queryLog.Clear();
    await service.GetCustomerNamesAsync();

    // Assert
    var selectQuery = queryLog.First(q => q.Contains("SELECT"));
    Assert.DoesNotContain("Address", selectQuery);
    Assert.DoesNotContain("OrderHistory", selectQuery);
    Assert.Contains("Id", selectQuery);
    Assert.Contains("Name", selectQuery);
}
```

### Performance Test: Response Size

```csharp
[Fact]
public async Task GetCustomerNames_ShouldReturnSmallResponse()
{
    // Arrange
    var client = _factory.CreateClient();

    // Act
    var response = await client.GetAsync("/api/customers/lookup");
    var content = await response.Content.ReadAsByteArrayAsync();

    // Assert - response should be small
    Assert.True(content.Length < 100_000,
        $"Response too large: {content.Length} bytes");
}
```

### Load Test: Memory Usage

```csharp
[Fact]
public async Task GetCustomers_ShouldNotCauseMemorySpike()
{
    var initialMemory = GC.GetTotalMemory(true);

    // Act - fetch customers
    await _service.GetCustomerNamesAsync();

    var finalMemory = GC.GetTotalMemory(true);
    var memoryIncrease = finalMemory - initialMemory;

    // Assert - memory increase should be reasonable
    Assert.True(memoryIncrease < 10_000_000,
        $"Memory increase too high: {memoryIncrease / 1_000_000}MB");
}
```

## Best Practices Checklist

- [ ] Never use SELECT * in production code
- [ ] Create DTOs for each use case
- [ ] Implement pagination for all list endpoints
- [ ] Use projection in LINQ queries
- [ ] Consider GraphQL for flexible data needs
- [ ] Implement field filtering in REST APIs
- [ ] Lazy load related data only when needed
- [ ] Set reasonable default and maximum page sizes
- [ ] Remove sensitive fields from general endpoints
- [ ] Monitor response sizes in production

## AWS to Azure Migration Note

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| DynamoDB ProjectionExpression | Cosmos DB SELECT projection |
| AppSync field selection | Hot Chocolate GraphQL |
| API Gateway response mapping | API Management policies |
| Lambda response filtering | Azure Functions + DTOs |

---

*Back to [Anti-Patterns Overview](README.md)*

*Previous: [Chatty I/O](03-chatty-io.md)*

*Next: [Improper Instantiation](05-improper-instantiation.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
