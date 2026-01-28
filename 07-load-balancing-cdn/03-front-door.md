# Azure Front Door Deep Dive

## Overview

Azure Front Door is a global, scalable entry-point that uses Microsoft's global edge network to create fast, secure, and highly available web applications. For AWS architects, it combines capabilities of CloudFront and Global Accelerator.

## AWS Comparison

| Feature | AWS CloudFront + Global Accelerator | Azure Front Door |
|---------|-------------------------------------|------------------|
| Global anycast | Yes (GA) | Yes |
| CDN | Yes (CloudFront) | Yes (built-in) |
| WAF | AWS WAF (separate) | Built-in |
| SSL termination | Yes | Yes |
| Path-based routing | Yes | Yes |
| Health probes | Yes | Yes |
| Private origins | Yes (OAI/OAC) | Yes (Premium + Private Link) |
| Real-time analytics | CloudWatch | Built-in reports |

## Front Door SKUs

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     FRONT DOOR SKU COMPARISON                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────┬───────────────────┬─────────────────────┐  │
│  │ Feature                     │ Standard          │ Premium             │  │
│  ├─────────────────────────────┼───────────────────┼─────────────────────┤  │
│  │ Global load balancing       │ Yes               │ Yes                 │  │
│  │ SSL offload                 │ Yes               │ Yes                 │  │
│  │ Custom domains              │ Yes               │ Yes                 │  │
│  │ Compression                 │ Yes               │ Yes                 │  │
│  │ URL redirect/rewrite        │ Yes               │ Yes                 │  │
│  │ Caching                     │ Yes               │ Yes                 │  │
│  │ WAF                         │ Standard rules    │ Custom + managed    │  │
│  │ Bot protection              │ No                │ Yes                 │  │
│  │ Private Link origins        │ No                │ Yes                 │  │
│  │ Advanced analytics          │ Basic             │ Enhanced reports    │  │
│  │ Price (base)                │ ~$35/month        │ ~$330/month         │  │
│  └─────────────────────────────┴───────────────────┴─────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     AZURE FRONT DOOR ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                            GLOBAL USERS                                      │
│                                 │                                            │
│        ┌────────────────────────┼────────────────────────┐                  │
│        │                        │                        │                  │
│        ▼                        ▼                        ▼                  │
│   ┌─────────┐              ┌─────────┐              ┌─────────┐            │
│   │ Edge    │              │ Edge    │              │ Edge    │            │
│   │ POP     │              │ POP     │              │ POP     │            │
│   │ (NYC)   │              │ (London)│              │ (Tokyo) │            │
│   └────┬────┘              └────┬────┘              └────┬────┘            │
│        │                        │                        │                  │
│        │         Microsoft Global Network                │                  │
│        │    (Anycast, <2ms between POPs)                │                  │
│        │                        │                        │                  │
│        └────────────────────────┼────────────────────────┘                  │
│                                 │                                            │
│                    ┌────────────┴────────────┐                              │
│                    │      FRONT DOOR         │                              │
│                    │                         │                              │
│                    │  • WAF (DDoS, Rules)    │                              │
│                    │  • SSL termination      │                              │
│                    │  • Routing rules        │                              │
│                    │  • Caching              │                              │
│                    │  • Health monitoring    │                              │
│                    └────────────┬────────────┘                              │
│                                 │                                            │
│         ┌───────────────────────┼───────────────────────┐                   │
│         │                       │                       │                   │
│         ▼                       ▼                       ▼                   │
│  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐           │
│  │ Origin      │         │ Origin      │         │ Origin      │           │
│  │ Group 1     │         │ Group 2     │         │ Group 3     │           │
│  │ (East US)   │         │ (West EU)   │         │ (SE Asia)   │           │
│  │             │         │             │         │             │           │
│  │ App Gateway │         │ App Gateway │         │ App Service │           │
│  │ or AKS      │         │ or AKS      │         │             │           │
│  └─────────────┘         └─────────────┘         └─────────────┘           │
│                                                                              │
│  ROUTING OPTIONS:                                                           │
│  • Latency-based (fastest origin)                                          │
│  • Priority-based (primary/secondary)                                       │
│  • Weighted (traffic splitting)                                            │
│  • Session affinity                                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Creating Front Door

### Azure CLI

