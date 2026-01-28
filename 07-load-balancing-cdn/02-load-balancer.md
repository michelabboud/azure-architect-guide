# Azure Load Balancer Deep Dive

## Overview

Azure Load Balancer is a Layer 4 (TCP/UDP) load balancer that provides high availability by distributing incoming traffic among healthy VMs. For AWS architects, it's comparable to Network Load Balancer (NLB).

## AWS Comparison

| Feature | AWS NLB | Azure Load Balancer |
|---------|---------|---------------------|
| Layer | 4 (TCP/UDP/TLS) | 4 (TCP/UDP) |
| TLS termination | Yes | No |
| Static IP | Yes | Yes (Standard SKU) |
| Cross-zone | Yes (extra cost) | Yes (zone-redundant) |
| HA Ports | Partial | Yes |
| Preserve client IP | Yes | Yes |
| Health probes | TCP, HTTP/HTTPS | TCP, HTTP/HTTPS |
| Target types | IP, Instance, ALB | IP, NIC |

## SKU Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    LOAD BALANCER SKU COMPARISON                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────┬──────────────────┬────────────────────┐  │
│  │ Feature                       │ Basic            │ Standard           │  │
│  ├───────────────────────────────┼──────────────────┼────────────────────┤  │
│  │ Backend pool size             │ Up to 300        │ Up to 1000         │  │
│  │ Backend pool type             │ Availability Set │ Any VMs in VNet    │  │
│  │                               │ or Scale Set     │                    │  │
│  │ Health probes                 │ TCP, HTTP        │ TCP, HTTP, HTTPS   │  │
│  │ Availability Zones            │ Not available    │ Zone-redundant     │  │
│  │ HA Ports                      │ Not available    │ Available          │  │
│  │ Outbound rules                │ Not available    │ Available          │  │
│  │ Multiple frontends            │ Not available    │ Available          │  │
│  │ Security                      │ Open by default  │ Closed by default  │  │
│  │ SLA                           │ Not available    │ 99.99%             │  │
│  │ Cost                          │ Free             │ Pay per rule/data  │  │
│  │ Recommended for               │ Dev/Test only    │ Production         │  │
│  └───────────────────────────────┴──────────────────┴────────────────────┘  │
│                                                                              │
│  IMPORTANT: Basic SKU will be retired. Always use Standard for new deploys. │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Architecture Types

### Public Load Balancer

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      PUBLIC LOAD BALANCER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                              INTERNET                                        │
│                                  │                                           │
│                                  ▼                                           │
│                    ┌─────────────────────────────┐                          │
│                    │      Public IP Address       │                          │
│                    │      (Static, Standard)      │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│                    ┌─────────────▼───────────────┐                          │
│                    │    LOAD BALANCER (Standard)  │                          │
│                    │                              │                          │
│                    │  Frontend: Public IP         │                          │
│                    │  Rules: TCP 80 → Backend:80  │                          │
│                    │         TCP 443 → Backend:443│                          │
│                    │  Health Probe: TCP 80        │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│         ┌────────────────────────┼────────────────────────┐                 │
│         │                        │                        │                 │
│         ▼                        ▼                        ▼                 │
│  ┌─────────────┐          ┌─────────────┐          ┌─────────────┐         │
│  │    VM 1     │          │    VM 2     │          │    VM 3     │         │
│  │   Zone 1    │          │   Zone 2    │          │   Zone 3    │         │
│  └─────────────┘          └─────────────┘          └─────────────┘         │
│                                                                              │
│  BACKEND POOL: All VMs across availability zones                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Internal Load Balancer

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     INTERNAL LOAD BALANCER                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                            VNET                                        │  │
│  │                                                                        │  │
│  │    ┌─────────────────┐                                                │  │
│  │    │   Web Tier      │                                                │  │
│  │    │   VMs           │                                                │  │
│  │    └────────┬────────┘                                                │  │
│  │             │                                                          │  │
│  │             ▼                                                          │  │
│  │    ┌─────────────────────────────┐                                    │  │
│  │    │  INTERNAL LOAD BALANCER     │                                    │  │
│  │    │  Frontend: 10.0.2.100       │                                    │  │
│  │    │  (Private IP in app subnet) │                                    │  │
│  │    └─────────────┬───────────────┘                                    │  │
│  │                  │                                                     │  │
│  │    ┌─────────────┼─────────────┐                                      │  │
│  │    │             │             │                                      │  │
│  │    ▼             ▼             ▼                                      │  │
│  │  ┌─────┐      ┌─────┐      ┌─────┐                                   │  │
│  │  │App 1│      │App 2│      │App 3│   ← App Tier Backend Pool         │  │
│  │  └─────┘      └─────┘      └─────┘                                   │  │
│  │                                                                        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Creating Load Balancers

### Public Load Balancer

