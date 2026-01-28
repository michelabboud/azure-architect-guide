# Azure Logic Apps

## What is Azure Logic Apps?

Azure Logic Apps is a cloud-based platform for creating and running automated workflows that integrate apps, data, services, and systems. It provides a visual designer and 400+ pre-built connectors for enterprise integration scenarios.

### Logic Apps vs AWS Step Functions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   LOGIC APPS vs AWS STEP FUNCTIONS                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS STEP FUNCTIONS                      AZURE LOGIC APPS                   │
│  ──────────────────                      ────────────────                   │
│                                                                              │
│  Approach:                               Approach:                          │
│  • Code-first (ASL JSON)                • Visual designer first            │
│  • State machine pattern                • Workflow pattern                 │
│  • Tightly coupled to Lambda            • 400+ connectors                  │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                                                                      │    │
│  │  STEP FUNCTIONS WORKFLOW:             LOGIC APPS WORKFLOW:          │    │
│  │                                                                      │    │
│  │  {                                    Visual Designer:               │    │
│  │    "StartAt": "ProcessOrder",         ┌─────────────────┐           │    │
│  │    "States": {                        │ When HTTP request│           │    │
│  │      "ProcessOrder": {                │    received     │           │    │
│  │        "Type": "Task",                └────────┬────────┘           │    │
│  │        "Resource": "arn:aws:          ┌────────▼────────┐           │    │
│  │          lambda:...",                 │   Parse JSON    │           │    │
│  │        "Next": "SendEmail"            └────────┬────────┘           │    │
│  │      },                               ┌────────▼────────┐           │    │
│  │      "SendEmail": {...}               │  Condition      │           │    │
│  │    }                                  └────┬───────┬────┘           │    │
│  │  }                                         │       │                │    │
│  │                                       ┌────▼──┐ ┌──▼────┐           │    │
│  │                                       │ Yes  │ │  No   │           │    │
│  │                                       └───────┘ └───────┘           │    │
│  │                                                                      │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
│  FEATURE COMPARISON:                                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │ Feature           │ Step Functions       │ Logic Apps              │  │
│  ├───────────────────┼──────────────────────┼─────────────────────────┤  │
│  │ Design approach   │ JSON/YAML (ASL)      │ Visual + JSON          │  │
│  │                   │                      │                         │  │
│  │ Connectors        │ AWS services +       │ 400+ built-in          │  │
│  │                   │ HTTP/Lambda          │ (SaaS, on-prem, custom)│  │
│  │                   │                      │                         │  │
│  │ Hosting           │ Serverless only      │ Consumption (serverless)│  │
│  │                   │                      │ Standard (dedicated)    │  │
│  │                   │                      │ ISE (isolated)          │  │
│  │                   │                      │                         │  │
│  │ Execution time    │ 1 year (Standard)    │ 90 days (Consumption)  │  │
│  │                   │ 5 min (Express)      │ Unlimited (Standard)   │  │
│  │                   │                      │                         │  │
│  │ B2B Integration   │ No built-in          │ Integration Accounts   │  │
│  │                   │                      │ (EDI, AS2, X12)        │  │
│  │                   │                      │                         │  │
│  │ Error handling    │ Catch/Retry states   │ Scope + Run After      │  │
│  │                   │                      │ Try-Catch patterns     │  │
│  │                   │                      │                         │  │
│  │ Pricing           │ Per state transition │ Per action execution   │  │
│  │                   │ $0.025/1000          │ $0.000025/action       │  │
│  │                   │                      │ + connector cost       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Logic Apps Hosting Options

### Plan Comparison

| Feature | Consumption | Standard | ISE (Deprecated) |
|---------|-------------|----------|-------------------|
| **Hosting** | Serverless | Dedicated | Isolated |
| **Scale** | Auto (0-N) | Manual/Auto | Manual |
| **Networking** | Public | VNet integration | Dedicated VNet |
| **Pricing** | Per execution | Per vCPU hour | Per scale unit |
| **Workflows per resource** | 1 | Multiple | Multiple |
| **Development** | Portal only | VS Code + Portal | Portal |
| **Stateful + Stateless** | Stateful only | Both | Stateful only |
| **Local debugging** | No | Yes | No |
| **Execution time** | 90 days | Unlimited | 90 days |
| **SLA** | 99.9% | 99.95% | 99.95% |

