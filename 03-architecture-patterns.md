# Cloud Architecture Patterns & Best Practices

Essential design patterns every Cloud Architect should know, with AWS and Azure implementation examples.

## 1. Microservices Architecture

### Pattern Overview

Breaking down applications into small, independently deployable services.

```
┌─────────────────────────────────────────────────────────────────┐
│                        API Gateway                               │
│                   (Rate limiting, Auth, Routing)                │
└─────────────────────────────────────────────────────────────────┘
          │              │              │              │
     ┌────┴────┐    ┌────┴────┐    ┌────┴────┐    ┌────┴────┐
     │ User    │    │ Product │    │ Order   │    │ Payment │
     │ Service │    │ Service │    │ Service │    │ Service │
     └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
          │              │              │              │
     [User DB]     [Product DB]   [Order DB]    [Payment DB]
```

### Implementation

| Component | AWS | Azure |
|-----------|-----|-------|
| API Gateway | API Gateway | API Management |
| Service Mesh | App Mesh | Azure Service Mesh |
| Service Discovery | Cloud Map | Azure Service Fabric |
| Container Orchestration | EKS/ECS | AKS |

### Best Practices

1. **Single Responsibility:** Each service does one thing well
2. **Database per Service:** Avoid shared databases
3. **API Contracts:** Use OpenAPI/Swagger specifications
4. **Circuit Breakers:** Prevent cascade failures
5. **Distributed Tracing:** Track requests across services
6. **Event-Driven Communication:** Use async messaging when possible

## 2. Event-Driven Architecture

### Pattern Overview

Services communicate through events rather than direct calls.

```
┌──────────┐    Event     ┌──────────────┐    Event     ┌──────────┐
│ Producer │───────────→│  Event Bus   │───────────→│ Consumer │
└──────────┘             └──────────────┘             └──────────┘
                              │
                              ├──→ Consumer 2
                              ├──→ Consumer 3
                              └──→ Event Store
```

### Event Patterns

**Event Notification:**
```json
{
  "eventType": "OrderCreated",
  "eventId": "12345",
  "timestamp": "2024-01-15T10:00:00Z",
  "data": {
    "orderId": "ORD-001"
  }
}
```

**Event-Carried State Transfer:**
```json
{
  "eventType": "OrderCreated",
  "eventId": "12345",
  "timestamp": "2024-01-15T10:00:00Z",
  "data": {
    "orderId": "ORD-001",
    "customerId": "CUST-123",
    "items": [...],
    "totalAmount": 99.99
  }
}
```

### Implementation

| Component | AWS | Azure |
|-----------|-----|-------|
| Event Bus | EventBridge | Event Grid |
| Message Queue | SQS | Service Bus Queues |
| Pub/Sub | SNS | Service Bus Topics |
| Streaming | Kinesis | Event Hubs |

### Best Practices

1. **Idempotent Consumers:** Handle duplicate events gracefully
2. **Event Versioning:** Support schema evolution
3. **Dead Letter Queues:** Handle failed messages
4. **Event Ordering:** When sequence matters, use partitioning
5. **Event Schema Registry:** Maintain event contracts

## 3. CQRS (Command Query Responsibility Segregation)

### Pattern Overview

Separate read and write models for optimized performance.

```
                    ┌─────────────┐
                    │   Client    │
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
         [Commands]                 [Queries]
              │                         │
        ┌─────┴─────┐            ┌──────┴──────┐
        │  Command  │            │    Query    │
        │  Handler  │            │   Handler   │
        └─────┬─────┘            └──────┬──────┘
              │                         │
        ┌─────┴─────┐            ┌──────┴──────┐
        │  Write    │───Events──→│    Read     │
        │  Store    │            │   Store     │
        └───────────┘            └─────────────┘
```

### When to Use CQRS

- Read/write patterns are significantly different
- Complex domain with heavy validation on writes
- High read-to-write ratio
- Need for different scaling strategies

### Implementation Example

