# Azure API Management (APIM)

## What is Azure API Management?

Azure API Management (APIM) is a fully managed service for publishing, securing, transforming, maintaining, and monitoring APIs. It acts as a facade in front of your backend services, providing a consistent entry point for API consumers.

### APIM vs AWS API Gateway

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   APIM vs AWS API GATEWAY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS API GATEWAY                         AZURE APIM                         │
│  ───────────────                         ──────────                         │
│                                                                              │
│  Types:                                  Types:                             │
│  • REST API (regional)                  • All-in-one service                │
│  • HTTP API (simpler, cheaper)          • Different SKUs for scale          │
│  • WebSocket API                        • WebSocket supported               │
│                                                                              │
│  Components:                             Components:                        │
│  ┌─────────────────────┐                ┌─────────────────────────────────┐ │
│  │ API Gateway         │                │ API Gateway                     │ │
│  │ (APIs + Stages)     │                │ (APIs + Products + Subscriptions│ │
│  │                     │                │  + Policies)                    │ │
│  │ + Usage Plans       │                │                                 │ │
│  │ + API Keys          │                │ Developer Portal (built-in)    │ │
│  │                     │                │                                 │ │
│  │ + Developer Portal  │                │ Analytics & Insights            │ │
│  │   (separate: Portal)│                │                                 │ │
│  └─────────────────────┘                └─────────────────────────────────┘ │
│                                                                              │
│  Policy/Transform:                       Policy/Transform:                  │
│  • VTL templates                         • XML-based policies              │
│  • Limited logic                         • C# expressions                  │
│  • Request/Response only                 • Inbound/Backend/Outbound/Error  │
│                                                                              │
│  Key Differences:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │ Feature          │ AWS API Gateway      │ Azure APIM                    ││
│  ├──────────────────┼──────────────────────┼───────────────────────────────┤│
│  │ Dev Portal       │ Separate service     │ Built-in, customizable       ││
│  │ Versioning       │ Stages               │ Versions + Revisions         ││
│  │ Monetization     │ Marketplace          │ Products + Subscriptions     ││
│  │ Caching          │ Separate (Elasticache│ Built-in cache              ││
│  │ Networking       │ VPC Link             │ VNet Integration + PL       ││
│  │ Auth             │ Cognito/Lambda       │ OAuth, JWT, Entra ID        ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## APIM Tiers and Features

### Tier Comparison

| Feature | Consumption | Developer | Basic | Standard | Premium |
|---------|-------------|-----------|-------|----------|---------|
| **SLA** | 99.95% | None | 99.95% | 99.95% | 99.99% |
| **Scale Units** | Auto | 1 | 2 | 4 | 10+ |
| **Cache** | External only | 10 MB | 50 MB | 1 GB | 5 GB |
| **VNet** | No | Yes | No | No | Yes |
| **Multi-region** | No | No | No | No | Yes |
| **Self-hosted Gateway** | No | Yes | No | No | Yes |
| **Dev Portal** | No | Yes | Yes | Yes | Yes |
| **Entra ID Integration** | No | Yes | No | Yes | Yes |
| **Price (approx)** | Pay per call | ~$50/mo | ~$150/mo | ~$700/mo | ~$2,800/mo |

### Choosing the Right Tier

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        APIM TIER SELECTION                                   │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  START HERE:                                                                │
│  ───────────                                                                │
│                                                                             │
│  Is this for development/testing only?                                     │
│  └── Yes ──► DEVELOPER TIER                                                │
│              • Full features for testing                                    │
│              • No SLA (not for production)                                 │
│              • Cheapest full-featured option                               │
│                                                                             │
│  Is traffic unpredictable/spiky?                                           │
│  └── Yes ──► CONSUMPTION TIER                                              │
│              • Pay-per-call pricing                                        │
│              • Auto-scales to zero                                         │
│              • Limited features (no VNet, no dev portal)                  │
│                                                                             │
│  Need VNet integration or multi-region?                                    │
│  └── Yes ──► PREMIUM TIER                                                  │
│              • Only tier with VNet support                                 │
│              • Multi-region deployment                                     │
│              • Self-hosted gateway for hybrid                              │
│                                                                             │
│  Standard enterprise API gateway?                                          │
│  └── Yes ──► STANDARD TIER                                                 │
│              • Good balance of features/cost                               │
│              • Entra ID integration                                        │
│              • 99.95% SLA                                                  │
│                                                                             │
│  Budget-conscious, simple needs?                                           │
│  └── Yes ──► BASIC TIER                                                    │
│              • Entry-level production tier                                 │
│              • No Entra ID integration                                     │
│              • Good for internal APIs                                      │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## APIM Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        APIM ARCHITECTURE                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  EXTERNAL CONSUMERS                                                         │
│  ──────────────────                                                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  Mobile  │  │   Web    │  │  Partner │  │   IoT    │                   │
│  │   Apps   │  │   Apps   │  │   Apps   │  │ Devices  │                   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘                   │
│       │             │             │             │                          │
│       └─────────────┴──────┬──────┴─────────────┘                          │
│                            │                                                │
│                            ▼                                                │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                    AZURE API MANAGEMENT                               │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │                    GATEWAY (Runtime)                             │ │  │
│  │  │  • Request/Response processing                                   │ │  │
│  │  │  • Policy execution                                              │ │  │
│  │  │  • Authentication/Authorization                                  │ │  │
│  │  │  • Rate limiting & throttling                                    │ │  │
│  │  │  • Caching                                                       │ │  │
│  │  │  • Logging & analytics                                           │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                       │  │
│  │  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐            │  │
│  │  │   Products    │  │ Subscriptions │  │    Policies   │            │  │
│  │  │  (API groups) │  │  (API keys)   │  │ (XML config)  │            │  │
│  │  └───────────────┘  └───────────────┘  └───────────────┘            │  │
│  │                                                                       │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐ │  │
│  │  │              DEVELOPER PORTAL (Optional)                         │ │  │
│  │  │  • API documentation                                             │ │  │
│  │  │  • Interactive testing                                           │ │  │
│  │  │  • Subscription management                                       │ │  │
│  │  │  • Customizable branding                                        │ │  │
│  │  └─────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                       │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                            │                                                │
│       ┌────────────────────┼────────────────────┐                          │
│       │                    │                    │                          │
│       ▼                    ▼                    ▼                          │
│  ┌──────────┐        ┌──────────┐        ┌──────────┐                     │
│  │   AKS    │        │ Functions│        │ App Svc  │                     │
│  │ Backend  │        │ Backend  │        │ Backend  │                     │
│  └──────────┘        └──────────┘        └──────────┘                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Products, APIs, and Subscriptions

### Conceptual Model

```
APIM RESOURCE HIERARCHY:
────────────────────────

APIM Instance
├── Products (Grouping & Access Control)
│   ├── "Free Tier"
│   │   ├── APIs: Weather API
│   │   └── Policies: 100 calls/hour
│   │
│   ├── "Standard"
│   │   ├── APIs: Weather API, Forecast API
│   │   └── Policies: 1000 calls/hour
│   │
│   └── "Premium"
│       ├── APIs: All APIs
│       └── Policies: Unlimited
│
├── APIs (Logical Grouping of Operations)
│   ├── Weather API (v1)
│   │   ├── GET /weather/current
│   │   ├── GET /weather/forecast
│   │   └── POST /weather/alerts
│   │
│   └── Orders API (v1)
│       ├── GET /orders
│       ├── POST /orders
│       └── GET /orders/{id}
│
├── Subscriptions (Access Credentials)
│   ├── "Contoso-Dev" → Free Tier
│   ├── "Contoso-Prod" → Premium
│   └── "Partner-ABC" → Standard
│
└── Users (Developer Portal)
    ├── developer@contoso.com
    └── partner@abc.com
```

### Versions and Revisions

```
┌────────────────────────────────────────────────────────────────────────────┐
│              VERSIONS vs REVISIONS                                          │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  VERSIONS = Breaking Changes (Different URLs)                              │
│  ────────────────────────────────────────────                              │
│                                                                             │
│  /api/v1/orders  ──────►  Version 1 (Original)                            │
│  /api/v2/orders  ──────►  Version 2 (New response format)                 │
│                                                                             │
│  • Different endpoints                                                      │
│  • Can run in parallel                                                      │
│  • Consumers choose which version                                          │
│  • Version in: URL path, query string, or header                          │
│                                                                             │
│  REVISIONS = Non-Breaking Changes (Same URL, internal)                     │
│  ─────────────────────────────────────────────────────                     │
│                                                                             │
│  /api/v1/orders                                                            │
│       │                                                                     │
│       ├── Revision 1 (Initial)                                             │
│       ├── Revision 2 (Added logging) ← Testing                            │
│       └── Revision 3 (Bug fix) ← Current                                   │
│                                                                             │
│  • Same endpoint                                                            │
│  • One active revision at a time                                           │
│  • Test revisions with special header                                      │
│  • Roll back by switching active revision                                  │
│                                                                             │
│  AWS Equivalent:                                                            │
│  • Versions ≈ API Gateway Stages                                           │
│  • Revisions ≈ API Gateway Deployments                                     │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## Policies and Transformations

### Policy Structure

```xml
<policies>
    <inbound>
        <!-- Executed BEFORE request goes to backend -->
        <!-- Authentication, rate limiting, request transformation -->
    </inbound>

    <backend>
        <!-- Executed when calling backend service -->
        <!-- Routing, load balancing -->
    </backend>

    <outbound>
        <!-- Executed AFTER response from backend -->
        <!-- Response transformation, header manipulation -->
    </outbound>

    <on-error>
        <!-- Executed when error occurs at any stage -->
        <!-- Custom error responses, logging -->
    </on-error>
</policies>
```

### Policy Execution Flow

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    POLICY EXECUTION FLOW                                     │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Client Request                                                            │
│       │                                                                     │
│       ▼                                                                     │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    INBOUND POLICIES                                  │   │
│  │                                                                      │   │
│  │  1. Global policies (all APIs)                                      │   │
│  │  2. Product policies (APIs in product)                              │   │
│  │  3. API policies (all operations in API)                            │   │
│  │  4. Operation policies (specific endpoint)                          │   │
│  │                                                                      │   │
│  │  <base/> = inherit from parent level                                │   │
│  └──────────────────────────┬──────────────────────────────────────────┘   │
│                             │                                               │
│                             ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    BACKEND POLICIES                                  │   │
│  │  (Request forwarded to backend service)                             │   │
│  └──────────────────────────┬──────────────────────────────────────────┘   │
│                             │                                               │
│                             ▼                                               │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    OUTBOUND POLICIES                                 │   │
│  │                                                                      │   │
│  │  4. Operation policies                                              │   │
│  │  3. API policies                                                    │   │
│  │  2. Product policies                                                │   │
│  │  1. Global policies                                                 │   │
│  └──────────────────────────┬──────────────────────────────────────────┘   │
│                             │                                               │
│                             ▼                                               │
│  Client Response                                                           │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Common Policy Examples

#### Rate Limiting and Throttling

```xml
<policies>
    <inbound>
        <!-- Rate limit by subscription key: 100 calls per minute -->
        <rate-limit-by-key calls="100"
                          renewal-period="60"
                          counter-key="@(context.Subscription.Key)"
                          increment-condition="@(context.Response.StatusCode >= 200 && context.Response.StatusCode < 400)" />

        <!-- Quota: 10,000 calls per day per subscription -->
        <quota-by-key calls="10000"
                      bandwidth="40000"
                      renewal-period="86400"
                      counter-key="@(context.Subscription.Key)" />

        <!-- Rate limit by IP address (for unauthenticated APIs) -->
        <rate-limit-by-key calls="20"
                          renewal-period="60"
                          counter-key="@(context.Request.IpAddress)" />
    </inbound>
</policies>
```

#### Request Transformation

```xml
<policies>
    <inbound>
        <!-- Add correlation ID header -->
        <set-header name="X-Correlation-ID" exists-action="skip">
            <value>@(Guid.NewGuid().ToString())</value>
        </set-header>

        <!-- Add timestamp to request body -->
        <set-body>@{
            var body = context.Request.Body.As<JObject>(preserveContent: true);
            body["requestTimestamp"] = DateTime.UtcNow.ToString("o");
            body["clientIp"] = context.Request.IpAddress;
            return body.ToString();
        }</set-body>

        <!-- Rewrite URL path -->
        <rewrite-uri template="/api/internal/v2/orders" />

        <!-- Set query parameter -->
        <set-query-parameter name="api-version" exists-action="override">
            <value>2024-01-01</value>
        </set-query-parameter>

        <!-- Remove sensitive headers before forwarding -->
        <set-header name="X-Internal-Secret" exists-action="delete" />
    </inbound>

    <outbound>
        <!-- Remove server info headers -->
        <set-header name="X-Powered-By" exists-action="delete" />
        <set-header name="X-AspNet-Version" exists-action="delete" />
        <set-header name="Server" exists-action="delete" />

        <!-- Add custom response headers -->
        <set-header name="X-Request-ID" exists-action="override">
            <value>@(context.RequestId.ToString())</value>
        </set-header>

        <!-- Transform XML response to JSON -->
        <xml-to-json kind="javascript-friendly" apply="always" />
    </outbound>
</policies>
```

#### Caching

```xml
<policies>
    <inbound>
        <!-- Cache lookup - check if response is cached -->
        <cache-lookup vary-by-developer="false"
                      vary-by-developer-groups="false"
                      downstream-caching-type="none"
                      must-revalidate="true">
            <vary-by-header>Accept</vary-by-header>
            <vary-by-header>Accept-Charset</vary-by-header>
            <vary-by-query-parameter>version</vary-by-query-parameter>
        </cache-lookup>
    </inbound>

    <outbound>
        <!-- Cache successful responses for 1 hour -->
        <cache-store duration="3600" />

        <!-- Or use conditional caching -->
        <choose>
            <when condition="@(context.Response.StatusCode == 200)">
                <cache-store duration="3600" />
            </when>
        </choose>
    </outbound>
</policies>
```

#### Backend Service Selection

```xml
<policies>
    <backend>
        <!-- Route to different backends based on header -->
        <choose>
            <when condition="@(context.Request.Headers.GetValueOrDefault("X-Region","") == "EU")">
                <set-backend-service base-url="https://api-eu.contoso.com" />
            </when>
            <when condition="@(context.Request.Headers.GetValueOrDefault("X-Region","") == "US")">
                <set-backend-service base-url="https://api-us.contoso.com" />
            </when>
            <otherwise>
                <set-backend-service base-url="https://api.contoso.com" />
            </otherwise>
        </choose>
    </backend>
</policies>
```

#### Mock Response

```xml
<policies>
    <inbound>
        <!-- Return mock response without calling backend -->
        <mock-response status-code="200" content-type="application/json" />
    </inbound>
</policies>
```

---

## Security Patterns

### OAuth 2.0 / JWT Validation

```xml
<policies>
    <inbound>
        <!-- Validate JWT token from Entra ID -->
        <validate-jwt header-name="Authorization"
                      require-scheme="Bearer"
                      failed-validation-httpcode="401"
                      failed-validation-error-message="Unauthorized. Access token is missing or invalid.">

            <!-- OpenID Connect configuration endpoint -->
            <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />

            <!-- Required audiences -->
            <audiences>
                <audience>api://my-api-client-id</audience>
            </audiences>

            <!-- Valid issuers -->
            <issuers>
                <issuer>https://login.microsoftonline.com/{tenant-id}/v2.0</issuer>
                <issuer>https://sts.windows.net/{tenant-id}/</issuer>
            </issuers>

            <!-- Required claims -->
            <required-claims>
                <claim name="roles" match="any">
                    <value>API.Read</value>
                    <value>API.Write</value>
                </claim>
            </required-claims>
        </validate-jwt>

        <!-- Extract user info from validated token -->
        <set-header name="X-User-ID" exists-action="override">
            <value>@(context.Request.Headers.GetValueOrDefault("Authorization","").AsJwt()?.Claims.GetValueOrDefault("oid", ""))</value>
        </set-header>
    </inbound>
</policies>
```

### API Key / Subscription Key Validation

```xml
<policies>
    <inbound>
        <!-- Subscription key is validated automatically when required -->
        <!-- These policies add custom key validation -->

        <!-- Validate custom API key from header -->
        <check-header name="X-API-Key" failed-check-httpcode="401" failed-check-error-message="API Key required">
            <value>secret-key-1</value>
            <value>secret-key-2</value>
        </check-header>

        <!-- Or validate from Key Vault -->
        <set-variable name="validApiKey" value="@{
            var namedValue = context.Variables.GetValueOrDefault<string>("api-key-from-keyvault");
            return namedValue;
        }" />

        <choose>
            <when condition="@(context.Request.Headers.GetValueOrDefault("X-API-Key","") != (string)context.Variables["validApiKey"])">
                <return-response>
                    <set-status code="401" reason="Unauthorized" />
                    <set-body>{"error": "Invalid API key"}</set-body>
                </return-response>
            </when>
        </choose>
    </inbound>