### When to Use Each

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    LOGIC APPS HOSTING SELECTION                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  CONSUMPTION (Serverless):                                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Best for:                                                            │   │
│  │ • Infrequent or unpredictable workloads                             │   │
│  │ • Quick prototyping and simple integrations                        │   │
│  │ • Cost-sensitive scenarios (pay-per-use)                           │   │
│  │ • Event-driven workflows with variable load                        │   │
│  │                                                                      │   │
│  │ Limitations:                                                        │   │
│  │ • No VNet integration                                               │   │
│  │ • No local development                                              │   │
│  │ • 90-day max execution                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  STANDARD (Dedicated):                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Best for:                                                            │   │
│  │ • Production enterprise workloads                                   │   │
│  │ • VNet connectivity requirements                                    │   │
│  │ • Multiple related workflows (like microservices)                  │   │
│  │ • Local development with VS Code                                   │   │
│  │ • Stateless workflows (high-performance)                           │   │
│  │ • DevOps/CI-CD integration                                         │   │
│  │                                                                      │   │
│  │ Advantages:                                                         │   │
│  │ • Run on Azure Functions runtime                                   │   │
│  │ • Deploy via ZIP, container, or ARM/Bicep                          │   │
│  │ • Better cost predictability                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Workflow Design Patterns

### Common Patterns

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    WORKFLOW DESIGN PATTERNS                                 │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. SEQUENTIAL WORKFLOW:                                                   │
│  ───────────────────────                                                   │
│                                                                             │
│  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐                   │
│  │ Trigger │──▶│ Action1 │──▶│ Action2 │──▶│ Action3 │                   │
│  └─────────┘   └─────────┘   └─────────┘   └─────────┘                   │
│                                                                             │
│  Use: Simple integrations, data processing pipelines                       │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────│
│                                                                             │
│  2. PARALLEL BRANCHES:                                                     │
│  ─────────────────────                                                     │
│                                                                             │
│                    ┌─────────────────┐                                     │
│                    │    Trigger      │                                     │
│                    └────────┬────────┘                                     │
│              ┌──────────────┼──────────────┐                               │
│              ▼              ▼              ▼                               │
│        ┌─────────┐   ┌─────────┐   ┌─────────┐                           │
│        │ Branch1 │   │ Branch2 │   │ Branch3 │   (parallel)              │
│        └────┬────┘   └────┬────┘   └────┬────┘                           │
│              └──────────────┼──────────────┘                               │
│                             ▼                                               │
│                    ┌─────────────────┐                                     │
│                    │ Join (wait all) │                                     │
│                    └─────────────────┘                                     │
│                                                                             │
│  Use: Fan-out processing, calling multiple APIs simultaneously            │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────│
│                                                                             │
│  3. CONDITIONAL BRANCHING:                                                 │
│  ─────────────────────────                                                 │
│                                                                             │
│                    ┌─────────────────┐                                     │
│                    │    Trigger      │                                     │
│                    └────────┬────────┘                                     │
│                             ▼                                               │
│                    ┌─────────────────┐                                     │
│                    │   Condition     │                                     │
│                    └───────┬─────────┘                                     │
│                      True  │  False                                        │
│                  ┌─────────┴─────────┐                                     │
│                  ▼                   ▼                                     │
│           ┌───────────┐       ┌───────────┐                               │
│           │ If True   │       │ If False  │                               │
│           └───────────┘       └───────────┘                               │
│                                                                             │
│  Use: Business rules, routing decisions                                   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────│
│                                                                             │
│  4. FOR EACH LOOP:                                                         │
│  ─────────────────                                                         │
│                                                                             │
│  ┌─────────┐   ┌──────────────────────────────────┐   ┌─────────┐        │
│  │ Trigger │──▶│ For Each item in array           │──▶│ Continue│        │
│  └─────────┘   │   ┌─────────┐   ┌─────────┐     │   └─────────┘        │
│                │   │Process 1│   │Process 2│     │                       │
│                │   └─────────┘   └─────────┘     │                       │
│                └──────────────────────────────────┘                       │
│                                                                             │
│  Use: Processing collections, batch operations                            │
│  Note: Concurrent by default (set Sequential if needed)                   │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────│
│                                                                             │
│  5. UNTIL LOOP (Polling):                                                  │
│  ────────────────────────                                                  │
│                                                                             │
│  ┌─────────┐   ┌──────────────────────────────────┐   ┌─────────┐        │
│  │ Trigger │──▶│ Until condition = true           │──▶│ Continue│        │
│  └─────────┘   │   ┌─────────┐   ┌─────────┐     │   └─────────┘        │
│                │   │Check    │──▶│ Delay   │──┐  │                       │
│                │   │status   │   │ 30 sec  │  │  │                       │
│                │   └─────────┘   └─────────┘  │  │                       │
│                │        ▲                     │  │                       │
│                │        └─────────────────────┘  │                       │
│                └──────────────────────────────────┘                       │
│                                                                             │
│  Use: Polling for completion, retry with delay                            │
│                                                                             │
│  ─────────────────────────────────────────────────────────────────────────│
│                                                                             │
│  6. SCOPE (Try-Catch):                                                     │
│  ─────────────────────                                                     │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Scope: Try                                                           │   │
│  │   ┌─────────┐   ┌─────────┐   ┌─────────┐                          │   │
│  │   │ Action1 │──▶│ Action2 │──▶│ Action3 │                          │   │
│  │   └─────────┘   └─────────┘   └─────────┘                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Scope: Catch (runAfter: Try = Failed)                               │   │
│  │   ┌─────────┐   ┌─────────┐                                        │   │
│  │   │  Log    │──▶│  Alert  │                                        │   │
│  │   │  Error  │   │  Team   │                                        │   │
│  │   └─────────┘   └─────────┘                                        │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  Use: Error handling, compensation logic                                  │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Retry Pattern

