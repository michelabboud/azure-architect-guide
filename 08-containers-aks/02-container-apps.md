# Azure Container Apps Deep Dive

## Overview

Azure Container Apps is a fully managed serverless container platform built on Kubernetes but without exposing the Kubernetes complexity. For AWS architects, think of it as the sweet spot between AWS App Runner's simplicity and ECS Fargate's flexibility, with built-in event-driven scaling (KEDA) and microservices capabilities (Dapr).

## AWS to Azure Container Services Mapping

| AWS Service | Azure Container Apps | Notes |
|-------------|---------------------|-------|
| AWS App Runner | Container Apps | Closest match - simple container deployments |
| ECS Fargate | Container Apps | Serverless containers with more features |
| ECS + Copilot | Container Apps | Simplified container orchestration |
| Lambda Container Images | Container Apps Jobs | Event-triggered container execution |
| Step Functions | Container Apps + Dapr | Workflow orchestration |
| AWS Batch | Container Apps Jobs | Batch/scheduled workloads |

## Container Apps vs AKS Decision Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONTAINER APPS vs AKS DECISION TREE                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│              ┌────────────────────────────────────────────┐                 │
│              │   Do you need full Kubernetes control?     │                 │
│              └────────────────────┬───────────────────────┘                 │
│                                   │                                         │
│                    ┌──────────────┴──────────────┐                         │
│                    │                             │                         │
│                    ▼                             ▼                         │
│               ┌────────┐                    ┌────────┐                     │
│               │  YES   │                    │   NO   │                     │
│               └────┬───┘                    └────┬───┘                     │
│                    │                             │                         │
│                    ▼                             ▼                         │
│           ┌────────────────┐         ┌────────────────────────┐           │
│           │     AKS        │         │ Scale to zero needed?   │           │
│           │                │         └───────────┬────────────┘           │
│           │ • Custom CNI   │                     │                         │
│           │ • Service mesh │         ┌───────────┴───────────┐            │
│           │ • Node control │         │                       │            │
│           │ • DaemonSets   │         ▼                       ▼            │
│           └────────────────┘    ┌────────┐              ┌────────┐        │
│                                 │  YES   │              │   NO   │        │
│                                 └────┬───┘              └────┬───┘        │
│                                      │                       │            │
│                                      ▼                       ▼            │
│                              ┌───────────────┐     ┌───────────────────┐  │
│                              │ Container Apps│     │ Event-driven?     │  │
│                              │               │     └─────────┬─────────┘  │
│                              │ • KEDA native │               │            │
│                              │ • Zero cost   │    ┌──────────┴──────────┐ │
│                              │   at idle     │    │                     │ │
│                              └───────────────┘    ▼                     ▼ │
│                                            ┌──────────┐         ┌────────┐│
│                                            │   YES    │         │   NO   ││
│                                            └────┬─────┘         └────┬───┘│
│                                                 │                    │    │
│                                                 ▼                    ▼    │
│                                          Container Apps        Either    │
│                                          (Built-in triggers)   works    │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Feature Comparison

| Feature | Container Apps | AKS |
|---------|---------------|-----|
| Kubernetes knowledge | None required | Required |
| Control plane cost | Included | Free |
| Infrastructure management | Fully managed | You manage nodes |
| Pricing model | Per vCPU-second + memory | Per VM node |
| Scale to zero | Yes (native) | With KEDA addon |
| Minimum cost | $0 when idle | Node VMs always running |
| Service mesh | Dapr built-in | Istio/Linkerd addon |
| Ingress | Built-in Envoy | NGINX/AGIC |
| Custom networking | VNet injection | Full VNet control |
| Node pools | N/A | Full control |
| DaemonSets | No | Yes |
| Helm charts | No | Yes |
| GitOps (Flux) | No | Yes |
| Max replicas | 300 | Unlimited |

## Architecture

### Container Apps Environment

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONTAINER APPS ARCHITECTURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                 CONTAINER APPS ENVIRONMENT                               ││
│  │                 (Secure boundary - like a K8s namespace)                 ││
│  │                                                                          ││
│  │  ┌─────────────────────────────────────────────────────────────────────┐││
│  │  │                    WORKLOAD PROFILES POOL                           │││
│  │  │                                                                      │││
│  │  │  Consumption (Serverless)    │    Dedicated (D4/D8/D16/etc.)       │││
│  │  │  ┌─────────────────────────┐ │ ┌──────────────────────────────────┐│││
│  │  │  │ • Scale to 0            │ │ │ • Reserved capacity              ││││
│  │  │  │ • Pay per use           │ │ │ • Predictable performance        ││││
│  │  │  │ • Auto-managed          │ │ │ • GPU support (NC-series)        ││││
│  │  │  │ • Max 4 vCPU / 8 GB     │ │ │ • Up to 32 vCPU / 128 GB        ││││
│  │  │  └─────────────────────────┘ │ └──────────────────────────────────┘│││
│  │  └─────────────────────────────────────────────────────────────────────┘││
│  │                                                                          ││
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────────┐  ││
│  │  │ Container App A  │  │ Container App B  │  │ Container App C      │  ││
│  │  │ (Frontend)       │  │ (API)            │  │ (Worker)             │  ││
│  │  │                  │  │                  │  │                      │  ││
│  │  │ ┌──────────────┐ │  │ ┌──────────────┐ │  │ ┌──────────────────┐ │  ││
│  │  │ │  Revision 1  │ │  │ │  Revision 1  │ │  │ │   Job: Batch     │ │  ││
│  │  │ │  (90% traffic)│ │  │ │  (Active)    │ │  │ │   (Event/Manual)│ │  ││
│  │  │ ├──────────────┤ │  │ └──────────────┘ │  │ └──────────────────┘ │  ││
│  │  │ │  Revision 2  │ │  │                  │  │                      │  ││
│  │  │ │  (10% traffic)│ │  │ ┌────────────┐  │  │                      │  ││
│  │  │ └──────────────┘ │  │ │ Dapr       │  │  │                      │  ││
│  │  │                  │  │ │ Sidecar    │  │  │                      │  ││
│  │  │ Ingress: External│  │ └────────────┘  │  │ Ingress: None        │  ││
│  │  └──────────────────┘  │ Ingress: Internal│  └──────────────────────┘  ││
│  │                        └──────────────────┘                            ││
│  │                                                                          ││
│  │  Shared Resources:                                                       ││
│  │  ┌────────────────┐ ┌─────────────────┐ ┌────────────────────────────┐ ││
│  │  │ Log Analytics  │ │ Container       │ │ VNet Integration           │ ││
│  │  │ Workspace      │ │ Registry (ACR)  │ │ (Optional)                 │ ││
│  │  └────────────────┘ └─────────────────┘ └────────────────────────────┘ ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Environment Setup

