# Gateway Patterns

## Gateway Routing Pattern

### Problem

Clients need to access multiple backend services but should use a single endpoint.

### Solution

```
GATEWAY ROUTING:
────────────────

WITHOUT GATEWAY:
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Client must know about each service:                                 │
│  ┌────────┐                                                           │
│  │ Client │──▶ orders.api.com                                        │
│  │        │──▶ products.api.com                                      │
│  │        │──▶ users.api.com                                         │
│  └────────┘                                                           │
│                                                                        │
│  Problems:                                                             │
│  • Client complexity                                                  │
│  • Cross-cutting concerns duplicated                                  │
│  • Hard to change service locations                                  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

WITH GATEWAY:
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌────────┐    ┌─────────┐    ┌────────────┐                        │
│  │ Client │───▶│ GATEWAY │───▶│ Orders     │                        │
│  │        │    │         │───▶│ Products   │                        │
│  │        │    │         │───▶│ Users      │                        │
│  └────────┘    └─────────┘    └────────────┘                        │
│                                                                        │
│  Single endpoint: api.myapp.com                                       │
│  /orders/* → Orders service                                          │
│  /products/* → Products service                                      │
│  /users/* → Users service                                            │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure API Management Implementation

```xml
<!-- API Management policy for routing -->
<policies>
    <inbound>
        <base />
        <choose>
            <when condition="@(context.Request.Url.Path.StartsWith("/orders"))">
                <set-backend-service base-url="https://orders-service.internal" />
            </when>
            <when condition="@(context.Request.Url.Path.StartsWith("/products"))">
                <set-backend-service base-url="https://products-service.internal" />
            </when>
            <when condition="@(context.Request.Url.Path.StartsWith("/users"))">
                <set-backend-service base-url="https://users-service.internal" />
            </when>
            <otherwise>
                <return-response>
                    <set-status code="404" reason="Not Found" />
                </return-response>
            </otherwise>
        </choose>
    </inbound>
</policies>
```

```bicep
// API Management with backend services
resource apim 'Microsoft.ApiManagement/service@2023-03-01-preview' = {
  name: 'apim-myapp'
  location: location
  sku: {
    name: 'Standard'
    capacity: 1
  }
}

resource ordersBackend 'Microsoft.ApiManagement/service/backends@2023-03-01-preview' = {
  parent: apim
  name: 'orders-backend'
  properties: {
    url: 'https://orders-service.azurewebsites.net'
    protocol: 'http'
  }
}

resource ordersApi 'Microsoft.ApiManagement/service/apis@2023-03-01-preview' = {
  parent: apim
  name: 'orders-api'
  properties: {
    displayName: 'Orders API'
    path: 'orders'
    protocols: ['https']
    serviceUrl: 'https://orders-service.azurewebsites.net'
  }
}
```

---

## Gateway Aggregation Pattern

### Problem

Client needs data from multiple services, requiring multiple round trips.

### Solution

```
GATEWAY AGGREGATION:
────────────────────

WITHOUT AGGREGATION:
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Client makes 3 requests for order details page:                      │
│                                                                        │
│  ┌────────┐                                                           │
│  │ Client │──1──▶ GET /orders/123                                    │
│  │        │◀─────                                                     │
│  │        │──2──▶ GET /customers/456                                 │
│  │        │◀─────                                                     │
│  │        │──3──▶ GET /products?ids=1,2,3                            │
│  │        │◀─────                                                     │
│  └────────┘                                                           │
│                                                                        │
│  3 round trips = high latency, especially on mobile                  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

WITH AGGREGATION:
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌────────┐    ┌─────────┐    ┌────────────┐                        │
│  │ Client │─1─▶│ GATEWAY │───▶│ Orders     │                        │
│  │        │    │         │───▶│ Customers  │                        │
│  │        │◀───│Aggregate│───▶│ Products   │                        │
│  └────────┘    └─────────┘    └────────────┘                        │
│                                                                        │
│  1 request: GET /order-details/123                                   │
│  Gateway calls 3 services in parallel, combines response             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Azure Function as aggregation gateway
public class OrderDetailsAggregator
{
    private readonly HttpClient _ordersClient;
    private readonly HttpClient _customersClient;
    private readonly HttpClient _productsClient;