```json
{
    "actions": {
        "HTTP_Call_API": {
            "type": "Http",
            "inputs": {
                "method": "POST",
                "uri": "https://api.example.com/orders",
                "body": "@body('Parse_JSON')"
            },
            "retryPolicy": {
                "type": "exponential",
                "count": 4,
                "interval": "PT10S",
                "minimumInterval": "PT5S",
                "maximumInterval": "PT1H"
            }
        }
    }
}
```

---

## Connectors

### Connector Categories

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    LOGIC APPS CONNECTORS (400+)                             │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  AZURE SERVICES:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Service Bus, Event Grid, Event Hubs, Storage, Cosmos DB,            │   │
│  │ Functions, Key Vault, Log Analytics, App Configuration,            │   │
│  │ Azure SQL, Data Lake, IoT Hub, Cognitive Services                  │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  MICROSOFT 365:                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Outlook, Teams, SharePoint, OneDrive, Excel, Planner,              │   │
│  │ Dynamics 365, Power BI, Power Automate                             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ENTERPRISE:                                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ SAP, Salesforce, ServiceNow, Oracle, IBM MQ, SFTP,                 │   │
│  │ FTP, AS2, X12, EDIFACT, RosettaNet                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SOCIAL & PRODUCTIVITY:                                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Slack, Twitter, Facebook, Gmail, Google Drive, Dropbox,            │   │
│  │ Box, Trello, Asana, GitHub, Jira, Zendesk                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  DATA:                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ SQL Server, MySQL, PostgreSQL, MongoDB, Snowflake,                 │   │
│  │ Teradata, Amazon Redshift, Google BigQuery                         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CUSTOM:                                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ HTTP (REST/SOAP), Custom Connectors (OpenAPI), Azure Functions,    │   │
│  │ On-premises Data Gateway                                           │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CONNECTOR TYPES:                                                          │
│  ─────────────────                                                         │
│                                                                             │
│  ┌────────────────────────────────────────────────────────────────┐        │
│  │ Type          │ Description              │ Examples            │        │
│  ├───────────────┼──────────────────────────┼─────────────────────┤        │
│  │ Built-in      │ Run in same process      │ HTTP, Schedule,     │        │
│  │               │ Best performance         │ Variables, Compose  │        │
│  │               │ No extra cost            │                     │        │
│  │               │                          │                     │        │
│  │ Standard      │ Managed by Microsoft     │ Office 365, SQL,    │        │
│  │               │ Includes most connectors │ SharePoint, Teams   │        │
│  │               │                          │                     │        │
│  │ Enterprise    │ Additional licensing     │ SAP, IBM MQ         │        │
│  │               │ B2B capabilities         │ X12, EDIFACT        │        │
│  │               │                          │                     │        │
│  │ Custom        │ Your own connectors      │ OpenAPI-based       │        │
│  │               │ Or from marketplace      │                     │        │
│  └────────────────────────────────────────────────────────────────┘        │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Connector Pricing (Consumption Plan)

