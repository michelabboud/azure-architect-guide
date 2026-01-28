# Chapter 09: Cloud Design Patterns

## Overview

Cloud design patterns are reusable solutions to common problems in cloud application architecture. These patterns help you build scalable, reliable, and maintainable applications on Azure. For AWS engineers, many patterns apply across clouds, but Azure services provide specific implementations.

## Pattern Categories

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CLOUD DESIGN PATTERN CATEGORIES                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  RELIABILITY PATTERNS                 PERFORMANCE PATTERNS                  │
│  ────────────────────                 ────────────────────                  │
│  • Circuit Breaker                    • Cache-Aside                         │
│  • Retry                              • CQRS                                │
│  • Bulkhead                           • Event Sourcing                      │
│  • Health Endpoint Monitoring         • Index Table                         │
│  • Queue-Based Load Leveling          • Materialized View                   │
│  • Compensating Transaction           • Sharding                            │
│  • Leader Election                    • Static Content Hosting              │
│                                                                              │
│  SECURITY PATTERNS                    MESSAGING PATTERNS                    │
│  ─────────────────                    ──────────────────                    │
│  • Federated Identity                 • Competing Consumers                 │
│  • Gatekeeper                         • Priority Queue                      │
│  • Valet Key                          • Publisher/Subscriber                │
│  • Claim Check                        • Pipes and Filters                   │
│  • Quarantine                         • Choreography                        │
│                                       • Saga                                │
│                                                                              │
│  DATA MANAGEMENT PATTERNS             DEPLOYMENT PATTERNS                   │
│  ────────────────────────             ───────────────────                   │
│  • Cache-Aside                        • Deployment Stamps                   │
│  • CQRS                               • Geode                               │
│  • Event Sourcing                     • Sidecar                             │
│  • Sharding                           • Ambassador                          │
│  • Materialized View                  • Strangler Fig                       │
│                                       • Anti-Corruption Layer               │
│                                                                              │
│  GATEWAY PATTERNS                     COST OPTIMIZATION PATTERNS            │
│  ────────────────                     ──────────────────────────            │
│  • Gateway Aggregation                • Throttling                          │
│  • Gateway Offloading                 • Queue-Based Load Leveling           │
│  • Gateway Routing                    • Static Content Hosting              │
│  • Backends for Frontends             • Compute Resource Consolidation      │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Well-Architected Framework Alignment

```
PATTERNS BY WELL-ARCHITECTED PILLAR:
────────────────────────────────────

RELIABILITY:
┌────────────────────────────────────────────────────────────────────────────┐
│ Pattern                    │ Purpose                                       │
├────────────────────────────┼───────────────────────────────────────────────┤
│ Circuit Breaker           │ Prevent cascade failures                       │
│ Retry                     │ Handle transient failures                      │
│ Bulkhead                  │ Isolate failures                               │
│ Health Endpoint           │ Detect issues early                            │
│ Compensating Transaction  │ Handle distributed failures                    │
│ Queue-Based Load Leveling │ Absorb traffic spikes                          │
└────────────────────────────┴───────────────────────────────────────────────┘

SECURITY:
┌────────────────────────────────────────────────────────────────────────────┐
│ Federated Identity        │ Delegate authentication                        │
│ Valet Key                 │ Limit resource access                          │
│ Gatekeeper                │ Protect backend services                       │
│ Quarantine                │ Validate external inputs                       │
│ Claim Check               │ Secure large message handling                  │
└────────────────────────────┴───────────────────────────────────────────────┘

PERFORMANCE EFFICIENCY:
┌────────────────────────────────────────────────────────────────────────────┐
│ Cache-Aside               │ Reduce database load                           │
│ CQRS                      │ Optimize read/write separately                 │
│ Sharding                  │ Scale data horizontally                        │
│ Static Content Hosting    │ Offload to CDN                                 │
│ Competing Consumers       │ Scale processing                               │
└────────────────────────────┴───────────────────────────────────────────────┘

OPERATIONAL EXCELLENCE:
┌────────────────────────────────────────────────────────────────────────────┐
│ Deployment Stamps         │ Isolated scale units                           │
│ Sidecar                   │ Modular capabilities                           │
│ External Configuration    │ Centralized config management                  │
│ Strangler Fig             │ Incremental migration                          │
└────────────────────────────┴───────────────────────────────────────────────┘

COST OPTIMIZATION:
┌────────────────────────────────────────────────────────────────────────────┐
│ Throttling                │ Control resource consumption                   │
│ Queue-Based Load Leveling │ Smooth demand, reduce over-provisioning       │
│ Compute Consolidation     │ Efficient resource usage                       │
│ Static Content Hosting    │ Reduce compute costs                           │
└────────────────────────────┴───────────────────────────────────────────────┘
```

## Chapter Contents