**Write Side (AWS):**
- API Gateway → Lambda → DynamoDB (writes)
- DynamoDB Streams → Lambda → Update read model

**Read Side (AWS):**
- API Gateway → Lambda → ElasticSearch/RDS Read Replica

**Azure Equivalent:**
- API Management → Functions → Cosmos DB (writes)
- Change Feed → Functions → Azure Search/SQL Read Replica

## 4. Saga Pattern

### Pattern Overview

Managing distributed transactions across multiple services.

```
Choreography (Event-based):
┌─────────┐     ┌─────────┐     ┌─────────┐     ┌─────────┐
│ Order   │────→│ Payment │────→│ Inventory│────→│ Shipping│
│ Service │     │ Service │     │ Service │     │ Service │
└─────────┘     └─────────┘     └─────────┘     └─────────┘
     ↑               │               │               │
     └───────────────┴───────────────┴───────────────┘
                    (Compensating events on failure)

Orchestration (Coordinator-based):
                    ┌─────────────┐
                    │    Saga     │
                    │ Orchestrator│
                    └──────┬──────┘
           ┌───────────────┼───────────────┐
           ↓               ↓               ↓
      ┌─────────┐    ┌─────────┐    ┌─────────┐
      │ Payment │    │ Inventory│    │ Shipping│
      │ Service │    │ Service │    │ Service │
      └─────────┘    └─────────┘    └─────────┘
```

### Implementation

| Approach | AWS | Azure |
|----------|-----|-------|
| Orchestration | Step Functions | Logic Apps / Durable Functions |
| Choreography | EventBridge + SQS | Event Grid + Service Bus |

### Compensation Example

```
Order Saga:
1. Create Order (Pending)
2. Reserve Inventory
   - Compensation: Release Inventory
3. Process Payment
   - Compensation: Refund Payment
4. Confirm Order
   - Compensation: Cancel Order
5. Ship Order
```

## 5. Strangler Fig Pattern

### Pattern Overview

Gradually replacing a legacy system by routing traffic to new services.

```
Phase 1: Facade
┌────────────────────────────────────────────────────────┐
│                       Facade/Proxy                      │
└────────────────────────────────────────────────────────┘
                           │
                    ┌──────┴──────┐
                    ↓             ↓
              [New Service]  [Legacy System]
                  (10%)          (90%)

Phase 2: Migration
┌────────────────────────────────────────────────────────┐
│                       Facade/Proxy                      │
└────────────────────────────────────────────────────────┘
                           │
                    ┌──────┴──────┐
                    ↓             ↓
              [New Service]  [Legacy System]
                  (70%)          (30%)

Phase 3: Complete
┌────────────────────────────────────────────────────────┐
│                       New System                        │
└────────────────────────────────────────────────────────┘
                       (100%)
```

### Implementation

**AWS:**
- ALB with weighted target groups
- API Gateway with stage variables
- Route 53 weighted routing

**Azure:**
- Application Gateway with URL path routing
- Traffic Manager weighted routing
- Azure Front Door rules engine

## 6. Circuit Breaker Pattern

### Pattern Overview

Preventing cascade failures by failing fast when a service is unhealthy.

```
States:
┌─────────┐     failures     ┌─────────┐    timeout    ┌───────────┐
│ Closed  │────────────────→│  Open   │──────────────→│ Half-Open │
│(Normal) │                  │ (Fail   │               │  (Test)   │
└─────────┘                  │  Fast)  │               └───────────┘
     ↑                       └─────────┘                     │
     │                            ↑                         │
     │   success                  │      failure            │
     └────────────────────────────┴─────────────────────────┘
```

### Implementation

| Tool | Language/Platform |
|------|-------------------|
| Hystrix | Java (deprecated, use Resilience4j) |
| Resilience4j | Java |
| Polly | .NET |
| Istio | Kubernetes service mesh |

### Configuration Example (Polly/.NET)

