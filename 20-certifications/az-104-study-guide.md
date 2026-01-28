# AZ-104: Azure Administrator - Study Guide

## Exam Overview

| Attribute | Details |
|-----------|---------|
| Exam Code | AZ-104 |
| Duration | 100 minutes |
| Questions | 40-60 questions |
| Passing Score | 700/1000 |
| Prerequisite | None (AZ-900 recommended) |
| Cost | $165 USD |

## Why AZ-104 for AWS Architects?

While you might jump directly to AZ-305, AZ-104 fills important operational gaps:

- **Azure-specific CLI/PowerShell commands**
- **Portal navigation and workflows**
- **Day-to-day administration tasks**
- **Troubleshooting and monitoring**

## Exam Domains

### Domain 1: Manage Azure Identities and Governance (20-25%)

#### Key Topics

| Topic | AWS Equivalent | Azure Focus |
|-------|---------------|-------------|
| Azure AD users/groups | IAM Users | User management, licenses |
| RBAC | IAM Policies | Built-in vs custom roles |
| Subscriptions | Accounts | Management hierarchy |
| Azure Policy | Config Rules | Policy assignment, compliance |
| Resource locks | Resource policies | Delete/ReadOnly locks |

#### Essential CLI Commands

```bash
# Create user
az ad user create --display-name "John Doe" \
  --user-principal-name john@contoso.com \
  --password "SecureP@ss123"

# Create group
az ad group create --display-name "Developers" \
  --mail-nickname "developers"

# Assign role
az role assignment create \
  --assignee john@contoso.com \
  --role "Contributor" \
  --scope /subscriptions/{sub-id}/resourceGroups/myRG

# Create policy assignment
az policy assignment create \
  --name "Require-Tag-Environment" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/..." \
  --scope /subscriptions/{sub-id}
```

#### PowerShell Equivalents

```powershell
# Create user
New-AzADUser -DisplayName "John Doe" `
  -UserPrincipalName "john@contoso.com" `
  -Password (ConvertTo-SecureString "SecureP@ss123" -AsPlainText -Force)

# Create role assignment
New-AzRoleAssignment -SignInName "john@contoso.com" `
  -RoleDefinitionName "Contributor" `
  -ResourceGroupName "myRG"

# Create policy assignment
New-AzPolicyAssignment -Name "Require-Tag" `
  -PolicyDefinition $policy `
  -Scope "/subscriptions/{sub-id}"
```

### Domain 2: Implement and Manage Storage (15-20%)

#### Key Topics

| Topic | AWS Equivalent | Key Differences |
|-------|---------------|-----------------|
| Storage accounts | S3 (partial) | Account-level config |
| Blob storage | S3 | Access tiers, lifecycle |
| File shares | EFS | SMB/NFS protocols |
| Storage security | Bucket policies | SAS tokens, access keys |
| Azure File Sync | None direct | Hybrid file sync |

#### Storage Account Types

| Type | Use Case | Replication Options |
|------|----------|-------------------|
| Standard GPv2 | General purpose | LRS, ZRS, GRS, GZRS |
| Premium Block Blob | High transaction | LRS, ZRS |
| Premium File Share | Enterprise files | LRS, ZRS |
| Premium Page Blob | VM disks (legacy) | LRS |

#### Essential CLI Commands

```bash
# Create storage account
az storage account create \
  --name mystorageaccount \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2

# Create container
az storage container create \
  --name mycontainer \
  --account-name mystorageaccount \
  --auth-mode login

# Upload blob
az storage blob upload \
  --account-name mystorageaccount \
  --container-name mycontainer \
  --file ./myfile.txt \
  --name myfile.txt

# Generate SAS token
az storage account generate-sas \
  --account-name mystorageaccount \
  --permissions rwdlac \
  --services b \
  --resource-types sco \
  --expiry 2024-12-31
```

### Domain 3: Deploy and Manage Azure Compute (20-25%)

#### Key Topics

| Topic | AWS Equivalent | Exam Focus |
|-------|---------------|------------|
| Virtual Machines | EC2 | Sizes, availability |
| VM Scale Sets | Auto Scaling Groups | Scaling rules |
| App Service | Elastic Beanstalk | Plans, deployment |
| Container Instances | Fargate (basic) | Quick containers |
| Azure Kubernetes Service | EKS | Basic management |

#### VM Size Categories

| Category | Use Case | Example |
|----------|----------|---------|
| B-series | Burstable | Dev/test, light workloads |
| D-series | General purpose | Web servers, databases |
| E-series | Memory optimized | In-memory analytics |
| F-series | Compute optimized | Batch processing |
| N-series | GPU | ML, rendering |

#### Essential CLI Commands

```bash
# Create VM
az vm create \
  --resource-group myRG \
  --name myVM \
  --image Ubuntu2204 \
  --size Standard_D2s_v3 \
  --admin-username azureuser \
  --generate-ssh-keys

# Create VMSS
az vmss create \
  --resource-group myRG \
  --name myVMSS \
  --image Ubuntu2204 \
  --instance-count 2 \
  --vm-sku Standard_D2s_v3 \
  --upgrade-policy-mode automatic

# Create App Service
az webapp create \
  --resource-group myRG \
  --plan myPlan \
  --name mywebapp \
  --runtime "DOTNET:6.0"

# Deploy to App Service
az webapp deploy \
  --resource-group myRG \
  --name mywebapp \
  --src-path ./app.zip \
  --type zip
```

### Domain 4: Implement and Manage Virtual Networking (15-20%)