| Connector Type | Price per Action |
|----------------|------------------|
| Built-in | Included in base |
| Standard | ~$0.000025/action |
| Enterprise | ~$0.001/action |

---

## Integration Accounts for B2B

### B2B Integration Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    B2B INTEGRATION WITH INTEGRATION ACCOUNTS               │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  INTEGRATION ACCOUNT COMPONENTS:                                           │
│  ───────────────────────────────                                           │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    INTEGRATION ACCOUNT                               │   │
│  │                                                                      │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐           │   │
│  │  │   PARTNERS    │  │  AGREEMENTS   │  │    SCHEMAS    │           │   │
│  │  │               │  │               │  │               │           │   │
│  │  │ • Contoso     │  │ • X12 (EDI)   │  │ • X12 schemas │           │   │
│  │  │ • Fabrikam    │  │ • EDIFACT     │  │ • EDIFACT     │           │   │
│  │  │ • Northwind   │  │ • AS2         │  │ • Custom XSD  │           │   │
│  │  │               │  │ • RosettaNet  │  │               │           │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘           │   │
│  │                                                                      │   │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐           │   │
│  │  │     MAPS      │  │ CERTIFICATES  │  │  ASSEMBLIES   │           │   │
│  │  │               │  │               │  │               │           │   │
│  │  │ • XSLT        │  │ • Public keys │  │ • Custom .NET │           │   │
│  │  │ • Liquid      │  │ • Private keys│  │   assemblies  │           │   │
│  │  │               │  │ • CA certs    │  │               │           │   │
│  │  └───────────────┘  └───────────────┘  └───────────────┘           │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  B2B MESSAGE FLOW (EDI Example):                                           │
│  ───────────────────────────────                                           │
│                                                                             │
│  Partner A                                             Partner B            │
│  (Sender)                                              (Receiver)           │
│     │                                                      ▲               │
│     │ X12 850 (Purchase Order)                             │               │
│     ▼                                                      │               │
│  ┌────────────────────────────────────────────────────────────────────┐   │
│  │                         LOGIC APP                                   │   │
│  │                                                                     │   │
│  │  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐           │   │
│  │  │ AS2     │──▶│ X12     │──▶│Transform│──▶│ AS2     │           │   │
│  │  │ Decode  │   │ Decode  │   │  (XSLT) │   │ Encode  │           │   │
│  │  └─────────┘   └─────────┘   └─────────┘   └─────────┘           │   │
│  │       │              │             │              │               │   │
│  │       │              │             │              │               │   │
│  │       ▼              ▼             ▼              ▼               │   │
│  │  ┌─────────────────────────────────────────────────────────────┐ │   │
│  │  │              INTEGRATION ACCOUNT                             │ │   │
│  │  │  (Certificates, Schemas, Agreements, Maps)                  │ │   │
│  │  └─────────────────────────────────────────────────────────────┘ │   │
│  │                                                                     │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  SUPPORTED STANDARDS:                                                      │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ • X12 (ANSI ASC X12) - US EDI standard                              │   │
│  │ • EDIFACT - International EDI standard                              │   │
│  │ • AS2 (Applicability Statement 2) - Secure HTTP transport          │   │
│  │ • RosettaNet - Supply chain standard                                │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Integration Account Tiers

| Feature | Free | Basic | Standard |
|---------|------|-------|----------|
| Partners | 25 | Unlimited | Unlimited |
| Agreements | 10 | Unlimited | Unlimited |
| Maps | 25 | 500 | 1000 |
| Schemas | 25 | 500 | 1000 |
| Assemblies | 10 | 25 | 50 |
| Certificates | 25 | 500 | 1000 |
| Price | Free | ~$300/mo | ~$1,200/mo |