</policies>
```

### IP Filtering

```xml
<policies>
    <inbound>
        <!-- Allow only specific IP addresses -->
        <ip-filter action="allow">
            <address-range from="10.0.0.0" to="10.0.0.255" />
            <address>203.0.113.42</address>
        </ip-filter>

        <!-- Or deny specific IPs -->
        <ip-filter action="forbid">
            <address-range from="192.168.1.0" to="192.168.1.255" />
        </ip-filter>
    </inbound>
</policies>
```

### Certificate Validation

```xml
<policies>
    <inbound>
        <!-- Validate client certificate -->
        <choose>
            <when condition="@(context.Request.Certificate == null ||
                              context.Request.Certificate.Thumbprint != "expected-thumbprint")">
                <return-response>
                    <set-status code="403" reason="Forbidden" />
                    <set-body>{"error": "Invalid client certificate"}</set-body>
                </return-response>
            </when>
        </choose>

        <!-- Validate certificate against CA -->
        <validate-client-certificate
            validate-revocation="true"
            validate-trust="true"
            validate-not-before="true"
            validate-not-after="true" />
    </inbound>
</policies>
```

### CORS Configuration

```xml
<policies>
    <inbound>
        <cors allow-credentials="true">
            <allowed-origins>
                <origin>https://app.contoso.com</origin>
                <origin>https://portal.contoso.com</origin>
            </allowed-origins>
            <allowed-methods preflight-result-max-age="300">
                <method>GET</method>
                <method>POST</method>
                <method>PUT</method>
                <method>DELETE</method>
                <method>OPTIONS</method>
            </allowed-methods>
            <allowed-headers>
                <header>Authorization</header>
                <header>Content-Type</header>
                <header>X-Requested-With</header>
            </allowed-headers>
            <expose-headers>
                <header>X-Request-ID</header>
            </expose-headers>
        </cors>
    </inbound>
