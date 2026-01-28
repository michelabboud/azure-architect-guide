# Application Gateway Deep Dive

## Overview

Azure Application Gateway is a Layer 7 (HTTP/HTTPS) load balancer that provides application delivery controller (ADC) functionality. For AWS architects, it combines features of AWS ALB with some capabilities of AWS CloudFront.

## AWS Comparison

| Feature | AWS ALB | Azure App Gateway |
|---------|---------|-------------------|
| Layer | 7 (HTTP/HTTPS) | 7 (HTTP/HTTPS) |
| Path-based routing | Yes | Yes |
| Host-based routing | Yes | Yes (multi-site) |
| WAF | Separate service | Integrated (WAF v2) |
| SSL termination | Yes | Yes |
| WebSocket | Yes | Yes |
| HTTP/2 | Yes | Yes |
| Autoscaling | Built-in | v2 SKU |
| Private deployment | Yes | Yes |
| Session affinity | Yes | Yes (cookie-based) |

## Architecture Components

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   APPLICATION GATEWAY COMPONENTS                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                         FRONTEND                                     │    │
│  │  ┌───────────────────┐    ┌───────────────────┐                     │    │
│  │  │ Frontend IP       │    │ Listeners         │                     │    │
│  │  │ • Public IP       │───►│ • HTTP (80)       │                     │    │
│  │  │ • Private IP      │    │ • HTTPS (443)     │                     │    │
│  │  │ (or both)         │    │ • Multi-site      │                     │    │
│  │  └───────────────────┘    └─────────┬─────────┘                     │    │
│  └─────────────────────────────────────┼───────────────────────────────┘    │
│                                        │                                     │
│  ┌─────────────────────────────────────▼───────────────────────────────┐    │
│  │                          ROUTING                                     │    │
│  │  ┌───────────────────────────────────────────────────────────────┐  │    │
│  │  │ Request Routing Rules                                          │  │    │
│  │  │ • Basic: Listener → Backend Pool                               │  │    │
│  │  │ • Path-based: /api/* → API Pool, /static/* → Static Pool      │  │    │
│  │  │ • Multi-site: app1.com → Pool1, app2.com → Pool2             │  │    │
│  │  └───────────────────────────────────────────────────────────────┘  │    │
│  │  ┌───────────────────────────────────────────────────────────────┐  │    │
│  │  │ URL Path Maps (Path-based routing)                             │  │    │
│  │  │ • /api/*     → apiBackendPool                                  │  │    │
│  │  │ • /images/*  → storageBackendPool                              │  │    │
│  │  │ • default    → webBackendPool                                  │  │    │
│  │  └───────────────────────────────────────────────────────────────┘  │    │
│  └─────────────────────────────────────┬───────────────────────────────┘    │
│                                        │                                     │
│  ┌─────────────────────────────────────▼───────────────────────────────┐    │
│  │                          BACKEND                                     │    │
│  │  ┌───────────────────┐    ┌───────────────────┐                     │    │
│  │  │ Backend Pools     │    │ HTTP Settings     │                     │    │
│  │  │ • VMs             │    │ • Port            │                     │    │
│  │  │ • VMSS            │    │ • Protocol        │                     │    │
│  │  │ • App Service     │    │ • Cookie affinity │                     │    │
│  │  │ • IP addresses    │    │ • Connection drain│                     │    │
│  │  │ • FQDN            │    │ • Custom probe    │                     │    │
│  │  └───────────────────┘    └───────────────────┘                     │    │
│  │                                                                      │    │
│  │  ┌───────────────────┐                                              │    │
│  │  │ Health Probes     │                                              │    │
│  │  │ • HTTP/HTTPS      │                                              │    │
│  │  │ • Custom path     │                                              │    │
│  │  │ • Match conditions│                                              │    │
│  │  └───────────────────┘                                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Deployment Options

### Public Application Gateway

```bash
# Create subnet (minimum /24 recommended)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name myVNet \
  --name appgw-subnet \
  --address-prefixes 10.0.100.0/24

# Create public IP
az network public-ip create \
  --resource-group myRG \
  --name appgw-pip \
  --sku Standard \
  --allocation-method Static

# Create Application Gateway
az network application-gateway create \
  --resource-group myRG \
  --name myAppGateway \
  --location eastus \
  --sku Standard_v2 \
  --capacity 2 \
  --vnet-name myVNet \
  --subnet appgw-subnet \
  --public-ip-address appgw-pip \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --servers 10.0.1.4 10.0.1.5
```

### Private Application Gateway

```bash
# Create Application Gateway with private frontend only
az network application-gateway create \
  --resource-group myRG \
  --name myPrivateAppGateway \
  --location eastus \
  --sku Standard_v2 \
  --capacity 2 \
  --vnet-name myVNet \
  --subnet appgw-subnet \
  --private-ip-address 10.0.100.10 \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --servers 10.0.1.4 10.0.1.5
```

## SSL/TLS Configuration

### SSL Termination

```bash
# Upload PFX certificate
az network application-gateway ssl-cert create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name mySslCert \
  --cert-file /path/to/certificate.pfx \
  --cert-password "YourPassword"

# Create HTTPS listener
az network application-gateway http-listener create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name httpsListener \
  --frontend-ip appGatewayFrontendIP \
  --frontend-port 443 \
  --ssl-cert mySslCert

# Create HTTPS frontend port
az network application-gateway frontend-port create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name port443 \
  --port 443
```

### End-to-End SSL

```bash
# Create HTTP settings with HTTPS to backend
az network application-gateway http-settings create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name httpsBackendSettings \
  --port 443 \
  --protocol Https \
  --cookie-based-affinity Disabled \
  --connection-draining-timeout 60

# Upload trusted root certificate (for backend validation)
az network application-gateway root-cert create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name backendRootCert \
  --cert-file /path/to/backend-root.cer
```

### SSL Policy

```bash
# Set predefined SSL policy
az network application-gateway ssl-policy set \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --policy-type Predefined \
  --policy-name AppGwSslPolicy20220101

# Set custom SSL policy
az network application-gateway ssl-policy set \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --policy-type Custom \
  --min-protocol-version TLSv1_2 \
  --cipher-suites TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384 TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

## Routing Configurations

### Path-Based Routing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PATH-BASED ROUTING EXAMPLE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                    https://app.contoso.com                                   │
│                              │                                               │
│                              ▼                                               │
│                    ┌─────────────────┐                                      │
│                    │ App Gateway     │                                      │
│                    │ URL Path Map    │                                      │
│                    └────────┬────────┘                                      │
│                             │                                               │
│         ┌───────────────────┼───────────────────┐                          │
│         │                   │                   │                          │
│    /api/*               /static/*           /* (default)                   │
│         │                   │                   │                          │
│         ▼                   ▼                   ▼                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                    │
│  │ API Backend │    │ Storage     │    │ Web Backend │                    │
│  │ Pool        │    │ Backend     │    │ Pool        │                    │
│  │             │    │ Pool        │    │             │                    │
│  │ 10.0.2.x    │    │ (Blob)      │    │ 10.0.1.x    │                    │
│  └─────────────┘    └─────────────┘    └─────────────┘                    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Create backend pools
az network application-gateway address-pool create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name apiBackendPool \
  --servers 10.0.2.4 10.0.2.5

az network application-gateway address-pool create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name storageBackendPool \
  --servers mystorageaccount.blob.core.windows.net

# Create URL path map
az network application-gateway url-path-map create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name urlPathMap \
  --paths /api/* \
  --address-pool apiBackendPool \
  --http-settings apiHttpSettings \
  --default-address-pool webBackendPool \
  --default-http-settings webHttpSettings

# Add additional path rule
az network application-gateway url-path-map rule create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --path-map-name urlPathMap \
  --name staticRule \
  --paths /static/* /images/* \
  --address-pool storageBackendPool \
  --http-settings storageHttpSettings
```

### Multi-Site Hosting

```bash
# Create listeners for different hostnames
az network application-gateway http-listener create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name app1Listener \
  --frontend-ip appGatewayFrontendIP \
  --frontend-port httpsPort \
  --host-name app1.contoso.com \
  --ssl-cert mySslCert

az network application-gateway http-listener create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name app2Listener \
  --frontend-ip appGatewayFrontendIP \
  --frontend-port httpsPort \
  --host-name app2.contoso.com \
  --ssl-cert mySslCert

# Create routing rules for each site
az network application-gateway rule create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name app1Rule \
  --http-listener app1Listener \
  --address-pool app1BackendPool \
  --http-settings app1HttpSettings \
  --priority 100

az network application-gateway rule create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name app2Rule \
  --http-listener app2Listener \
  --address-pool app2BackendPool \
  --http-settings app2HttpSettings \
  --priority 200
```

## WAF Configuration

### Enable WAF

```bash
# Create WAF policy
az network application-gateway waf-policy create \
  --resource-group myRG \
  --name myWafPolicy

# Configure managed rules
az network application-gateway waf-policy managed-rule rule-set add \
  --resource-group myRG \
  --policy-name myWafPolicy \
  --type OWASP \
  --version 3.2

# Create App Gateway with WAF
az network application-gateway create \
  --resource-group myRG \
  --name myWafAppGateway \
  --location eastus \
  --sku WAF_v2 \
  --capacity 2 \
  --vnet-name myVNet \
  --subnet appgw-subnet \
  --public-ip-address appgw-pip \
  --waf-policy myWafPolicy
```

### Custom WAF Rules

```bash
# Create custom rule to block specific IPs
az network application-gateway waf-policy custom-rule create \
  --resource-group myRG \
  --policy-name myWafPolicy \
  --name blockBadIPs \
  --priority 10 \
  --action Block \
  --rule-type MatchRule

# Add match condition
az network application-gateway waf-policy custom-rule match-condition add \
  --resource-group myRG \
  --policy-name myWafPolicy \
  --name blockBadIPs \
  --match-variables RemoteAddr \
  --operator IPMatch \
  --values "192.168.1.0/24" "10.0.0.0/8"
```

## Autoscaling

```bash
# Configure autoscaling
az network application-gateway update \
  --resource-group myRG \
  --name myAppGateway \
  --set autoscaleConfiguration.minCapacity=2 \
  --set autoscaleConfiguration.maxCapacity=10

# Set capacity to 0 for fully automatic scaling (2-125)
az network application-gateway update \
  --resource-group myRG \
  --name myAppGateway \
  --capacity 0 \
  --set autoscaleConfiguration.minCapacity=2 \
  --set autoscaleConfiguration.maxCapacity=125
```

## Bicep Template

```bicep
@description('Location for resources')
param location string = resourceGroup().location

@description('Application Gateway name')
param appGatewayName string

@description('VNet name')
param vnetName string

@description('Subnet name')
param subnetName string

@description('Backend server IPs')
param backendServers array = ['10.0.1.4', '10.0.1.5']

// Public IP
resource publicIp 'Microsoft.Network/publicIPAddresses@2023-05-01' = {
  name: '${appGatewayName}-pip'
  location: location
  sku: { name: 'Standard' }
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

// Application Gateway
resource appGateway 'Microsoft.Network/applicationGateways@2023-05-01' = {
  name: appGatewayName
  location: location
  properties: {
    sku: {
      name: 'Standard_v2'
      tier: 'Standard_v2'
    }
    autoscaleConfiguration: {
      minCapacity: 2
      maxCapacity: 10
    }
    gatewayIPConfigurations: [
      {
        name: 'gatewayIP'
        properties: {
          subnet: {
            id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, subnetName)
          }
        }
      }
    ]
    frontendIPConfigurations: [
      {
        name: 'frontendIP'
        properties: {
          publicIPAddress: { id: publicIp.id }
        }
      }
    ]
    frontendPorts: [
      {
        name: 'port80'
        properties: { port: 80 }
      }
      {
        name: 'port443'
        properties: { port: 443 }
      }
    ]
    backendAddressPools: [
      {
        name: 'backendPool'
        properties: {
          backendAddresses: [for server in backendServers: {
            ipAddress: server
          }]
        }
      }
    ]
    backendHttpSettingsCollection: [
      {
        name: 'httpSettings'
        properties: {
          port: 80
          protocol: 'Http'
          cookieBasedAffinity: 'Disabled'
          requestTimeout: 30
          probe: { id: resourceId('Microsoft.Network/applicationGateways/probes', appGatewayName, 'healthProbe') }
        }
      }
    ]
    httpListeners: [
      {
        name: 'httpListener'
        properties: {
          frontendIPConfiguration: { id: resourceId('Microsoft.Network/applicationGateways/frontendIPConfigurations', appGatewayName, 'frontendIP') }
          frontendPort: { id: resourceId('Microsoft.Network/applicationGateways/frontendPorts', appGatewayName, 'port80') }
          protocol: 'Http'
        }
      }
    ]
    requestRoutingRules: [
      {
        name: 'routingRule'
        properties: {
          priority: 100
          ruleType: 'Basic'
          httpListener: { id: resourceId('Microsoft.Network/applicationGateways/httpListeners', appGatewayName, 'httpListener') }
          backendAddressPool: { id: resourceId('Microsoft.Network/applicationGateways/backendAddressPools', appGatewayName, 'backendPool') }
          backendHttpSettings: { id: resourceId('Microsoft.Network/applicationGateways/backendHttpSettingsCollection', appGatewayName, 'httpSettings') }
        }
      }
    ]
    probes: [
      {
        name: 'healthProbe'
        properties: {
          protocol: 'Http'
          host: '127.0.0.1'
          path: '/health'
          interval: 30
          timeout: 30
          unhealthyThreshold: 3
        }
      }
    ]
  }
}

output appGatewayId string = appGateway.id
output publicIpAddress string = publicIp.properties.ipAddress
```

## Troubleshooting

### Check Backend Health

```bash
# View backend health status
az network application-gateway show-backend-health \
  --resource-group myRG \
  --name myAppGateway \
  --output table
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| 502 Bad Gateway | Backend unhealthy | Check probe configuration and backend connectivity |
| 504 Gateway Timeout | Backend slow | Increase request timeout in HTTP settings |
| SSL errors | Certificate mismatch | Verify certificate chain and hostname |
| 403 Forbidden | WAF blocking | Check WAF logs and adjust rules |

### Diagnostic Logs

```bash
# Enable diagnostics
az monitor diagnostic-settings create \
  --resource /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/applicationGateways/myAppGateway \
  --name appgw-diagnostics \
  --logs '[{"category":"ApplicationGatewayAccessLog","enabled":true},{"category":"ApplicationGatewayPerformanceLog","enabled":true},{"category":"ApplicationGatewayFirewallLog","enabled":true}]' \
  --workspace /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.OperationalInsights/workspaces/myWorkspace
```

---

*Continue to [Azure Load Balancer](02-load-balancer.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
