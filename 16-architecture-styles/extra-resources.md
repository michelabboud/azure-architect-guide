# Extra Resources: Architecture Styles

## Official Microsoft Documentation

### Azure Architecture Center
- [Architecture Styles Overview](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/)
- [N-Tier Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/n-tier)
- [Web-Queue-Worker](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/web-queue-worker)
- [Microservices](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/microservices)
- [Event-Driven Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
- [Big Data Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/big-data)
- [Big Compute Architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/big-compute)

### Design Principles
- [Design Principles](https://learn.microsoft.com/en-us/azure/architecture/guide/design-principles/)
- [Self-Healing](https://learn.microsoft.com/en-us/azure/architecture/guide/design-principles/self-healing)
- [Design for Scale Out](https://learn.microsoft.com/en-us/azure/architecture/guide/design-principles/scale-out)
- [Use Managed Services](https://learn.microsoft.com/en-us/azure/architecture/guide/design-principles/managed-services)

### Technology Choices
- [Choosing a Compute Service](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/compute-decision-tree)
- [Choosing a Data Store](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/data-store-overview)
- [Choosing a Messaging Service](https://learn.microsoft.com/en-us/azure/architecture/guide/technology-choices/messaging)

## Reference Architectures

### N-Tier
- [N-Tier Application with SQL Server](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/n-tier/n-tier-sql-server)
- [Multi-Region N-Tier](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/n-tier/multi-region-sql-server)

### Microservices
- [Microservices on AKS](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/containers/aks-microservices/aks-microservices)
- [Microservices with Container Apps](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/serverless/microservices-with-container-apps)
- [Serverless Microservices](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/serverless/cloud-automation-event-based)

### Event-Driven
- [Event-Driven Architecture Style](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven)
- [Serverless Event Processing](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/serverless/event-processing)
- [Stream Processing with Azure Stream Analytics](https://learn.microsoft.com/en-us/azure/architecture/reference-architectures/data/stream-processing-stream-analytics)

### Big Data
- [Modern Data Warehouse](https://learn.microsoft.com/en-us/azure/architecture/solution-ideas/articles/modern-data-warehouse)
- [Real-Time Analytics](https://learn.microsoft.com/en-us/azure/architecture/solution-ideas/articles/real-time-analytics)
- [Data Lakehouse](https://learn.microsoft.com/en-us/azure/architecture/example-scenario/data/data-lakehouse)

## Cloud Design Patterns

### Resilience Patterns
- [Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Retry Pattern](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)
- [Compensating Transaction](https://learn.microsoft.com/en-us/azure/architecture/patterns/compensating-transaction)
- [Health Endpoint Monitoring](https://learn.microsoft.com/en-us/azure/architecture/patterns/health-endpoint-monitoring)

### Scalability Patterns
- [Sharding](https://learn.microsoft.com/en-us/azure/architecture/patterns/sharding)
- [Queue-Based Load Leveling](https://learn.microsoft.com/en-us/azure/architecture/patterns/queue-based-load-leveling)
- [Throttling](https://learn.microsoft.com/en-us/azure/architecture/patterns/throttling)

### Data Patterns
- [CQRS](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- [Event Sourcing](https://learn.microsoft.com/en-us/azure/architecture/patterns/event-sourcing)
- [Materialized View](https://learn.microsoft.com/en-us/azure/architecture/patterns/materialized-view)

## Microsoft Learn Paths

### Architecture
- [Architect Great Solutions in Azure](https://learn.microsoft.com/en-us/training/paths/architect-great-solutions-in-azure/)
- [Build Great Solutions with Well-Architected Framework](https://learn.microsoft.com/en-us/training/paths/azure-well-architected-framework/)

### Microservices
- [Architect Modern Apps with Microservices](https://learn.microsoft.com/en-us/training/paths/architect-modern-apps/)
- [Build Microservices with AKS](https://learn.microsoft.com/en-us/training/paths/build-microservices-with-aks/)

### Data
- [Architect Data Platform in Azure](https://learn.microsoft.com/en-us/training/paths/architect-data-platform/)

## Books

### Architecture
- *Fundamentals of Software Architecture* - Mark Richards, Neal Ford
- *Software Architecture: The Hard Parts* - Neal Ford et al.
- *Building Evolutionary Architectures* - Neal Ford et al.
- *Clean Architecture* - Robert C. Martin

### Microservices
- *Building Microservices* - Sam Newman
- *Microservices Patterns* - Chris Richardson
- *Monolith to Microservices* - Sam Newman

### Distributed Systems
- *Designing Data-Intensive Applications* - Martin Kleppmann
- *Distributed Systems* - Maarten van Steen, Andrew Tanenbaum

## Video Resources

### Microsoft Learn
- [Architecture Sessions](https://www.youtube.com/results?search_query=azure+architecture+patterns)
- [Microservices on Azure](https://www.youtube.com/results?search_query=azure+microservices)

### Conference Talks
- [GOTO Conferences - Software Architecture](https://www.youtube.com/playlist?list=PLEx5khR4g7PI89_ZS_wz5suqCoqFgv-gO)
- [QCon Presentations](https://www.youtube.com/c/InfoQ/videos)

## Tools and Frameworks

### Diagramming
- [Azure Architecture Icons](https://learn.microsoft.com/en-us/azure/architecture/icons/)
- [Draw.io](https://app.diagrams.net/)
- [Visio Online](https://www.microsoft.com/en-us/microsoft-365/visio/flowchart-software)

### Modeling
- [C4 Model](https://c4model.com/)
- [Structurizr](https://structurizr.com/)

### Validation
- [Azure Well-Architected Review](https://learn.microsoft.com/en-us/assessments/azure-architecture-review/)
- [Architecture Decision Records](https://adr.github.io/)

## AWS to Azure Comparison

| AWS Pattern | Azure Equivalent |
|-------------|-----------------|
| Three-tier on AWS | N-tier on Azure |
| Lambda + SQS | Functions + Service Bus |
| EKS Microservices | AKS Microservices |
| Kinesis Streaming | Event Hubs Streaming |
| EMR/Redshift | Synapse/Databricks |
| AWS Batch | Azure Batch |

---

*Back to [Chapter Overview](README.md)*