### Create Container Apps Environment

```bash
# Create resource group
az group create --name container-apps-rg --location eastus

# Create Log Analytics workspace
az monitor log-analytics workspace create \
  --resource-group container-apps-rg \
  --workspace-name container-apps-logs

# Get workspace credentials
LOG_ANALYTICS_WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group container-apps-rg \
  --workspace-name container-apps-logs \
  --query customerId -o tsv)

LOG_ANALYTICS_KEY=$(az monitor log-analytics workspace get-shared-keys \
  --resource-group container-apps-rg \
  --workspace-name container-apps-logs \
  --query primarySharedKey -o tsv)

# Create Container Apps environment (Consumption only)
az containerapp env create \
  --name prod-env \
  --resource-group container-apps-rg \
  --location eastus \
  --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_ID \
  --logs-workspace-key $LOG_ANALYTICS_KEY

# Create environment with workload profiles (Consumption + Dedicated)
az containerapp env create \
  --name prod-env-profiles \
  --resource-group container-apps-rg \
  --location eastus \
  --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_ID \
  --logs-workspace-key $LOG_ANALYTICS_KEY \
  --enable-workload-profiles

# Add dedicated workload profile
az containerapp env workload-profile add \
  --name prod-env-profiles \
  --resource-group container-apps-rg \
  --workload-profile-name "dedicated-d4" \
  --workload-profile-type D4 \
  --min-nodes 1 \
  --max-nodes 10
```

### VNet Integration

```bash
# Create VNet for Container Apps
az network vnet create \
  --name container-apps-vnet \
  --resource-group container-apps-rg \
  --location eastus \
  --address-prefix 10.0.0.0/16

# Create infrastructure subnet (minimum /23 for consumption, /27 for workload profiles)
az network vnet subnet create \
  --name infrastructure-subnet \
  --vnet-name container-apps-vnet \
  --resource-group container-apps-rg \
  --address-prefix 10.0.0.0/23

# Create environment with VNet (internal only - no public ingress)
az containerapp env create \
  --name secure-env \
  --resource-group container-apps-rg \
  --location eastus \
  --infrastructure-subnet-resource-id /subscriptions/{sub}/resourceGroups/container-apps-rg/providers/Microsoft.Network/virtualNetworks/container-apps-vnet/subnets/infrastructure-subnet \
  --internal-only true \
  --logs-workspace-id $LOG_ANALYTICS_WORKSPACE_ID \
  --logs-workspace-key $LOG_ANALYTICS_KEY
```

### Bicep Template

```bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Environment name')
param environmentName string = 'prod-env'

// Log Analytics Workspace
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: '${environmentName}-logs'
  location: location
  properties: {
    sku: {
      name: 'PerGB2018'
    }
    retentionInDays: 30
  }
}

// VNet for Container Apps
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: '${environmentName}-vnet'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: 'infrastructure'
        properties: {
          addressPrefix: '10.0.0.0/23'
          delegations: [
            {
              name: 'Microsoft.App.environments'
              properties: {
                serviceName: 'Microsoft.App/environments'
              }
            }
          ]
        }
      }
    ]
  }
}

// Container Apps Environment
resource containerAppEnv 'Microsoft.App/managedEnvironments@2024-03-01' = {
  name: environmentName
  location: location
  properties: {
    appLogsConfiguration: {
      destination: 'log-analytics'
      logAnalyticsConfiguration: {
        customerId: logAnalytics.properties.customerId
        sharedKey: logAnalytics.listKeys().primarySharedKey
      }
    }
    vnetConfiguration: {
      infrastructureSubnetId: vnet.properties.subnets[0].id
      internal: false
    }
    workloadProfiles: [
      {
        name: 'Consumption'
        workloadProfileType: 'Consumption'
      }
      {
        name: 'dedicated-d4'
        workloadProfileType: 'D4'
        minimumCount: 1
        maximumCount: 10
      }
    ]
    zoneRedundant: true
  }
}

output environmentId string = containerAppEnv.id
output defaultDomain string = containerAppEnv.properties.defaultDomain
output staticIp string = containerAppEnv.properties.staticIp
```

## Deploy Container Apps

### Basic Deployment

```bash
# Deploy from public image
az containerapp create \
  --name frontend-app \
  --resource-group container-apps-rg \
  --environment prod-env \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --cpu 0.5 \
  --memory 1.0Gi

# Deploy from ACR with system-assigned identity
az containerapp create \
  --name api-app \
  --resource-group container-apps-rg \
  --environment prod-env \
  --image myacr.azurecr.io/api:v1 \
  --registry-server myacr.azurecr.io \
  --registry-identity system \
  --target-port 8080 \
  --ingress internal \
  --min-replicas 2 \
  --max-replicas 50 \
  --cpu 1 \
  --memory 2Gi \
  --env-vars "ASPNETCORE_ENVIRONMENT=Production" "API_VERSION=v1"
```

### Bicep Container App