    [FunctionName("GetOrderDetails")]
    public async Task<IActionResult> GetOrderDetails(
        [HttpTrigger(AuthorizationLevel.Function, "get", Route = "order-details/{orderId}")]
        HttpRequest req,
        string orderId)
    {
        // Fetch all data in parallel
        var orderTask = _ordersClient.GetFromJsonAsync<Order>($"/orders/{orderId}");
        var order = await orderTask;

        var customerTask = _customersClient.GetFromJsonAsync<Customer>(
            $"/customers/{order.CustomerId}");
        var productsTask = _productsClient.GetFromJsonAsync<List<Product>>(
            $"/products?ids={string.Join(",", order.ProductIds)}");

        await Task.WhenAll(customerTask, productsTask);

        // Aggregate into single response
        var response = new OrderDetailsResponse
        {
            Order = order,
            Customer = customerTask.Result,
            Products = productsTask.Result
        };

        return new OkObjectResult(response);
    }
}
```

```xml
<!-- API Management policy for aggregation -->
<policies>
    <inbound>
        <base />
    </inbound>
    <backend>
        <!-- Call order service -->
        <send-request mode="new" response-variable-name="orderResponse" timeout="10">
            <set-url>@($"https://orders-service/orders/{context.Request.MatchedParameters["orderId"]}")</set-url>
            <set-method>GET</set-method>
        </send-request>

        <!-- Parse order to get IDs -->
        <set-variable name="order" value="@(((IResponse)context.Variables["orderResponse"]).Body.As<JObject>())" />

        <!-- Call customer and products in parallel -->
        <send-request mode="new" response-variable-name="customerResponse" timeout="10">
            <set-url>@($"https://customers-service/customers/{((JObject)context.Variables["order"])["customerId"]}")</set-url>
            <set-method>GET</set-method>
        </send-request>

        <send-request mode="new" response-variable-name="productsResponse" timeout="10">
            <set-url>@($"https://products-service/products?ids={string.Join(",", ((JArray)((JObject)context.Variables["order"])["productIds"]).Select(p => p.ToString()))}")</set-url>
            <set-method>GET</set-method>
        </send-request>
    </backend>
    <outbound>
        <!-- Aggregate responses -->
        <return-response>
            <set-status code="200" />
            <set-header name="Content-Type" exists-action="override">
                <value>application/json</value>
            </set-header>
            <set-body>@{
                var order = ((IResponse)context.Variables["orderResponse"]).Body.As<JObject>();
                var customer = ((IResponse)context.Variables["customerResponse"]).Body.As<JObject>();
                var products = ((IResponse)context.Variables["productsResponse"]).Body.As<JArray>();

                return new JObject(
                    new JProperty("order", order),
                    new JProperty("customer", customer),
                    new JProperty("products", products)
                ).ToString();
            }</set-body>
        </return-response>
    </outbound>
</policies>
```

---

## Backends for Frontends (BFF) Pattern

### Problem

Different clients (web, mobile, IoT) have different API needs.

### Solution

```
BACKENDS FOR FRONTENDS:
───────────────────────

WITHOUT BFF (One API for all):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌─────────┐                                                          │
│  │   Web   │──┐                                                       │
│  └─────────┘  │                                                       │
│  ┌─────────┐  │    ┌─────────────┐    ┌──────────────┐              │
│  │ Mobile  │──┼───▶│ General API │───▶│   Services   │              │
│  └─────────┘  │    └─────────────┘    └──────────────┘              │
│  ┌─────────┐  │                                                       │
│  │   IoT   │──┘                                                       │
│  └─────────┘                                                          │
│                                                                        │
│  Problems:                                                             │
│  • Mobile gets too much data (battery, bandwidth)                    │
│  • Web misses features it needs                                      │
│  • IoT can't parse complex responses                                 │
│  • Changes affect all clients                                        │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

