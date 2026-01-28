# Quick Reference: Network Security

## NSG Commands

### Create NSG

```bash
# Create NSG
az network nsg create \
  --resource-group myRG \
  --name myNSG \
  --location eastus

# Create rule - allow HTTP
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowHTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes '*' \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 80

# Create rule - deny all (explicit)
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name DenyAll \
  --priority 4096 \
  --direction Inbound \
  --access Deny \
  --protocol '*' \
  --source-address-prefixes '*' \
  --destination-address-prefixes '*'
```

### Associate NSG

```bash
# Associate with subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name mySubnet \
  --network-security-group myNSG

# Associate with NIC
az network nic update \
  --resource-group myRG \
  --name myNIC \
  --network-security-group myNSG
```

### Application Security Groups

```bash
# Create ASG
az network asg create \
  --resource-group myRG \
  --name webServersASG

# Add NIC to ASG
az network nic ip-config update \
  --resource-group myRG \
  --nic-name myNIC \
  --name ipconfig1 \
  --application-security-groups webServersASG

# NSG rule with ASG
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowWebTraffic \
  --priority 100 \
  --source-asgs webServersASG \
  --destination-port-ranges 80 443 \
  --protocol Tcp \
  --access Allow
```

## Azure Firewall Commands

### Deploy Firewall

```bash
# Create firewall subnet (must be named AzureFirewallSubnet)
az network vnet subnet create \
  --resource-group myRG \
  --vnet-name hubVNet \
  --name AzureFirewallSubnet \
  --address-prefix 10.0.1.0/24

# Create public IP for firewall
az network public-ip create \
  --resource-group myRG \
  --name fw-pip \
  --sku Standard \
  --allocation-method Static

# Create firewall
az network firewall create \
  --resource-group myRG \
  --name myFirewall \
  --location eastus \
  --sku AZFW_VNet \
  --tier Standard
```

### Firewall Rules

```bash
# Application rule (FQDN)
az network firewall application-rule create \
  --resource-group myRG \
  --firewall-name myFirewall \
  --collection-name AllowWeb \
  --name AllowGoogle \
  --protocols Http=80 Https=443 \
  --source-addresses 10.0.0.0/16 \
  --target-fqdns "*.google.com" "*.microsoft.com" \
  --action Allow \
  --priority 100

# Network rule
az network firewall network-rule create \
  --resource-group myRG \
  --firewall-name myFirewall \
  --collection-name AllowDNS \
  --name AllowDNS \
  --protocols UDP \
  --source-addresses 10.0.0.0/16 \
  --destination-addresses 168.63.129.16 \
  --destination-ports 53 \
  --action Allow \
  --priority 100

# NAT rule (DNAT)
az network firewall nat-rule create \
  --resource-group myRG \
  --firewall-name myFirewall \
  --collection-name NATRules \
  --name DNAT-HTTP \
  --protocols TCP \
  --source-addresses '*' \
  --destination-addresses <firewall-public-ip> \
  --destination-ports 80 \
  --translated-address 10.0.2.4 \
  --translated-port 80 \
  --action Dnat \
  --priority 100
```

## DDoS Protection Commands

```bash
# Create DDoS protection plan
az network ddos-protection create \
  --resource-group myRG \
  --name myDDoSPlan

# Associate with VNet
az network vnet update \
  --resource-group myRG \
  --name myVNet \
  --ddos-protection-plan myDDoSPlan
```

## Private Link Commands

### Private Endpoint

```bash
# Create private endpoint for storage
az network private-endpoint create \
  --resource-group myRG \
  --name myStoragePE \
  --vnet-name myVNet \
  --subnet privateEndpointSubnet \
  --private-connection-resource-id /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/mystorageaccount \
  --group-id blob \
  --connection-name myStorageConnection

# Create private DNS zone
az network private-dns zone create \
  --resource-group myRG \
  --name "privatelink.blob.core.windows.net"

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group myRG \
  --zone-name "privatelink.blob.core.windows.net" \
  --name myDNSLink \
  --virtual-network myVNet \
  --registration-enabled false

# Create DNS record
az network private-endpoint dns-zone-group create \
  --resource-group myRG \
  --endpoint-name myStoragePE \
  --name myZoneGroup \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name blob
```

## Flow Logs Commands

