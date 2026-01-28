# Containers Quick Reference

## Service Selection Matrix

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   CONTAINER SERVICE COMPARISON                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────────────┬───────────┬─────────────────┬───────────────────────┐ │
│  │ Feature          │ AKS       │ Container Apps  │ ACI                   │ │
│  ├──────────────────┼───────────┼─────────────────┼───────────────────────┤ │
│  │ Kubernetes API   │ Full      │ Hidden          │ None                  │ │
│  │ Control plane    │ Free      │ Managed         │ None                  │ │
│  │ Auto-scaling     │ HPA/KEDA  │ KEDA built-in   │ Manual                │ │
│  │ Min cost         │ Node VMs  │ ~$0 (scale to 0)│ Per-second            │ │
│  │ Networking       │ Full CNI  │ VNet integration│ VNet integration      │ │
│  │ Service mesh     │ Yes       │ Dapr            │ No                    │ │
│  │ GPU support      │ Yes       │ No              │ Yes                   │ │
│  │ Windows          │ Yes       │ No              │ Yes                   │ │
│  │ Best for         │ Complex   │ Microservices   │ Batch/CI jobs         │ │
│  └──────────────────┴───────────┴─────────────────┴───────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## AWS to Azure Commands

| Operation | AWS | Azure |
|-----------|-----|-------|
| Create cluster | `eksctl create cluster` | `az aks create` |
| Get credentials | `aws eks update-kubeconfig` | `az aks get-credentials` |
| List clusters | `aws eks list-clusters` | `az aks list` |
| Scale nodes | `eksctl scale nodegroup` | `az aks nodepool scale` |
| Push image | `docker push xxx.dkr.ecr...` | `az acr login && docker push` |

---

## Azure Container Registry (ACR)

```bash
# Create ACR
az acr create \
  --resource-group myRG \
  --name myacr \
  --sku Premium \
  --admin-enabled false

# Login to ACR
az acr login --name myacr

# Build and push image
az acr build \
  --registry myacr \
  --image myapp:v1 \
  --file Dockerfile .

# Import from Docker Hub
az acr import \
  --name myacr \
  --source docker.io/library/nginx:latest \
  --image nginx:latest

# Enable geo-replication (Premium SKU)
az acr replication create \
  --registry myacr \
  --location westeurope

# Enable private endpoint
az network private-endpoint create \
  --name acr-pe \
  --resource-group myRG \
  --vnet-name myVNet \
  --subnet pe-subnet \
  --private-connection-resource-id $(az acr show --name myacr --query id -o tsv) \
  --group-id registry \
  --connection-name acr-connection
```

### ACR SKU Comparison

| Feature | Basic | Standard | Premium |
|---------|-------|----------|---------|
| Storage | 10 GB | 100 GB | 500 GB |
| Webhooks | 2 | 10 | 500 |
| Geo-replication | No | No | Yes |
| Private Link | No | No | Yes |
| Content trust | No | No | Yes |
| Price/month | ~$5 | ~$20 | ~$50 |

---

## AKS Quick Commands

```bash
# Create AKS cluster
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_D4s_v3 \
  --enable-managed-identity \
  --network-plugin azure \
  --network-policy azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --generate-ssh-keys \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10

# Get credentials
az aks get-credentials \
  --resource-group myRG \
  --name myAKSCluster

# Add node pool
az aks nodepool add \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name gpupool \
  --node-count 1 \
  --node-vm-size Standard_NC6 \
  --node-taints sku=gpu:NoSchedule \
  --labels hardware=gpu

# Scale node pool
az aks nodepool scale \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name nodepool1 \
  --node-count 5

# Enable add-ons
az aks enable-addons \
  --resource-group myRG \
  --name myAKSCluster \
  --addons monitoring,azure-policy

# Attach ACR
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --attach-acr myacr

# Upgrade cluster
az aks upgrade \
  --resource-group myRG \
  --name myAKSCluster \
  --kubernetes-version 1.28.0
```

---

## Container Apps Quick Commands

