# Deployment Patterns

## Deployment Stamps Pattern

### Problem

You need to scale an application globally with tenant isolation and regional data residency.

### Solution

```
DEPLOYMENT STAMPS:
──────────────────

Each "stamp" is a complete, independent deployment:

┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  GLOBAL ROUTER (Azure Front Door / Traffic Manager)                   │
│                          │                                            │
│          ┌───────────────┼───────────────┐                           │
│          ▼               ▼               ▼                           │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                  │
│  │  STAMP: US   │ │  STAMP: EU   │ │  STAMP: APAC │                  │
│  │              │ │              │ │              │                  │
│  │ ┌──────────┐ │ │ ┌──────────┐ │ │ ┌──────────┐ │                  │
│  │ │ App Svc  │ │ │ │ App Svc  │ │ │ │ App Svc  │ │                  │
│  │ ├──────────┤ │ │ ├──────────┤ │ │ ├──────────┤ │                  │
│  │ │ Database │ │ │ │ Database │ │ │ │ Database │ │                  │
│  │ ├──────────┤ │ │ ├──────────┤ │ │ ├──────────┤ │                  │
│  │ │ Cache    │ │ │ │ Cache    │ │ │ │ Cache    │ │                  │
│  │ ├──────────┤ │ │ ├──────────┤ │ │ ├──────────┤ │                  │
│  │ │ Storage  │ │ │ │ Storage  │ │ │ │ Storage  │ │                  │
│  │ └──────────┘ │ │ └──────────┘ │ │ └──────────┘ │                  │
│  │              │ │              │ │              │                  │
│  │ Tenants:     │ │ Tenants:     │ │ Tenants:     │                  │
│  │ A, B, C      │ │ D, E, F      │ │ G, H, I      │                  │
│  └──────────────┘ └──────────────┘ └──────────────┘                  │
│                                                                        │
│  Benefits:                                                            │
│  • Complete isolation between stamps                                  │
│  • Independent scaling per region                                    │
│  • Data residency compliance                                         │
│  • Blast radius limited to single stamp                              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```bicep
// stamp.bicep - Reusable stamp template
@description('Stamp identifier')
param stampId string

@description('Location for the stamp')
param location string

@description('Environment')
param environment string

var stampPrefix = '${stampId}-${environment}'

// App Service Plan
resource appServicePlan 'Microsoft.Web/serverfarms@2022-09-01' = {
  name: 'asp-${stampPrefix}'
  location: location
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
  }
}

// App Service
resource appService 'Microsoft.Web/sites@2022-09-01' = {
  name: 'app-${stampPrefix}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      appSettings: [
        {
          name: 'STAMP_ID'
          value: stampId
        }
      ]
    }
  }
}

// Cosmos DB with regional write
resource cosmosAccount 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' = {
  name: 'cosmos-${stampPrefix}'
  location: location
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: location
        failoverPriority: 0
      }
    ]
  }
}

// Redis Cache
resource redis 'Microsoft.Cache/redis@2023-04-01' = {
  name: 'redis-${stampPrefix}'
  location: location
  properties: {
    sku: {
      name: 'Premium'
      family: 'P'
      capacity: 1
    }
  }
}

// Storage Account
resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'st${replace(stampPrefix, '-', '')}'
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}

output appServiceHostname string = appService.properties.defaultHostName
output stampId string = stampId
```

```bicep
// main.bicep - Deploy multiple stamps
var stamps = [
  { id: 'us', location: 'eastus' }
  { id: 'eu', location: 'westeurope' }
  { id: 'apac', location: 'southeastasia' }
]

// Deploy each stamp
module stampDeployments 'stamp.bicep' = [for stamp in stamps: {
  name: 'stamp-${stamp.id}'
  params: {
    stampId: stamp.id
    location: stamp.location
    environment: environment
  }
}]

// Global routing with Front Door
resource frontDoor 'Microsoft.Cdn/profiles@2023-05-01' = {
  name: 'fd-global'
  location: 'global'
  sku: {
    name: 'Standard_AzureFrontDoor'
  }
}

resource frontDoorEndpoint 'Microsoft.Cdn/profiles/afdEndpoints@2023-05-01' = {
  parent: frontDoor
  name: 'api-endpoint'
  location: 'global'
  properties: {
    enabledState: 'Enabled'
  }
}
```

---

## Sidecar Pattern

### Problem

You need to add capabilities (logging, monitoring, security) to applications without modifying them.

### Solution

```
SIDECAR PATTERN:
────────────────

