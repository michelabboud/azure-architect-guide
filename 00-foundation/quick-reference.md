# Foundation Quick Reference

> **Purpose**: One-page cheat sheet for AWS architects transitioning to Azure. Use this to quickly map your AWS mental model to Azure concepts.

---

## AWS to Azure Service Mapping (Quick Lookup)

| AWS Service | Azure Equivalent | Key Difference | Notes |
|-------------|------------------|-----------------|-------|
| **EC2** | Virtual Machines | Pay per minute vs per hour | Image Gallery has pre-configured options |
| **RDS** | Azure SQL Database | Managed service, no instance selection | Hyperscale option for massive scale |
| **S3** | Blob Storage | Container = bucket, Hot/Cool/Archive tiers | Lifecycle policies similar |
| **DynamoDB** | Cosmos DB | Global distribution native | Azure Table Storage for simpler needs |
| **Lambda** | Azure Functions | Consumption plan available | Durable Functions for stateful workflows |
| **API Gateway** | API Management | More features for API governance | Application Gateway simpler alternative |
| **CloudFront** | Azure CDN / Front Door | Front Door has DDoS + WAF built-in | CDN for origin failover |
| **VPC** | Virtual Network (VNet) | No implicit subnets | Network Security Groups = security groups |
| **ECS/EKS** | Container Instances / AKS | AKS is managed Kubernetes | Spot instances = Low-priority VMs |
| **Route 53** | Azure DNS | Simple zone management | Traffic Manager for load balancing |
| **CloudWatch** | Azure Monitor + Log Analytics | Unified with Application Insights | Query language: KQL vs CloudWatch Insights |
| **IAM** | Entra ID + RBAC | Identity-first model | Entra ID is the security perimeter |
| **Secrets Manager** | Key Vault | Includes keys + secrets + certificates | Unified secrets management |
| **S3 Transfer Acceleration** | Blob Storage Transfer Accelerator | Use ExpressRoute for consistent speed | For high-volume migrations |
| **SQS** | Storage Queues / Service Bus | Service Bus has Topics/Subscriptions | Queues simpler, Service Bus more powerful |

---

## Key Terminology Differences

### Identity & Access

| AWS | Azure | Meaning |
|-----|-------|---------|
| **IAM User** | **Entra ID User** | Individual identity |
| **IAM Role** | **Managed Identity** | Service-to-service authentication (no credentials) |
| **IAM Policy** | **RBAC Role Definition + Azure Policy** | RBAC = permissions, Policy = governance |
| **Access Key** | **Credentials (Secret/Certificate)** | Authentication material |
| **AssumeRole** | **Managed Identity / Workload Identity** | Cross-service access without secrets |

### Infrastructure & Hierarchy

| AWS | Azure | Meaning |
|-----|-------|---------|
| **AWS Organization** | **Entra ID Tenant** | Top-level container |
| **Organization Unit (OU)** | **Management Group** | Logical grouping for policy |
| **AWS Account** | **Subscription** | Billing & resource boundary |
| **VPC** | **Virtual Network** | Network isolation |
| **Region** | **Region** | Physical location (same concept) |
| **Availability Zone** | **Availability Zone** | Physical isolation (same concept) |

### Observability

| AWS | Azure | Meaning |
|-----|-------|---------|
| **CloudWatch Metrics** | **Azure Monitor Metrics** | Time-series performance data |
| **CloudWatch Logs** | **Log Analytics Workspace** | Centralized log storage |
| **CloudWatch Insights** | **Kusto Query Language (KQL)** | Query language for logs |
| **CloudTrail** | **Activity Log + Audit Logs** | Split: Activity Log for resources, Audit Logs for identity |
| **X-Ray** | **Application Insights** | Distributed tracing |

---