WITH BFF (Dedicated backends):
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌─────────┐    ┌─────────────┐                                      │
│  │   Web   │───▶│  Web BFF    │──┐                                   │
│  └─────────┘    └─────────────┘  │                                   │
│                                   │    ┌──────────────┐              │
│  ┌─────────┐    ┌─────────────┐  ├───▶│   Services   │              │
│  │ Mobile  │───▶│ Mobile BFF  │──┤    └──────────────┘              │
│  └─────────┘    └─────────────┘  │                                   │
│                                   │                                   │
│  ┌─────────┐    ┌─────────────┐  │                                   │
│  │   IoT   │───▶│  IoT BFF    │──┘                                   │
│  └─────────┘    └─────────────┘                                      │
│                                                                        │
│  Each BFF optimized for its client:                                  │
│  • Web BFF: Full data, complex features                              │
│  • Mobile BFF: Minimal data, optimized payloads                      │
│  • IoT BFF: Binary protocols, simple responses                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```csharp
// Web BFF - Full featured
[ApiController]
[Route("api/web/orders")]
public class WebOrdersController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<WebOrderDetailsDto> GetOrderDetails(string id)
    {
        // Rich response with all details
        return new WebOrderDetailsDto
        {
            Order = await _orderService.GetAsync(id),
            Customer = await _customerService.GetAsync(order.CustomerId),
            Products = await _productService.GetManyAsync(order.ProductIds),
            ShippingHistory = await _shippingService.GetHistoryAsync(id),
            RelatedOrders = await _orderService.GetRelatedAsync(id),
            Recommendations = await _recommendationService.GetAsync(order.CustomerId),
            Analytics = await _analyticsService.GetOrderAnalyticsAsync(id)
        };
    }
}

// Mobile BFF - Optimized for bandwidth
[ApiController]
[Route("api/mobile/orders")]
public class MobileOrdersController : ControllerBase
{
    [HttpGet("{id}")]
    public async Task<MobileOrderSummaryDto> GetOrderSummary(string id)
    {
        var order = await _orderService.GetAsync(id);

        // Minimal response - only what's needed on mobile
        return new MobileOrderSummaryDto
        {
            Id = order.Id,
            Status = order.Status,
            Total = order.Total,
            ItemCount = order.Items.Count,
            EstimatedDelivery = order.EstimatedDelivery,
            TrackingUrl = order.TrackingUrl  // Deep link
        };
    }
}
```

```bicep
// Separate API Management products for each BFF
resource webProduct 'Microsoft.ApiManagement/service/products@2023-03-01-preview' = {
  parent: apim
  name: 'web-api'
  properties: {
    displayName: 'Web API'
    subscriptionRequired: true
    state: 'published'
  }
}

resource mobileProduct 'Microsoft.ApiManagement/service/products@2023-03-01-preview' = {
  parent: apim
  name: 'mobile-api'
  properties: {
    displayName: 'Mobile API'
    subscriptionRequired: true
    state: 'published'
  }
}
```

---

## Gateway Offloading Pattern

### Problem

Cross-cutting concerns (auth, SSL, rate limiting) are duplicated across services.

### Solution

