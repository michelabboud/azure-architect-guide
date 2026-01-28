# Load Balancing Quick Reference

## Service Selection Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCING DECISION MATRIX                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┬────────────────┬──────────────┬─────────────────────┐ │
│  │ Requirement      │ Load Balancer  │ App Gateway  │ Front Door          │ │
│  ├──────────────────┼────────────────┼──────────────┼─────────────────────┤ │
│  │ Layer            │ L4 (TCP/UDP)   │ L7 (HTTP/S)  │ L7 (HTTP/S)         │ │
│  │ Scope            │ Regional       │ Regional     │ Global              │ │
│  │ SSL termination  │ No             │ Yes          │ Yes                 │ │
│  │ WAF              │ No             │ Yes (v2)     │ Yes                 │ │
│  │ URL routing      │ No             │ Yes          │ Yes                 │ │
│  │ Session affinity │ Hash-based     │ Cookie-based │ Cookie-based        │ │
│  │ WebSocket        │ No             │ Yes          │ Yes                 │ │
│  │ Private only     │ Yes            │ Yes          │ No (Premium: Yes)   │ │
│  │ Multi-region     │ No             │ No           │ Yes                 │ │
│  │ CDN              │ No             │ No           │ Built-in            │ │
│  └──────────────────┴────────────────┴──────────────┴─────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## AWS to Azure Command Mapping

| Operation | AWS CLI | Azure CLI |
|-----------|---------|-----------|
| Create ALB | `aws elbv2 create-load-balancer --type application` | `az network application-gateway create` |
| Create NLB | `aws elbv2 create-load-balancer --type network` | `az network lb create --sku Standard` |
| Create target group | `aws elbv2 create-target-group` | `az network lb address-pool create` |
| Add listener | `aws elbv2 create-listener` | `az network application-gateway http-listener create` |
| Health check | `aws elbv2 create-target-group --health-check-*` | `az network lb probe create` |

---

## Azure Load Balancer Quick Commands

```bash
# Create Standard Load Balancer (public)
az network lb create \
  --resource-group myRG \
  --name myLoadBalancer \
  --sku Standard \
  --public-ip-address myPublicIP \
  --frontend-ip-name myFrontEnd \
  --backend-pool-name myBackEndPool

# Create Internal Load Balancer
az network lb create \
  --resource-group myRG \
  --name myInternalLB \
  --sku Standard \
  --vnet-name myVNet \
  --subnet mySubnet \
  --frontend-ip-name myFrontEnd \
  --backend-pool-name myBackEndPool

# Add health probe
az network lb probe create \
  --resource-group myRG \
  --lb-name myLoadBalancer \
  --name myHealthProbe \
  --protocol Tcp \
  --port 80

# Add load balancing rule
az network lb rule create \
  --resource-group myRG \
  --lb-name myLoadBalancer \
  --name myHTTPRule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name myFrontEnd \
  --backend-pool-name myBackEndPool \
  --probe-name myHealthProbe

# HA Ports rule (for NVAs)
az network lb rule create \
  --resource-group myRG \
  --lb-name myNvaLB \
  --name haPortsRule \
  --protocol All \
  --frontend-port 0 \
  --backend-port 0 \
  --frontend-ip-name myFrontEnd \
  --backend-pool-name myBackEndPool \
  --probe-name myHealthProbe
```

---

## Application Gateway Quick Commands

```bash
# Create Application Gateway v2
az network application-gateway create \
  --resource-group myRG \
  --name myAppGateway \
  --location eastus \
  --sku Standard_v2 \
  --capacity 2 \
  --vnet-name myVNet \
  --subnet appgw-subnet \
  --public-ip-address appgw-pip \
  --http-settings-cookie-based-affinity Disabled \
  --frontend-port 80 \
  --http-settings-port 80 \
  --http-settings-protocol Http \
  --servers 10.0.1.4 10.0.1.5

# Add HTTPS listener with certificate
az network application-gateway ssl-cert create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name myCert \
  --cert-file /path/to/cert.pfx \
  --cert-password "password"

az network application-gateway http-listener create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name httpsListener \
  --frontend-port 443 \
  --ssl-cert myCert

# Add path-based routing
az network application-gateway url-path-map create \
  --resource-group myRG \
  --gateway-name myAppGateway \
  --name urlPathMap \
  --paths /api/* \
  --address-pool apiBackendPool \
  --http-settings apiHttpSettings \
  --default-address-pool defaultBackendPool \
  --default-http-settings defaultHttpSettings

# Enable autoscaling
az network application-gateway update \
  --resource-group myRG \
  --name myAppGateway \
  --set sku.name=Standard_v2 \
  --set autoscaleConfiguration.minCapacity=1 \
  --set autoscaleConfiguration.maxCapacity=10
```