```csharp
Policy
    .Handle<HttpRequestException>()
    .CircuitBreaker(
        exceptionsAllowedBeforeBreaking: 3,
        durationOfBreak: TimeSpan.FromSeconds(30)
    );
```

## 7. Sidecar Pattern

### Pattern Overview

Deploying helper components alongside your application.

```
┌────────────────────────────────────┐
│              Pod/Host              │
│  ┌─────────────┐  ┌─────────────┐  │
│  │    Main     │  │   Sidecar   │  │
│  │ Application │←→│   Proxy     │  │
│  └─────────────┘  └─────────────┘  │
└────────────────────────────────────┘
                         │
                    Network Traffic
```

### Common Use Cases

1. **Service Mesh Proxy (Envoy):** Traffic management, observability
2. **Log Aggregation:** Shipping logs to central system
3. **Configuration:** Dynamically updating config
4. **Secrets Management:** Vault agent for secrets
5. **Monitoring:** Metrics collection and export

### Implementation

**Kubernetes:**
```yaml
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: main-app
    image: myapp:latest
  - name: sidecar-proxy
    image: envoyproxy/envoy:latest
```

## 8. Bulkhead Pattern

### Pattern Overview

Isolating components to prevent failures from spreading.

```
Without Bulkhead:
┌────────────────────────────────────────────┐
│            Shared Thread Pool              │
│  [Request 1] [Request 2] ... [Request N]   │
│         (One slow service blocks all)      │
└────────────────────────────────────────────┘

With Bulkhead:
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Service A  │  │  Service B  │  │  Service C  │
│  Pool (10)  │  │  Pool (20)  │  │  Pool (15)  │
└─────────────┘  └─────────────┘  └─────────────┘
    (Isolated - failure in A doesn't affect B,C)
```

### Implementation Strategies

1. **Thread Pool Isolation:** Separate thread pools per service
2. **Process Isolation:** Separate processes/containers
3. **Resource Quotas:** Kubernetes resource limits
4. **Rate Limiting:** API Gateway throttling

## 9. Backend for Frontend (BFF)

### Pattern Overview

Custom backends optimized for specific frontend applications.

```
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │   Web    │    │  Mobile  │    │   IoT    │
        │   App    │    │   App    │    │ Devices  │
        └────┬─────┘    └────┬─────┘    └────┬─────┘
             │               │               │
        ┌────┴─────┐    ┌────┴─────┐    ┌────┴─────┐
        │ Web BFF  │    │Mobile BFF│    │ IoT BFF  │
        │(GraphQL) │    │ (REST)   │    │ (MQTT)   │
        └────┬─────┘    └────┬─────┘    └────┬─────┘
             │               │               │
             └───────────────┼───────────────┘
                             │
                    ┌────────┴────────┐
                    │ Shared Services │
                    └─────────────────┘
```

### Benefits

- Optimized response payloads per client
- Client-specific authentication
- Different caching strategies
- Independent deployment and scaling

## 10. Data Patterns

### Database per Service

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   User      │    │   Order     │    │   Product   │
│   Service   │    │   Service   │    │   Service   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
  ┌────┴────┐        ┌────┴────┐        ┌────┴────┐
  │ User DB │        │Order DB │        │Product  │
  │ (SQL)   │        │(NoSQL)  │        │DB(Cache)│
  └─────────┘        └─────────┘        └─────────┘
```

### Shared Database (Anti-pattern for Microservices)

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   User      │    │   Order     │    │   Product   │
│   Service   │    │   Service   │    │   Service   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                   ┌──────┴──────┐
                   │ Shared DB   │  ← Tight coupling
                   │             │  ← Schema changes affect all
                   └─────────────┘
```

### Event Sourcing

```
Traditional:
┌─────────┐  UPDATE  ┌─────────────────┐
│ Service │─────────→│ Current State   │
└─────────┘          │ (overwritten)   │
                     └─────────────────┘

Event Sourcing:
┌─────────┐  APPEND  ┌─────────────────────────────────┐
│ Service │─────────→│ Event Store                     │
└─────────┘          │ [Created][Updated][Updated]...  │
                     └─────────────────────────────────┘
                                    │
                              (Replay to get current state)
```