</policies>
```

---

## Developer Portal

### Portal Architecture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                    DEVELOPER PORTAL                                         │
├────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  BUILT-IN FEATURES:                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │   │
│  │  │   API Docs   │  │   Try It     │  │  Subscribe   │             │   │
│  │  │              │  │   Console    │  │  to APIs     │             │   │
│  │  │ • OpenAPI    │  │              │  │              │             │   │
│  │  │ • Operations │  │ • Send req.  │  │ • Get keys   │             │   │
│  │  │ • Schemas    │  │ • View resp. │  │ • Manage     │             │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘             │   │
│  │                                                                      │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐             │   │
│  │  │   User       │  │  Analytics   │  │   Custom     │             │   │
│  │  │   Profile    │  │              │  │   Pages      │             │   │
│  │  │              │  │              │  │              │             │   │
│  │  │ • Sign up    │  │ • Usage      │  │ • Markdown   │             │   │
│  │  │ • Sign in    │  │ • Reports    │  │ • Widgets    │             │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘             │   │
│  │                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  CUSTOMIZATION OPTIONS:                                                    │
│  • Custom domain (portal.yourcompany.com)                                 │
│  • Custom CSS/styling                                                      │
│  • Logo and branding                                                       │
│  • Custom pages and navigation                                            │
│  • OAuth/Social sign-in (Google, Facebook, Entra ID)                     │
│  • Delegation (custom sign-up/sign-in)                                    │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Portal Setup CLI

```bash
# Enable developer portal
az apim update \
  --resource-group myRG \
  --name my-apim \
  --enable-client-certificate false

