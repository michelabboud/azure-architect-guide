# Extra Resources: Performance Anti-Patterns

## Official Microsoft Documentation

### Azure Architecture Center
- [Performance Anti-patterns](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/)
- [Best Practices for Cloud Applications](https://learn.microsoft.com/en-us/azure/architecture/best-practices/)
- [Performance Testing](https://learn.microsoft.com/en-us/azure/architecture/framework/scalability/performance-test)

### Individual Anti-Pattern Guides
- [Busy Database](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/busy-database/)
- [Busy Front End](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/busy-front-end/)
- [Chatty I/O](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/chatty-io/)
- [Extraneous Fetching](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/extraneous-fetching/)
- [Improper Instantiation](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/improper-instantiation/)
- [Monolithic Persistence](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/monolithic-persistence/)
- [No Caching](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/no-caching/)
- [Noisy Neighbor](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/noisy-neighbor/)
- [Retry Storm](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/retry-storm/)
- [Synchronous I/O](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/synchronous-io/)

## Performance Testing Tools

### Load Testing
- [Azure Load Testing](https://learn.microsoft.com/en-us/azure/load-testing/)
- [JMeter](https://jmeter.apache.org/)
- [k6](https://k6.io/)
- [Locust](https://locust.io/)

### Profiling Tools
- [Application Insights Profiler](https://learn.microsoft.com/en-us/azure/azure-monitor/profiler/profiler-overview)
- [Visual Studio Performance Profiler](https://learn.microsoft.com/en-us/visualstudio/profiling/)
- [dotTrace](https://www.jetbrains.com/profiler/)

## Resilience Libraries

### .NET
- [Polly](https://github.com/App-vNext/Polly) - Resilience and transient fault handling
- [Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/en-us/dotnet/core/resilience/)

### Java
- [Resilience4j](https://resilience4j.readme.io/)

### Node.js
- [cockatiel](https://github.com/connor4312/cockatiel)
- [opossum](https://github.com/nodeshift/opossum)

## Monitoring and Detection

### Azure Monitor
- [Azure Monitor Documentation](https://learn.microsoft.com/en-us/azure/azure-monitor/)
- [Application Insights](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)
- [Log Analytics KQL](https://learn.microsoft.com/en-us/azure/data-explorer/kusto/query/)

### Database Monitoring
- [Query Performance Insight](https://learn.microsoft.com/en-us/azure/azure-sql/database/query-performance-insight-use)
- [Intelligent Insights](https://learn.microsoft.com/en-us/azure/azure-sql/database/intelligent-insights-overview)
- [Cosmos DB Metrics](https://learn.microsoft.com/en-us/azure/cosmos-db/monitor-cosmos-db)

## Best Practices Guides

### Caching
- [Caching Best Practices](https://learn.microsoft.com/en-us/azure/architecture/best-practices/caching)
- [Azure Cache for Redis Best Practices](https://learn.microsoft.com/en-us/azure/azure-cache-for-redis/cache-best-practices)

### Database
- [Azure SQL Performance](https://learn.microsoft.com/en-us/azure/azure-sql/database/performance-guidance)
- [Cosmos DB Performance Tips](https://learn.microsoft.com/en-us/azure/cosmos-db/performance-tips)

### HTTP Clients
- [IHttpClientFactory](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-factory)
- [HttpClient Guidelines](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines)

## Books and Articles

### Architecture
- *Designing Data-Intensive Applications* - Martin Kleppmann
- *Release It!* - Michael Nygard
- *Building Microservices* - Sam Newman

### Performance
- *High Performance Browser Networking* - Ilya Grigorik
- *Systems Performance* - Brendan Gregg

## Video Resources

### Microsoft Learn
- [Performance Optimization Patterns](https://www.youtube.com/results?search_query=azure+performance+patterns)
- [Azure Friday - Performance](https://www.youtube.com/results?search_query=azure+friday+performance)

### Conferences
- [Microsoft Build Sessions](https://www.youtube.com/results?search_query=microsoft+build+performance)
- [.NET Conf Performance Sessions](https://www.youtube.com/results?search_query=dotnet+conf+performance)

## Related Design Patterns

| Anti-Pattern | Related Patterns to Apply |
|-------------|---------------------------|
| Busy Database | CQRS, Command Query Separation |
| Busy Front End | Async Request-Reply, Web Worker |
| Chatty I/O | Ambassador, Gateway Aggregation |
| Extraneous Fetching | Materialized View, CQRS |
| Improper Instantiation | Singleton, Object Pool |
| Monolithic Persistence | Polyglot Persistence |
| No Caching | Cache-Aside, Write-Through |
| Noisy Neighbor | Bulkhead, Throttling |
| Retry Storm | Circuit Breaker, Queue-Based Load Leveling |
| Synchronous I/O | Async Request-Reply, Event-Driven |

---

*Back to [Anti-Patterns Overview](README.md)*
