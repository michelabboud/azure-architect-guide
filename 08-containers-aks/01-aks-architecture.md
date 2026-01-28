# AKS Architecture Deep Dive

## Overview

Azure Kubernetes Service (AKS) is Azure's managed Kubernetes offering. For AWS architects familiar with EKS, AKS provides similar functionality with some key differences in networking, identity integration, and management.

## AWS EKS vs Azure AKS

| Feature | AWS EKS | Azure AKS |
|---------|---------|-----------|
| Control plane cost | ~$72/month | Free |
| Control plane SLA | 99.95% | 99.95% (Standard tier) |
| Node management | Managed/Self-managed | Managed node pools |
| Identity | IAM/IRSA | Entra ID + Workload Identity |
| Networking | VPC CNI | Azure CNI/kubenet |
| Load Balancer | AWS LB Controller | Azure LB (integrated) |
| Storage | EBS CSI Driver | Azure Disk/File CSI |
| Registry | ECR | ACR |

## AKS Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           AKS ARCHITECTURE                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                 AZURE MANAGED (Control Plane)                        │    │
│  │                                                                      │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │    │
│  │  │   API Server │  │    etcd      │  │  Scheduler   │               │    │
│  │  │              │  │   (Managed)  │  │              │               │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘               │    │
│  │                                                                      │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │    │
│  │  │  Controller  │  │  Cloud       │  │  Admission   │               │    │
│  │  │  Manager     │  │  Controller  │  │  Controllers │               │    │
│  │  └──────────────┘  └──────────────┘  └──────────────┘               │    │
│  │                                                                      │    │
│  │  Cost: FREE (Standard tier: $0.10/cluster/hour for SLA)             │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                               │
│                              │ API Server Endpoint                          │
│                              │ (Public or Private)                          │
│                              │                                               │
│  ┌───────────────────────────┴───────────────────────────────────────────┐  │
│  │                 CUSTOMER MANAGED (Data Plane - VNet)                   │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  SYSTEM NODE POOL (Required)                                     │  │  │
│  │  │  • CoreDNS, metrics-server, kube-proxy                          │  │  │
│  │  │  • Tainted: CriticalAddonsOnly=true:NoSchedule                  │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                       │  │  │
│  │  │  │  Node 1  │  │  Node 2  │  │  Node 3  │                       │  │  │
│  │  │  │  (AZ 1)  │  │  (AZ 2)  │  │  (AZ 3)  │                       │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘                       │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  USER NODE POOL 1 (Application workloads)                       │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐        │  │  │
│  │  │  │  Node 1  │  │  Node 2  │  │  Node 3  │  │  Node N  │        │  │  │
│  │  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘        │  │  │
│  │  │                                                                  │  │  │
│  │  │  Autoscaler: min=3, max=20                                      │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │  USER NODE POOL 2 (GPU workloads)                               │  │  │
│  │  │  Taint: sku=gpu:NoSchedule                                      │  │  │
│  │  │  Label: hardware=gpu                                            │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌──────────┐  ┌──────────┐                                     │  │  │
│  │  │  │  NC6 VM  │  │  NC6 VM  │                                     │  │  │
│  │  │  └──────────┘  └──────────┘                                     │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                        │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Cluster Configuration

### Creating Production AKS Cluster

```bash
# Create resource group
az group create --name aks-rg --location eastus

# Create AKS cluster
az aks create \
  --resource-group aks-rg \
  --name prod-aks \
  --kubernetes-version 1.28.0 \
  --tier standard \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --enable-managed-identity \
  --enable-aad \
  --aad-admin-group-object-ids <admin-group-id> \
  --network-plugin azure \
  --network-policy azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --service-cidr 10.100.0.0/16 \
  --dns-service-ip 10.100.0.10 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 20 \
  --enable-addons monitoring,azure-policy,azure-keyvault-secrets-provider \
  --enable-oidc-issuer \
  --enable-workload-identity \
  --enable-defender \
  --generate-ssh-keys

# Add user node pool
az aks nodepool add \
  --resource-group aks-rg \
  --cluster-name prod-aks \
  --name apppool \
  --node-count 3 \
  --node-vm-size Standard_D8s_v5 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 50 \
  --labels workload=app \
  --mode User

# Add GPU node pool
az aks nodepool add \
  --resource-group aks-rg \
  --cluster-name prod-aks \
  --name gpupool \
  --node-count 0 \
  --node-vm-size Standard_NC6s_v3 \
  --enable-cluster-autoscaler \
  --min-count 0 \
  --max-count 5 \
  --node-taints sku=gpu:NoSchedule \
  --labels hardware=gpu \
  --mode User
```