```bicep
@description('Container App name')
param appName string

@description('Container image')
param containerImage string

@description('Environment resource ID')
param environmentId string

@description('User-assigned managed identity ID')
param userIdentityId string

resource containerApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: appName
  location: location
  identity: {
    type: 'SystemAssigned,UserAssigned'
    userAssignedIdentities: {
      '${userIdentityId}': {}
    }
  }
  properties: {
    managedEnvironmentId: environmentId
    workloadProfileName: 'Consumption'
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
        transport: 'http'
        allowInsecure: false
        traffic: [
          {
            latestRevision: true
            weight: 100
          }
        ]
        corsPolicy: {
          allowedOrigins: ['https://myapp.com']
          allowedMethods: ['GET', 'POST', 'PUT', 'DELETE']
          allowedHeaders: ['*']
          maxAge: 3600
        }
      }
      registries: [
        {
          server: 'myacr.azurecr.io'
          identity: userIdentityId
        }
      ]
      secrets: [
        {
          name: 'db-connection'
          keyVaultUrl: 'https://mykv.vault.azure.net/secrets/db-connection'
          identity: userIdentityId
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'main'
          image: containerImage
          resources: {
            cpu: json('1.0')
            memory: '2Gi'
          }
          env: [
            {
              name: 'ASPNETCORE_ENVIRONMENT'
              value: 'Production'
            }
            {
              name: 'DB_CONNECTION'
              secretRef: 'db-connection'
            }
          ]
          probes: [
            {
              type: 'Liveness'
              httpGet: {
                path: '/health/live'
                port: 8080
              }
              initialDelaySeconds: 10
              periodSeconds: 10
            }
            {
              type: 'Readiness'
              httpGet: {
                path: '/health/ready'
                port: 8080
              }
              initialDelaySeconds: 5
              periodSeconds: 5
            }
          ]
        }
      ]
      scale: {
        minReplicas: 2
        maxReplicas: 100
        rules: [
          {
            name: 'http-rule'
            http: {
              metadata: {
                concurrentRequests: '100'
              }
            }
          }
        ]
      }
    }
  }
}

output fqdn string = containerApp.properties.configuration.ingress.fqdn
output latestRevision string = containerApp.properties.latestRevisionName
```

## Dapr Integration

### Dapr Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    DAPR INTEGRATION IN CONTAINER APPS                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     CONTAINER APP WITH DAPR                          │   │
│  │                                                                      │   │
│  │  ┌─────────────────────┐    ┌─────────────────────────────────────┐ │   │
│  │  │  YOUR APPLICATION   │    │         DAPR SIDECAR                 │ │   │
│  │  │                     │    │                                      │ │   │
│  │  │  ┌───────────────┐  │    │  ┌─────────────────────────────────┐│ │   │
│  │  │  │    Code       │──┼────┼──│ Service Invocation              ││ │   │
│  │  │  │               │  │    │  │ • HTTP/gRPC calls between apps  ││ │   │
│  │  │  │ localhost:3500│  │    │  │ • Service discovery             ││ │   │
│  │  │  │ (Dapr HTTP)   │  │    │  │ • mTLS encryption               ││ │   │
│  │  │  │               │  │    │  └─────────────────────────────────┘│ │   │
│  │  │  │ localhost:50001│ │    │                                      │ │   │
│  │  │  │ (Dapr gRPC)   │  │    │  ┌─────────────────────────────────┐│ │   │
│  │  │  └───────────────┘  │    │  │ State Management                ││ │   │
│  │  │                     │    │  │ • Azure Cosmos DB               ││ │   │
│  │  │                     │    │  │ • Azure Table Storage           ││ │   │
│  │  │                     │    │  │ • Redis Cache                   ││ │   │
│  │  │                     │    │  └─────────────────────────────────┘│ │   │
│  │  │                     │    │                                      │ │   │
│  │  │                     │    │  ┌─────────────────────────────────┐│ │   │
│  │  │                     │    │  │ Pub/Sub Messaging               ││ │   │
│  │  │                     │    │  │ • Azure Service Bus             ││ │   │
│  │  │                     │    │  │ • Azure Event Hubs              ││ │   │
│  │  │                     │    │  │ • Azure Event Grid              ││ │   │
│  │  │                     │    │  └─────────────────────────────────┘│ │   │
│  │  │                     │    │                                      │ │   │
│  │  │                     │    │  ┌─────────────────────────────────┐│ │   │
│  │  │                     │    │  │ Bindings (Input/Output)         ││ │   │
│  │  │                     │    │  │ • Azure Blob Storage            ││ │   │
│  │  │                     │    │  │ • Azure SignalR                 ││ │   │
│  │  │                     │    │  │ • SendGrid, Twilio              ││ │   │
│  │  │                     │    │  └─────────────────────────────────┘│ │   │
│  │  │                     │    │                                      │ │   │
│  │  │                     │    │  ┌─────────────────────────────────┐│ │   │
│  │  │                     │    │  │ Secrets                         ││ │   │
│  │  │                     │    │  │ • Azure Key Vault               ││ │   │
│  │  │                     │    │  └─────────────────────────────────┘│ │   │
│  │  │                     │    │                                      │ │   │
│  │  └─────────────────────┘    └─────────────────────────────────────┘ │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Enable Dapr

```bash
# Create app with Dapr enabled
az containerapp create \
  --name order-api \
  --resource-group container-apps-rg \
  --environment prod-env \
  --image myacr.azurecr.io/order-api:v1 \
  --target-port 8080 \
  --ingress internal \
  --enable-dapr \
  --dapr-app-id order-api \
  --dapr-app-port 8080 \
  --dapr-app-protocol http
```

### Dapr Components

```yaml
# State store component - Azure Cosmos DB
componentType: state.azure.cosmosdb
version: v1
metadata:
  - name: url
    value: https://myaccount.documents.azure.com:443/
  - name: database
    value: ordersdb
  - name: collection
    value: orders
  - name: masterKey
    secretRef: cosmos-key
secrets:
  - name: cosmos-key
    keyVaultUrl: https://mykv.vault.azure.net/secrets/cosmos-key
    identity: system
scopes:
  - order-api
  - inventory-api
```

```bash
# Create Dapr component via CLI
az containerapp env dapr-component set \
  --name prod-env \
  --resource-group container-apps-rg \
  --dapr-component-name statestore \
  --yaml statestore.yaml
```

### Pub/Sub Component

```yaml
# Pub/Sub component - Azure Service Bus
componentType: pubsub.azure.servicebus.topics
version: v1
metadata:
  - name: connectionString
    secretRef: servicebus-connection
  - name: consumerID
    value: order-processor
secrets:
  - name: servicebus-connection
    keyVaultUrl: https://mykv.vault.azure.net/secrets/servicebus-connection
    identity: system
scopes:
  - order-api
  - notification-service
```