---

## Enterprise Integration Scenarios

### Scenario 1: Order Processing

```
┌────────────────────────────────────────────────────────────────────────────┐
│              SCENARIO: E-COMMERCE ORDER PROCESSING                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌──────────┐                                                              │
│  │ Web App  │──HTTP POST (Order JSON)                                      │
│  └──────────┘           │                                                   │
│                         ▼                                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    LOGIC APP: Order Processor                        │   │
│  │                                                                      │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ HTTP Trigger  │                                                  │   │
│  │  └───────┬───────┘                                                  │   │
│  │          ▼                                                          │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ Parse JSON    │                                                  │   │
│  │  └───────┬───────┘                                                  │   │
│  │          ▼                                                          │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ Validate Order│ (Custom connector or Azure Function)            │   │
│  │  └───────┬───────┘                                                  │   │
│  │          ▼                                                          │   │
│  │  ┌───────────────────────────────────────────────────────────────┐ │   │
│  │  │ Parallel Branch                                                │ │   │
│  │  │                                                                │ │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │ │   │
│  │  │  │ Save to     │  │ Send to     │  │ Check       │           │ │   │
│  │  │  │ Cosmos DB   │  │ Service Bus │  │ Inventory   │           │ │   │
│  │  │  │             │  │ (Fulfillment│  │ (SAP)       │           │ │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘           │ │   │
│  │  └───────────────────────────────────────────────────────────────┘ │   │
│  │          ▼                                                          │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ Condition:    │                                                  │   │
│  │  │ In Stock?     │                                                  │   │
│  │  └───────┬───────┘                                                  │   │
│  │    Yes   │   No                                                     │   │
│  │   ┌──────┴──────┐                                                   │   │
│  │   ▼             ▼                                                   │   │
│  │  ┌─────────┐  ┌─────────┐                                          │   │
│  │  │ Process │  │ Queue   │                                          │   │
│  │  │ Payment │  │ Backord │                                          │   │
│  │  │(Stripe) │  │         │                                          │   │
│  │  └────┬────┘  └────┬────┘                                          │   │
│  │       │            │                                                │   │
│  │       └────────────┘                                                │   │
│  │                ▼                                                    │   │
│  │  ┌─────────────────────────────────────────────────────────────┐   │   │
│  │  │ Send Confirmation Email (Office 365)                         │   │   │
│  │  └─────────────────────────────────────────────────────────────┘   │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Scenario 2: File Integration

```
┌────────────────────────────────────────────────────────────────────────────┐
│              SCENARIO: SFTP FILE PROCESSING                                 │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    LOGIC APP: File Processor                         │   │
│  │                                                                      │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ SFTP Trigger  │  (When file is added to /incoming folder)       │   │
│  │  │ [Recurrence]  │                                                  │   │
│  │  └───────┬───────┘                                                  │   │
│  │          ▼                                                          │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ Get File      │                                                  │   │
│  │  │ Content       │                                                  │   │
│  │  └───────┬───────┘                                                  │   │
│  │          ▼                                                          │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ Switch on     │                                                  │   │
│  │  │ File Extension│                                                  │   │
│  │  └───────┬───────┘                                                  │   │
│  │    .csv  │  .xml  .json                                             │   │
│  │   ┌──────┼──────┬──────┐                                            │   │
│  │   ▼      ▼      ▼      ▼                                            │   │
│  │ ┌─────┐┌─────┐┌─────┐┌─────┐                                       │   │
│  │ │Parse││Parse││Parse││Other│                                       │   │
│  │ │CSV  ││XML  ││JSON ││Error│                                       │   │
│  │ └──┬──┘└──┬──┘└──┬──┘└──┬──┘                                       │   │
│  │    └──────┴──────┴──────┘                                           │   │
│  │                ▼                                                    │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ Transform     │  (XSLT or Liquid template)                      │   │
│  │  │ to Standard   │                                                  │   │
│  │  └───────┬───────┘                                                  │   │
│  │          ▼                                                          │   │
│  │  ┌───────────────────────────────────────────────────────────────┐ │   │
│  │  │ For Each Record                                                │ │   │
│  │  │                                                                │ │   │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐           │ │   │
│  │  │  │ Upsert to   │──│ Send Event  │──│ Log to      │           │ │   │
│  │  │  │ SQL Server  │  │ to Event    │  │ Application │           │ │   │
│  │  │  │             │  │ Grid        │  │ Insights    │           │ │   │
│  │  │  └─────────────┘  └─────────────┘  └─────────────┘           │ │   │
│  │  └───────────────────────────────────────────────────────────────┘ │   │
│  │          ▼                                                          │   │
│  │  ┌───────────────┐                                                  │   │
│  │  │ Move File to  │                                                  │   │
│  │  │ /processed    │                                                  │   │
│  │  └───────────────┘                                                  │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Bicep/CLI Deployment Examples