```bash
# Create Front Door profile
az afd profile create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --sku Premium_AzureFrontDoor

# Create endpoint
az afd endpoint create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name myapp \
  --enabled-state Enabled

# Create origin group with health probes
az afd origin-group create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name webOrigins \
  --probe-request-type HEAD \
  --probe-protocol Https \
  --probe-interval-in-seconds 30 \
  --sample-size 4 \
  --successful-samples-required 3 \
  --additional-latency-in-milliseconds 50

# Add origins
az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name webOrigins \
  --origin-name eastus \
  --host-name myapp-eastus.azurewebsites.net \
  --origin-host-header myapp-eastus.azurewebsites.net \
  --http-port 80 \
  --https-port 443 \
  --priority 1 \
  --weight 1000 \
  --enabled-state Enabled

az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name webOrigins \
  --origin-name westeurope \
  --host-name myapp-westeu.azurewebsites.net \
  --origin-host-header myapp-westeu.azurewebsites.net \
  --http-port 80 \
  --https-port 443 \
  --priority 1 \
  --weight 1000 \
  --enabled-state Enabled

# Create route
az afd route create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name myapp \
  --route-name defaultRoute \
  --origin-group webOrigins \
  --supported-protocols Https Http \
  --patterns-to-match "/*" \
  --forwarding-protocol HttpsOnly \
  --https-redirect Enabled
```

### Private Link Origin (Premium SKU)

```bash
# Create Private Link origin
az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name webOrigins \
  --origin-name privateOrigin \
  --host-name myapp.azurewebsites.net \
  --origin-host-header myapp.azurewebsites.net \
  --enable-private-link true \
  --private-link-resource /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Web/sites/myapp \
  --private-link-location eastus \
  --private-link-request-message "Front Door Private Link"
```

## Custom Domains and SSL

```bash
# Add custom domain
az afd custom-domain create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --custom-domain-name www-contoso \
  --host-name www.contoso.com \
  --certificate-type ManagedCertificate

# Associate with route
az afd route update \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name myapp \
  --route-name defaultRoute \
  --custom-domains www-contoso
```

### BYOC (Bring Your Own Certificate)

```bash
# Create Key Vault and certificate first, then:
az afd secret create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --secret-name mySecret \
  --use-latest-version true \
  --secret-source /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{vault}/secrets/{cert}

az afd custom-domain create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --custom-domain-name apex-contoso \
  --host-name contoso.com \
  --certificate-type CustomerCertificate \
  --secret mySecret
```

## WAF Configuration

```bash
# Create WAF policy
az afd waf-policy create \
  --resource-group myRG \
  --policy-name myWafPolicy \
  --sku Premium_AzureFrontDoor \
  --mode Prevention

# Add managed rule set
az afd waf-policy managed-rule-set add \
  --resource-group myRG \
  --policy-name myWafPolicy \
  --type Microsoft_DefaultRuleSet \
  --version 2.1

# Add bot protection
az afd waf-policy managed-rule-set add \
  --resource-group myRG \
  --policy-name myWafPolicy \
  --type Microsoft_BotManagerRuleSet \
  --version 1.0

# Create custom rule (rate limiting)
az afd waf-policy rule create \
  --resource-group myRG \
  --policy-name myWafPolicy \
  --rule-name rateLimitRule \
  --priority 100 \
  --action Block \
  --rule-type RateLimitRule \
  --rate-limit-threshold 1000 \
  --rate-limit-duration-in-minutes 1

# Associate with security policy
az afd security-policy create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --security-policy-name mySecPolicy \
  --domains /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Cdn/profiles/myFrontDoor/afdEndpoints/myapp \
  --waf-policy /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Network/FrontDoorWebApplicationFirewallPolicies/myWafPolicy
```

## Routing Rules

### Path-Based Routing

```bash
# Create additional origin groups
az afd origin-group create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name apiOrigins \
  --probe-request-type GET \
  --probe-path /health

az afd origin-group create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name staticOrigins \
  --probe-request-type HEAD

# Create routes for different paths
az afd route create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name myapp \
  --route-name apiRoute \
  --origin-group apiOrigins \
  --patterns-to-match "/api/*" \
  --supported-protocols Https \
  --forwarding-protocol HttpsOnly

az afd route create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name myapp \
  --route-name staticRoute \
  --origin-group staticOrigins \
  --patterns-to-match "/static/*" "/images/*" \
  --supported-protocols Https Http \
  --forwarding-protocol MatchRequest \
  --enable-caching true \
  --query-string-caching-behavior IgnoreQueryString
```

### Rule Sets (URL Rewrite/Redirect)

```bash
# Create rule set
az afd rule-set create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --rule-set-name redirectRules

# Create redirect rule (HTTP to HTTPS)
az afd rule create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --rule-set-name redirectRules \
  --rule-name httpsRedirect \
  --order 1 \
  --match-variable RequestScheme \
  --operator Equal \
  --match-values HTTP \
  --action-name UrlRedirect \
  --redirect-protocol Https \
  --redirect-type Moved

# Create URL rewrite rule
az afd rule create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --rule-set-name redirectRules \
  --rule-name apiRewrite \
  --order 2 \
  --match-variable UrlPath \
  --operator BeginsWith \
  --match-values "/v1/api" \
  --action-name UrlRewrite \
  --source-pattern "/v1/api" \
  --destination "/api"
```

