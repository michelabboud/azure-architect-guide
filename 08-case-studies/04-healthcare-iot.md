# Case Study 4: Healthcare IoT Platform

## Executive Summary

**Company**: MedWatch Healthcare Systems
**Industry**: Healthcare / Medical Devices
**Challenge**: Build HIPAA-compliant IoT platform for patient monitoring devices
**Outcome**: 500K connected devices, real-time alerting <2 seconds, zero PHI breaches

### The Business Context

MedWatch manufactures remote patient monitoring devices (heart monitors, glucose monitors, blood pressure cuffs). They needed a cloud platform to:

1. Ingest data from 500,000+ medical devices globally
2. Process real-time alerts for critical patient conditions
3. Integrate with hospital EHR systems via FHIR
4. Maintain HIPAA, HITECH, and FDA 21 CFR Part 11 compliance

## Architecture Overview

```
                                    ┌─────────────────────────────────────┐
                                    │           Device Layer               │
                                    │  ┌───────┐ ┌───────┐ ┌───────┐     │
                                    │  │ Heart │ │Glucose│ │ BP    │     │
                                    │  │Monitor│ │Monitor│ │Monitor│     │
                                    │  └───┬───┘ └───┬───┘ └───┬───┘     │
                                    └──────┼─────────┼─────────┼──────────┘
                                           │         │         │
                                           └─────────┼─────────┘
                                                     │ (MQTT/HTTPS)
                                    ┌────────────────▼────────────────────┐
                                    │         Azure IoT Hub               │
                                    │    • Device provisioning            │
                                    │    • Bi-directional messaging       │
                                    │    • Device twin state              │
                                    └────────────────┬────────────────────┘
                                                     │
                         ┌───────────────────────────┼───────────────────────────┐
                         │                           │                           │
              ┌──────────▼──────────┐    ┌──────────▼──────────┐    ┌──────────▼──────────┐
              │   Hot Path          │    │   Warm Path         │    │   Cold Path         │
              │   (Real-time)       │    │   (Near-real-time)  │    │   (Batch)           │
              │                     │    │                     │    │                     │
              │ ┌─────────────────┐ │    │ ┌─────────────────┐ │    │ ┌─────────────────┐ │
              │ │Stream Analytics │ │    │ │ Azure Functions │ │    │ │  Data Factory   │ │
              │ │  (Anomaly Det.) │ │    │ │ (FHIR Transform)│ │    │ │  (Daily ETL)    │ │
              │ └────────┬────────┘ │    │ └────────┬────────┘ │    │ └────────┬────────┘ │
              │          │          │    │          │          │    │          │          │
              │ ┌────────▼────────┐ │    │ ┌────────▼────────┐ │    │ ┌────────▼────────┐ │
              │ │  Event Hub      │ │    │ │ FHIR API        │ │    │ │  ADLS Gen2      │ │
              │ │  (Alerts)       │ │    │ │ (Azure API for  │ │    │ │  (PHI Data Lake)│ │
              │ └────────┬────────┘ │    │ │  FHIR)          │ │    │ └────────┬────────┘ │
              └──────────┼──────────┘    │ └────────┬────────┘ │    └──────────┼──────────┘
                         │               └──────────┼──────────┘               │
                         │                          │                          │
              ┌──────────▼──────────┐    ┌──────────▼──────────┐    ┌──────────▼──────────┐
              │  Notification Hub   │    │  Cosmos DB          │    │  Synapse Analytics  │
              │  (Push to Clinicians)│   │  (Patient Records)  │    │  (Population Health)│
              └─────────────────────┘    └─────────────────────┘    └─────────────────────┘
```

## AWS to Azure Service Mapping

| Component | AWS Equivalent | Azure Implementation | Healthcare Feature |
|-----------|---------------|---------------------|-------------------|
| Device Gateway | IoT Core | Azure IoT Hub | X.509 certificate auth |
| Device Provisioning | IoT Core Registry | DPS | Zero-touch provisioning |
| Streaming | Kinesis | Event Hubs + Stream Analytics | Anomaly detection ML |
| FHIR API | None (custom) | Azure API for FHIR | Built-in compliance |
| PHI Storage | S3 (encrypted) | ADLS Gen2 + Purview | Access audit, lineage |
| Patient Records | DynamoDB | Cosmos DB | Multi-region writes |
| ML Models | SageMaker | Azure ML | Responsible AI toolkit |

