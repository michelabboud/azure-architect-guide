# DDoS Protection

## Overview

Azure DDoS Protection safeguards Azure resources from distributed denial-of-service attacks. It provides always-on traffic monitoring and automatic mitigation at the network edge.

## AWS Comparison

| Feature | AWS Shield Standard | AWS Shield Advanced | Azure DDoS Basic | Azure DDoS Standard |
|---------|--------------------|--------------------|------------------|---------------------|
| Cost | Free | $3,000/mo | Free | ~$2,944/mo |
| SLA | No | Yes | No | Yes |
| Cost Protection | No | Yes | No | Yes |
| Response Team | No | Yes | No | Yes (Rapid Response) |
| Analytics | Basic | Detailed | Basic | Detailed |
| Custom Mitigation | No | Yes | No | Yes |

## Protection Tiers

### Basic (Free)

- **Included**: Automatically enabled for all Azure services
- **Protection**: Always-on network layer mitigation
- **Scope**: Platform-level protection
- **Use Case**: Non-critical workloads

### Standard (Paid)

- **Cost**: ~$2,944/month per protected VNet
- **Protection**: Enhanced mitigation with adaptive tuning
- **Features**:
  - Attack telemetry and reporting
  - Cost protection guarantee
  - DDoS Rapid Response team access
  - SLA-backed availability
- **Use Case**: Production workloads

## Architecture

```
                                    Attack Traffic
                                         │
                                         ▼
                    ┌────────────────────────────────────────┐
                    │         Azure Global Network           │
                    │                                        │
                    │   ┌──────────────────────────────┐    │
                    │   │    DDoS Protection Layer     │    │
                    │   │                              │    │
                    │   │  • Volumetric mitigation    │    │
                    │   │  • Protocol attacks         │    │
                    │   │  • Application layer        │    │
                    │   │  • Always-on monitoring     │    │
                    │   └──────────────────────────────┘    │
                    │                   │                    │
                    │                   │ Clean Traffic      │
                    │                   ▼                    │
                    │   ┌──────────────────────────────┐    │
                    │   │      Azure Edge Network      │    │
                    │   └──────────────────────────────┘    │
                    │                   │                    │
                    └───────────────────┼────────────────────┘
                                        │
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
                    ▼                   ▼                   ▼
            ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
            │   VNet 1      │  │   VNet 2      │  │   VNet 3      │
            │   (Protected) │  │   (Protected) │  │   (Protected) │
            └───────────────┘  └───────────────┘  └───────────────┘
```

## Attack Types Mitigated

| Attack Type | Description | Mitigation |
|-------------|-------------|------------|
| **Volumetric** | Floods bandwidth (UDP, ICMP) | Traffic scrubbing |
| **Protocol** | Exploits protocol weaknesses (SYN flood) | Connection limiting |
| **Application** | HTTP floods, slow attacks | Rate limiting, WAF integration |

## Configuration

### Enable DDoS Protection

```bicep
// Create DDoS Protection Plan
resource ddosPlan 'Microsoft.Network/ddosProtectionPlans@2023-05-01' = {
  name: 'ddos-plan-${environment}'
  location: location
  properties: {}
}

// Associate with VNet
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'protected-vnet'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    ddosProtectionPlan: {
      id: ddosPlan.id
    }
    enableDdosProtection: true
    subnets: [
      {
        name: 'default'
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
    ]
  }
}
```

### CLI Commands

```bash
# Create DDoS protection plan
az network ddos-protection create \
  --resource-group myRG \
  --name myDDoSPlan

# Associate with VNet
az network vnet update \
  --resource-group myRG \
  --name myVNet \
  --ddos-protection true \
  --ddos-protection-plan /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/ddosProtectionPlans/myDDoSPlan

# View protection status
az network ddos-protection show \
  --resource-group myRG \
  --name myDDoSPlan
```

## Monitoring and Analytics

### Metrics

| Metric | Description | Alert Threshold |
|--------|-------------|-----------------|
| InDDoSAttack | Binary attack indicator | > 0 |
| DroppedPackets | Packets dropped during attack | Baseline + 20% |
| ForwardedPackets | Clean traffic forwarded | Baseline |
| TCPBytesDropped | TCP bytes blocked | > 0 during attack |
| UDPBytesDropped | UDP bytes blocked | > 0 during attack |

### Alert Configuration

```bicep
resource ddosAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'DDoS-Attack-Alert'
  location: 'global'
  properties: {
    description: 'Alert when DDoS attack detected'
    severity: 1
    enabled: true
    scopes: [
      publicIP.id
    ]
    evaluationFrequency: 'PT1M'
    windowSize: 'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'DDoSAttackDetected'
          metricName: 'IfUnderDDoSAttack'
          operator: 'GreaterThan'
          threshold: 0
          timeAggregation: 'Maximum'
        }
      ]
    }
    actions: [
      {
        actionGroupId: actionGroup.id
      }
    ]
  }
}
```

### KQL Queries

```kusto
// DDoS attack events
AzureDiagnostics
| where Category == "DDoSProtectionNotifications"
| project TimeGenerated, Resource, Action_s, Message
| order by TimeGenerated desc

// Attack traffic analysis
AzureDiagnostics
| where Category == "DDoSMitigationReports"
| extend
    AttackVectors = parse_json(tostring(AttackVectors_s)),
    SourceCountry = SourceContinentOrCountry_s,
    TotalDropped = TotalDroppedPackets_d
| summarize
    TotalDroppedPackets = sum(TotalDropped),
    count()
    by SourceCountry
| order by TotalDroppedPackets desc
```

## Best Practices

### 1. Architecture Design

```
┌──────────────────────────────────────────────────────────────┐
│              DDoS Protection Best Practices                   │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  1. Deploy Azure Front Door / CDN                            │
│     └── Absorbs attacks at edge before reaching origin       │
│                                                               │
│  2. Enable DDoS Standard on hub VNets                        │
│     └── Protects all peered spoke VNets                      │
│                                                               │
│  3. Use Azure WAF with DDoS                                  │
│     └── Layered protection for application attacks           │
│                                                               │
│  4. Configure auto-scaling                                    │
│     └── Handle legitimate traffic during attacks             │
│                                                               │
│  5. Monitor baseline metrics                                  │
│     └── Establish normal traffic patterns                    │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### 2. Response Planning

| Phase | Actions |
|-------|---------|
| **Preparation** | Document runbooks, test alerts |
| **Detection** | Automated alerts, monitoring |
| **Analysis** | Identify attack vectors |
| **Mitigation** | Automatic + manual tuning |
| **Recovery** | Verify services restored |
| **Post-Incident** | Review, update policies |

## Cost Protection

DDoS Standard includes cost protection for:

- **Compute**: VM scale-out during attack
- **Bandwidth**: Data transfer during attack
- **Application Gateway**: Scaling during attack
- **Load Balancer**: Additional resources

**Note**: Must file claim within 30 days of attack

---

*Continue to [Private Link](04-private-link.md)*

*Back to [Network Security Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