### Dapr Service Invocation

```python
# Python example - calling another service via Dapr
import requests

DAPR_HTTP_PORT = 3500

def get_inventory(product_id: str):
    # Dapr service invocation
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/invoke/inventory-api/method/products/{product_id}"
    response = requests.get(url)
    return response.json()

def publish_order_event(order: dict):
    # Dapr pub/sub
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/publish/orderpubsub/orders"
    requests.post(url, json=order)

def save_state(order_id: str, order: dict):
    # Dapr state management
    url = f"http://localhost:{DAPR_HTTP_PORT}/v1.0/state/statestore"
    state = [{"key": order_id, "value": order}]
    requests.post(url, json=state)
```

### Bicep with Dapr

```bicep
// Dapr-enabled Container App
resource orderApi 'Microsoft.App/containerApps@2024-03-01' = {
  name: 'order-api'
  location: location
  properties: {
    managedEnvironmentId: containerAppEnv.id
    configuration: {
      dapr: {
        enabled: true
        appId: 'order-api'
        appPort: 8080
        appProtocol: 'http'
        enableApiLogging: true
        logLevel: 'info'
      }
      ingress: {
        external: false
        targetPort: 8080
      }
    }
    template: {
      containers: [
        {
          name: 'order-api'
          image: 'myacr.azurecr.io/order-api:v1'
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
      scale: {
        minReplicas: 2
        maxReplicas: 20
      }
    }
  }
}

// Dapr State Store Component
resource stateStore 'Microsoft.App/managedEnvironments/daprComponents@2024-03-01' = {
  parent: containerAppEnv
  name: 'statestore'
  properties: {
    componentType: 'state.azure.cosmosdb'
    version: 'v1'
    metadata: [
      { name: 'url', value: cosmosAccount.properties.documentEndpoint }
      { name: 'database', value: 'ordersdb' }
      { name: 'collection', value: 'orders' }
    ]
    secretStoreComponent: 'secretstore'
    scopes: ['order-api', 'inventory-api']
  }
}
```

## KEDA Scaling

### KEDA Scale Rules

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    KEDA SCALING IN CONTAINER APPS                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     EVENT SOURCES                                      │  │
│  │                                                                        │  │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ │  │
│  │  │ Service Bus  │ │ Event Hubs   │ │ Storage Queue│ │ HTTP Traffic │ │  │
│  │  │ Queue: 500   │ │ Lag: 1000    │ │ Length: 200  │ │ RPS: 1000    │ │  │
│  │  │ messages     │ │ messages     │ │ messages     │ │ concurrent   │ │  │
│  │  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ └──────┬───────┘ │  │
│  │         │                │                │                │         │  │
│  └─────────┼────────────────┼────────────────┼────────────────┼─────────┘  │
│            │                │                │                │            │
│            └────────────────┴────────────────┴────────────────┘            │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     KEDA SCALER                                        │  │
│  │                                                                        │  │
│  │  Calculate: (Event Count / Threshold) = Desired Replicas              │  │
│  │  Example:   (500 messages / 10 per pod) = 50 replicas                 │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                    │                                        │
│                                    ▼                                        │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     CONTAINER APP REPLICAS                             │  │
│  │                                                                        │  │
│  │  Scale to Zero ──► 1 ──► 5 ──► 10 ──► 50 ──► 100 (max)              │  │
│  │                                                                        │  │
│  │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐         ┌─────┐            │  │
│  │  │ Pod │ │ Pod │ │ Pod │ │ Pod │ │ Pod │  ...    │ Pod │            │  │
│  │  │  1  │ │  2  │ │  3  │ │  4  │ │  5  │         │ 100 │            │  │
│  │  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘         └─────┘            │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Scale Rules Configuration

```bash
# HTTP scaling rule
az containerapp create \
  --name web-api \
  --resource-group container-apps-rg \
  --environment prod-env \
  --image myacr.azurecr.io/web-api:v1 \
  --min-replicas 0 \
  --max-replicas 100 \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 100

# Azure Service Bus queue scaling
az containerapp create \
  --name queue-processor \
  --resource-group container-apps-rg \
  --environment prod-env \
  --image myacr.azurecr.io/queue-processor:v1 \
  --min-replicas 0 \
  --max-replicas 50 \
  --scale-rule-name servicebus-rule \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata "queueName=orders" "messageCount=10" \
  --scale-rule-auth "connection=servicebus-connection" \
  --secrets "servicebus-connection=<connection-string>"
```

### Bicep Scale Rules

```bicep
resource queueProcessor 'Microsoft.App/containerApps@2024-03-01' = {
  name: 'queue-processor'
  location: location
  properties: {
    managedEnvironmentId: containerAppEnv.id
    configuration: {
      secrets: [
        {
          name: 'servicebus-connection'
          keyVaultUrl: 'https://mykv.vault.azure.net/secrets/servicebus-connection'
          identity: 'system'
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'processor'
          image: 'myacr.azurecr.io/queue-processor:v1'
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
      scale: {
        minReplicas: 0  // Scale to zero when queue is empty
        maxReplicas: 100
        rules: [
          {
            name: 'servicebus-queue-rule'
            azureQueue: null
            custom: {
              type: 'azure-servicebus'
              metadata: {
                queueName: 'orders'
                messageCount: '10'
                activationMessageCount: '1'
              }
              auth: [
                {
                  secretRef: 'servicebus-connection'
                  triggerParameter: 'connection'
                }
              ]
            }
          }
        ]
      }
    }
  }
}
```

### Common KEDA Scalers

| Scaler | Use Case | Key Metadata |
|--------|----------|--------------|
| azure-servicebus | Queue processing | queueName, messageCount |
| azure-eventhub | Event streaming | consumerGroup, unprocessedEventThreshold |
| azure-queue | Storage queue | queueName, queueLength |
| azure-blob | Blob triggers | blobContainerName, blobCount |
| http | Web APIs | concurrentRequests |
| cpu | CPU-based | type: Utilization, value: 70 |
| memory | Memory-based | type: Utilization, value: 70 |
| cron | Scheduled scaling | timezone, start, end, desiredReplicas |

## Ingress and Networking