# Publish the developer portal (makes it accessible)
# Note: This is typically done via Azure Portal or REST API

# Set custom domain for portal
az apim custom-domain create \
  --resource-group myRG \
  --service-name my-apim \
  --domain-name portal.contoso.com \
  --certificate-path ./portal-cert.pfx \
  --certificate-password "secure-password"
```

---

## Bicep/CLI Deployment Examples

### Complete APIM Deployment (Bicep)

```bicep
// main.bicep - Complete APIM deployment

@description('The name of the API Management instance')
param apimName string = 'apim-${uniqueString(resourceGroup().id)}'

@description('Location for all resources')
param location string = resourceGroup().location

@description('The pricing tier of the API Management instance')
@allowed(['Consumption', 'Developer', 'Basic', 'Standard', 'Premium'])
param sku string = 'Developer'

@description('Publisher email address')
param publisherEmail string

@description('Publisher organization name')
param publisherName string

@description('Application Insights resource ID')
param appInsightsId string = ''

@description('Application Insights instrumentation key')
param appInsightsKey string = ''

// API Management instance
resource apim 'Microsoft.ApiManagement/service@2023-05-01-preview' = {
  name: apimName
  location: location
  sku: {
    name: sku
    capacity: sku == 'Consumption' ? 0 : 1
  }
  properties: {
    publisherEmail: publisherEmail
    publisherName: publisherName
    customProperties: {
      'Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA': 'false'
      'Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Ciphers.TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA': 'false'
      'Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls10': 'false'
      'Microsoft.WindowsAzure.ApiManagement.Gateway.Security.Protocols.Tls11': 'false'
    }
  }

  // Named Values (configuration)
  resource backendUrl 'namedValues' = {
    name: 'backend-url'
    properties: {
      displayName: 'Backend URL'
      value: 'https://api-backend.azurewebsites.net'
      secret: false
    }
  }

  // Logger for Application Insights
  resource logger 'loggers' = if (!empty(appInsightsId)) {
    name: 'app-insights-logger'
    properties: {
      loggerType: 'applicationInsights'
      credentials: {
        instrumentationKey: appInsightsKey
      }
      resourceId: appInsightsId
    }
  }
}

