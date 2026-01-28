# Chapter 04: Observability

## Overview

Azure's observability stack provides comprehensive monitoring, logging, and diagnostics across your entire environment. For AWS engineers, think of this as CloudWatch, X-Ray, and CloudTrail unified into a single platform with a powerful query language.

## AWS to Azure Mapping

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY: AWS vs AZURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS                                   AZURE                                │
│  ───                                   ─────                                │
│                                                                              │
│  CloudWatch Metrics      ──────────▶   Azure Monitor Metrics                │
│  CloudWatch Logs         ──────────▶   Log Analytics (Azure Monitor Logs)   │
│  CloudWatch Alarms       ──────────▶   Azure Monitor Alerts                 │
│  CloudWatch Dashboards   ──────────▶   Azure Dashboards / Workbooks        │
│  CloudWatch Insights     ──────────▶   Log Analytics + KQL                  │
│  X-Ray                   ──────────▶   Application Insights                 │
│  CloudTrail              ──────────▶   Activity Log + Diagnostic Settings  │
│  EventBridge             ──────────▶   Event Grid + Azure Monitor           │
│  AWS Health              ──────────▶   Azure Service Health                 │
│                                                                              │
│  KEY DIFFERENCE:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  AWS: Separate services with different query syntaxes                │   │
│  │       CloudWatch Logs Insights vs X-Ray vs CloudTrail queries       │   │
│  │                                                                       │   │
│  │  Azure: Unified platform with ONE query language (KQL)              │   │
│  │         Same KQL for logs, metrics, traces, and security            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Azure Monitor Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AZURE MONITOR ARCHITECTURE                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  DATA SOURCES                         AZURE MONITOR                         │
│  ────────────                         ─────────────                         │
│                                                                              │
│  ┌─────────────────┐                 ┌───────────────────────────────────┐ │
│  │ Applications    │────────────────▶│                                   │ │
│  │ (App Insights)  │                 │     ┌─────────────────────────┐   │ │
│  └─────────────────┘                 │     │       METRICS           │   │ │
│                                      │     │  (Time-series data)     │   │ │
│  ┌─────────────────┐                 │     │  • Platform metrics     │   │ │
│  │ Infrastructure  │────────────────▶│     │  • Custom metrics       │   │ │
│  │ (VMs, Storage)  │                 │     │  • 93 days retention    │   │ │
│  └─────────────────┘                 │     └─────────────────────────┘   │ │
│                                      │                                   │ │
│  ┌─────────────────┐                 │     ┌─────────────────────────┐   │ │
│  │ Azure Platform  │────────────────▶│     │        LOGS             │   │ │
│  │ (Activity logs) │                 │     │  (Log Analytics)        │   │ │
│  └─────────────────┘                 │     │  • Resource logs        │   │ │
│                                      │     │  • Activity logs        │   │ │
│  ┌─────────────────┐                 │     │  • App traces           │   │ │
│  │ Operating System│────────────────▶│     │  • 2 years+ retention   │   │ │
│  │ (Agents)        │                 │     └─────────────────────────┘   │ │
│  └─────────────────┘                 │                                   │ │
│                                      │     ┌─────────────────────────┐   │ │
│  ┌─────────────────┐                 │     │  APPLICATION INSIGHTS   │   │ │
│  │ Custom Sources  │────────────────▶│     │  (APM - Distributed     │   │ │
│  │ (APIs, SDKs)    │                 │     │   tracing & telemetry)  │   │ │
│  └─────────────────┘                 │     └─────────────────────────┘   │ │
│                                      │                                   │ │
│                                      └─────────────────┬─────────────────┘ │
│                                                        │                   │
│                                                        ▼                   │
│  CONSUMPTION                                                               │
│  ───────────                                                               │
│  ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ ┌───────────────┐ │
│  │   INSIGHTS    │ │    ALERTS     │ │  DASHBOARDS   │ │   EXPORT      │ │
│  │               │ │               │ │               │ │               │ │
│  │ • VM Insights │ │ • Metric      │ │ • Azure       │ │ • Event Hub   │ │
│  │ • Container   │ │ • Log         │ │ • Workbooks   │ │ • Storage     │ │
│  │   Insights    │ │ • Activity    │ │ • Grafana     │ │ • SIEM        │ │
│  │ • Network     │ │ • Smart       │ │ • Power BI    │ │ • Third-party │ │
│  │   Insights    │ │   Detection   │ │               │ │               │ │
│  └───────────────┘ └───────────────┘ └───────────────┘ └───────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Chapter Contents

### Quick Reference
- [Quick Reference](quick-reference.md) - Essential commands, KQL queries, and comparison tables

### Deep Dive Topics
1. [Azure Monitor Fundamentals](01-azure-monitor.md) - Metrics, logs, and platform architecture
2. [Log Analytics & KQL](02-log-analytics-kql.md) - Workspace design and query mastery
3. [Application Insights](03-application-insights.md) - APM, distributed tracing, and availability
4. [Alerting & Automation](04-alerting.md) - Alert rules, action groups, and auto-remediation
5. [Case Studies](case-studies.md) - Real-world observability architectures

## Key Concepts for AWS Engineers

### Metric vs Log Data