## Azure Hierarchy (Management Model)

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ENTRA ID TENANT (Top Level)                    │
│                   (Your organization identity)                      │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              MANAGEMENT GROUPS (Optional Layer)                │ │
│  │         (Organize subscriptions by business need)              │ │
│  │                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │  Management Group: "Production"                         │  │ │
│  │  │  ├─ Subscription: "Prod-Americas"                       │  │ │
│  │  │  └─ Subscription: "Prod-EMEA"                           │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  │                                                                │ │
│  │  ┌─────────────────────────────────────────────────────────┐  │ │
│  │  │  Management Group: "Non-Production"                     │  │ │
│  │  │  ├─ Subscription: "Dev"                                 │  │ │
│  │  │  └─ Subscription: "Test"                                │  │ │
│  │  └─────────────────────────────────────────────────────────┘  │ │
│  │                                                                │ │
│  │  └─ Policy applied at MG level cascades to all subscriptions  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                                                                      │
│  ┌─ SUBSCRIPTIONS (Billing + Resource Boundary)                    │
│  │  └─ Each subscription has its own resources, quotas, billing   │
│  │                                                                 │
│  └─ RESOURCE GROUPS (Logical Grouping)                            │
│     └─ Within each subscription, organize resources by RG         │
│        • RG determines region, access controls, RBAC              │
│        • Resources in same RG = easier management                 │
└─────────────────────────────────────────────────────────────────────┘
```

### Quick Rules

- **Entra ID Tenant**: One per organization (not per subscription like AWS account)
- **Management Groups**: Optional, but recommended for multi-subscription governance
- **Subscriptions**: Billing unit (think AWS account)
- **Resource Groups**: Logical container (no AWS equivalent, closest = Tags)

---

## Common CLI Commands

### Authentication & Context

```bash
# Login to Azure
az login

# Login with specific tenant
az login --tenant <tenant-id>

# Get current subscription
az account show

# List all subscriptions
az account list

# Switch subscription
az account set --subscription <subscription-id>

# Get current user/service principal
az ad signed-in-user show

# Get access token (for API calls)
az account get-access-token
```

### Resource Management

```bash
# Create resource group
az group create --name <rg-name> --location eastus

# List resource groups
az group list

# Delete resource group (deletes all resources)
az group delete --name <rg-name> --yes

# List all resources in subscription
az resource list

# List resources in specific RG
az resource list --resource-group <rg-name>

# Get resource details
az resource show --resource-group <rg-name> --name <resource-name> \
  --resource-type Microsoft.Compute/virtualMachines
```

### Virtual Machines

```bash
# Create VM
az vm create \
  --resource-group <rg-name> \
  --name <vm-name> \
  --image UbuntuLTS \
  --admin-username azureuser

# List VMs
az vm list

# Start VM
az vm start --resource-group <rg-name> --name <vm-name>

# Stop VM (deallocate to avoid charges)
az vm deallocate --resource-group <rg-name> --name <vm-name>

# Delete VM
az vm delete --resource-group <rg-name> --name <vm-name> --yes

# Get VM details
az vm show --resource-group <rg-name> --name <vm-name>
```

### Storage & Networking

```bash
# Create storage account
az storage account create \
  --name <storage-name> \
  --resource-group <rg-name> \
  --location eastus \
  --sku Standard_LRS

# Create virtual network
az network vnet create \
  --resource-group <rg-name> \
  --name <vnet-name> \
  --address-prefix 10.0.0.0/16

# Create subnet
az network vnet subnet create \
  --resource-group <rg-name> \
  --vnet-name <vnet-name> \
  --name <subnet-name> \
  --address-prefix 10.0.1.0/24

# Create Network Security Group (firewall rules)
az network nsg create \
  --resource-group <rg-name> \
  --name <nsg-name>

# Add NSG rule (allow SSH)
az network nsg rule create \
  --resource-group <rg-name> \
  --nsg-name <nsg-name> \
  --name allow-ssh \
  --priority 100 \
  --source-address-prefixes '*' \
  --destination-port-ranges 22 \
  --access Allow
```

### Identity & RBAC

```bash
# Get current identity
az ad signed-in-user show

# Create Entra ID user
az ad user create \
  --display-name "John Doe" \
  --user-principal-name john@contoso.com \
  --password <password>

# Assign RBAC role
az role assignment create \
  --assignee <user-id> \
  --role Contributor \
  --scope /subscriptions/<subscription-id>

# List role assignments
az role assignment list

# List available roles
az role definition list --output table

# Create managed identity
az identity create \
  --resource-group <rg-name> \
  --name <identity-name>

# Assign role to managed identity
az role assignment create \
  --assignee <identity-object-id> \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<subscription-id>/resourcegroups/<rg-name>