// Products
resource freeProduct 'Microsoft.ApiManagement/service/products@2023-05-01-preview' = {
  parent: apim
  name: 'free-tier'
  properties: {
    displayName: 'Free Tier'
    description: 'Free tier with limited access'
    subscriptionRequired: true
    approvalRequired: false
    state: 'published'
    subscriptionsLimit: 1
  }
}

resource premiumProduct 'Microsoft.ApiManagement/service/products@2023-05-01-preview' = {
  parent: apim
  name: 'premium-tier'
  properties: {
    displayName: 'Premium Tier'
    description: 'Premium tier with full access'
    subscriptionRequired: true
    approvalRequired: true
    state: 'published'
  }
}

// API Definition
resource ordersApi 'Microsoft.ApiManagement/service/apis@2023-05-01-preview' = {
  parent: apim
  name: 'orders-api'
  properties: {
    displayName: 'Orders API'
    description: 'API for managing orders'
    path: 'orders'
    protocols: ['https']
    subscriptionRequired: true
    serviceUrl: 'https://api-backend.azurewebsites.net/api'
    apiVersion: 'v1'
    apiVersionSetId: ordersVersionSet.id
  }
}

// API Version Set
resource ordersVersionSet 'Microsoft.ApiManagement/service/apiVersionSets@2023-05-01-preview' = {
  parent: apim
  name: 'orders-version-set'
  properties: {
    displayName: 'Orders API'
    versioningScheme: 'Segment'
  }
}