```bash
# Create public IP
az network public-ip create \
  --resource-group myRG \
  --name lb-public-ip \
  --sku Standard \
  --allocation-method Static \
  --zone 1 2 3

# Create load balancer
az network lb create \
  --resource-group myRG \
  --name myPublicLB \
  --sku Standard \
  --public-ip-address lb-public-ip \
  --frontend-ip-name frontendPool \
  --backend-pool-name backendPool

# Create health probe
az network lb probe create \
  --resource-group myRG \
  --lb-name myPublicLB \
  --name healthProbe \
  --protocol Tcp \
  --port 80 \
  --interval 5 \
  --threshold 2

# Create load balancing rule
az network lb rule create \
  --resource-group myRG \
  --lb-name myPublicLB \
  --name httpRule \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name frontendPool \
  --backend-pool-name backendPool \
  --probe-name healthProbe \
  --idle-timeout 4 \
  --enable-tcp-reset true

# Add VMs to backend pool
az network nic ip-config address-pool add \
  --resource-group myRG \
  --nic-name vm1-nic \
  --ip-config-name ipconfig1 \
  --lb-name myPublicLB \
  --address-pool backendPool
```

### Internal Load Balancer

```bash
# Create internal load balancer
az network lb create \
  --resource-group myRG \
  --name myInternalLB \
  --sku Standard \
  --vnet-name myVNet \
  --subnet app-subnet \
  --frontend-ip-name frontendPool \
  --private-ip-address 10.0.2.100 \
  --backend-pool-name backendPool

# Create health probe
az network lb probe create \
  --resource-group myRG \
  --lb-name myInternalLB \
  --name healthProbe \
  --protocol Http \
  --port 80 \
  --path /health

# Create load balancing rule
az network lb rule create \
  --resource-group myRG \
  --lb-name myInternalLB \
  --name appRule \
  --protocol Tcp \
  --frontend-port 8080 \
  --backend-port 8080 \
  --frontend-ip-name frontendPool \
  --backend-pool-name backendPool \
  --probe-name healthProbe
```

## HA Ports

HA Ports allow load balancing of all TCP and UDP ports simultaneously. Essential for NVA deployments.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        HA PORTS FOR NVA                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│                           SPOKE VNETS                                        │
│                               │                                              │
│                               │ All traffic (any port)                       │
│                               ▼                                              │
│                    ┌─────────────────────────────┐                          │
│                    │  INTERNAL LOAD BALANCER     │                          │
│                    │  (HA Ports Rule)            │                          │
│                    │                             │                          │
│                    │  Protocol: All              │                          │
│                    │  Frontend Port: 0           │                          │
│                    │  Backend Port: 0            │                          │
│                    │  Floating IP: Enabled       │                          │
│                    └─────────────┬───────────────┘                          │
│                                  │                                           │
│                    ┌─────────────┼─────────────┐                            │
│                    │             │             │                            │
│                    ▼             ▼             ▼                            │
│             ┌───────────┐ ┌───────────┐ ┌───────────┐                      │
│             │   NVA 1   │ │   NVA 2   │ │   NVA 3   │                      │
│             │  Active   │ │  Standby  │ │  Standby  │                      │
│             └───────────┘ └───────────┘ └───────────┘                      │
│                                                                              │
│  USE CASES:                                                                 │
│  • Third-party firewalls (Palo Alto, Fortinet, Check Point)                │
│  • SD-WAN appliances                                                        │
│  • Custom routing appliances                                                │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Create HA Ports rule for NVA
az network lb rule create \
  --resource-group myRG \
  --lb-name nvaLoadBalancer \
  --name haPortsRule \
  --protocol All \
  --frontend-port 0 \
  --backend-port 0 \
  --frontend-ip-name frontendPool \
  --backend-pool-name nvaBackendPool \
  --probe-name nvaHealthProbe \
  --enable-floating-ip true \
  --idle-timeout 4
```

## Outbound Rules

Control outbound NAT behavior for backend pool members.

```bash
# Create outbound rule
az network lb outbound-rule create \
  --resource-group myRG \
  --lb-name myPublicLB \
  --name outboundRule \
  --frontend-ip-configs frontendPool \
  --protocol All \
  --outbound-ports 10000 \
  --idle-timeout 4 \
  --address-pool backendPool

# Create with multiple frontend IPs for more SNAT ports
az network lb outbound-rule create \
  --resource-group myRG \
  --lb-name myPublicLB \
  --name outboundRule \
  --frontend-ip-configs frontend1 frontend2 frontend3 \
  --protocol All \
  --outbound-ports 30000 \
  --address-pool backendPool
```

### SNAT Port Calculation

```
SNAT Ports per VM = (Number of Frontend IPs × 64000) / Backend Pool Size