---

## Front Door Quick Commands

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
  --endpoint-name myEndpoint \
  --enabled-state Enabled

# Create origin group
az afd origin-group create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --probe-request-type HEAD \
  --probe-protocol Http \
  --probe-interval-in-seconds 30 \
  --sample-size 4 \
  --successful-samples-required 3

# Add origin
az afd origin create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --origin-group-name myOriginGroup \
  --origin-name myOrigin1 \
  --host-name myapp.azurewebsites.net \
  --origin-host-header myapp.azurewebsites.net \
  --http-port 80 \
  --https-port 443 \
  --priority 1 \
  --weight 1000

# Create route
az afd route create \
  --resource-group myRG \
  --profile-name myFrontDoor \
  --endpoint-name myEndpoint \
  --route-name myRoute \
  --origin-group myOriginGroup \
  --supported-protocols Https Http \
  --patterns-to-match "/*" \
  --forwarding-protocol HttpsOnly
```

---

## Azure CDN Quick Commands

```bash
# Create CDN profile
az cdn profile create \
  --resource-group myRG \
  --name myCdnProfile \
  --sku Standard_Microsoft

# Create CDN endpoint
az cdn endpoint create \
  --resource-group myRG \
  --profile-name myCdnProfile \
  --name myCdnEndpoint \
  --origin myapp.azurewebsites.net \
  --origin-host-header myapp.azurewebsites.net

# Add custom domain
az cdn custom-domain create \
  --resource-group myRG \
  --profile-name myCdnProfile \
  --endpoint-name myCdnEndpoint \
  --name myCustomDomain \
  --hostname www.contoso.com

# Enable HTTPS on custom domain
az cdn custom-domain enable-https \
  --resource-group myRG \
  --profile-name myCdnProfile \
  --endpoint-name myCdnEndpoint \
  --name myCustomDomain

# Purge CDN cache
az cdn endpoint purge \
  --resource-group myRG \
  --profile-name myCdnProfile \
  --name myCdnEndpoint \
  --content-paths "/*"
```

---

## SKU Comparison

### Load Balancer SKUs

| Feature | Basic | Standard |
|---------|-------|----------|
| Backend pool size | 300 | 1000 |
| Health probes | TCP, HTTP | TCP, HTTP, HTTPS |
| Availability Zones | No | Yes (zone-redundant) |
| HA Ports | No | Yes |
| Outbound rules | No | Yes |
| SLA | None | 99.99% |
| Security | Open by default | Closed by default (NSG required) |

### Application Gateway SKUs

| Feature | Standard v2 | WAF v2 |
|---------|-------------|--------|
| Autoscaling | Yes | Yes |
| Zone redundancy | Yes | Yes |
| Static VIP | Yes | Yes |
| WAF | No | Yes |
| Header rewrite | Yes | Yes |
| Private Link | Yes | Yes |

### Front Door SKUs

| Feature | Standard | Premium |
|---------|----------|---------|
| CDN | Yes | Yes |
| WAF | Standard policies | Custom + managed rules |
| Private Link origins | No | Yes |
| Bot protection | No | Yes |
| Advanced reports | No | Yes |

---

## Common Patterns

### Pattern 1: Multi-tier with Internal LB

```
Internet → App Gateway → Web VMs → Internal LB → App VMs → Internal LB → DB
```

### Pattern 2: Global with Regional Backends

```
Users → Front Door → [Region 1: App Gateway → VMs]
                  → [Region 2: App Gateway → VMs]
```

### Pattern 3: NVA High Availability

```
Spokes → Internal LB (HA Ports) → Active/Standby NVAs → Internet
```

---

## Troubleshooting Quick Reference

| Issue | Check | Command |
|-------|-------|---------|
| Health probe failing | Probe configuration | `az network lb probe show` |
| 502 Bad Gateway | Backend health | `az network application-gateway show-backend-health` |
| Front Door 503 | Origin health | `az afd origin show` |
| SSL errors | Certificate expiry | `az network application-gateway ssl-cert show` |
| Slow response | Backend pool latency | Check metrics in portal |

---

*Deep Dive: [Application Gateway](01-application-gateway.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