┌────────────────────────────────────────────────────────────────────────┐
│                              POD                                       │
│  ┌────────────────────┐    ┌────────────────────┐                    │
│  │  PRIMARY CONTAINER │    │  SIDECAR CONTAINER │                    │
│  │                    │    │                    │                    │
│  │  Application       │◀──▶│  Responsibilities: │                    │
│  │  (business logic)  │    │  • Logging agent   │                    │
│  │                    │    │  • Proxy (Envoy)   │                    │
│  │                    │    │  • Secrets sync    │                    │
│  │                    │    │  • Config reload   │                    │
│  │                    │    │  • TLS termination │                    │
│  └────────────────────┘    └────────────────────┘                    │
│                                                                        │
│  Shared:                                                              │
│  • Network namespace (localhost communication)                        │
│  • Volume mounts                                                      │
│  • Lifecycle                                                          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

Benefits:
• Separation of concerns
• Independent deployment/updates
• Reusable across services
• Technology agnostic (any language)
```

### Azure Kubernetes Service Implementation

```yaml
# Deployment with sidecar containers
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        # Primary application container
        - name: order-service
          image: myregistry.azurecr.io/order-service:v1
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: shared-logs
              mountPath: /var/log/app

        # Logging sidecar
        - name: log-forwarder
          image: fluent/fluent-bit:latest
          volumeMounts:
            - name: shared-logs
              mountPath: /var/log/app
            - name: fluent-config
              mountPath: /fluent-bit/etc
          env:
            - name: LOG_ANALYTICS_WORKSPACE_ID
              valueFrom:
                secretKeyRef:
                  name: log-analytics
                  key: workspace-id

        # Envoy proxy sidecar (service mesh)
        - name: envoy-proxy
          image: envoyproxy/envoy:v1.28.0
          ports:
            - containerPort: 9901  # Admin
            - containerPort: 10000 # Inbound
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy

      volumes:
        - name: shared-logs
          emptyDir: {}
        - name: fluent-config
          configMap:
            name: fluent-bit-config
        - name: envoy-config
          configMap:
            name: envoy-config