Example:
- 3 Frontend IPs × 64000 = 192,000 total ports
- 10 VMs in backend pool
- 192,000 / 10 = 19,200 SNAT ports per VM
```

## Session Persistence

| Distribution Mode | Description | Use Case |
|-------------------|-------------|----------|
| None (default) | 5-tuple hash | Stateless apps |
| Client IP | 2-tuple (source IP, dest IP) | Legacy apps |
| Client IP and Protocol | 3-tuple | Protocol-specific affinity |

```bash
# Create rule with session persistence
az network lb rule create \
  --resource-group myRG \
  --lb-name myPublicLB \
  --name stickyRule \
  --protocol Tcp \
  --frontend-port 443 \
  --backend-port 443 \
  --frontend-ip-name frontendPool \
  --backend-pool-name backendPool \
  --probe-name healthProbe \
  --load-distribution SourceIP
```

## Bicep Template

```bicep
@description('Location for resources')
param location string = resourceGroup().location

@description('VNet name for internal LB')
param vnetName string

@description('Subnet name for internal LB')
param subnetName string

// Public IP for public LB
resource publicIp 'Microsoft.Network/publicIPAddresses@2023-05-01' = {
  name: 'lb-pip'
  location: location
  sku: { name: 'Standard' }
  zones: ['1', '2', '3']
  properties: {
    publicIPAllocationMethod: 'Static'
  }
}

// Public Load Balancer
resource publicLb 'Microsoft.Network/loadBalancers@2023-05-01' = {
  name: 'public-lb'
  location: location
  sku: { name: 'Standard' }
  properties: {
    frontendIPConfigurations: [
      {
        name: 'frontend'
        properties: {
          publicIPAddress: { id: publicIp.id }
        }
      }
    ]
    backendAddressPools: [
      { name: 'backend' }
    ]
    probes: [
      {
        name: 'healthProbe'
        properties: {
          protocol: 'Tcp'
          port: 80
          intervalInSeconds: 5
          numberOfProbes: 2
        }
      }
    ]
    loadBalancingRules: [
      {
        name: 'httpRule'
        properties: {
          frontendIPConfiguration: {
            id: resourceId('Microsoft.Network/loadBalancers/frontendIPConfigurations', 'public-lb', 'frontend')
          }
          backendAddressPool: {
            id: resourceId('Microsoft.Network/loadBalancers/backendAddressPools', 'public-lb', 'backend')
          }
          probe: {
            id: resourceId('Microsoft.Network/loadBalancers/probes', 'public-lb', 'healthProbe')
          }
          protocol: 'Tcp'
          frontendPort: 80
          backendPort: 80
          enableFloatingIP: false
          enableTcpReset: true
          idleTimeoutInMinutes: 4
        }
      }
    ]
  }
}

// Internal Load Balancer
resource internalLb 'Microsoft.Network/loadBalancers@2023-05-01' = {
  name: 'internal-lb'
  location: location
  sku: { name: 'Standard' }
  properties: {
    frontendIPConfigurations: [
      {
        name: 'frontend'
        properties: {
          subnet: {
            id: resourceId('Microsoft.Network/virtualNetworks/subnets', vnetName, subnetName)
          }
          privateIPAllocationMethod: 'Static'
          privateIPAddress: '10.0.2.100'
        }
      }
    ]
    backendAddressPools: [
      { name: 'backend' }
    ]
    probes: [
      {
        name: 'healthProbe'
        properties: {
          protocol: 'Http'
          port: 8080
          requestPath: '/health'
          intervalInSeconds: 5
          numberOfProbes: 2
        }
      }
    ]
    loadBalancingRules: [
      {
        name: 'appRule'
        properties: {
          frontendIPConfiguration: {
            id: resourceId('Microsoft.Network/loadBalancers/frontendIPConfigurations', 'internal-lb', 'frontend')
          }
          backendAddressPool: {
            id: resourceId('Microsoft.Network/loadBalancers/backendAddressPools', 'internal-lb', 'backend')
          }
          probe: {
            id: resourceId('Microsoft.Network/loadBalancers/probes', 'internal-lb', 'healthProbe')
          }
          protocol: 'Tcp'
          frontendPort: 8080
          backendPort: 8080
          enableFloatingIP: false
          idleTimeoutInMinutes: 4
        }
      }
    ]
  }
}

output publicLbId string = publicLb.id
output internalLbId string = internalLb.id
output publicIpAddress string = publicIp.properties.ipAddress
```

## Troubleshooting

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| VMs not receiving traffic | NSG blocking | Allow LB probe source (168.63.129.16) |
| Uneven distribution | Session persistence | Check load distribution settings |
| SNAT exhaustion | Too few ports | Add more frontend IPs or use NAT Gateway |
| Health probe failing | Wrong port/path | Verify probe configuration matches app |

### Diagnostic Commands

```bash
# View backend pool health
az network lb show \
  --resource-group myRG \
  --name myPublicLB \
  --query "backendAddressPools[].backendIPConfigurations[].id"

# Check probe status (via metrics)
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/loadBalancers/myPublicLB \
  --metric "HealthProbeStatus" \
  --interval PT1M

# View SNAT port usage
az monitor metrics list \
  --resource /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Network/loadBalancers/myPublicLB \
  --metric "SnatConnectionCount" \
  --interval PT1M
```

---

*Continue to [Front Door](03-front-door.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