### Complete Logic Apps Deployment (Bicep)

```bicep
// main.bicep - Complete Logic App deployment (Consumption)

@description('Location for all resources')
param location string = resourceGroup().location

@description('Name of the Logic App')
param logicAppName string = 'logic-${uniqueString(resourceGroup().id)}'

@description('Office 365 connection name')
param office365ConnectionName string = 'office365'

@description('Service Bus connection name')
param serviceBusConnectionName string = 'servicebus'

@description('Service Bus namespace connection string')
@secure()
param serviceBusConnectionString string

// API Connections (managed connectors)
resource office365Connection 'Microsoft.Web/connections@2016-06-01' = {
  name: office365ConnectionName
  location: location
  properties: {
    displayName: 'Office 365 Outlook'
    api: {
      id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'office365')
    }
    // Note: OAuth connections require consent in portal
  }
}

resource serviceBusConnection 'Microsoft.Web/connections@2016-06-01' = {
  name: serviceBusConnectionName
  location: location
  properties: {
    displayName: 'Service Bus'
    api: {
      id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'servicebus')
    }
    parameterValues: {
      connectionString: serviceBusConnectionString
    }
  }
}

// Logic App (Consumption)
resource logicApp 'Microsoft.Logic/workflows@2019-05-01' = {
  name: logicAppName
  location: location
  properties: {
    state: 'Enabled'
    definition: {
      '$schema': 'https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#'
      contentVersion: '1.0.0.0'
      parameters: {
        '$connections': {
          defaultValue: {}
          type: 'Object'
        }
      }
      triggers: {
        When_a_message_is_received_in_a_queue: {
          type: 'ApiConnection'
          inputs: {
            host: {
              connection: {
                name: '@parameters(\'$connections\')[\'servicebus\'][\'connectionId\']'
              }
            }
            method: 'get'
            path: '/@{encodeURIComponent(encodeURIComponent(\'orders\'))}/messages/head'
            queries: {
              queueType: 'Main'
            }
          }
          recurrence: {
            frequency: 'Minute'
            interval: 1
          }
        }
      }
      actions: {
        Parse_JSON: {
          type: 'ParseJson'
          inputs: {
            content: '@base64ToString(triggerBody()?[\'ContentData\'])'
            schema: {
              type: 'object'
              properties: {
                orderId: { type: 'string' }
                customerEmail: { type: 'string' }
                amount: { type: 'number' }
                items: {
                  type: 'array'
                  items: { type: 'string' }
                }
              }
            }
          }
          runAfter: {}
        }
        Condition_High_Value: {
          type: 'If'
          expression: {
            and: [
              {
                greater: [
                  '@body(\'Parse_JSON\')?[\'amount\']'
                  500
                ]
              }
            ]
          }
          actions: {
            Send_email_High_Value: {
              type: 'ApiConnection'
              inputs: {
                host: {
                  connection: {
                    name: '@parameters(\'$connections\')[\'office365\'][\'connectionId\']'
                  }
                }
                method: 'post'
                path: '/v2/Mail'
                body: {
                  To: 'sales@company.com'
                  Subject: 'High Value Order: @{body(\'Parse_JSON\')?[\'orderId\']}'
                  Body: '<p>Order Amount: $@{body(\'Parse_JSON\')?[\'amount\']}</p><p>Customer: @{body(\'Parse_JSON\')?[\'customerEmail\']}</p>'
                }
              }
            }
          }
          else: {
            actions: {
              Send_email_Standard: {
                type: 'ApiConnection'
                inputs: {
                  host: {
                    connection: {
                      name: '@parameters(\'$connections\')[\'office365\'][\'connectionId\']'
                    }
                  }
                  method: 'post'
                  path: '/v2/Mail'
                  body: {
                    To: '@body(\'Parse_JSON\')?[\'customerEmail\']'
                    Subject: 'Order Confirmation: @{body(\'Parse_JSON\')?[\'orderId\']}'
                    Body: '<p>Thank you for your order!</p>'
                  }
                }
              }
            }
          }
          runAfter: {
            Parse_JSON: ['Succeeded']
          }
        }
        Complete_the_message: {
          type: 'ApiConnection'
          inputs: {
            host: {
              connection: {
                name: '@parameters(\'$connections\')[\'servicebus\'][\'connectionId\']'
              }
            }
            method: 'delete'
            path: '/@{encodeURIComponent(encodeURIComponent(\'orders\'))}/messages/complete'
            queries: {
              lockToken: '@triggerBody()?[\'LockToken\']'
              queueType: 'Main'
            }
          }
          runAfter: {
            Condition_High_Value: ['Succeeded']
          }
        }
      }
      outputs: {}
    }
    parameters: {
      '$connections': {
        value: {
          servicebus: {
            connectionId: serviceBusConnection.id
            connectionName: serviceBusConnectionName
            id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'servicebus')
          }
          office365: {
            connectionId: office365Connection.id
            connectionName: office365ConnectionName
            id: subscriptionResourceId('Microsoft.Web/locations/managedApis', location, 'office365')
          }
        }
      }
    }
  }
}

// Outputs
output logicAppId string = logicApp.id
output logicAppName string = logicApp.name
output triggerUrl string = listCallbackUrl('${logicApp.id}/triggers/manual', '2019-05-01').value
```

