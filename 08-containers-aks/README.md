# Chapter 13: Containers & Azure Kubernetes Service (AKS)

## AWS to Azure Container Services Map

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AWS vs AZURE CONTAINER SERVICES                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS                              Azure                                      │
│  ───                              ─────                                      │
│                                                                              │
│  CONTAINER REGISTRY:                                                         │
│  ┌─────────────────────┐         ┌──────────────────────────────────────┐   │
│  │ ECR                 │  ←───→  │ Azure Container Registry (ACR)       │   │
│  │ (Elastic Container  │         │ • Geo-replication                    │   │
│  │  Registry)          │         │ • Private Link support               │   │
│  └─────────────────────┘         │ • Image scanning                     │   │
│                                  │ • Tasks (build automation)           │   │
│                                  └──────────────────────────────────────┘   │
│                                                                              │
│  MANAGED KUBERNETES:                                                         │
│  ┌─────────────────────┐         ┌──────────────────────────────────────┐   │
│  │ EKS                 │  ←───→  │ Azure Kubernetes Service (AKS)       │   │
│  │ (Elastic Kubernetes │         │ • Free control plane                 │   │
│  │  Service)           │         │ • Entra ID integration               │   │
│  └─────────────────────┘         │ • Azure Policy for K8s               │   │
│                                  │ • Virtual Nodes (serverless)         │   │
│                                  └──────────────────────────────────────┘   │
│                                                                              │
│  SERVERLESS CONTAINERS:                                                      │
│  ┌─────────────────────┐         ┌──────────────────────────────────────┐   │
│  │ Fargate             │  ←───→  │ Azure Container Instances (ACI)      │   │
│  │                     │         │ • Per-second billing                 │   │
│  └─────────────────────┘         │ • No cluster management              │   │
│                                  │ • Virtual Nodes in AKS               │   │
│                                  └──────────────────────────────────────┘   │
│                                                                              │
│  SIMPLE CONTAINER APPS:                                                      │
│  ┌─────────────────────┐         ┌──────────────────────────────────────┐   │
│  │ ECS (Elastic        │  ←───→  │ Azure Container Apps                 │   │
│  │ Container Service)  │         │ • Managed Kubernetes under the hood  │   │
│  │                     │         │ • KEDA auto-scaling                  │   │
│  │ App Runner          │  ←───→  │ • Dapr integration                   │   │
│  │                     │         │ • Event-driven scaling               │   │
│  └─────────────────────┘         └──────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Decision Tree: Which Container Service?

```
                    ┌─────────────────────────────────┐
                    │ What's your container workload?  │
                    └────────────────┬────────────────┘
                                     │
        ┌────────────────────────────┼────────────────────────────┐
        │                            │                            │
        ▼                            ▼                            ▼
┌───────────────────┐      ┌───────────────────┐      ┌───────────────────┐
│ Need full K8s     │      │ Simple containers │      │ Batch/scheduled   │
│ control?          │      │ with auto-scale?  │      │ jobs?             │
└────────┬──────────┘      └────────┬──────────┘      └────────┬──────────┘
         │                          │                          │
         ▼                          ▼                          ▼
┌───────────────────┐      ┌───────────────────┐      ┌───────────────────┐
│       AKS         │      │  Container Apps   │      │       ACI         │
│                   │      │                   │      │                   │
│ • Full K8s API    │      │ • Managed K8s     │      │ • Serverless      │
│ • Custom configs  │      │ • KEDA scaling    │      │ • Per-second      │
│ • Service mesh    │      │ • Dapr sidecar    │      │ • Quick start     │
│ • Multi-node pools│      │ • No K8s exposure │      │ • No cluster      │
└───────────────────┘      └───────────────────┘      └───────────────────┘
```

---