```
WHEN TO USE METRICS vs LOGS:
────────────────────────────

METRICS (Time-series):
┌────────────────────────────────────────────────────────────────────────────┐
│ • Numbers over time (CPU %, request count, latency)                         │
│ • Fast queries (pre-aggregated)                                             │
│ • 93-day default retention                                                  │
│ • Best for: Dashboards, real-time alerting, autoscaling                    │
│ • Cost: Lower (included with resources)                                     │
│                                                                             │
│ AWS Equivalent: CloudWatch Metrics                                          │
└────────────────────────────────────────────────────────────────────────────┘

LOGS (Event data):
┌────────────────────────────────────────────────────────────────────────────┐
│ • Text/structured events (errors, requests, audit trails)                  │
│ • Flexible queries with KQL                                                 │
│ • Configurable retention (up to 2 years interactive, 12 years archive)    │
│ • Best for: Debugging, compliance, complex analysis                        │
│ • Cost: Pay per GB ingested + retention                                    │
│                                                                             │
│ AWS Equivalent: CloudWatch Logs                                             │
└────────────────────────────────────────────────────────────────────────────┘

TRACES (Distributed):
┌────────────────────────────────────────────────────────────────────────────┐
│ • Request flow across services                                              │
│ • Parent-child relationships                                                │
│ • Stored in Application Insights                                           │
│ • Best for: Debugging microservices, finding bottlenecks                   │
│                                                                             │
│ AWS Equivalent: X-Ray                                                       │
└────────────────────────────────────────────────────────────────────────────┘
```

### Log Analytics Workspace Design

```
WORKSPACE DESIGN PATTERNS:
──────────────────────────

PATTERN 1: CENTRALIZED (Recommended for most)
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              CENTRAL LOG ANALYTICS WORKSPACE                          │   │
│  │                                                                       │   │
│  │  All subscriptions → One workspace                                   │   │
│  │  • Single pane of glass                                              │   │
│  │  • Cross-resource queries                                            │   │
│  │  • Simplified RBAC at workspace level                               │   │
│  │  • Cost optimization (commitment tiers)                              │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Best for: Most organizations, unified operations teams                    │
└────────────────────────────────────────────────────────────────────────────┘

PATTERN 2: REGIONAL
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐  │
│  │  East US          │    │  West Europe      │    │  Southeast Asia   │  │
│  │  Workspace        │    │  Workspace        │    │  Workspace        │  │
│  └───────────────────┘    └───────────────────┘    └───────────────────┘  │
│                                                                             │
│  • Data sovereignty requirements                                           │
│  • Reduced egress costs                                                    │
│  • Regional operations teams                                               │
│                                                                             │
│  Best for: Global enterprises, data residency compliance                   │
└────────────────────────────────────────────────────────────────────────────┘

PATTERN 3: ENVIRONMENT-BASED
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  ┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐  │
│  │  Production       │    │  Non-Production   │    │  Security         │  │
│  │  Workspace        │    │  Workspace        │    │  Workspace        │  │
│  │                   │    │  (Dev, Test, QA)  │    │  (Sentinel)       │  │
│  └───────────────────┘    └───────────────────┘    └───────────────────┘  │
│                                                                             │
│  • Strict prod/non-prod separation                                         │
│  • Different retention policies                                            │
│  • Security team has dedicated workspace                                   │
│                                                                             │
│  Best for: Regulated industries, strict access controls                    │
└────────────────────────────────────────────────────────────────────────────┘
```

## Learning Path

```
RECOMMENDED STUDY ORDER:
────────────────────────

Week 1: Fundamentals
├── Read: Quick Reference (commands and comparisons)
├── Do: Create Log Analytics workspace
├── Do: Enable diagnostic settings for 3 resources
└── Practice: 5 basic KQL queries

Week 2: Log Analytics & KQL
├── Read: Log Analytics & KQL deep dive
├── Do: Configure VM Insights
├── Do: Create custom log table
└── Practice: 20 intermediate KQL queries

Week 3: Application Insights
├── Read: Application Insights deep dive
├── Do: Instrument a sample application
├── Do: Create availability test
└── Practice: Distributed tracing analysis

Week 4: Alerting & Integration
├── Read: Alerting & Automation deep dive
├── Do: Create metric and log alerts
├── Do: Configure auto-remediation
├── Review: Case studies
└── Practice: Build complete monitoring solution
```

## Key Services Summary

| Service | Purpose | AWS Equivalent | Cost Model |
|---------|---------|----------------|------------|
| Azure Monitor | Platform for all monitoring | CloudWatch | Per resource (included) |
| Log Analytics | Log storage and query | CloudWatch Logs | Per GB ingested |
| Application Insights | APM and tracing | X-Ray + CloudWatch | Per GB + features |
| Azure Monitor Alerts | Alerting engine | CloudWatch Alarms | Per alert rule |
| Azure Workbooks | Interactive reports | CloudWatch Dashboards | Free |
| Azure Dashboards | Overview dashboards | CloudWatch Dashboards | Free |

## Cost Optimization Tips

```
LOG ANALYTICS COST OPTIMIZATION:
────────────────────────────────

1. Use commitment tiers (100GB+/day = 15% discount)
2. Configure appropriate retention (not everything needs 2 years)
3. Use Basic logs for high-volume, low-query data
4. Filter at source (don't ingest what you won't query)
5. Archive old logs to storage account
6. Use sampling in Application Insights for high-traffic apps

TYPICAL COSTS (approximate):
• Log ingestion: ~$2.76/GB
• Log retention beyond 31 days: ~$0.12/GB/month
• Basic logs: ~$0.65/GB (limited query)
• Commitment tier 100GB/day: ~$196/day (vs $276 pay-as-you-go)
```

---

*Next: [Quick Reference](quick-reference.md)* | *Back to [Chapter 03: Compliance](../03-compliance/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