## Key Architectural Decisions

### Decision 1: IoT Hub vs Event Hubs for Device Ingestion

**The Debate:**

| Criteria | IoT Hub | Event Hubs |
|----------|---------|------------|
| Device Management | Built-in twins, C2D | No native support |
| Protocol Support | MQTT, AMQP, HTTPS | AMQP, Kafka, HTTPS |
| Authentication | X.509, SAS, TPM | SAS tokens |
| Cost | Higher per-message | Lower per-message |
| Scale | 1M devices/unit | Higher throughput |

**Decision: IoT Hub with DPS for device provisioning**

**Rationale:**
1. Medical devices require individual identity management
2. Device twins store configuration and compliance state
3. Cloud-to-device messaging needed for firmware updates
4. X.509 certificates required for FDA compliance

**Device Provisioning Configuration:**
```csharp
// Device Provisioning Service Enrollment
public class DeviceProvisioning
{
    private readonly ProvisioningServiceClient _provisioningClient;

    public async Task<IndividualEnrollment> RegisterDeviceAsync(
        string deviceId,
        string attestationType,
        X509Certificate2 certificate)
    {
        var attestation = new X509Attestation(certificate);

        var enrollment = new IndividualEnrollment(
            deviceId,
            attestation)
        {
            DeviceId = deviceId,
            ProvisioningStatus = ProvisioningStatus.Enabled,
            ReprovisionPolicy = new ReprovisionPolicy
            {
                MigrateDeviceData = true,
                UpdateHubAssignment = true
            },
            InitialTwinState = new TwinState(
                tags: new TwinCollection(JsonSerializer.Serialize(new
                {
                    deviceType = "HeartMonitor",
                    firmwareVersion = "2.1.0",
                    complianceStatus = "pending"
                })),
                desiredProperties: new TwinCollection(JsonSerializer.Serialize(new
                {
                    telemetryInterval = 30,
                    alertThresholds = new
                    {
                        heartRateHigh = 120,
                        heartRateLow = 50
                    }
                })))
        };

        return await _provisioningClient.CreateOrUpdateIndividualEnrollmentAsync(enrollment);
    }
}
```

### Decision 2: Real-time Alert Processing

**The Debate:**

| Option | Latency | Complexity | ML Integration | Cost |
|--------|---------|------------|----------------|------|
| Stream Analytics | Low (<1s) | Low | Built-in ML | $$ |
| Azure Functions | Medium (1-5s) | Medium | Custom | $ |
| Databricks Streaming | Medium | High | Excellent | $$$ |
| AKS with Kafka | Very Low | Very High | Custom | $$ |

**Decision: Stream Analytics with built-in anomaly detection + Azure Functions for escalation**

**Rationale:**
1. Stream Analytics provides sub-second latency for critical alerts
2. Built-in ML anomaly detection reduces false positives
3. No Spark cluster management overhead
4. Azure Functions handle notification routing logic