## 11. Caching Patterns

### Cache-Aside (Lazy Loading)

```
┌────────┐    1. Check Cache    ┌─────────┐
│ Client │─────────────────────→│  Cache  │
└────────┘                      └─────────┘
     │                               │
     │ 2. Cache Miss                 │
     │                               │
     ↓                               │
┌────────┐    3. Query DB      ┌─────────┐
│   DB   │←────────────────────│ Client  │
└────────┘                      └─────────┘
     │                               │
     │ 4. Return Data                │
     │                               │
     ↓                               ↓
┌────────┐    5. Update Cache  ┌─────────┐
│ Client │────────────────────→│  Cache  │
└────────┘                      └─────────┘
```

### Write-Through

```
┌────────┐    1. Write    ┌─────────┐    2. Write    ┌────────┐
│ Client │───────────────→│  Cache  │───────────────→│   DB   │
└────────┘                └─────────┘                └────────┘
```

### Write-Behind (Write-Back)

```
┌────────┐    1. Write    ┌─────────┐    2. Async Write    ┌────────┐
│ Client │───────────────→│  Cache  │─ ─ ─ ─ ─ ─ ─ ─ ─ ─ →│   DB   │
└────────┘                └─────────┘    (Batched)         └────────┘
```

### Implementation

| Service | AWS | Azure |
|---------|-----|-------|
| In-Memory Cache | ElastiCache (Redis/Memcached) | Azure Cache for Redis |
| CDN Cache | CloudFront | Azure CDN / Front Door |
| API Cache | API Gateway Cache | API Management Cache |

## 12. Multi-Region Architecture

### Active-Passive

```
┌─────────────────────┐         ┌─────────────────────┐
│    Region A         │         │    Region B         │
│    (Active)         │         │    (Passive)        │
│  ┌─────────────┐    │         │  ┌─────────────┐    │
│  │    App      │    │         │  │    App      │    │
│  └─────────────┘    │         │  │  (Standby)  │    │
│  ┌─────────────┐    │  Async  │  └─────────────┘    │
│  │   Primary   │────│────────→│  ┌─────────────┐    │
│  │     DB      │    │  Repl   │  │   Replica   │    │
│  └─────────────┘    │         │  │     DB      │    │
└─────────────────────┘         └─────────────────────┘
```

### Active-Active

```
         ┌─────────────────────────────────────────┐
         │         Global Load Balancer            │
         │     (Route 53 / Traffic Manager)        │
         └────────────────┬────────────────────────┘
                          │
         ┌────────────────┴────────────────┐
         ↓                                 ↓
┌─────────────────────┐         ┌─────────────────────┐
│    Region A         │         │    Region B         │
│    (Active)         │         │    (Active)         │
│  ┌─────────────┐    │         │  ┌─────────────┐    │
│  │    App      │    │  Sync   │  │    App      │    │
│  └─────────────┘    │←───────→│  └─────────────┘    │
│  ┌─────────────┐    │  Repl   │  ┌─────────────┐    │
│  │   Multi-    │────│────────→│  │   Multi-    │    │
│  │ Master DB   │←───│─────────│  │ Master DB   │    │
│  └─────────────┘    │         │  └─────────────┘    │
└─────────────────────┘         └─────────────────────┘
```

### Global Services

| Use Case | AWS | Azure |
|----------|-----|-------|
| Global DB | DynamoDB Global Tables, Aurora Global | Cosmos DB (multi-region) |
| DNS Routing | Route 53 | Traffic Manager |
| CDN | CloudFront | Azure CDN / Front Door |
| Global LB | Global Accelerator | Azure Front Door |

---

*Next: See [04-hands-on-labs.md](04-hands-on-labs.md) for practical exercises*