// API Operations
resource getOrdersOperation 'Microsoft.ApiManagement/service/apis/operations@2023-05-01-preview' = {
  parent: ordersApi
  name: 'get-orders'
  properties: {
    displayName: 'Get Orders'
    method: 'GET'
    urlTemplate: '/'
    description: 'Retrieve all orders'
    responses: [
      {
        statusCode: 200
        description: 'Success'
        representations: [
          {
            contentType: 'application/json'
          }
        ]
      }
    ]
  }
}

resource getOrderByIdOperation 'Microsoft.ApiManagement/service/apis/operations@2023-05-01-preview' = {
  parent: ordersApi
  name: 'get-order-by-id'
  properties: {
    displayName: 'Get Order by ID'
    method: 'GET'
    urlTemplate: '/{orderId}'
    description: 'Retrieve a specific order'
    templateParameters: [
      {
        name: 'orderId'
        type: 'string'
        required: true
        description: 'The order ID'
      }
    ]
    responses: [
      {
        statusCode: 200
        description: 'Success'
      }
      {
        statusCode: 404
        description: 'Order not found'
      }
    ]
  }
}

resource createOrderOperation 'Microsoft.ApiManagement/service/apis/operations@2023-05-01-preview' = {
  parent: ordersApi
  name: 'create-order'
  properties: {
    displayName: 'Create Order'
    method: 'POST'
    urlTemplate: '/'
    description: 'Create a new order'
    request: {
      representations: [
        {
          contentType: 'application/json'
        }
      ]
    }
    responses: [
      {
        statusCode: 201
        description: 'Order created'
      }
      {
        statusCode: 400
        description: 'Bad request'
      }
    ]
  }
}