**Stream Analytics Alert Query:**
```sql
-- Real-time anomaly detection for heart rate
WITH HeartRateReadings AS (
    SELECT
        deviceId,
        patientId,
        heartRate,
        timestamp,
        IoTHub.ConnectionDeviceId as connectionId,
        AnomalyDetection_SpikeAndDip(
            heartRate,
            95,  -- confidence level
            30   -- history points
        ) OVER (PARTITION BY patientId LIMIT DURATION(minute, 5)) as anomalyResult
    FROM IoTHubInput TIMESTAMP BY timestamp
    WHERE sensorType = 'HeartMonitor'
),
CriticalAlerts AS (
    SELECT
        deviceId,
        patientId,
        heartRate,
        timestamp,
        anomalyResult.Score as anomalyScore,
        anomalyResult.IsAnomaly as isAnomaly,
        CASE
            WHEN heartRate > 150 THEN 'CRITICAL_HIGH'
            WHEN heartRate < 40 THEN 'CRITICAL_LOW'
            WHEN heartRate > 120 AND anomalyResult.IsAnomaly = 1 THEN 'ANOMALY_HIGH'
            WHEN heartRate < 50 AND anomalyResult.IsAnomaly = 1 THEN 'ANOMALY_LOW'
            ELSE NULL
        END as alertType
    FROM HeartRateReadings
)

-- Output critical alerts to Event Hub for immediate action
SELECT
    deviceId,
    patientId,
    heartRate,
    timestamp,
    alertType,
    anomalyScore,
    'IMMEDIATE' as priority
INTO AlertEventHub
FROM CriticalAlerts
WHERE alertType IN ('CRITICAL_HIGH', 'CRITICAL_LOW')

-- Output anomaly alerts for clinical review
SELECT
    deviceId,
    patientId,
    heartRate,
    timestamp,
    alertType,
    anomalyScore,
    'REVIEW' as priority
INTO ReviewQueue
FROM CriticalAlerts
WHERE alertType IN ('ANOMALY_HIGH', 'ANOMALY_LOW')
```

### Decision 3: FHIR Integration Strategy

**The Debate:**

| Approach | Interoperability | Compliance | Complexity | Cost |
|----------|-----------------|------------|------------|------|
| Custom API | Low | Manual | High | $$ |
| Azure API for FHIR | High | Built-in | Low | $$ |
| FHIR Server OSS | High | Manual | Medium | $ |
| Firely Server | High | Built-in | Low | $$$ |

**Decision: Azure API for FHIR with Azure Functions for data transformation**

**Rationale:**
1. Native HIPAA compliance with BAA
2. Built-in SMART on FHIR support for EHR integration
3. Managed service reduces operational burden
4. $export for bulk data access to analytics

**FHIR Data Transformation:**
```csharp
// Azure Function to transform IoT telemetry to FHIR Observation
public class FhirTransformFunction
{
    private readonly FhirClient _fhirClient;
    private readonly ILogger<FhirTransformFunction> _logger;

    [FunctionName("TransformToFhirObservation")]
    public async Task Run(
        [EventHubTrigger("iot-telemetry", Connection = "EventHubConnection")]
        EventData[] events,
        ILogger log)
    {
        var observations = new List<Observation>();

        foreach (var eventData in events)
        {
            var telemetry = JsonSerializer.Deserialize<DeviceTelemetry>(
                eventData.Body.ToArray());

            var observation = new Observation
            {
                Status = ObservationStatus.Final,
                Code = new CodeableConcept
                {
                    Coding = new List<Coding>
                    {
                        new Coding
                        {
                            System = "http://loinc.org",
                            Code = GetLoincCode(telemetry.SensorType),
                            Display = telemetry.SensorType
                        }
                    }
                },
                Subject = new ResourceReference($"Patient/{telemetry.PatientId}"),
                Device = new ResourceReference($"Device/{telemetry.DeviceId}"),
                Effective = new FhirDateTime(telemetry.Timestamp),
                Value = CreateValue(telemetry)
            };

            // Add provenance for audit trail
            observation.Meta = new Meta
            {
                Source = $"Device/{telemetry.DeviceId}",
                LastUpdated = DateTimeOffset.UtcNow
            };

            observations.Add(observation);
        }

        // Batch upload to FHIR
        var bundle = new Bundle
        {
            Type = Bundle.BundleType.Transaction,
            Entry = observations.Select(o => new Bundle.EntryComponent
            {
                Resource = o,
                Request = new Bundle.RequestComponent
                {
                    Method = Bundle.HTTPVerb.POST,
                    Url = "Observation"
                }
            }).ToList()
        };

        await _fhirClient.TransactionAsync(bundle);
    }

    private string GetLoincCode(string sensorType) => sensorType switch
    {
        "HeartMonitor" => "8867-4",      // Heart rate
        "GlucoseMonitor" => "2339-0",    // Blood glucose
        "BPMonitor" => "85354-9",        // Blood pressure panel
        _ => throw new ArgumentException($"Unknown sensor type: {sensorType}")
    };
}
```

