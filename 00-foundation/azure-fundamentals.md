# Azure Fundamentals for AWS Architects

## Core Concepts: The Azure Mental Model

### The Azure Resource Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ENTRA ID TENANT                                    │
│  (Your identity boundary - like AWS Organization but for identity)          │
│                                                                              │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                      MANAGEMENT GROUPS                                 │  │
│  │  (Organize subscriptions - like AWS OUs)                              │  │
│  │                                                                        │  │
│  │  ┌─────────────────────────────────────────────────────────────────┐  │  │
│  │  │                     SUBSCRIPTIONS                                │  │  │
│  │  │  (Billing + access boundary - like AWS Accounts)                │  │  │
│  │  │                                                                  │  │  │
│  │  │  ┌───────────────────────────────────────────────────────────┐  │  │  │
│  │  │  │                 RESOURCE GROUPS                            │  │  │  │
│  │  │  │  (Logical containers - NO AWS equivalent)                  │  │  │  │
│  │  │  │                                                            │  │  │  │
│  │  │  │  ┌─────────────────────────────────────────────────────┐  │  │  │  │
│  │  │  │  │                   RESOURCES                          │  │  │  │  │
│  │  │  │  │  (VMs, Storage, Databases, etc.)                    │  │  │  │  │
│  │  │  │  └─────────────────────────────────────────────────────┘  │  │  │  │
│  │  │  └───────────────────────────────────────────────────────────┘  │  │  │
│  │  └─────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### AWS to Azure Hierarchy Mapping

| AWS Concept | Azure Concept | Key Difference |
|-------------|---------------|----------------|
| Organization | Entra ID Tenant | Tenant is identity-centric |
| Organizational Unit (OU) | Management Group | Policy inheritance, not account grouping |
| Account | Subscription | Subscription = billing + access boundary |
| (No equivalent) | Resource Group | Lifecycle management container |
| Resource | Resource | Same concept |
| Region | Region | Same concept |
| Availability Zone | Availability Zone | Same concept |

---

## Authentication vs Authorization

### The Azure Model: Separation of Concerns

```
┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│   WHO ARE YOU?              CAN YOU COME IN?           WHAT CAN YOU DO?    │
│   ─────────────             ────────────────           ─────────────────    │
│                                                                             │
│   ┌─────────────┐          ┌─────────────────┐        ┌─────────────────┐  │
│   │  Entra ID   │    →     │  Conditional    │   →    │      RBAC       │  │
│   │             │          │  Access         │        │                 │  │
│   │ • Users     │          │                 │        │ • Role          │  │
│   │ • Groups    │          │ • Device state  │        │   Assignments   │  │
│   │ • Service   │          │ • Location      │        │ • Role          │  │
│   │   Principals│          │ • Risk level    │        │   Definitions   │  │
│   │ • Managed   │          │ • App           │        │ • Scopes        │  │
│   │   Identities│          │ • MFA required  │        │                 │  │
│   └─────────────┘          └─────────────────┘        └─────────────────┘  │
│                                                                             │
│   AUTHENTICATION            CONDITIONAL AUTH          AUTHORIZATION         │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

### Comparison with AWS

```
AWS Model (Combined):                   Azure Model (Separated):
─────────────────────                   ───────────────────────

┌─────────────────────┐                 ┌─────────────────────┐
│       IAM           │                 │      Entra ID       │ ← WHO
├─────────────────────┤                 └──────────┬──────────┘
│ Users/Roles/Policies│                            ↓
│ (all in one)        │                 ┌─────────────────────┐
└─────────────────────┘                 │ Conditional Access  │ ← SHOULD THEY
                                        └──────────┬──────────┘
                                                   ↓
                                        ┌─────────────────────┐
                                        │       RBAC          │ ← WHAT CAN THEY DO
                                        └──────────┬──────────┘
                                                   ↓
                                        ┌─────────────────────┐
                                        │    Azure Policy     │ ← HOW MUST IT BE
                                        └─────────────────────┘
```

---

## Azure RBAC Deep Dive

### RBAC Components

```
Role Assignment = Security Principal + Role Definition + Scope
                  ─────────────────   ───────────────   ─────
                  WHO                 WHAT              WHERE
```

### Scope Hierarchy

```
                    Management Group
                          │
                          ↓
                    ┌─────────────┐
                    │ Subscription│
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │Resource Group│
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │  Resource   │
                    └─────────────┘