### Bicep Template

```bicep
@description('Location for resources')
param location string = resourceGroup().location

@description('Kubernetes version')
param kubernetesVersion string = '1.28.0'

@description('AKS subnet ID')
param aksSubnetId string

@description('Entra ID admin group')
param aadAdminGroupId string

// Managed Identity
resource aksIdentity 'Microsoft.ManagedIdentity/userAssignedIdentities@2023-01-31' = {
  name: 'aks-identity'
  location: location
}

// AKS Cluster
resource aksCluster 'Microsoft.ContainerService/managedClusters@2024-01-01' = {
  name: 'prod-aks'
  location: location
  sku: {
    name: 'Base'
    tier: 'Standard'
  }
  identity: {
    type: 'UserAssigned'
    userAssignedIdentities: {
      '${aksIdentity.id}': {}
    }
  }
  properties: {
    kubernetesVersion: kubernetesVersion
    dnsPrefix: 'prod-aks'
    enableRBAC: true
    aadProfile: {
      managed: true
      enableAzureRBAC: true
      adminGroupObjectIDs: [aadAdminGroupId]
    }
    networkProfile: {
      networkPlugin: 'azure'
      networkPolicy: 'azure'
      serviceCidr: '10.100.0.0/16'
      dnsServiceIP: '10.100.0.10'
      loadBalancerSku: 'standard'
    }
    agentPoolProfiles: [
      {
        name: 'system'
        count: 3
        vmSize: 'Standard_D4s_v5'
        osType: 'Linux'
        mode: 'System'
        vnetSubnetID: aksSubnetId
        availabilityZones: ['1', '2', '3']
        enableAutoScaling: true
        minCount: 3
        maxCount: 5
        nodeTaints: ['CriticalAddonsOnly=true:NoSchedule']
      }
      {
        name: 'apppool'
        count: 3
        vmSize: 'Standard_D8s_v5'
        osType: 'Linux'
        mode: 'User'
        vnetSubnetID: aksSubnetId
        availabilityZones: ['1', '2', '3']
        enableAutoScaling: true
        minCount: 3
        maxCount: 50
        nodeLabels: {
          workload: 'app'
        }
      }
    ]
    addonProfiles: {
      omsagent: {
        enabled: true
        config: {
          logAnalyticsWorkspaceResourceID: logAnalytics.id
        }
      }
      azurepolicy: {
        enabled: true
      }
      azureKeyvaultSecretsProvider: {
        enabled: true
      }
    }
    securityProfile: {
      defender: {
        securityMonitoring: {
          enabled: true
        }
      }
      workloadIdentity: {
        enabled: true
      }
    }
    oidcIssuerProfile: {
      enabled: true
    }
  }
}

// Log Analytics
resource logAnalytics 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: 'aks-logs'
  location: location
  properties: {
    sku: { name: 'PerGB2018' }
    retentionInDays: 30
  }
}

output clusterName string = aksCluster.name
output clusterFqdn string = aksCluster.properties.fqdn
output oidcIssuerUrl string = aksCluster.properties.oidcIssuerProfile.issuerURL
```

## Networking Options

### Azure CNI

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      AZURE CNI NETWORKING                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  VNet: 10.0.0.0/16                                                          │
│  │                                                                           │
│  ├── AKS Subnet: 10.0.0.0/22 (1019 IPs)                                    │
│  │   │                                                                       │
│  │   ├── Node 1 (10.0.0.4)                                                  │
│  │   │   ├── Pod A (10.0.0.10)                                              │
│  │   │   ├── Pod B (10.0.0.11)                                              │
│  │   │   └── Pod C (10.0.0.12)                                              │
│  │   │                                                                       │
│  │   ├── Node 2 (10.0.0.5)                                                  │
│  │   │   ├── Pod D (10.0.0.20)                                              │
│  │   │   └── Pod E (10.0.0.21)                                              │
│  │   │                                                                       │
│  │   └── Node 3 (10.0.0.6)                                                  │
│  │       └── Pod F (10.0.0.30)                                              │
│  │                                                                           │
│  ├── App Gateway Subnet: 10.0.4.0/24                                       │
│  │                                                                           │
│  └── Database Subnet: 10.0.5.0/24                                          │
│      └── SQL Server can reach pods directly (same VNet)                    │
│                                                                              │
│  IP CALCULATION:                                                             │
│  • Max pods per node: 30 (default) or 250                                   │
│  • IPs needed = Nodes × Max_Pods + Nodes + Reserved                        │
│  • Example: 20 nodes × 30 pods + 20 + 5 = 625 IPs minimum                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Private Cluster