## Technical Deep Dive

### Device Twin for Configuration Management

```json
{
  "deviceId": "HM-001-A1B2C3",
  "etag": "AAAAAAAAAAA=",
  "properties": {
    "desired": {
      "telemetryInterval": 30,
      "alertThresholds": {
        "heartRateHigh": 120,
        "heartRateLow": 50,
        "spo2Low": 92
      },
      "firmwareVersion": "2.2.0",
      "$metadata": {
        "$lastUpdated": "2024-01-15T10:30:00.000Z"
      }
    },
    "reported": {
      "telemetryInterval": 30,
      "batteryLevel": 85,
      "signalStrength": -65,
      "firmwareVersion": "2.1.0",
      "firmwareUpdateStatus": "downloading",
      "lastCalibration": "2024-01-10T08:00:00.000Z",
      "$metadata": {
        "$lastUpdated": "2024-01-15T10:35:00.000Z"
      }
    }
  },
  "tags": {
    "deviceType": "HeartMonitor",
    "patientId": "P-12345",
    "facilityId": "F-789",
    "complianceStatus": "compliant",
    "lastAudit": "2024-01-01"
  }
}
```

### PHI Data Protection Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PHI Data Protection Layers                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  Layer 1: Transport Security                                                 │
│  ├── TLS 1.3 for all device communication                                   │
│  ├── Certificate pinning on devices                                          │
│  └── Private endpoints for all Azure services                                │
│                                                                              │
│  Layer 2: Data Encryption                                                    │
│  ├── Customer-managed keys (CMK) for all storage                            │
│  ├── Always Encrypted for SQL (deterministic for search)                    │
│  └── Double encryption enabled (infrastructure + service)                    │
│                                                                              │
│  Layer 3: Access Control                                                     │
│  ├── Azure AD authentication for all services                                │
│  ├── RBAC with least privilege                                               │
│  ├── Managed identities (no secrets in code)                                 │
│  └── Conditional Access policies (location, device compliance)               │
│                                                                              │
│  Layer 4: Data Governance                                                    │
│  ├── Microsoft Purview for data classification                              │
│  ├── Sensitivity labels (auto-applied to PHI)                               │
│  ├── Data loss prevention policies                                           │
│  └── Retention policies (7-year minimum for HIPAA)                          │
│                                                                              │
│  Layer 5: Monitoring & Audit                                                 │
│  ├── Azure Monitor for security events                                       │
│  ├── Microsoft Defender for Cloud                                           │
│  ├── Immutable audit logs (append-only blob)                                │
│  └── SIEM integration (Microsoft Sentinel)                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Bicep Infrastructure for Healthcare Compliance

