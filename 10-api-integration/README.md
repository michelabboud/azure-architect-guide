# Chapter 14: API Integration

## Overview

This chapter covers Azure services for API management, integration, and messaging. These services enable you to build connected applications, manage APIs, and implement event-driven architectures.

## AWS Architect Quick Reference

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| API Gateway | Azure API Management | Full lifecycle management |
| Amazon MQ | Azure Service Bus | Queues + Topics + Relay |
| Amazon SQS | Azure Queue Storage / Service Bus | Service Bus for enterprise |
| Amazon SNS | Azure Event Grid / Service Bus Topics | Event Grid for events |
| EventBridge | Azure Event Grid | Native Azure integration |
| Step Functions | Azure Logic Apps / Durable Functions | Visual designer vs code |
| AppFlow | Azure Logic Apps + Connectors | 400+ connectors |

## Chapter Contents

### [Quick Reference](quick-reference.md)
- CLI commands
- Common patterns
- Code snippets

### [01 - API Management](01-api-management.md)
- APIM tiers and features
- Policies and transformations
- Developer portal
- Security patterns

### [02 - Service Bus](02-service-bus.md)
- Queues and topics
- Message sessions
- Dead-letter handling
- Geo-disaster recovery

### [03 - Event Grid](03-event-grid.md)
- Event sources and handlers
- Custom topics
- Event domains
- Filtering and routing

### [04 - Logic Apps](04-logic-apps.md)
- Workflow design
- Connectors
- Integration accounts
- B2B scenarios

## Architecture Patterns

### API Gateway Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         API Gateway Pattern                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   Clients                                                                    │
│   ───────                                                                    │
│   ┌────────┐  ┌────────┐  ┌────────┐                                        │
│   │ Mobile │  │  Web   │  │Partner │                                        │
│   │  App   │  │  App   │  │  API   │                                        │
│   └───┬────┘  └───┬────┘  └───┬────┘                                        │
│       │           │           │                                              │
│       └───────────┼───────────┘                                              │
│                   │                                                          │
│                   ▼                                                          │
│   ┌───────────────────────────────────────────────────────────────┐        │
│   │                Azure API Management                            │        │
│   │  ┌─────────────┬─────────────┬─────────────┬─────────────┐   │        │
│   │  │   Rate      │   Auth      │  Transform  │   Cache     │   │        │
│   │  │  Limiting   │   (OAuth)   │  (XML→JSON) │   (Redis)   │   │        │
│   │  └─────────────┴─────────────┴─────────────┴─────────────┘   │        │
│   │                                                                │        │
│   │  Policies: Inbound → Backend → Outbound → On-Error            │        │
│   └───────────────────────────────┬───────────────────────────────┘        │
│                                   │                                          │
│          ┌────────────────────────┼────────────────────────┐                │
│          │                        │                        │                │
│          ▼                        ▼                        ▼                │
│   ┌─────────────┐          ┌─────────────┐          ┌─────────────┐        │
│   │  Orders     │          │  Products   │          │  Customers  │        │
│   │  Service    │          │  Service    │          │  Service    │        │
│   │  (AKS)      │          │  (Functions)│          │  (App Svc)  │        │
│   └─────────────┘          └─────────────┘          └─────────────┘        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Event-Driven Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     Event-Driven Architecture                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Event Sources                                                               │
│  ─────────────                                                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │ Storage  │ │ Service  │ │   IoT    │ │  Custom  │                        │
│  │ Account  │ │   Bus    │ │   Hub    │ │   App    │                        │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘                        │
│       │            │            │            │                               │
│       └────────────┼────────────┼────────────┘                               │
│                    │            │                                            │
│                    ▼            ▼                                            │
│            ┌──────────────────────────────────┐                             │
│            │        Azure Event Grid          │                             │
│            │                                  │                             │
│            │  • Push-based delivery           │                             │
│            │  • Filtering by event type       │                             │
│            │  • Fan-out to multiple handlers  │                             │
│            │  • Retry with dead-letter        │                             │
│            └──────────────┬───────────────────┘                             │
│                           │                                                  │
│        ┌──────────────────┼──────────────────┐                              │
│        │                  │                  │                              │
│        ▼                  ▼                  ▼                              │
│  ┌──────────┐      ┌──────────┐      ┌──────────┐                          │
│  │ Azure    │      │  Logic   │      │  Event   │                          │
│  │ Function │      │   App    │      │   Hub    │                          │
│  │(Process) │      │(Workflow)│      │(Stream)  │                          │
│  └──────────┘      └──────────┘      └──────────┘                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Message Queue Pattern

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Message Queue Pattern                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Producer                  Service Bus Queue                Consumer        │
│  ────────                  ─────────────────                ────────        │
│                                                                              │
│  ┌──────────┐             ┌──────────────────┐            ┌──────────┐     │
│  │  Order   │             │                  │            │  Order   │     │
│  │ Service  │────Send────▶│    ═══════════   │───Receive──│ Processor│     │
│  │          │             │    ═══════════   │            │          │     │
│  └──────────┘             │    ═══════════   │            └──────────┘     │
│                           │                  │                              │
│                           │  ┌────────────┐  │                              │
│                           │  │Dead Letter │  │  Failed messages             │
│                           │  │   Queue    │  │  for investigation           │
│                           │  └────────────┘  │                              │
│                           └──────────────────┘                              │
│                                                                              │
│  Features:                                                                   │
│  • At-least-once / At-most-once delivery                                    │
│  • Message sessions (FIFO per session)                                      │
│  • Scheduled delivery                                                        │
│  • Auto-forwarding                                                           │
│  • Duplicate detection                                                       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Service Comparison

| Feature | Storage Queue | Service Bus Queue | Event Grid | Event Hub |
|---------|--------------|-------------------|------------|-----------|
| Max Message | 64KB | 256KB/1MB | 1MB | 1MB |
| Ordering | No | Yes (sessions) | No | Partition-level |
| Delivery | At-least-once | At-least/most-once | At-least-once | At-least-once |
| Throughput | High | Medium-High | High | Very High |
| Cost | $ | $$ | $ | $$-$$$ |
| Use Case | Simple queuing | Enterprise messaging | Events | Streaming |

## When to Use What

```
┌────────────────────────────────────────────────────────────────────┐
│                    Integration Service Selection                    │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Need API Management?                                               │
│  └── Yes ──► Azure API Management                                  │
│                                                                     │
│  Need point-to-point messaging?                                     │
│  └── Simple ──► Storage Queue                                      │
│  └── Enterprise features ──► Service Bus Queue                     │
│                                                                     │
│  Need pub/sub?                                                      │
│  └── Few subscribers ──► Service Bus Topics                        │
│  └── Many subscribers + events ──► Event Grid                      │
│                                                                     │
│  Need high-throughput streaming?                                    │
│  └── Yes ──► Event Hubs                                            │
│                                                                     │
│  Need workflow orchestration?                                       │
│  └── Visual designer ──► Logic Apps                                │
│  └── Code-first ──► Durable Functions                              │
│                                                                     │
│  Need B2B integration?                                              │
│  └── Yes ──► Logic Apps + Integration Account                      │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

## Learning Objectives

After completing this chapter, you will:

1. **Design** API-first architectures using APIM
2. **Implement** reliable messaging with Service Bus
3. **Build** event-driven systems with Event Grid
4. **Create** integration workflows with Logic Apps
5. **Choose** the right messaging service for each scenario
6. **Map** AWS integration patterns to Azure equivalents

---

*Continue to [Quick Reference](quick-reference.md)*

*Back to [Main Guide](../README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