```bash
# Create storage account for logs
az storage account create \
  --resource-group myRG \
  --name flowlogsstorage \
  --sku Standard_LRS

# Enable NSG flow logs
az network watcher flow-log create \
  --resource-group myRG \
  --name myFlowLog \
  --nsg myNSG \
  --storage-account flowlogsstorage \
  --enabled true \
  --format JSON \
  --log-version 2 \
  --retention 30
```

## Common Service Tags

| Service Tag | Description | Use For |
|-------------|-------------|---------|
| Internet | Internet traffic | Outbound access |
| VirtualNetwork | VNet + peered VNets | Internal traffic |
| AzureLoadBalancer | Health probes | Allow LB probes |
| AzureCloud | All Azure IPs | Azure services |
| AzureCloud.EastUS | Regional Azure IPs | Regional services |
| Storage | Azure Storage | Storage access |
| Sql | Azure SQL | Database access |
| AzureKeyVault | Key Vault | Secrets access |
| AzureActiveDirectory | Azure AD | Identity |
| EventHub | Event Hubs | Messaging |
| ServiceBus | Service Bus | Messaging |

## Default NSG Rules (Cannot Delete)

### Inbound
| Priority | Name | Source | Dest | Port | Action |
|----------|------|--------|------|------|--------|
| 65000 | AllowVNetInBound | VirtualNetwork | VirtualNetwork | Any | Allow |
| 65001 | AllowAzureLBInBound | AzureLoadBalancer | Any | Any | Allow |
| 65500 | DenyAllInBound | Any | Any | Any | Deny |

### Outbound
| Priority | Name | Source | Dest | Port | Action |
|----------|------|--------|------|------|--------|
| 65000 | AllowVNetOutBound | VirtualNetwork | VirtualNetwork | Any | Allow |
| 65001 | AllowInternetOutBound | Any | Internet | Any | Allow |
| 65500 | DenyAllOutBound | Any | Any | Any | Deny |

## Troubleshooting Commands

```bash
# View effective NSG rules
az network nic show-effective-nsg \
  --resource-group myRG \
  --name myNIC

# Test IP flow
az network watcher show-next-hop \
  --resource-group myRG \
  --vm myVM \
  --source-ip 10.0.0.4 \
  --dest-ip 10.0.1.4

# Verify connectivity
az network watcher test-connectivity \
  --resource-group myRG \
  --source-resource myVM \
  --dest-address www.microsoft.com \
  --dest-port 443

# Check flow logs
az network watcher flow-log show \
  --resource-group myRG \
  --name myFlowLog
```

## Bicep Snippets

### NSG with Rules

```bicep
resource nsg 'Microsoft.Network/networkSecurityGroups@2023-05-01' = {
  name: 'web-nsg'
  location: location
  properties: {
    securityRules: [
      {
        name: 'AllowHTTPS'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '443'
        }
      }
      {
        name: 'AllowHTTP'
        properties: {
          priority: 110
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourceAddressPrefix: '*'
          sourcePortRange: '*'
          destinationAddressPrefix: '*'
          destinationPortRange: '80'
        }
      }
    ]
  }
}
```

### Azure Firewall Policy

```bicep
resource firewallPolicy 'Microsoft.Network/firewallPolicies@2023-05-01' = {
  name: 'fw-policy'
  location: location
  properties: {
    sku: {
      tier: 'Standard'
    }
    threatIntelMode: 'Alert'
  }
}

resource ruleCollectionGroup 'Microsoft.Network/firewallPolicies/ruleCollectionGroups@2023-05-01' = {
  parent: firewallPolicy
  name: 'DefaultRuleCollectionGroup'
  properties: {
    priority: 100
    ruleCollections: [
      {
        name: 'AllowWeb'
        ruleCollectionType: 'FirewallPolicyFilterRuleCollection'
        priority: 100
        action: { type: 'Allow' }
        rules: [
          {
            name: 'AllowHTTPS'
            ruleType: 'ApplicationRule'
            sourceAddresses: ['10.0.0.0/16']
            protocols: [{ protocolType: 'Https', port: 443 }]
            targetFqdns: ['*.microsoft.com', '*.azure.com']
          }
        ]
      }
    ]
  }
}
```

---

*Back to [Network Security Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