```
GATEWAY OFFLOADING:
───────────────────

OFFLOADED CONCERNS:
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                         API GATEWAY                              │  │
│  │                                                                   │  │
│  │  ✓ SSL/TLS termination                                          │  │
│  │  ✓ Authentication (JWT validation, OAuth)                       │  │
│  │  ✓ Authorization (role checks)                                  │  │
│  │  ✓ Rate limiting & throttling                                   │  │
│  │  ✓ Request/response transformation                              │  │
│  │  ✓ Caching                                                       │  │
│  │  ✓ Logging & monitoring                                         │  │
│  │  ✓ CORS handling                                                │  │
│  │  ✓ Compression                                                  │  │
│  │                                                                   │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                │                                      │
│                                ▼                                      │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                      BACKEND SERVICES                           │  │
│  │                                                                   │  │
│  │  Focus only on business logic                                   │  │
│  │  Internal network (no SSL required)                             │  │
│  │  Trust authenticated requests                                   │  │
│  │                                                                   │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure API Management Policies

```xml
<policies>
    <inbound>
        <!-- Rate limiting -->
        <rate-limit-by-key
            calls="100"
            renewal-period="60"
            counter-key="@(context.Subscription.Key)"
            increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 400)" />

        <!-- JWT validation -->
        <validate-jwt header-name="Authorization" require-scheme="Bearer">
            <openid-config url="https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud" match="all">
                    <value>api://myapp</value>
                </claim>
            </required-claims>
        </validate-jwt>

        <!-- Add authenticated user to header for backend -->
        <set-header name="X-User-Id" exists-action="override">
            <value>@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims.GetValueOrDefault("sub", ""))</value>
        </set-header>

        <!-- CORS -->
        <cors allow-credentials="true">
            <allowed-origins>
                <origin>https://myapp.com</origin>
            </allowed-origins>
            <allowed-methods>
                <method>GET</method>
                <method>POST</method>
                <method>PUT</method>
                <method>DELETE</method>
            </allowed-methods>
            <allowed-headers>
                <header>*</header>
            </allowed-headers>
        </cors>

        <!-- Request logging -->
        <log-to-eventhub logger-id="eventhub-logger">
        @{
            return new JObject(
                new JProperty("requestId", context.RequestId),
                new JProperty("method", context.Request.Method),
                new JProperty("url", context.Request.Url.ToString()),
                new JProperty("user", context.User?.Id ?? "anonymous")
            ).ToString();
        }
        </log-to-eventhub>
    </inbound>

    <outbound>
        <!-- Response caching -->
        <cache-store duration="300" />

        <!-- Compression -->
        <set-header name="Content-Encoding" exists-action="override">
            <value>gzip</value>
        </set-header>
    </outbound>
</policies>
```

---

## Gatekeeper Pattern

### Problem

Backend services are exposed to potential attacks and need protection.

### Solution

```
GATEKEEPER ARCHITECTURE:
────────────────────────

┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  Internet ──▶ [ GATEKEEPER ] ──▶ [ TRUSTED HOST ] ──▶ [ SERVICES ]   │
│               (Public facing)    (DMZ)              (Internal)       │
│                                                                        │
│  GATEKEEPER responsibilities:                                         │
│  • Validate all requests                                              │
│  • Sanitize inputs                                                    │
│  • Rate limit                                                         │
│  • Block malicious patterns                                          │
│  • NO business logic access                                          │
│                                                                        │
│  TRUSTED HOST responsibilities:                                       │
│  • Perform business operations                                        │
│  • Access sensitive data                                              │
│  • Only accepts requests from Gatekeeper                             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Azure Implementation

```bicep
// Network isolation for gatekeeper pattern
resource appGatewaySubnet 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = {
  name: 'AppGatewaySubnet'
  properties: {
    addressPrefix: '10.0.1.0/24'
  }
}

resource appServiceSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = {
  name: 'AppServiceSubnet'
  properties: {
    addressPrefix: '10.0.2.0/24'
    delegations: [
      {
        name: 'delegation'
        properties: {
          serviceName: 'Microsoft.Web/serverFarms'
        }
      }
    ]
  }
}

// App Gateway as Gatekeeper (public-facing)
resource appGateway 'Microsoft.Network/applicationGateways@2023-05-01' = {
  name: 'gatekeeper-appgw'
  properties: {
    webApplicationFirewallConfiguration: {
      enabled: true
      firewallMode: 'Prevention'
      ruleSetType: 'OWASP'
      ruleSetVersion: '3.2'
    }
    // ... gateway configuration
  }
}

// App Service as Trusted Host (internal only)
resource appService 'Microsoft.Web/sites@2022-09-01' = {
  name: 'trusted-host'
  properties: {
    virtualNetworkSubnetId: appServiceSubnet.id
    publicNetworkAccess: 'Disabled'  // No direct internet access
  }
}

// Private endpoint for App Service
resource privateEndpoint 'Microsoft.Network/privateEndpoints@2023-05-01' = {
  name: 'app-service-pe'
  properties: {
    subnet: {
      id: appServiceSubnet.id
    }
    privateLinkServiceConnections: [
      {
        name: 'app-service-connection'
        properties: {
          privateLinkServiceId: appService.id
          groupIds: ['sites']
        }
      }
    ]
  }
}
```

---

*Next: [Deployment Patterns](05-deployment-patterns.md)* | *Back to [Data Patterns](03-data-patterns.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