## Azure Kubernetes Service (AKS) Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AKS CLUSTER ARCHITECTURE                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    CONTROL PLANE (Azure-Managed, FREE)              │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐  │    │
│  │  │ API Server  │  │   etcd      │  │ Scheduler   │  │Controllers│  │    │
│  │  │             │  │   (HA)      │  │             │  │           │  │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘  └───────────┘  │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                         │                                    │
│                                         │ Kubernetes API                     │
│                                         ▼                                    │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │                    WORKER NODES (You Pay For These)                 │    │
│  │                                                                      │    │
│  │  NODE POOL: system (system workloads)                               │    │
│  │  ┌─────────────┐  ┌─────────────┐                                  │    │
│  │  │   Node 1    │  │   Node 2    │   VM Size: Standard_DS2_v2       │    │
│  │  │   (AZ 1)    │  │   (AZ 2)    │   Taints: CriticalAddonsOnly     │    │
│  │  └─────────────┘  └─────────────┘                                  │    │
│  │                                                                      │    │
│  │  NODE POOL: apppool (application workloads)                         │    │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                │    │
│  │  │   Node 1    │  │   Node 2    │  │   Node 3    │                │    │
│  │  │   (AZ 1)    │  │   (AZ 2)    │  │   (AZ 3)    │                │    │
│  │  │             │  │             │  │             │                │    │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │                │    │
│  │  │ │   Pod   │ │  │ │   Pod   │ │  │ │   Pod   │ │                │    │
│  │  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │                │    │
│  │  └─────────────┘  └─────────────┘  └─────────────┘                │    │
│  │                                                                      │    │
│  │  NODE POOL: gpupool (GPU workloads) - Optional                      │    │
│  │  ┌─────────────┐                                                   │    │
│  │  │   Node 1    │   VM Size: Standard_NC6s_v3                       │    │
│  │  │   (GPU)     │   Labels: gpu=nvidia                              │    │
│  │  └─────────────┘                                                   │    │
│  │                                                                      │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Creating an AKS Cluster

### Basic Cluster

```bash
# Create AKS cluster
az aks create \
  --resource-group myRG \
  --name myAKSCluster \
  --node-count 3 \
  --node-vm-size Standard_DS2_v2 \
  --enable-managed-identity \
  --network-plugin azure \
  --network-policy azure \
  --zones 1 2 3 \
  --generate-ssh-keys

# Get credentials
az aks get-credentials --resource-group myRG --name myAKSCluster

# Verify
kubectl get nodes
```

### Production Cluster

```bash
# Production AKS with all features
az aks create \
  --resource-group myRG \
  --name prod-aks \
  --location eastus \
  --kubernetes-version 1.28 \
  \
  # Identity
  --enable-managed-identity \
  --enable-aad \
  --aad-admin-group-object-ids {admin-group-id} \
  --enable-azure-rbac \
  \
  # Networking
  --network-plugin azure \
  --network-policy azure \
  --vnet-subnet-id /subscriptions/.../subnets/aks-subnet \
  --service-cidr 10.2.0.0/16 \
  --dns-service-ip 10.2.0.10 \
  --load-balancer-sku standard \
  \
  # Node configuration
  --node-count 3 \
  --node-vm-size Standard_DS3_v2 \
  --zones 1 2 3 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10 \
  \
  # Observability
  --enable-addons monitoring \
  --workspace-resource-id /subscriptions/.../workspaces/logs \
  --enable-defender \
  \
  # Security
  --enable-private-cluster \
  --enable-oidc-issuer \
  --enable-workload-identity
```

---

## AKS Scaling Options

### Cluster Autoscaler (Node Scaling)

```yaml
# Scales NODES based on pending pods
┌─────────────────────────────────────────────────────────────────────────────┐
│                        CLUSTER AUTOSCALER                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  BEFORE: 3 nodes, pods requesting more resources than available            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     ┌──────────┐                │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │     │ Pending  │                │
│  │  [Full]  │  │  [Full]  │  │  [Full]  │     │   Pod    │                │
│  └──────────┘  └──────────┘  └──────────┘     └──────────┘                │
│                                                     │                       │
│                                    Cluster Autoscaler detects               │
│                                                     │                       │
│  AFTER: New node added                              ▼                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐                   │
│  │  Node 1  │  │  Node 2  │  │  Node 3  │  │  Node 4  │                   │
│  │  [Full]  │  │  [Full]  │  │  [Full]  │  │ [NewPod] │                   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Enable cluster autoscaler on node pool
az aks nodepool update \
  --resource-group myRG \
  --cluster-name myAKSCluster \
  --name nodepool1 \
  --enable-cluster-autoscaler \
  --min-count 3 \
  --max-count 10
```

### Horizontal Pod Autoscaler (HPA)

```yaml
# Scales PODS based on metrics
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 3
  maxReplicas: 100
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
```