## Caching Configuration

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        FRONT DOOR CACHING                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  CACHE BEHAVIOR OPTIONS:                                                    │
│                                                                              │
│  ┌─────────────────────────────┬────────────────────────────────────────┐   │
│  │ Query String Setting        │ Description                            │   │
│  ├─────────────────────────────┼────────────────────────────────────────┤   │
│  │ IgnoreQueryString           │ Cache ignores query strings            │   │
│  │ UseQueryString              │ Each unique query string cached        │   │
│  │ IgnoreSpecifiedQueryStrings │ Ignore specified params, cache rest   │   │
│  │ IncludeSpecifiedQueryStrings│ Only use specified params for key     │   │
│  └─────────────────────────────┴────────────────────────────────────────┘   │
│                                                                              │
│  CACHE DURATION:                                                            │
│  • Respect origin headers (Cache-Control, Expires)                          │
│  • Override with Front Door rules                                           │
│  • Default: Use origin headers                                              │
│                                                                              │
│  CACHE PURGE:                                                               │
│  az afd endpoint purge \                                                    │
│    --resource-group myRG \                                                  │
│    --profile-name myFrontDoor \                                             │
│    --endpoint-name myapp \                                                  │
│    --content-paths "/*"                                                     │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Bicep Template

```bicep
@description('Location for resources')
param location string = resourceGroup().location

@description('Front Door profile name')
param profileName string

@description('Origin hostnames')
param originHostnames array = [
  'myapp-eastus.azurewebsites.net'
  'myapp-westeu.azurewebsites.net'
]

// Front Door Profile
resource frontDoorProfile 'Microsoft.Cdn/profiles@2023-05-01' = {
  name: profileName
  location: 'global'
  sku: {
    name: 'Premium_AzureFrontDoor'
  }
}

// Endpoint
resource endpoint 'Microsoft.Cdn/profiles/afdEndpoints@2023-05-01' = {
  parent: frontDoorProfile
  name: 'myapp'
  location: 'global'
  properties: {
    enabledState: 'Enabled'
  }
}

// Origin Group
resource originGroup 'Microsoft.Cdn/profiles/originGroups@2023-05-01' = {
  parent: frontDoorProfile
  name: 'webOrigins'
  properties: {
    loadBalancingSettings: {
      sampleSize: 4
      successfulSamplesRequired: 3
      additionalLatencyInMilliseconds: 50
    }
    healthProbeSettings: {
      probePath: '/health'
      probeRequestType: 'HEAD'
      probeProtocol: 'Https'
      probeIntervalInSeconds: 30
    }
  }
}

// Origins
resource origins 'Microsoft.Cdn/profiles/originGroups/origins@2023-05-01' = [for (hostname, i) in originHostnames: {
  parent: originGroup
  name: 'origin${i}'
  properties: {
    hostName: hostname
    originHostHeader: hostname
    httpPort: 80
    httpsPort: 443
    priority: 1
    weight: 1000
    enabledState: 'Enabled'
  }
}]

// Route
resource route 'Microsoft.Cdn/profiles/afdEndpoints/routes@2023-05-01' = {
  parent: endpoint
  name: 'defaultRoute'
  properties: {
    originGroup: {
      id: originGroup.id
    }
    supportedProtocols: ['Http', 'Https']
    patternsToMatch: ['/*']
    forwardingProtocol: 'HttpsOnly'
    httpsRedirect: 'Enabled'
    cacheConfiguration: {
      queryStringCachingBehavior: 'UseQueryString'
      compressionSettings: {
        isCompressionEnabled: true
        contentTypesToCompress: [
          'text/html'
          'text/css'
          'application/javascript'
          'application/json'
        ]
      }
    }
  }
  dependsOn: origins
}

output frontDoorEndpoint string = endpoint.properties.hostName
output frontDoorId string = frontDoorProfile.id
```

## Monitoring and Diagnostics

```bash
# Enable diagnostics
az monitor diagnostic-settings create \
  --resource /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Cdn/profiles/myFrontDoor \
  --name fd-diagnostics \
  --logs '[{"category":"FrontDoorAccessLog","enabled":true},{"category":"FrontDoorHealthProbeLog","enabled":true},{"category":"FrontDoorWebApplicationFirewallLog","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]' \
  --workspace /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace
```

### Key Metrics

| Metric | Description |
|--------|-------------|
| RequestCount | Total requests |
| TotalLatency | End-to-end latency |
| OriginLatency | Time to fetch from origin |
| OriginHealthPercentage | Origin availability |
| BytesSent | Egress bytes |
| WebApplicationFirewallRequestCount | WAF processed requests |

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| 503 Service Unavailable | All origins unhealthy | Check origin health probes |
| High latency | Wrong routing | Check latency settings |
| Cache not working | Cache-Control headers | Verify origin headers or override |
| WAF blocking legitimate traffic | False positives | Check WAF logs, add exclusions |

---

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