// API Policy
resource ordersApiPolicy 'Microsoft.ApiManagement/service/apis/policies@2023-05-01-preview' = {
  parent: ordersApi
  name: 'policy'
  properties: {
    format: 'xml'
    value: '''
<policies>
    <inbound>
        <base />
        <set-header name="X-Request-ID" exists-action="skip">
            <value>@(Guid.NewGuid().ToString())</value>
        </set-header>
        <rate-limit-by-key calls="100" renewal-period="60" counter-key="@(context.Subscription.Key)" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
        <set-header name="X-Powered-By" exists-action="delete" />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
'''
  }
}

// Product-API Association
resource freeProductApi 'Microsoft.ApiManagement/service/products/apis@2023-05-01-preview' = {
  parent: freeProduct
  name: ordersApi.name
}

resource premiumProductApi 'Microsoft.ApiManagement/service/products/apis@2023-05-01-preview' = {
  parent: premiumProduct
  name: ordersApi.name
}

// Product Policies
resource freeProductPolicy 'Microsoft.ApiManagement/service/products/policies@2023-05-01-preview' = {
  parent: freeProduct
  name: 'policy'
  properties: {
    format: 'xml'
    value: '''
<policies>
    <inbound>
        <base />
        <rate-limit calls="100" renewal-period="3600" />
        <quota calls="1000" renewal-period="86400" />
    </inbound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
'''
  }
}

// Outputs
output apimName string = apim.name
output apimGatewayUrl string = apim.properties.gatewayUrl
output apimPortalUrl string = apim.properties.developerPortalUrl
output apimManagementUrl string = apim.properties.managementApiUrl
```

### Azure CLI Commands

```bash
# Create APIM instance
az apim create \
  --resource-group myRG \
  --name my-apim \
  --location eastus \
  --publisher-email admin@contoso.com \
  --publisher-name "Contoso" \
  --sku-name Developer \
  --enable-client-certificate false