Inheritance: Permissions assigned at a higher scope are inherited by lower scopes
```

### Built-in Roles (Key Ones to Know)

| Role | What It Does | AWS Equivalent |
|------|--------------|----------------|
| **Owner** | Full access + can assign roles | AdministratorAccess |
| **Contributor** | Full access, cannot assign roles | PowerUserAccess |
| **Reader** | View only | ReadOnlyAccess |
| **User Access Administrator** | Manage user access only | IAMFullAccess |
| **Security Admin** | Security Center permissions | SecurityAudit |
| **Network Contributor** | Manage networks | NetworkAdministrator |
| **Storage Blob Data Contributor** | Read/write blob data | S3FullAccess |

### RBAC vs AWS IAM Policies

```
AWS IAM Policy:                        Azure RBAC:
───────────────                        ──────────

{                                      Role Definition:
  "Effect": "Allow",                   {
  "Action": [                            "Name": "Custom Role",
    "s3:GetObject",                      "Actions": [
    "s3:PutObject"                         "Microsoft.Storage/*/read",
  ],                                       "Microsoft.Storage/*/write"
  "Resource": "arn:aws:s3:::bucket/*"    ],
}                                        "NotActions": [],
                                         "AssignableScopes": [
                                           "/subscriptions/xxx"
                                         ]
                                       }

                                       Role Assignment:
                                       {
                                         "Principal": "user-id",
                                         "Role": "Custom Role",
                                         "Scope": "/subscriptions/xxx/
                                                   resourceGroups/rg"
                                       }
```

---

## Azure Policy vs AWS Config/SCPs

### Azure Policy Capabilities

| Effect | What It Does | AWS Equivalent |
|--------|--------------|----------------|
| **Deny** | Block non-compliant resources | SCP Deny |
| **Audit** | Log non-compliance | Config Rules |
| **Append** | Add properties to resources | (No equivalent) |
| **Modify** | Change resource properties | (No equivalent) |
| **DeployIfNotExists** | Auto-remediate | Config + Lambda |
| **AuditIfNotExists** | Audit missing resources | Config Rules |

### Policy Example: Require Tags

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "field": "[concat('tags[', parameters('tagName'), ']')]",
      "exists": "false"
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {
    "tagName": {
      "type": "String",
      "metadata": {
        "description": "Name of the tag to require"
      }
    }
  }
}
```

### AWS Equivalent (SCP + Config)

```json
// SCP (preventive)
{
  "Effect": "Deny",
  "Action": "*",
  "Resource": "*",
  "Condition": {
    "Null": {
      "aws:RequestTag/Environment": "true"
    }
  }
}

// Config Rule (detective)
// Requires separate Lambda function
```

---

## Networking Fundamentals

### VNet vs VPC

| Feature | AWS VPC | Azure VNet |
|---------|---------|------------|
| Default connectivity | Isolated by default | Isolated by default |
| Subnets | AZ-specific | Span all AZs in region |
| Internet access | Requires IGW | Implicit (with public IP) |
| Private connectivity | NAT Gateway | NAT Gateway |
| DNS | Route 53 Resolver | Azure DNS (built-in) |
| Security | Security Groups + NACLs | NSGs only |

### Subnet Design Difference

```
AWS VPC:                               Azure VNet:
────────                               ──────────

Region: us-east-1                      Region: East US
┌─────────────────────────────┐        ┌─────────────────────────────┐
│          VPC                │        │          VNet               │
│  ┌──────────┐ ┌──────────┐  │        │  ┌───────────────────────┐  │
│  │ Subnet   │ │ Subnet   │  │        │  │       Subnet          │  │
│  │ AZ-1a    │ │ AZ-1b    │  │        │  │  (Spans all AZs)      │  │
│  │          │ │          │  │        │  │                       │  │
│  │ 10.0.1.0 │ │ 10.0.2.0 │  │        │  │    10.0.1.0/24        │  │
│  │ /24      │ │ /24      │  │        │  │                       │  │
│  └──────────┘ └──────────┘  │        │  │  [AZ1] [AZ2] [AZ3]    │  │
└─────────────────────────────┘        │  └───────────────────────┘  │
                                       └─────────────────────────────┘

AWS: Create subnet per AZ             Azure: Subnet spans AZs,
for high availability                 use Availability Zones in
                                      resource deployment
```