### Logic App Standard (Bicep)

```bicep
// standard-logic-app.bicep - Logic App Standard with VNet

@description('Location for all resources')
param location string = resourceGroup().location

@description('Name of the Logic App')
param logicAppName string = 'logic-std-${uniqueString(resourceGroup().id)}'

@description('App Service Plan SKU')
param skuName string = 'WS1'

@description('VNet subnet resource ID')
param subnetId string = ''

// Storage Account (required for Logic App Standard)
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'st${uniqueString(resourceGroup().id)}'
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    supportsHttpsTrafficOnly: true
  }
}

// App Service Plan (Workflow Standard)
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'asp-${logicAppName}'
  location: location
  sku: {
    tier: 'WorkflowStandard'
    name: skuName
  }
  properties: {
    maximumElasticWorkerCount: 20
    zoneRedundant: false
  }
}

// Logic App Standard
resource logicApp 'Microsoft.Web/sites@2023-01-01' = {
  name: logicAppName
  location: location
  kind: 'functionapp,workflowapp'
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    serverFarmId: appServicePlan.id
    virtualNetworkSubnetId: empty(subnetId) ? null : subnetId
    siteConfig: {
      netFrameworkVersion: 'v6.0'
      appSettings: [
        {
          name: 'FUNCTIONS_EXTENSION_VERSION'
          value: '~4'
        }
        {
          name: 'FUNCTIONS_WORKER_RUNTIME'
          value: 'node'
        }
        {
          name: 'WEBSITE_NODE_DEFAULT_VERSION'
          value: '~18'
        }
        {
          name: 'AzureWebJobsStorage'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTAZUREFILECONNECTIONSTRING'
          value: 'DefaultEndpointsProtocol=https;AccountName=${storageAccount.name};EndpointSuffix=${environment().suffixes.storage};AccountKey=${storageAccount.listKeys().keys[0].value}'
        }
        {
          name: 'WEBSITE_CONTENTSHARE'
          value: logicAppName
        }
        {
          name: 'AzureFunctionsJobHost__extensionBundle__id'
          value: 'Microsoft.Azure.Functions.ExtensionBundle.Workflows'
        }
        {
          name: 'AzureFunctionsJobHost__extensionBundle__version'
          value: '[1.*, 2.0.0)'
        }
        {
          name: 'APP_KIND'
          value: 'workflowApp'
        }
      ]
      use32BitWorkerProcess: false
      ftpsState: 'Disabled'
      vnetRouteAllEnabled: !empty(subnetId)
    }
    httpsOnly: true
  }
}

// Outputs
output logicAppName string = logicApp.name
output logicAppDefaultHostname string = logicApp.properties.defaultHostName
output principalId string = logicApp.identity.principalId
```