### Quick Reference
- [Quick Reference](quick-reference.md) - Pattern decision matrix and cheat sheet

### Deep Dive Topics
1. [Reliability Patterns](01-reliability-patterns.md) - Circuit breaker, retry, bulkhead
2. [Messaging Patterns](02-messaging-patterns.md) - Queues, pub/sub, saga
3. [Data Patterns](03-data-patterns.md) - CQRS, event sourcing, sharding
4. [Gateway Patterns](04-gateway-patterns.md) - API gateway, BFF, aggregation
5. [Deployment Patterns](05-deployment-patterns.md) - Stamps, sidecar, strangler fig

## Pattern Selection Guide

```
WHEN TO USE WHICH PATTERN:
──────────────────────────

"My service calls are failing intermittently"
→ Retry Pattern + Circuit Breaker

"My database is a bottleneck for reads"
→ Cache-Aside + CQRS + Read Replicas

"I need to process messages reliably"
→ Queue-Based Load Leveling + Competing Consumers

"I need to handle traffic spikes"
→ Queue-Based Load Leveling + Throttling + Autoscaling

"I'm migrating from a monolith"
→ Strangler Fig + Anti-Corruption Layer

"I need to coordinate distributed transactions"
→ Saga Pattern + Compensating Transaction

"I want to decouple services"
→ Publisher/Subscriber + Event-Driven Architecture

"I need multi-region deployment"
→ Deployment Stamps + Geode

"I need to handle large messages"
→ Claim Check Pattern

"I need to validate external data"
→ Quarantine Pattern
```

## AWS to Azure Pattern Implementation

```
PATTERN IMPLEMENTATION MAPPING:
──────────────────────────────

┌─────────────────────┬────────────────────────┬──────────────────────────────┐
│ Pattern             │ AWS Implementation     │ Azure Implementation          │
├─────────────────────┼────────────────────────┼──────────────────────────────┤
│ Circuit Breaker     │ App code + CloudWatch  │ App code + App Insights      │
│ Queue Load Leveling │ SQS + Lambda           │ Service Bus + Functions      │
│ Pub/Sub             │ SNS + SQS              │ Event Grid + Service Bus     │
│ Cache-Aside         │ ElastiCache            │ Azure Cache for Redis        │
│ CQRS                │ DynamoDB + Aurora      │ Cosmos DB + SQL Database     │
│ Saga                │ Step Functions         │ Durable Functions            │
│ API Gateway         │ API Gateway            │ API Management               │
│ Sidecar             │ ECS/EKS sidecar        │ AKS sidecar / Container Apps │
│ Federated Identity  │ Cognito + SAML         │ Entra ID + B2C               │
│ Valet Key           │ Pre-signed URLs        │ SAS Tokens                   │
│ Static Hosting      │ S3 + CloudFront        │ Blob Storage + CDN           │
│ Event Sourcing      │ Kinesis + DynamoDB     │ Event Hubs + Cosmos DB       │
└─────────────────────┴────────────────────────┴──────────────────────────────┘
```

## Learning Path

```
RECOMMENDED STUDY ORDER:
────────────────────────

Week 1: Reliability Foundation
├── Study: Circuit Breaker, Retry, Bulkhead
├── Implement: Add Polly to .NET app
├── Azure: Configure App Insights for failure tracking
└── Practice: Simulate failures and test resilience

Week 2: Messaging & Async
├── Study: Queue Load Leveling, Pub/Sub, Competing Consumers
├── Implement: Service Bus queue processing
├── Azure: Set up Event Grid topics
└── Practice: Build event-driven workflow

Week 3: Data Patterns
├── Study: Cache-Aside, CQRS, Event Sourcing
├── Implement: Redis caching layer
├── Azure: Configure read replicas
└── Practice: Separate read/write paths

Week 4: Gateway & Deployment
├── Study: API Gateway patterns, Sidecar, Strangler Fig
├── Implement: API Management policies
├── Azure: Deploy multi-region stamp
└── Practice: Migrate legacy endpoint with strangler
```

## Key Patterns Summary

| Pattern | Problem Solved | Azure Services |
|---------|----------------|----------------|
| Circuit Breaker | Cascade failures | Polly + App Insights |
| Retry | Transient failures | Polly + SDK retries |
| Cache-Aside | Database bottleneck | Azure Cache for Redis |
| CQRS | Read/write contention | Cosmos DB + SQL |
| Saga | Distributed transactions | Durable Functions |
| Queue Load Leveling | Traffic spikes | Service Bus |
| Pub/Sub | Service coupling | Event Grid |
| Sidecar | Cross-cutting concerns | AKS, Container Apps |
| Strangler Fig | Legacy migration | API Management |
| Deployment Stamps | Regional scaling | Bicep/Terraform modules |

---

*Next: [Quick Reference](quick-reference.md)* | *Back to [Chapter 07: FinOps](../07-finops/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