### Ingress Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONTAINER APPS INGRESS                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  EXTERNAL INGRESS                           INTERNAL INGRESS                │
│  ───────────────                           ───────────────                  │
│                                                                              │
│  ┌──────────────┐                          ┌──────────────┐                │
│  │   Internet   │                          │  Other Apps  │                │
│  │    Users     │                          │   in VNet    │                │
│  └──────┬───────┘                          └──────┬───────┘                │
│         │                                         │                         │
│         ▼                                         ▼                         │
│  ┌──────────────────────────────┐       ┌──────────────────────────────┐  │
│  │  Public FQDN                 │       │  Internal FQDN               │  │
│  │  myapp.{region}.azurecontai- │       │  myapp.internal.{env-id}.    │  │
│  │  nerapps.io                  │       │  {region}.azurecontainer-    │  │
│  │                              │       │  apps.io                      │  │
│  │  • Auto TLS (managed cert)   │       │                              │  │
│  │  • Custom domain support     │       │  • Only accessible within    │  │
│  │  • IP restrictions           │       │    VNet/environment          │  │
│  └──────────────────────────────┘       │  • Service-to-service calls  │  │
│                                         └──────────────────────────────┘  │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                    ENVOY PROXY (Built-in)                              │  │
│  │                                                                        │  │
│  │  Features:                                                             │  │
│  │  • TLS termination              • Request timeout                     │  │
│  │  • Path-based routing           • Retry policies                      │  │
│  │  • Traffic splitting            • CORS                                │  │
│  │  • Session affinity             • IP restrictions                     │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Ingress Configuration

```bash
# External ingress with custom domain
az containerapp ingress enable \
  --name frontend-app \
  --resource-group container-apps-rg \
  --type external \
  --target-port 8080 \
  --transport auto

# Add custom domain
az containerapp hostname add \
  --name frontend-app \
  --resource-group container-apps-rg \
  --hostname app.contoso.com

# Bind managed certificate
az containerapp hostname bind \
  --name frontend-app \
  --resource-group container-apps-rg \
  --hostname app.contoso.com \
  --environment prod-env \
  --validation-method CNAME

# IP restrictions
az containerapp ingress access-restriction set \
  --name frontend-app \
  --resource-group container-apps-rg \
  --rule-name allow-corporate \
  --ip-address 203.0.113.0/24 \
  --action Allow
```

### Bicep Ingress

```bicep
resource frontendApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: 'frontend-app'
  location: location
  properties: {
    managedEnvironmentId: containerAppEnv.id
    configuration: {
      ingress: {
        external: true
        targetPort: 8080
        transport: 'auto'
        allowInsecure: false

        // Traffic splitting
        traffic: [
          {
            revisionName: 'frontend-app--v1'
            weight: 90
            label: 'stable'
          }
          {
            revisionName: 'frontend-app--v2'
            weight: 10
            label: 'canary'
          }
        ]

        // Custom domain
        customDomains: [
          {
            name: 'app.contoso.com'
            certificateId: certificate.id
            bindingType: 'SniEnabled'
          }
        ]

        // IP restrictions
        ipSecurityRestrictions: [
          {
            name: 'allow-corporate'
            ipAddressRange: '203.0.113.0/24'
            action: 'Allow'
          }
          {
            name: 'deny-all'
            ipAddressRange: '0.0.0.0/0'
            action: 'Deny'
          }
        ]

        // Sticky sessions
        stickySessions: {
          affinity: 'sticky'
        }

        // CORS
        corsPolicy: {
          allowedOrigins: ['https://contoso.com', 'https://www.contoso.com']
          allowedMethods: ['GET', 'POST', 'PUT', 'DELETE', 'OPTIONS']
          allowedHeaders: ['*']
          exposeHeaders: ['X-Custom-Header']
          maxAge: 3600
          allowCredentials: true
        }
      }
    }
    template: {
      // ... container configuration
    }
  }
}
```

## Managed Identity Integration

### Identity Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MANAGED IDENTITY IN CONTAINER APPS                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     CONTAINER APP                                    │   │
│  │                                                                      │   │
│  │  ┌─────────────────────────┐    ┌──────────────────────────────┐  │   │
│  │  │  SYSTEM-ASSIGNED        │    │  USER-ASSIGNED               │  │   │
│  │  │  MANAGED IDENTITY       │    │  MANAGED IDENTITY            │  │   │
│  │  │                         │    │                              │  │   │
│  │  │  • Auto-created         │    │  • Pre-created               │  │   │
│  │  │  • Tied to app lifecycle│    │  • Shared across apps        │  │   │
│  │  │  • Can't be shared      │    │  • Independent lifecycle     │  │   │
│  │  │                         │    │  • Best for multi-app access │  │   │
│  │  └────────────┬────────────┘    └──────────────┬───────────────┘  │   │
│  │               │                                │                   │   │
│  └───────────────┼────────────────────────────────┼───────────────────┘   │
│                  │                                │                        │
│                  └────────────────┬───────────────┘                        │
│                                   │                                        │
│                                   ▼                                        │
│                  ┌────────────────────────────────────┐                    │
│                  │           ENTRA ID                 │                    │
│                  │                                    │                    │
│                  │   Identity → Azure RBAC Role      │                    │
│                  │                                    │                    │
│                  └────────────────────────────────────┘                    │
│                                   │                                        │
│           ┌───────────────────────┼───────────────────────┐               │
│           │                       │                       │               │
│           ▼                       ▼                       ▼               │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐       │
│  │   Key Vault     │    │  Storage Acct   │    │   Azure SQL     │       │
│  │                 │    │                 │    │                 │       │
│  │ Role: Secrets   │    │ Role: Blob      │    │ Role: SQL       │       │
│  │ User            │    │ Contributor     │    │ Contributor     │       │
│  └─────────────────┘    └─────────────────┘    └─────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Configure Managed Identity