# Import API from OpenAPI specification
az apim api import \
  --resource-group myRG \
  --service-name my-apim \
  --api-id orders-api \
  --path orders \
  --display-name "Orders API" \
  --specification-format OpenApiJson \
  --specification-url "https://raw.githubusercontent.com/contoso/api-specs/main/orders-api.json" \
  --service-url "https://orders-backend.azurewebsites.net/api"

# Create product
az apim product create \
  --resource-group myRG \
  --service-name my-apim \
  --product-id premium \
  --display-name "Premium API Access" \
  --description "Full access to all APIs" \
  --subscription-required true \
  --approval-required true \
  --state published

# Add API to product
az apim product api add \
  --resource-group myRG \
  --service-name my-apim \
  --product-id premium \
  --api-id orders-api

# Create subscription
az apim subscription create \
  --resource-group myRG \
  --service-name my-apim \
  --subscription-id contoso-prod \
  --display-name "Contoso Production" \
  --scope "/products/premium" \
  --state active

# Get subscription keys
az apim subscription list \
  --resource-group myRG \
  --service-name my-apim \
  --query "[].{Name:displayName, PrimaryKey:primaryKey, SecondaryKey:secondaryKey}" \
  --output table

# Apply policy to API
az apim api policy create \
  --resource-group myRG \
  --service-name my-apim \
  --api-id orders-api \
  --policy-file ./api-policy.xml

# Set named value (configuration)
az apim nv create \
  --resource-group myRG \
  --service-name my-apim \
  --named-value-id backend-url \
  --display-name "Backend URL" \
  --value "https://orders-backend.azurewebsites.net"

# Set secret named value (from Key Vault)
az apim nv create \
  --resource-group myRG \
  --service-name my-apim \
  --named-value-id db-connection-string \
  --display-name "Database Connection" \
  --secret true \
  --value "Server=..." # Or use Key Vault reference

# Enable Application Insights logging
az apim api diagnostic create \
  --resource-group myRG \
  --service-name my-apim \
  --api-id orders-api \
  --diagnostic-id applicationinsights \
  --logger-id app-insights-logger \
  --always-log allErrors \
  --log-client-ip true
```

---

## Best Practices

```
APIM BEST PRACTICES:
────────────────────

1. SECURITY
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Always require subscriptions for production APIs                   │
   │ • Use OAuth 2.0 / JWT validation for user context                   │
   │ • Enable HTTPS only (disable HTTP)                                   │
   │ • Use managed identities for backend authentication                 │
   │ • Store secrets in Key Vault, reference via named values           │
   │ • Implement IP filtering for sensitive APIs                         │
   │ • Remove server/version headers in outbound policies               │
   └─────────────────────────────────────────────────────────────────────┘

2. PERFORMANCE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Enable caching for read-heavy endpoints                           │
   │ • Use Premium tier for VNet integration (reduces latency)          │
   │ • Implement rate limiting to protect backends                       │
   │ • Use circuit breaker pattern for unreliable backends              │
   │ • Minimize policy complexity (C# expressions cost CPU)             │
   │ • Use async policies when possible                                  │
   └─────────────────────────────────────────────────────────────────────┘

3. OPERATIONS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Use revisions for testing changes before production               │
   │ • Implement proper versioning strategy from day one                │
   │ • Enable diagnostic logging to Application Insights                 │
   │ • Set up alerts for error rates and latency                        │
   │ • Use products to group APIs and manage access                     │
   │ • Automate deployments with Bicep/ARM templates                    │
   └─────────────────────────────────────────────────────────────────────┘

4. DEVELOPER EXPERIENCE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Publish comprehensive API documentation                           │
   │ • Provide example requests/responses                                │
   │ • Customize developer portal with branding                         │
   │ • Enable self-service subscription for free tiers                  │
   │ • Include error response schemas                                    │
   │ • Offer SDKs in popular languages                                  │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Service Bus](02-service-bus.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