```bicep
// Healthcare IoT Platform - HIPAA Compliant Infrastructure
@description('IoT Hub with HIPAA configurations')
resource iotHub 'Microsoft.Devices/IotHubs@2023-06-30' = {
  name: 'iot-${prefix}-healthcare'
  location: location
  sku: {
    name: 'S3'  // Standard tier for production
    capacity: 4
  }
  properties: {
    minTlsVersion: '1.2'
    disableLocalAuth: false  // Required for device auth
    publicNetworkAccess: 'Disabled'
    networkRuleSets: {
      defaultAction: 'Deny'
      applyToBuiltInEventHubEndpoint: true
    }
    routing: {
      endpoints: {
        eventHubs: [
          {
            name: 'alerts-eh'
            connectionString: alertEventHub.listKeys().primaryConnectionString
            authenticationType: 'identityBased'
          }
        ]
        storageContainers: [
          {
            name: 'phi-storage'
            containerName: 'device-telemetry'
            fileNameFormat: '{iothub}/{partition}/{YYYY}/{MM}/{DD}/{HH}/{mm}'
            authenticationType: 'identityBased'
            endpointUri: storageAccount.properties.primaryEndpoints.blob
          }
        ]
      }
      routes: [
        {
          name: 'critical-alerts'
          source: 'DeviceMessages'
          condition: 'alertType = \'CRITICAL\''
          endpointNames: ['alerts-eh']
          isEnabled: true
        }
        {
          name: 'all-telemetry'
          source: 'DeviceMessages'
          condition: 'true'
          endpointNames: ['phi-storage']
          isEnabled: true
        }
      ]
    }
  }
  identity: {
    type: 'SystemAssigned'
  }
}

// Azure API for FHIR
resource fhirService 'Microsoft.HealthcareApis/workspaces/fhirservices@2023-09-06' = {
  name: 'fhir-${prefix}'
  parent: healthcareWorkspace
  location: location
  kind: 'fhir-R4'
  properties: {
    authenticationConfiguration: {
      authority: 'https://login.microsoftonline.com/${subscription().tenantId}'
      audience: 'https://${healthcareWorkspace.name}-fhir-${prefix}.fhir.azurehealthcareapis.com'
      smartProxyEnabled: true  // Enable SMART on FHIR
    }
    corsConfiguration: {
      origins: ['https://*.medwatch.com']
      headers: ['Authorization', 'Content-Type']
      methods: ['GET', 'POST', 'PUT', 'DELETE']
      maxAge: 600
      allowCredentials: false
    }
    exportConfiguration: {
      storageAccountName: exportStorageAccount.name
    }
    encryption: {
      customerManagedKeyEncryption: {
        keyEncryptionKeyUrl: keyVault::fhirKey.properties.keyUri
      }
    }
  }
}

// HIPAA-compliant Storage Account
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'st${prefix}phi'
  location: location
  sku: {
    name: 'Standard_GZRS'  // Geo-zone redundant
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    publicNetworkAccess: 'Disabled'
    encryption: {
      requireInfrastructureEncryption: true  // Double encryption
      keySource: 'Microsoft.Keyvault'
      keyvaultproperties: {
        keyvaulturi: keyVault.properties.vaultUri
        keyname: 'storage-cmk'
      }
      services: {
        blob: { enabled: true, keyType: 'Account' }
        file: { enabled: true, keyType: 'Account' }
        table: { enabled: true, keyType: 'Account' }
        queue: { enabled: true, keyType: 'Account' }
      }
    }
    immutableStorageWithVersioning: {
      enabled: true  // Immutable for audit trail
    }
  }
}
```

## Cost Analysis

### Monthly Cost Breakdown (500K Devices)

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| IoT Hub | S3, 4 units | $8,000 |
| Device Provisioning Service | S1, 1M operations | $200 |
| Event Hubs | Dedicated, 4 CU | $6,500 |
| Stream Analytics | 12 SU | $1,500 |
| Azure Functions | Premium EP2, 3 instances | $1,200 |
| Azure API for FHIR | 100K transactions | $2,500 |
| Cosmos DB | 50K RU/s | $15,000 |
| ADLS Gen2 | 50TB + transactions | $3,000 |
| Key Vault Premium | HSM operations | $800 |
| Microsoft Defender | IoT + Cloud | $4,000 |
| **Total** | | **$42,700** |

### Per-Device Cost Analysis

| Metric | Value |
|--------|-------|
| Total monthly cost | $42,700 |
| Active devices | 500,000 |
| **Cost per device** | **$0.085/month** |
| Messages per device/day | ~100 |
| Cost per 1000 messages | $0.028 |

## Lessons Learned

### What Worked Well

1. **IoT Hub Device Twins**: Simplified configuration management across 500K devices
2. **Stream Analytics Anomaly Detection**: Reduced false positive alerts by 60%
3. **Azure API for FHIR**: Accelerated EHR integration by 3 months

### Challenges Encountered

1. **Device Certificate Management**: Needed custom certificate rotation solution
2. **FHIR Data Volume**: Bulk export for analytics required careful planning
3. **Regional Latency**: Added IoT Hub in EU for GDPR data residency

### Recommendations

1. **Plan certificate lifecycle**: Automate certificate renewal before expiry
2. **Test at scale early**: 500K device simulation revealed bottlenecks
3. **Use IoT Edge for offline**: Critical for devices in poor connectivity areas

---

*Continue to [Case Study 5: SaaS Multi-Tenant Architecture](05-saas-multitenant.md)*

*Back to [Chapter 08: Case Studies](README.md)*