### Network Security Groups (NSGs)

```
AWS Security Group:                    Azure NSG:
───────────────────                    ──────────

• Attached to ENI (instance)           • Attached to NIC or Subnet
• Stateful                             • Stateful
• Allow rules only                     • Allow + Deny rules
• No explicit deny                     • Priority-based (100-4096)
• Implicit deny all                    • Default rules included

AWS SG Rule:                           Azure NSG Rule:
{                                      {
  "IpProtocol": "tcp",                   "name": "AllowHTTPS",
  "FromPort": 443,                       "priority": 100,
  "ToPort": 443,                         "direction": "Inbound",
  "CidrIp": "10.0.0.0/8"                 "access": "Allow",
}                                        "protocol": "Tcp",
                                         "sourcePortRange": "*",
                                         "destinationPortRange": "443",
                                         "sourceAddressPrefix": "10.0.0.0/8",
                                         "destinationAddressPrefix": "*"
                                       }
```

---

## Resource Naming and Tagging

### Azure Naming Conventions

```
Pattern: {resource-type}-{workload}-{environment}-{region}-{instance}

Examples:
• vm-web-prod-eastus-001          (Virtual Machine)
• st-data-prod-eastus             (Storage Account - no hyphens allowed)
• vnet-hub-prod-eastus            (Virtual Network)
• rg-app-prod-eastus              (Resource Group)
• kv-secrets-prod-eastus          (Key Vault)
```

### Tagging Strategy

```
Recommended Tags:
┌─────────────────┬─────────────────────────────────────────┐
│ Tag             │ Purpose                                 │
├─────────────────┼─────────────────────────────────────────┤
│ Environment     │ prod, staging, dev, test                │
│ Owner           │ Team or individual responsible          │
│ CostCenter      │ Billing allocation                      │
│ Project         │ Project or application name             │
│ DataClass       │ public, internal, confidential, secret  │
│ Compliance      │ pci, hipaa, sox                         │
│ CreatedBy       │ IaC pipeline or manual                  │
│ CreatedDate     │ Resource creation date                  │
└─────────────────┴─────────────────────────────────────────┘
```

---

## Azure CLI vs AWS CLI

### Common Operations Comparison

| Operation | AWS CLI | Azure CLI |
|-----------|---------|-----------|
| List VMs | `aws ec2 describe-instances` | `az vm list` |
| Create VM | `aws ec2 run-instances` | `az vm create` |
| List storage | `aws s3 ls` | `az storage account list` |
| Create storage | `aws s3 mb s3://bucket` | `az storage account create` |
| Get identity | `aws sts get-caller-identity` | `az account show` |
| Switch context | `aws configure --profile` | `az account set` |
| List regions | `aws ec2 describe-regions` | `az account list-locations` |

### Azure CLI Structure

```bash
az <service> <subcommand> <action> --parameters

# Examples:
az vm create --name myVM --resource-group myRG --image Ubuntu2204
az storage account create --name mystorageacct --resource-group myRG
az network vnet create --name myVNet --resource-group myRG
az ad user list  # Entra ID commands
az role assignment create --assignee user@domain.com --role "Reader"
```

### Resource Group Context

```bash
# AWS: Resources are region-scoped
aws ec2 describe-instances --region us-east-1

# Azure: Resources are resource-group-scoped
az vm list --resource-group myRG

# Set default resource group
az configure --defaults group=myRG

# Then commands don't need --resource-group
az vm list
```

---

## Key Architectural Differences Summary

| Concept | AWS Approach | Azure Approach |
|---------|--------------|----------------|
| **Identity** | IAM is permission-focused | Entra ID is identity-platform |
| **Access Control** | IAM Policies attached to resources/principals | RBAC separate from identity |
| **Conditional Access** | Limited (IP conditions in policies) | Full policy engine |
| **Governance** | SCPs + Config Rules | Azure Policy (richer effects) |
| **Network Security** | SGs (allow) + NACLs (allow/deny) | NSGs only (allow/deny) |
| **Resource Organization** | Account = boundary | Subscription + Resource Group |
| **Compliance** | Multiple tools (Config, Macie, etc.) | Unified in Purview |
| **Security Platform** | Multiple tools (GuardDuty, Inspector, etc.) | Unified in Defender |

---

*Next: [Chapter 01: Identity](../01-identity/README.md)* | *Back to [Foundation Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