```bash
# Enable system-assigned identity
az containerapp identity assign \
  --name api-app \
  --resource-group container-apps-rg \
  --system-assigned

# Create and assign user-assigned identity
az identity create \
  --name container-app-identity \
  --resource-group container-apps-rg

az containerapp identity assign \
  --name api-app \
  --resource-group container-apps-rg \
  --user-assigned /subscriptions/{sub}/resourceGroups/container-apps-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/container-app-identity

# Grant Key Vault access
az keyvault set-policy \
  --name mykv \
  --object-id $(az containerapp show -n api-app -g container-apps-rg --query identity.principalId -o tsv) \
  --secret-permissions get list

# Grant Storage access
az role assignment create \
  --role "Storage Blob Data Contributor" \
  --assignee $(az containerapp show -n api-app -g container-apps-rg --query identity.principalId -o tsv) \
  --scope /subscriptions/{sub}/resourceGroups/container-apps-rg/providers/Microsoft.Storage/storageAccounts/mystorageaccount
```

### Using Identity in Code

```csharp
// C# - Azure SDK with DefaultAzureCredential
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Azure.Storage.Blobs;

// Works automatically with managed identity in Container Apps
var credential = new DefaultAzureCredential();

// Key Vault
var secretClient = new SecretClient(
    new Uri("https://mykv.vault.azure.net/"),
    credential);
KeyVaultSecret secret = await secretClient.GetSecretAsync("my-secret");

// Blob Storage
var blobServiceClient = new BlobServiceClient(
    new Uri("https://mystorageaccount.blob.core.windows.net"),
    credential);
```

```python
# Python - Azure SDK with DefaultAzureCredential
from azure.identity import DefaultAzureCredential
from azure.keyvault.secrets import SecretClient
from azure.storage.blob import BlobServiceClient

# Works automatically with managed identity
credential = DefaultAzureCredential()

# Key Vault
secret_client = SecretClient(
    vault_url="https://mykv.vault.azure.net/",
    credential=credential
)
secret = secret_client.get_secret("my-secret")

# Blob Storage
blob_service = BlobServiceClient(
    account_url="https://mystorageaccount.blob.core.windows.net",
    credential=credential
)
```

## Revisions and Traffic Splitting