```bash
# Create Container Apps environment
az containerapp env create \
  --resource-group myRG \
  --name myEnvironment \
  --location eastus

# Create container app
az containerapp create \
  --resource-group myRG \
  --name myapp \
  --environment myEnvironment \
  --image myacr.azurecr.io/myapp:v1 \
  --registry-server myacr.azurecr.io \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --cpu 0.5 \
  --memory 1.0Gi

# Configure scaling rule
az containerapp update \
  --resource-group myRG \
  --name myapp \
  --scale-rule-name http-rule \
  --scale-rule-type http \
  --scale-rule-http-concurrency 100

# Create revision (new deployment)
az containerapp update \
  --resource-group myRG \
  --name myapp \
  --image myacr.azurecr.io/myapp:v2

# Traffic split
az containerapp ingress traffic set \
  --resource-group myRG \
  --name myapp \
  --revision-weight myapp--v1=80 myapp--v2=20
```

---

## ACI Quick Commands

```bash
# Run container
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image nginx:latest \
  --ports 80 \
  --dns-name-label myapp \
  --cpu 1 \
  --memory 1.5

# Run in VNet
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image myacr.azurecr.io/myapp:v1 \
  --vnet myVNet \
  --subnet aci-subnet \
  --ports 8080

# Run with managed identity
az container create \
  --resource-group myRG \
  --name mycontainer \
  --image myacr.azurecr.io/myapp:v1 \
  --assign-identity [system] \
  --acr-identity [system]

# Execute command in container
az container exec \
  --resource-group myRG \
  --name mycontainer \
  --exec-command "/bin/bash"

# View logs
az container logs \
  --resource-group myRG \
  --name mycontainer \
  --follow
```

---

## Networking Comparison

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    CONTAINER NETWORKING OPTIONS                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AKS NETWORK PLUGINS:                                                        │
│  ┌──────────────┬─────────────────────────────────────────────────────────┐ │
│  │ kubenet      │ • Pods get IPs from separate CIDR                       │ │
│  │ (Basic)      │ • UDR required for pod-to-pod across nodes             │ │
│  │              │ • Limited to 400 nodes                                  │ │
│  ├──────────────┼─────────────────────────────────────────────────────────┤ │
│  │ Azure CNI    │ • Pods get IPs from VNet subnet                        │ │
│  │ (Advanced)   │ • Native Azure networking                               │ │
│  │              │ • Required for Windows, Virtual Nodes                  │ │
│  ├──────────────┼─────────────────────────────────────────────────────────┤ │
│  │ Azure CNI    │ • Dynamic IP allocation                                │ │
│  │ Overlay      │ • More efficient IP usage                              │ │
│  │              │ • Pod CIDR separate from VNet                          │ │
│  └──────────────┴─────────────────────────────────────────────────────────┘ │
│                                                                              │
│  CONTAINER APPS:                                                             │
│  • VNet integration optional                                                │
│  • Internal/External ingress                                                │
│  • No direct pod networking control                                         │
│                                                                              │
│  ACI:                                                                        │
│  • VNet integration supported                                               │
│  • Dedicated subnet required (/24 minimum)                                  │
│  • No NSG support on ACI subnet                                             │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Common Patterns

### Pattern 1: AKS with ACR
```
ACR (Premium) → Private Endpoint → AKS
     ↑
   Build pipeline
```

### Pattern 2: Container Apps Microservices
```
Front Door → Container Apps Environment
                    ├── API App (external ingress)
                    ├── Worker App (no ingress)
                    └── Background Job (KEDA scaling)
```

### Pattern 3: ACI for CI/CD Agents
```
Azure DevOps → ACI (VNet) → Deploy to AKS
```

---

## Troubleshooting

| Issue | Command | Check |
|-------|---------|-------|
| Pod not starting | `kubectl describe pod <pod>` | Events section |
| ACR pull fail | `az aks check-acr --acr myacr` | ACR attachment |
| Node issues | `kubectl describe node <node>` | Conditions |
| Networking | `kubectl exec -it <pod> -- curl <service>` | Connectivity |
| Logs | `kubectl logs <pod> -f` | Application logs |

---

*Deep Dive: [AKS Architecture](01-aks-architecture.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