### KEDA (Event-Driven Scaling)

```yaml
# Scales based on event sources (queue length, etc.)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: order-processor
spec:
  scaleTargetRef:
    name: order-processor-deployment
  minReplicaCount: 0    # Scale to zero!
  maxReplicaCount: 100
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: orders
      messageCount: "5"  # Scale when > 5 messages
```

---

## AKS Security

### Workload Identity (Recommended over Pod Identity)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        WORKLOAD IDENTITY FLOW                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. Pod requests token from projected service account                       │
│  2. AKS OIDC issuer validates the token                                    │
│  3. Token exchanged for Entra ID token                                     │
│  4. Pod uses Entra ID token to access Azure resources                      │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                         AKS CLUSTER                                    │ │
│  │  ┌─────────────────────┐                                              │ │
│  │  │        Pod          │                                              │ │
│  │  │  ┌───────────────┐  │    1. Get token                              │ │
│  │  │  │ App Container │──┼────────────────────────────────┐             │ │
│  │  │  └───────────────┘  │                                │             │ │
│  │  │  ┌───────────────┐  │                                │             │ │
│  │  │  │ Service Acct  │  │                                │             │ │
│  │  │  │ Token (JWT)   │  │                                ▼             │ │
│  │  │  └───────────────┘  │                    ┌─────────────────────┐   │ │
│  │  └─────────────────────┘                    │   OIDC Issuer       │   │ │
│  └─────────────────────────────────────────────┤   (AKS managed)     │───┘ │
│                                                └──────────┬──────────┘     │
│                                                           │ 2. Validate    │
│                                                           ▼                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                         ENTRA ID                                       │ │
│  │  ┌─────────────────────┐                  ┌─────────────────────────┐ │ │
│  │  │ Federated Credential│──────────────────│ Managed Identity        │ │ │
│  │  │ (Trust K8s SA)      │                  │ (Has Azure permissions) │ │ │
│  │  └─────────────────────┘                  └─────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                           │ 3. Azure token │
│                                                           ▼                │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │  AZURE RESOURCES (Key Vault, Storage, SQL, etc.)                      │ │
│  │  4. Access with Azure token                                            │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

```bash
# Enable workload identity on cluster
az aks update \
  --resource-group myRG \
  --name myAKSCluster \
  --enable-oidc-issuer \
  --enable-workload-identity

# Create managed identity
az identity create --name aks-app-identity --resource-group myRG

# Create federated credential
az identity federated-credential create \
  --name k8s-federated \
  --identity-name aks-app-identity \
  --resource-group myRG \
  --issuer $(az aks show -n myAKSCluster -g myRG --query oidcIssuerProfile.issuerUrl -o tsv) \
  --subject system:serviceaccount:default:my-app-sa
```

---

## Container Apps vs AKS

| Feature | Container Apps | AKS |
|---------|---------------|-----|
| K8s knowledge needed | No | Yes |
| Control plane cost | Included | Free |
| Pricing | Per vCPU/memory/second | Per node VM |
| Scale to zero | Yes (KEDA built-in) | With KEDA addon |
| Service mesh | Dapr built-in | Istio/Linkerd addon |
| Ingress | Built-in | NGINX/App Gateway |
| Use case | Microservices, APIs, event-driven | Full K8s control |

```bash
# Create Container App (simpler alternative to AKS)
az containerapp create \
  --name myapp \
  --resource-group myRG \
  --environment myenv \
  --image myregistry.azurecr.io/myapp:v1 \
  --target-port 8080 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --cpu 0.5 \
  --memory 1.0Gi
```

---

## AWS EKS vs Azure AKS Comparison

| Feature | AWS EKS | Azure AKS |
|---------|---------|-----------|
| Control plane cost | $0.10/hour (~$73/month) | Free |
| Managed node groups | Yes | Yes (node pools) |
| Fargate integration | Yes | ACI (Virtual Nodes) |
| IAM integration | IRSA | Workload Identity |
| Network plugin | VPC CNI, Calico | Azure CNI, Kubenet |
| Service mesh | App Mesh | Istio addon |
| GitOps | Flux (addon) | Flux (addon) |
| Cluster autoscaler | Yes | Yes |
| Private cluster | Yes | Yes |

---

*Next Chapter: [Databases](../09-databases/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