```

### Azure Container Apps with Dapr Sidecar

```bicep
// Container App with Dapr sidecar
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: 'order-service'
  location: location
  properties: {
    configuration: {
      dapr: {
        enabled: true
        appId: 'order-service'
        appPort: 8080
        appProtocol: 'http'
      }
    }
    template: {
      containers: [
        {
          name: 'order-service'
          image: 'myregistry.azurecr.io/order-service:v1'
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

```csharp
// Application using Dapr sidecar for state and pub/sub
public class OrderService
{
    private readonly DaprClient _daprClient;

    public OrderService(DaprClient daprClient)
    {
        _daprClient = daprClient;
    }

    public async Task CreateOrderAsync(Order order)
    {
        // Dapr handles state storage (Redis, Cosmos, etc.)
        await _daprClient.SaveStateAsync("statestore", order.Id, order);

        // Dapr handles pub/sub (Service Bus, Event Hub, etc.)
        await _daprClient.PublishEventAsync("pubsub", "orders", new OrderCreatedEvent
        {
            OrderId = order.Id,
            CustomerId = order.CustomerId
        });
    }

    public async Task<Order?> GetOrderAsync(string orderId)
    {
        return await _daprClient.GetStateAsync<Order>("statestore", orderId);
    }
}
```

---

## Strangler Fig Pattern

### Problem

You need to migrate from a monolith to microservices without a big-bang rewrite.

### Solution

```
STRANGLER FIG MIGRATION:
────────────────────────

PHASE 1: Facade
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Clients ──▶ API Gateway ──▶ Monolith (all traffic)                  │
│                                                                        │
│  Gateway routes everything to legacy system                           │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

PHASE 2: Extract first service
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Clients ──▶ API Gateway ──┬──▶ Orders Service (new)                 │
│                            │                                          │
│                            └──▶ Monolith (remaining)                 │
│                                                                        │
│  /orders/* → New service                                              │
│  /* → Monolith                                                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

PHASE 3: Continue extraction
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Clients ──▶ API Gateway ──┬──▶ Orders Service                       │
│                            ├──▶ Products Service                      │
│                            ├──▶ Users Service                         │
│                            └──▶ Monolith (shrinking)                 │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

PHASE 4: Monolith retired
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Clients ──▶ API Gateway ──┬──▶ Orders Service                       │
│                            ├──▶ Products Service                      │
│                            ├──▶ Users Service                         │
│                            └──▶ Payments Service                      │
│                                                                        │
│  Monolith decommissioned                                              │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure API Management Implementation

```xml
<!-- Phase 2: Route orders to new service, rest to monolith -->
<policies>
    <inbound>
        <base />
        <choose>
            <!-- New Orders service -->
            <when condition="@(context.Request.Url.Path.StartsWith("/api/orders"))">
                <set-backend-service base-url="https://orders-service.azurewebsites.net" />
                <!-- Transform if needed -->
                <rewrite-uri template="@(context.Request.Url.Path.Replace("/api/orders", "/orders"))" />
            </when>

            <!-- Legacy monolith for everything else -->
            <otherwise>
                <set-backend-service base-url="https://legacy-monolith.azurewebsites.net" />
            </otherwise>
        </choose>
    </inbound>
</policies>
```

```csharp
// Feature flags to control traffic split
public class StranglerRouter
{
    private readonly IFeatureManager _featureManager;
    private readonly IOrdersServiceClient _newOrdersService;
    private readonly ILegacyMonolithClient _legacyMonolith;

    public async Task<OrderResponse> GetOrderAsync(string orderId)
    {
        // Gradual rollout with feature flags
        if (await _featureManager.IsEnabledAsync("UseNewOrdersService"))
        {
            try
            {
                return await _newOrdersService.GetOrderAsync(orderId);
            }
            catch (Exception ex)
            {
                // Fallback to legacy on error
                _logger.LogWarning(ex, "New service failed, falling back to legacy");
                return await _legacyMonolith.GetOrderAsync(orderId);
            }
        }

        return await _legacyMonolith.GetOrderAsync(orderId);
    }
}
```

---

## Anti-Corruption Layer Pattern

### Problem

New system needs to integrate with legacy system without adopting its bad practices or data models.

### Solution

```
ANTI-CORRUPTION LAYER:
──────────────────────

┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌────────────────┐     ┌──────────────────┐     ┌────────────────┐  │
│  │   NEW SYSTEM   │     │ ANTI-CORRUPTION  │     │ LEGACY SYSTEM  │  │
│  │                │     │     LAYER        │     │                │  │
│  │  Clean models  │◀───▶│                  │◀───▶│  Messy models  │  │
│  │  Modern API    │     │  Translates:     │     │  Old protocols │  │
│  │  Domain-driven │     │  • Data formats  │     │  Tech debt     │  │
│  │                │     │  • Protocols     │     │                │  │
│  └────────────────┘     │  • Concepts      │     └────────────────┘  │
│                         └──────────────────┘                          │
│                                                                        │
│  ACL protects new system from legacy complexity                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Anti-corruption layer service
public class LegacyCustomerAdapter : ICustomerService
{
    private readonly LegacySoapClient _legacyClient;
    private readonly IMapper _mapper;
    private readonly ILogger _logger;

    // Clean interface for new system
    public async Task<Customer> GetCustomerAsync(string customerId)
    {
        // Call legacy SOAP service
        var legacyRequest = new GetCustomerInfoRequest
        {
            CUST_ID = customerId,
            INCLUDE_ORDERS = "N",
            INCLUDE_ADDR = "Y"
        };

        var legacyResponse = await _legacyClient.GetCustomerInfoAsync(legacyRequest);

        // Translate to clean domain model
        return MapToCustomer(legacyResponse);
    }

    private Customer MapToCustomer(CustomerInfoResponse legacy)
    {
        return new Customer
        {
            Id = legacy.CUST_ID,
            Name = $"{legacy.FIRST_NM} {legacy.LAST_NM}".Trim(),
            Email = NormalizeEmail(legacy.EMAIL_ADDR),
            Status = MapStatus(legacy.CUST_STATUS_CD),
            CreatedAt = ParseLegacyDate(legacy.CUST_SINCE_DT),
            Address = new Address
            {
                Line1 = legacy.ADDR_LINE_1,
                Line2 = legacy.ADDR_LINE_2,
                City = legacy.CITY_NM,
                State = legacy.STATE_CD,
                PostalCode = NormalizePostalCode(legacy.ZIP_CD),
                Country = MapCountryCode(legacy.CNTRY_CD)
            }
        };
    }

    private CustomerStatus MapStatus(string legacyCode)
    {
        return legacyCode switch
        {
            "A" => CustomerStatus.Active,
            "I" => CustomerStatus.Inactive,
            "S" => CustomerStatus.Suspended,
            "D" => CustomerStatus.Deleted,
            _ => CustomerStatus.Unknown
        };
    }

    private string NormalizeEmail(string legacyEmail)
    {
        // Legacy system stores emails in uppercase
        return legacyEmail?.ToLowerInvariant().Trim();
    }

    private DateTime ParseLegacyDate(string legacyDate)
    {
        // Legacy format: YYYYMMDD
        return DateTime.ParseExact(legacyDate, "yyyyMMdd", CultureInfo.InvariantCulture);
    }
}
```

```csharp
// Facade that decides which system to use
public class CustomerServiceFacade : ICustomerService
{
    private readonly ICustomerService _newService;
    private readonly ICustomerService _legacyAdapter;
    private readonly ICustomerMigrationTracker _migrationTracker;

    public async Task<Customer> GetCustomerAsync(string customerId)
    {
        // Check if customer has been migrated
        if (await _migrationTracker.IsMigratedAsync(customerId))
        {
            return await _newService.GetCustomerAsync(customerId);
        }

        // Use legacy through ACL
        return await _legacyAdapter.GetCustomerAsync(customerId);
    }

    public async Task UpdateCustomerAsync(Customer customer)
    {
        if (await _migrationTracker.IsMigratedAsync(customer.Id))
        {
            await _newService.UpdateCustomerAsync(customer);
        }
        else
        {
            // Write to both during migration
            await _legacyAdapter.UpdateCustomerAsync(customer);
            await _newService.UpdateCustomerAsync(customer);
            await _migrationTracker.MarkAsMigratedAsync(customer.Id);
        }
    }
}
```

---

## Ambassador Pattern

### Problem

Application needs network capabilities (retries, circuit breaking, monitoring) but you don't want to add them to every service.

### Solution

```
AMBASSADOR PATTERN:
───────────────────

Similar to Sidecar, but specifically for network proxy duties.

┌────────────────────────────────────────────────────────────────────────┐
│                              POD                                       │
│                                                                        │
│  ┌───────────────┐         ┌─────────────────────┐                   │
│  │  APPLICATION  │──local──▶│     AMBASSADOR      │──network──▶ Services │
│  │               │         │                     │                   │
│  │  Simple HTTP  │         │  • Connection pool  │                   │
│  │  calls only   │         │  • Retries          │                   │
│  │               │         │  • Circuit breaker  │                   │
│  │               │         │  • TLS termination  │                   │
│  │               │         │  • Service discovery│                   │
│  │               │         │  • Load balancing   │                   │
│  │               │         │  • Tracing          │                   │
│  └───────────────┘         └─────────────────────┘                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

Application doesn't know about network complexity - just calls localhost.
```

### Azure Implementation with Envoy

```yaml
# Envoy as ambassador sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
spec:
  template:
    spec:
      containers:
        - name: my-service
          image: my-service:v1
          env:
            # App calls ambassador on localhost
            - name: EXTERNAL_API_URL
              value: "http://localhost:10000/external-api"

        - name: ambassador
          image: envoyproxy/envoy:v1.28.0
          ports:
            - containerPort: 10000
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy

      volumes:
        - name: envoy-config
          configMap:
            name: envoy-ambassador-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-ambassador-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
        - name: external_api_listener
          address:
            socket_address:
              address: 0.0.0.0
              port_value: 10000
          filter_chains:
            - filters:
                - name: envoy.filters.network.http_connection_manager
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                    stat_prefix: external_api
                    route_config:
                      virtual_hosts:
                        - name: external_api
                          domains: ["*"]
                          routes:
                            - match: { prefix: "/external-api" }
                              route:
                                cluster: external_api_cluster
                                retry_policy:
                                  retry_on: "5xx,connect-failure"
                                  num_retries: 3

      clusters:
        - name: external_api_cluster
          connect_timeout: 5s
          type: STRICT_DNS
          lb_policy: ROUND_ROBIN
          circuit_breakers:
            thresholds:
              - max_connections: 100
                max_pending_requests: 100
                max_retries: 3
          load_assignment:
            cluster_name: external_api_cluster
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: api.external.com
                          port_value: 443
          transport_socket:
            name: envoy.transport_sockets.tls
```

---

*Back to [Quick Reference](quick-reference.md)* | *Chapter Overview: [Design Patterns](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