#### Key Topics

| Topic | AWS Equivalent | Key Differences |
|-------|---------------|-----------------|
| Virtual Networks | VPC | Address space, subnets |
| NSGs | Security Groups | Subnet or NIC level |
| VNet Peering | VPC Peering | Non-transitive by default |
| VPN Gateway | VPN Gateway | SKU-based pricing |
| Load Balancer | NLB/ALB | Standard vs Basic |
| Application Gateway | ALB | Layer 7, WAF |

#### Network Security Group Rules

```bash
# Create NSG
az network nsg create \
  --resource-group myRG \
  --name myNSG

# Add rule - allow SSH
az network nsg rule create \
  --resource-group myRG \
  --nsg-name myNSG \
  --name AllowSSH \
  --priority 1000 \
  --destination-port-ranges 22 \
  --access Allow \
  --protocol Tcp

# Associate with subnet
az network vnet subnet update \
  --resource-group myRG \
  --vnet-name myVNet \
  --name mySubnet \
  --network-security-group myNSG
```

#### Load Balancer Configuration

```bash
# Create public load balancer
az network lb create \
  --resource-group myRG \
  --name myLB \
  --sku Standard \
  --frontend-ip-name myFrontEnd \
  --backend-pool-name myBackendPool \
  --public-ip-address myPublicIP

# Create health probe
az network lb probe create \
  --resource-group myRG \
  --lb-name myLB \
  --name myHealthProbe \
  --protocol tcp \
  --port 80

# Create load balancing rule
az network lb rule create \
  --resource-group myRG \
  --lb-name myLB \
  --name myHTTPRule \
  --frontend-ip-name myFrontEnd \
  --backend-pool-name myBackendPool \
  --probe-name myHealthProbe \
  --protocol tcp \
  --frontend-port 80 \
  --backend-port 80
```

### Domain 5: Monitor and Maintain Azure Resources (10-15%)

#### Key Topics

| Topic | AWS Equivalent | Exam Focus |
|-------|---------------|------------|
| Azure Monitor | CloudWatch | Metrics, alerts |
| Log Analytics | CloudWatch Logs | KQL queries |
| Azure Backup | AWS Backup | Policies, recovery |
| Azure Site Recovery | DRS | DR setup |

#### Essential KQL Queries

```kusto
// Find failed operations
AzureActivity
| where ActivityStatus == "Failed"
| summarize count() by ResourceGroup, OperationName
| order by count_ desc

// VM heartbeat check
Heartbeat
| summarize LastHeartbeat = max(TimeGenerated) by Computer
| where LastHeartbeat < ago(5m)

// Storage account transactions
StorageAccountLogs
| where TimeGenerated > ago(1h)
| summarize Transactions = count() by bin(TimeGenerated, 5m)
| render timechart

// Alert on high CPU
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where CounterValue > 90
| summarize AggregatedValue = avg(CounterValue) by bin(TimeGenerated, 5m), Computer
```

#### Backup Configuration

```bash
# Create Recovery Services vault
az backup vault create \
  --resource-group myRG \
  --name myVault \
  --location eastus

# Enable backup for VM
az backup protection enable-for-vm \
  --resource-group myRG \
  --vault-name myVault \
  --vm myVM \
  --policy-name DefaultPolicy

# Trigger backup
az backup protection backup-now \
  --resource-group myRG \
  --vault-name myVault \
  --container-name myVM \
  --item-name myVM \
  --retain-until 31-12-2024
```

## Practice Labs

### Lab 1: Identity and Governance (2 hours)
1. Create Azure AD users and groups
2. Configure RBAC with custom role
3. Create and assign Azure Policy
4. Set up resource locks

### Lab 2: Storage (1.5 hours)
1. Create storage account with different tiers
2. Configure blob lifecycle management
3. Set up Azure File Sync
4. Implement SAS tokens

### Lab 3: Compute (2 hours)
1. Deploy VM with availability zones
2. Create and configure VMSS
3. Deploy App Service with slots
4. Configure container instances

### Lab 4: Networking (2 hours)
1. Create hub-spoke VNet topology
2. Configure VNet peering
3. Set up NSGs with application rules
4. Deploy Application Gateway with WAF

### Lab 5: Monitoring (1.5 hours)
1. Configure Azure Monitor alerts
2. Create Log Analytics workspace
3. Write KQL queries
4. Set up Azure Backup

## Study Checklist

### Week 1: Identity & Governance
- [ ] Azure AD user/group management
- [ ] RBAC roles and assignments
- [ ] Azure Policy basics
- [ ] Management groups and subscriptions
- [ ] Lab: Configure RBAC

### Week 2: Storage
- [ ] Storage account types
- [ ] Blob, file, queue, table storage
- [ ] Storage security and networking
- [ ] Replication options
- [ ] Lab: Storage configuration

### Week 3: Compute
- [ ] VM creation and management
- [ ] Availability sets and zones
- [ ] VM Scale Sets
- [ ] App Service configuration
- [ ] Lab: Deploy scalable app

### Week 4: Networking
- [ ] VNet and subnet design
- [ ] NSG configuration
- [ ] Load balancing options
- [ ] VPN and ExpressRoute basics
- [ ] Lab: Network topology

### Week 5: Monitoring & Review
- [ ] Azure Monitor configuration
- [ ] Log Analytics and KQL
- [ ] Backup and recovery
- [ ] Full practice exam
- [ ] Review weak areas

---

*Continue to [Certification Tips](certification-tips.md)*

*Back to [Certifications Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