```

---

## Azure Portal Navigation (Key Destinations)

### Finding What You Need

| Task | Path | Hot Key |
|------|------|---------|
| **Create new resource** | Home > Create a resource | + |
| **Search for resources** | Search bar (top center) | Cmd+K / Ctrl+K |
| **Your subscriptions** | Home > Subscriptions | — |
| **Billing & cost** | Cost Management + Billing | — |
| **Role assignments** | Resource > Access Control (IAM) | — |
| **Alerts & monitoring** | Monitor > Alerts | — |
| **Audit logs** | Activity Log (per resource) | — |
| **Policy assignments** | Policy > Assignments | — |
| **Management Groups** | Management Groups | — |
| **Marketplace** | Home > Marketplace | — |

### Power User Tips

1. **Favorites Bar**: Pin frequently used services to the left sidebar
   - Click ⭐ on any service in the left menu

2. **Portal Settings** (⚙️ top-right):
   - Change theme (light/dark)
   - Set default subscription
   - Set timezone

3. **Quick Create**: Home page has shortcuts for common resources
   - Customize which resources appear

4. **Azure Cloud Shell**:
   - >_ icon top-right opens bash/PowerShell terminal
   - Pre-authenticated, Azure CLI pre-installed

5. **Links Everywhere**:
   - Resource names are clickable (navigate to resource)
   - Metrics are clickable (drill into detailed view)
   - Related resources show as links

6. **Cost Analysis**:
   - Cost Management > Cost Analysis
   - Filter by resource, resource group, service type
   - Set up budgets & alerts

---

## Decision Tree: Which Azure Service?

```
┌──────────────────────────────────────┐
│  Need to run code/applications?      │
└────┬─────────────────────────────────┘
     │
     ├─→ Full control over OS/infrastructure?
     │   └─→ Virtual Machines (VMs)
     │
     ├─→ Just need to run functions/scripts?
     │   └─→ Azure Functions (serverless)
     │
     └─→ Need to run containers?
         └─→ AKS (Kubernetes) or Container Instances

┌──────────────────────────────────────┐
│  Need to store data?                 │
└────┬─────────────────────────────────┘
     │
     ├─→ Files/Objects (unstructured)?
     │   └─→ Blob Storage
     │
     ├─→ Structured data / SQL?
     │   └─→ Azure SQL Database
     │
     ├─→ NoSQL / Key-value?
     │   └─→ Cosmos DB
     │
     └─→ Backups / Archives?
         └─→ Blob Storage (Archive tier)

┌──────────────────────────────────────┐
│  Need secure secret management?      │
└────┬─────────────────────────────────┘
     │
     └─→ Key Vault (keys + secrets + certificates)
        └─→ All access via Entra ID
        └─→ Audit all access attempts
```

---

## Common Gotchas (AWS → Azure)

| Gotcha | AWS Default | Azure Default | Watch Out For |
|--------|-------------|----------------|---------------|
| **VM Billing** | Pay by hour | Pay by minute | Deallocate VMs to stop charges (stopping ≠ deallocating) |
| **Storage Cleanup** | S3 doesn't auto-delete | Blob Storage doesn't auto-delete | Set lifecycle policies or manually delete |
| **Network Security** | Security groups deny-by-default | NSGs deny-by-default | Need explicit "Allow" rules |
| **Access Control** | IAM is access gate | Entra ID is access gate | Role assignments != Entra group membership |
| **Regions** | Must explicitly choose | Must explicitly choose | Storage replication decisions matter for cost |
| **Quotas** | High defaults | Strict quotas by subscription | Request limit increase early |
| **Free Tier** | 12-month free | 12-month free (limited) | Some services always paid |

---

## Key Concepts Cheat Sheet

| Concept | Meaning | AWS Equivalent | Example |
|---------|---------|---|---------|
| **Managed Identity** | Azure-native service authentication (no credentials) | IAM Role + AssumeRole | VM → Key Vault without secrets |
| **Entra ID** | Azure's identity provider (NOT just IAM) | IAM + Cognito + Single Sign-On | All user authentication flows |
| **Resource Group** | Logical container for resources (same region) | Tags + account boundaries | GroupBy="project-name", Region="eastus" |
| **Subscription** | Billing boundary + quota container | AWS Account | Each Subscription has separate bill |
| **Policy** | Compliance enforcement (audit/deny) | AWS Config + SCPs | "All VMs must have backup" |
| **RBAC** | Role-based access control | IAM roles + policies | "Developers" group gets Contributor role |
| **NSG** | Network firewall (inbound/outbound rules) | Security Group | Port 443 inbound, port 3306 deny |
| **UDR** | User-defined routes (custom routing) | Route tables | Force traffic through firewall appliance |
| **Service Endpoint** | VNet → Azure Service tunnel | VPC endpoints | Access Storage securely without public IP |

---

## Next Steps

1. **Complete the Getting Started guide** → [Azure Fundamentals](azure-fundamentals.md)
2. **Map all your services** → [AWS-Azure Mapping](aws-azure-mapping.md)
3. **Set up your tenant** → [Chapter 01: Identity](../02-identity/README.md)
4. **Explore the full guide** → [Back to Foundation](README.md)

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