### Revision Management

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    REVISIONS & TRAFFIC SPLITTING                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                     CONTAINER APP: frontend-app                        │  │
│  │                                                                        │  │
│  │  Revision Mode: Multiple                                               │  │
│  │                                                                        │  │
│  │  ┌────────────────────┐  ┌────────────────────┐  ┌──────────────────┐ │  │
│  │  │ Revision v1        │  │ Revision v2        │  │ Revision v3      │ │  │
│  │  │ (frontend-app--v1) │  │ (frontend-app--v2) │  │ (frontend-app--v3│ │  │
│  │  │                    │  │                    │  │                  │ │  │
│  │  │ Image: app:1.0     │  │ Image: app:2.0     │  │ Image: app:3.0   │ │  │
│  │  │ Created: Jan 1     │  │ Created: Jan 15    │  │ Created: Jan 28  │ │  │
│  │  │                    │  │                    │  │                  │ │  │
│  │  │ Traffic: 0%        │  │ Traffic: 90%       │  │ Traffic: 10%     │ │  │
│  │  │ Label: deprecated  │  │ Label: stable      │  │ Label: canary    │ │  │
│  │  │ Status: Inactive   │  │ Status: Active     │  │ Status: Active   │ │  │
│  │  └────────────────────┘  └────────────────────┘  └──────────────────┘ │  │
│  │                                                                        │  │
│  │  Traffic Distribution:                                                 │  │
│  │  ═══════════════════════════════════════════════════ 90% → v2         │  │
│  │  ══════ 10% → v3 (canary)                                             │  │
│  │                                                                        │  │
│  │  Direct URL Access:                                                    │  │
│  │  • stable.frontend-app.{region}.azurecontainerapps.io → v2            │  │
│  │  • canary.frontend-app.{region}.azurecontainerapps.io → v3            │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Manage Revisions

```bash
# Set revision mode to multiple
az containerapp revision set-mode \
  --name frontend-app \
  --resource-group container-apps-rg \
  --mode multiple

# Create new revision with update
az containerapp update \
  --name frontend-app \
  --resource-group container-apps-rg \
  --image myacr.azurecr.io/frontend:v2 \
  --revision-suffix v2

# List revisions
az containerapp revision list \
  --name frontend-app \
  --resource-group container-apps-rg \
  --output table

# Split traffic (blue-green)
az containerapp ingress traffic set \
  --name frontend-app \
  --resource-group container-apps-rg \
  --revision-weight frontend-app--v1=50 frontend-app--v2=50

# Canary deployment (10% to new)
az containerapp ingress traffic set \
  --name frontend-app \
  --resource-group container-apps-rg \
  --revision-weight frontend-app--v1=90 frontend-app--v2=10

# Full rollout to new revision
az containerapp ingress traffic set \
  --name frontend-app \
  --resource-group container-apps-rg \
  --revision-weight frontend-app--v2=100

# Add label for direct access
az containerapp revision label add \
  --name frontend-app \
  --resource-group container-apps-rg \
  --revision frontend-app--v2 \
  --label stable

az containerapp revision label add \
  --name frontend-app \
  --resource-group container-apps-rg \
  --revision frontend-app--v3 \
  --label canary

# Deactivate old revision
az containerapp revision deactivate \
  --name frontend-app \
  --resource-group container-apps-rg \
  --revision frontend-app--v1
```

### Bicep Traffic Configuration

```bicep
resource frontendApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: 'frontend-app'
  location: location
  properties: {
    managedEnvironmentId: containerAppEnv.id
    configuration: {
      activeRevisionsMode: 'Multiple'
      ingress: {
        external: true
        targetPort: 8080
        traffic: [
          {
            revisionName: 'frontend-app--v2'
            weight: 90
            label: 'stable'
          }
          {
            revisionName: 'frontend-app--v3'
            weight: 10
            label: 'canary'
          }
        ]
      }
    }
    template: {
      revisionSuffix: 'v3'
      containers: [
        {
          name: 'frontend'
          image: 'myacr.azurecr.io/frontend:v3'
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
    }
  }
}
```

## Secrets and Environment Variables

### Secret Management Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    SECRETS MANAGEMENT OPTIONS                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  OPTION 1: Container Apps Secrets (Simple)                                  │
│  ──────────────────────────────────────────                                 │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Container App                                                       │   │
│  │  ├── secrets:                                                        │   │
│  │  │   ├── db-password: "stored-encrypted-in-container-apps"          │   │
│  │  │   └── api-key: "stored-encrypted-in-container-apps"              │   │
│  │  └── env:                                                            │   │
│  │      ├── DB_PASSWORD: secretRef=db-password                         │   │
│  │      └── API_KEY: secretRef=api-key                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  OPTION 2: Key Vault References (Recommended for Production)                │
│  ───────────────────────────────────────────────────────────                │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Container App                          Azure Key Vault              │   │
│  │  ├── secrets:                           ┌──────────────────────┐    │   │
│  │  │   ├── db-password ──────────────────►│ db-password          │    │   │
│  │  │   │   keyVaultUrl: https://kv/...    │ (Version controlled) │    │   │
│  │  │   │   identity: system               └──────────────────────┘    │   │
│  │  │   │                                                               │   │
│  │  │   └── api-key ──────────────────────►┌──────────────────────┐    │   │
│  │  │       keyVaultUrl: https://kv/...    │ api-key              │    │   │
│  │  │       identity: user-assigned        │ (Rotation enabled)   │    │   │
│  │  │                                      └──────────────────────┘    │   │
│  │  └── env:                                                            │   │
│  │      ├── DB_PASSWORD: secretRef=db-password                         │   │
│  │      └── API_KEY: secretRef=api-key                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  OPTION 3: Volume Mounts (For Files/Certificates)                           │
│  ─────────────────────────────────────────────────                          │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Container App                          Azure Key Vault              │   │
│  │  ├── volumes:                           ┌──────────────────────┐    │   │
│  │  │   └── secrets-volume ───────────────►│ tls-cert             │    │   │
│  │  │       (Azure Key Vault)              │ (Certificate)        │    │   │
│  │  │                                      └──────────────────────┘    │   │
│  │  └── volumeMounts:                                                   │   │
│  │      └── /mnt/secrets → secrets-volume                              │   │
│  │                                                                      │   │
│  │  Files available at:                                                 │   │
│  │  /mnt/secrets/tls-cert                                              │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Configure Secrets

```bash
# Create secret directly
az containerapp secret set \
  --name api-app \
  --resource-group container-apps-rg \
  --secrets "db-password=MySecretPassword123"

# Create secret from Key Vault
az containerapp secret set \
  --name api-app \
  --resource-group container-apps-rg \
  --secrets "db-connection=keyvaultref:https://mykv.vault.azure.net/secrets/db-connection,identityref:system"

# Use secret as environment variable
az containerapp update \
  --name api-app \
  --resource-group container-apps-rg \
  --set-env-vars "DB_CONNECTION=secretref:db-connection"

# List secrets
az containerapp secret list \
  --name api-app \
  --resource-group container-apps-rg

# Remove secret
az containerapp secret remove \
  --name api-app \
  --resource-group container-apps-rg \
  --secret-names db-password
```

### Bicep Secrets Configuration

```bicep
resource apiApp 'Microsoft.App/containerApps@2024-03-01' = {
  name: 'api-app'
  location: location
  identity: {
    type: 'SystemAssigned,UserAssigned'
    userAssignedIdentities: {
      '${userIdentity.id}': {}
    }
  }
  properties: {
    managedEnvironmentId: containerAppEnv.id
    configuration: {
      secrets: [
        // Direct secret (not recommended for production)
        {
          name: 'simple-secret'
          value: 'my-value'
        }
        // Key Vault reference with system identity
        {
          name: 'db-connection'
          keyVaultUrl: 'https://mykv.vault.azure.net/secrets/db-connection'
          identity: 'system'
        }
        // Key Vault reference with user-assigned identity
        {
          name: 'api-key'
          keyVaultUrl: 'https://mykv.vault.azure.net/secrets/api-key'
          identity: userIdentity.id
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'api'
          image: 'myacr.azurecr.io/api:v1'
          env: [
            // Plain environment variable
            {
              name: 'ASPNETCORE_ENVIRONMENT'
              value: 'Production'
            }
            // Secret reference
            {
              name: 'DB_CONNECTION'
              secretRef: 'db-connection'
            }
            {
              name: 'API_KEY'
              secretRef: 'api-key'
            }
          ]
          volumeMounts: [
            {
              volumeName: 'secrets-volume'
              mountPath: '/mnt/secrets'
            }
          ]
        }
      ]
      volumes: [
        {
          name: 'secrets-volume'
          storageType: 'AzureFile'
          storageName: 'myfileshare'
        }
      ]
    }
  }
}
```

## Event-Driven Scenarios

### Container Apps Jobs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    EVENT-DRIVEN ARCHITECTURES                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  SCENARIO 1: Queue-Triggered Processing                                     │
│  ────────────────────────────────────────                                   │
│                                                                              │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────────┐  │
│  │ Service Bus  │───►│ Container App    │───►│ Storage / Database       │  │
│  │ Queue        │    │ (KEDA: 0-100)    │    │                          │  │
│  │              │    │                  │    │ Processed results        │  │
│  │ Orders queue │    │ Order processor  │    │                          │  │
│  └──────────────┘    └──────────────────┘    └──────────────────────────┘  │
│                                                                              │
│  SCENARIO 2: Scheduled Batch Jobs                                           │
│  ────────────────────────────────                                           │
│                                                                              │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────────┐  │
│  │ Cron         │───►│ Container Apps   │───►│ Generate reports         │  │
│  │ Schedule     │    │ JOB              │    │ Send emails              │  │
│  │              │    │                  │    │ Cleanup data             │  │
│  │ 0 2 * * *    │    │ Batch processor  │    │                          │  │
│  └──────────────┘    └──────────────────┘    └──────────────────────────┘  │
│                                                                              │
│  SCENARIO 3: Event-Driven Microservices                                     │
│  ───────────────────────────────────────                                    │
│                                                                              │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────────┐  │
│  │ Event Grid   │───►│ Container App    │    │                          │  │
│  │              │    │ (Event handler)  │───►│ Trigger workflows        │  │
│  │ Blob created │    │                  │    │ Process images           │  │
│  │ IoT events   │    │ Dapr pub/sub     │    │ Send notifications       │  │
│  └──────────────┘    └──────────────────┘    └──────────────────────────┘  │
│                                                                              │
│  SCENARIO 4: HTTP-Triggered Scale                                           │
│  ────────────────────────────────                                           │
│                                                                              │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────────┐  │
│  │ API Gateway  │───►│ Container App    │───►│ Backend services         │  │
│  │ / Front Door │    │ (Scale 0-300)    │    │                          │  │
│  │              │    │                  │    │ Databases                │  │
│  │ HTTP traffic │    │ Scale on RPS     │    │ Cache                    │  │
│  └──────────────┘    └──────────────────┘    └──────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Container Apps Jobs

```bash
# Create scheduled job (cron)
az containerapp job create \
  --name daily-report-job \
  --resource-group container-apps-rg \
  --environment prod-env \
  --image myacr.azurecr.io/report-generator:v1 \
  --trigger-type Schedule \
  --cron-expression "0 2 * * *" \
  --replica-timeout 1800 \
  --replica-retry-limit 3 \
  --replica-completion-count 1 \
  --parallelism 1 \
  --cpu 1 \
  --memory 2Gi

# Create event-triggered job (Service Bus)
az containerapp job create \
  --name order-processor-job \
  --resource-group container-apps-rg \
  --environment prod-env \
  --image myacr.azurecr.io/order-processor:v1 \
  --trigger-type Event \
  --min-executions 0 \
  --max-executions 50 \
  --scale-rule-name servicebus \
  --scale-rule-type azure-servicebus \
  --scale-rule-metadata "queueName=orders" "messageCount=1" \
  --scale-rule-auth "connection=servicebus-connection" \
  --secrets "servicebus-connection=<connection-string>" \
  --replica-timeout 300 \
  --cpu 0.5 \
  --memory 1Gi

# Create manual job
az containerapp job create \
  --name migration-job \
  --resource-group container-apps-rg \
  --environment prod-env \
  --image myacr.azurecr.io/migration:v1 \
  --trigger-type Manual \
  --replica-timeout 3600 \
  --cpu 2 \
  --memory 4Gi

# Start manual job
az containerapp job start \
  --name migration-job \
  --resource-group container-apps-rg

# List job executions
az containerapp job execution list \
  --name daily-report-job \
  --resource-group container-apps-rg
```

### Bicep Jobs

```bicep
// Scheduled job
resource scheduledJob 'Microsoft.App/jobs@2024-03-01' = {
  name: 'daily-cleanup-job'
  location: location
  properties: {
    environmentId: containerAppEnv.id
    workloadProfileName: 'Consumption'
    configuration: {
      triggerType: 'Schedule'
      scheduleTriggerConfig: {
        cronExpression: '0 3 * * *'  // 3 AM daily
        parallelism: 1
        replicaCompletionCount: 1
      }
      replicaTimeout: 1800
      replicaRetryLimit: 3
      secrets: [
        {
          name: 'db-connection'
          keyVaultUrl: 'https://mykv.vault.azure.net/secrets/db-connection'
          identity: 'system'
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'cleanup'
          image: 'myacr.azurecr.io/cleanup:v1'
          resources: {
            cpu: json('1.0')
            memory: '2Gi'
          }
          env: [
            {
              name: 'DB_CONNECTION'
              secretRef: 'db-connection'
            }
          ]
        }
      ]
    }
  }
}

// Event-driven job
resource eventJob 'Microsoft.App/jobs@2024-03-01' = {
  name: 'image-processor-job'
  location: location
  properties: {
    environmentId: containerAppEnv.id
    configuration: {
      triggerType: 'Event'
      eventTriggerConfig: {
        parallelism: 10
        replicaCompletionCount: 1
        scale: {
          minExecutions: 0
          maxExecutions: 100
          pollingInterval: 30
          rules: [
            {
              name: 'blob-trigger'
              type: 'azure-blob'
              metadata: {
                blobContainerName: 'images'
                blobCount: '1'
              }
              auth: [
                {
                  secretRef: 'storage-connection'
                  triggerParameter: 'connection'
                }
              ]
            }
          ]
        }
      }
      replicaTimeout: 300
      secrets: [
        {
          name: 'storage-connection'
          keyVaultUrl: 'https://mykv.vault.azure.net/secrets/storage-connection'
          identity: 'system'
        }
      ]
    }
    template: {
      containers: [
        {
          name: 'processor'
          image: 'myacr.azurecr.io/image-processor:v1'
          resources: {
            cpu: json('0.5')
            memory: '1Gi'
          }
        }
      ]
    }
  }
}
```

## Best Practices Summary

| Category | Best Practice |
|----------|---------------|
| **Identity** | Use managed identity for all Azure resource access; prefer user-assigned for shared access |
| **Secrets** | Store secrets in Key Vault with references; never hardcode secrets |
| **Scaling** | Set appropriate min/max replicas; use scale to zero for cost savings |
| **Networking** | Use VNet integration for enterprise; configure IP restrictions |
| **Revisions** | Use multiple revision mode for blue-green/canary deployments |
| **Monitoring** | Enable Application Insights; configure alerts on key metrics |
| **Dapr** | Use Dapr for service-to-service communication and state management |
| **Cost** | Use Consumption tier for variable workloads; Dedicated for predictable loads |
| **Images** | Use ACR with managed identity; enable vulnerability scanning |
| **Jobs** | Use jobs for batch processing; scheduled tasks; event-driven processing |

## Migration from AWS

| AWS Pattern | Azure Container Apps Equivalent |
|-------------|--------------------------------|
| ECS Task Definition | Container App template |
| ECS Service | Container App (always-on) |
| ECS Scheduled Task | Container Apps Job (scheduled) |
| App Runner Service | Container App with external ingress |
| Lambda Container | Container Apps Job (event-triggered) |
| Fargate Spot | Consumption workload profile |
| ALB Target Group | Built-in ingress with traffic splitting |
| CloudMap | Dapr service invocation |
| EventBridge Rules | KEDA scale rules |
| Secrets Manager | Key Vault references |

---

*Continue to [Quick Reference](quick-reference.md)*

*Back to [AKS Architecture](01-aks-architecture.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