```bash
# Create private AKS cluster
az aks create \
  --resource-group aks-rg \
  --name private-aks \
  --enable-private-cluster \
  --private-dns-zone system \
  --network-plugin azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --enable-managed-identity

# Access private cluster via:
# 1. Jump box in VNet
# 2. VPN/ExpressRoute connection
# 3. az aks command invoke (Azure CLI proxy)
az aks command invoke \
  --resource-group aks-rg \
  --name private-aks \
  --command "kubectl get nodes"
```

## Identity and Security

### Workload Identity

```yaml
# 1. Create Kubernetes Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: workload-identity-sa
  namespace: default
  annotations:
    azure.workload.identity/client-id: <managed-identity-client-id>
---
# 2. Create Pod with Workload Identity
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: default
  labels:
    azure.workload.identity/use: "true"
spec:
  serviceAccountName: workload-identity-sa
  containers:
  - name: app
    image: myacr.azurecr.io/myapp:v1
    env:
    - name: AZURE_CLIENT_ID
      valueFrom:
        secretKeyRef:
          name: azure-identity
          key: client-id
```

```bash
# Create federated credential
az identity federated-credential create \
  --name aks-federated-credential \
  --identity-name my-identity \
  --resource-group aks-rg \
  --issuer $(az aks show -g aks-rg -n prod-aks --query oidcIssuerProfile.issuerUrl -o tsv) \
  --subject system:serviceaccount:default:workload-identity-sa \
  --audience api://AzureADTokenExchange
```

### Entra ID RBAC

```bash
# Enable Azure RBAC for Kubernetes
az aks update \
  --resource-group aks-rg \
  --name prod-aks \
  --enable-azure-rbac

# Assign cluster admin role
az role assignment create \
  --role "Azure Kubernetes Service Cluster Admin Role" \
  --assignee <user-or-group-id> \
  --scope /subscriptions/.../resourceGroups/aks-rg/providers/Microsoft.ContainerService/managedClusters/prod-aks

# Assign namespace-specific role
az role assignment create \
  --role "Azure Kubernetes Service Cluster User Role" \
  --assignee <user-or-group-id> \
  --scope /subscriptions/.../resourceGroups/aks-rg/providers/Microsoft.ContainerService/managedClusters/prod-aks/namespaces/production
```

## Storage Options

| Storage Type | Use Case | Access Mode | Performance |
|--------------|----------|-------------|-------------|
| Azure Disk | Database, stateful apps | ReadWriteOnce | Premium SSD: 7,500 IOPS |
| Azure Files | Shared storage | ReadWriteMany | Premium: 100K IOPS |
| Azure Blob (NFS) | Large datasets | ReadWriteMany | High throughput |

```yaml
# PersistentVolumeClaim for Azure Disk
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: db-storage
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: managed-csi-premium
  resources:
    requests:
      storage: 100Gi
---
# PersistentVolumeClaim for Azure Files
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-storage
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile-csi-premium
  resources:
    requests:
      storage: 100Gi
```

## Monitoring and Diagnostics

```bash
# Enable Container Insights
az aks enable-addons \
  --resource-group aks-rg \
  --name prod-aks \
  --addons monitoring \
  --workspace-resource-id /subscriptions/.../workspaces/my-workspace

# Enable Prometheus metrics (managed)
az aks update \
  --resource-group aks-rg \
  --name prod-aks \
  --enable-azure-monitor-metrics
```

### Key Metrics to Monitor

| Metric | Alert Threshold | Description |
|--------|-----------------|-------------|
| Node CPU | >80% | Scale up nodes |
| Node Memory | >80% | Scale up or optimize |
| Pod restarts | >5/hour | Application issues |
| API server latency | >500ms | Control plane issues |
| Disk pressure | Any | Add storage |

## Best Practices

1. **Separate node pools** - System vs user workloads
2. **Use availability zones** - Deploy across 3 zones
3. **Enable autoscaling** - Both cluster and pod autoscaling
4. **Private cluster** - For production workloads
5. **Workload Identity** - No stored credentials
6. **Network policies** - Implement zero-trust
7. **Resource quotas** - Prevent noisy neighbors
8. **Regular upgrades** - Stay within N-2 support

---

*Continue to [Container Apps](02-container-apps.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