### Azure CLI Commands

```bash
# Create Logic App (Consumption) from template
az logic workflow create \
  --resource-group myRG \
  --location eastus \
  --name my-logic-app \
  --definition @workflow-definition.json

# Create API connection
az resource create \
  --resource-group myRG \
  --resource-type Microsoft.Web/connections \
  --name servicebus-connection \
  --location eastus \
  --properties '{
    "displayName": "Service Bus Connection",
    "api": {
      "id": "/subscriptions/{sub}/providers/Microsoft.Web/locations/eastus/managedApis/servicebus"
    },
    "parameterValues": {
      "connectionString": "Endpoint=sb://..."
    }
  }'

# Enable Logic App
az logic workflow update \
  --resource-group myRG \
  --name my-logic-app \
  --state Enabled

# Disable Logic App
az logic workflow update \
  --resource-group myRG \
  --name my-logic-app \
  --state Disabled

# Get callback URL for HTTP trigger
az rest --method post \
  --uri "https://management.azure.com/subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Logic/workflows/my-logic-app/triggers/manual/listCallbackUrl?api-version=2019-05-01"

# View run history
az logic workflow-run list \
  --resource-group myRG \
  --workflow-name my-logic-app \
  --top 10

# Create Logic App Standard
az logicapp create \
  --resource-group myRG \
  --name my-logic-app-std \
  --storage-account mystorageaccount \
  --plan my-asp \
  --runtime-version ~4

# Deploy workflow to Standard
az logicapp deployment source config-zip \
  --resource-group myRG \
  --name my-logic-app-std \
  --src ./workflow.zip
```

---

## Best Practices

```
LOGIC APPS BEST PRACTICES:
──────────────────────────

1. DESIGN
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Keep workflows focused (single responsibility)                    │
   │ • Use child workflows for reusable logic                           │
   │ • Implement proper error handling with scopes                      │
   │ • Use meaningful action names for readability                      │
   │ • Add comments and documentation in definitions                   │
   │ • Version control your workflow definitions                        │
   └─────────────────────────────────────────────────────────────────────┘

2. PERFORMANCE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use parallel branches when possible                               │
   │ • Limit data in variables (use references)                         │
   │ • Set concurrency controls on For Each loops                       │
   │ • Use Standard plan for high-throughput scenarios                  │
   │ • Enable chunking for large file transfers                         │
   │ • Use batching for bulk operations                                 │
   └─────────────────────────────────────────────────────────────────────┘

3. RELIABILITY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Configure retry policies on actions                               │
   │ • Use Run After conditions for error handling                      │
   │ • Implement idempotent operations                                  │
   │ • Set appropriate timeouts                                          │
   │ • Use Terminate action to fail fast on errors                     │
   │ • Monitor with Application Insights                                │
   └─────────────────────────────────────────────────────────────────────┘

4. SECURITY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use managed identities for Azure resources                       │
   │ • Store secrets in Key Vault                                       │
   │ • Secure trigger endpoints (SAS tokens, IP restrictions)          │
   │ • Enable secure inputs/outputs to hide sensitive data             │
   │ • Use Private Link for Standard plan                              │
   │ • Audit connector permissions regularly                           │
   └─────────────────────────────────────────────────────────────────────┘

5. OPERATIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Export definitions for version control                           │
   │ • Use ARM/Bicep for deployment automation                          │
   │ • Set up alerts for failures and long-running workflows           │
   │ • Review run history and analytics regularly                      │
   │ • Clean up disabled/unused workflows                              │
   │ • Use tags for organization and cost tracking                     │
   └─────────────────────────────────────────────────────────────────────┘

6. COST OPTIMIZATION
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use built-in actions over connectors when possible              │
   │ • Batch similar operations together                                │
   │ • Avoid unnecessary loops and conditions                          │
   │ • Use polling triggers appropriately (not too frequent)           │
   │ • Consider Standard plan for high-volume scenarios                │
   │ • Monitor action execution counts                                  │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Back to [Event Grid](03-event-grid.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
