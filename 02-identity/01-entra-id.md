# Microsoft Entra ID Deep Dive

## What is Entra ID?

Microsoft Entra ID (formerly Azure Active Directory) is Microsoft's cloud-based identity and access management service. It's fundamentally different from AWS IAM—it's a complete identity platform, not just a permission system.

### AWS IAM vs Entra ID Mental Model

```
AWS IAM:                               Entra ID:
────────                               ─────────

"Permission management for             "Identity platform for
 AWS resources"                         everything Microsoft + Azure"

┌─────────────────────┐                ┌─────────────────────────────────┐
│ • Users             │                │ • Users (workforce + external)  │
│ • Groups            │                │ • Groups (security + M365)      │
│ • Roles             │                │ • Applications (enterprise +    │
│ • Policies          │                │   custom)                       │
│ • (AWS-focused)     │                │ • Devices                       │
└─────────────────────┘                │ • Conditional Access            │
                                       │ • B2B/B2C                       │
                                       │ • SSO to 1000s of apps          │
                                       │ • Microsoft Graph API           │
                                       └─────────────────────────────────┘
```

---

## Core Components

### 1. Users

#### Member Users (Internal)
Full members of your organization, typically employees.

```bash
# Create a member user
az ad user create \
  --display-name "John Doe" \
  --user-principal-name john.doe@contoso.com \
  --password "TempP@ssw0rd!" \
  --force-change-password-next-sign-in true
```

#### Guest Users (External)
External collaborators from other organizations (B2B).

```bash
# Invite a guest user
az ad user create \
  --display-name "External Partner" \
  --user-principal-name partner@external.com#EXT#@contoso.onmicrosoft.com \
  --mail partner@external.com
```

#### Comparison with AWS

| Scenario | AWS Approach | Azure Approach |
|----------|--------------|----------------|
| Internal employee | IAM User | Entra ID Member User |
| External contractor | IAM User (separate account) or IAM Identity Center | Guest User (B2B) |
| Customer identity | Cognito User Pool | Entra ID B2C |
| Service account | IAM User with access keys | Service Principal or Managed Identity |

### 2. Groups

#### Security Groups
Used for access control (RBAC assignments, app access).

```bash
# Create security group
az ad group create \
  --display-name "Project-Alpha-Team" \
  --mail-nickname "project-alpha-team"

# Add member
az ad group member add \
  --group "Project-Alpha-Team" \
  --member-id {user-object-id}
```

#### Microsoft 365 Groups
Collaboration groups (Teams, SharePoint, shared mailbox).

#### Dynamic Groups
Membership based on user attributes (requires Entra ID P1/P2).

```
Dynamic membership rule examples:

// All users in Sales department
user.department -eq "Sales"

// All users with Manager title in US
(user.jobTitle -eq "Manager") and (user.country -eq "United States")

// All guest users
user.userType -eq "Guest"

// All users with specific attribute
user.extensionAttribute1 -eq "ProjectAlpha"
```

### 3. Applications

#### Enterprise Applications
Pre-integrated apps from the gallery (Salesforce, ServiceNow, etc.) or custom apps.

#### App Registrations
Applications you build that need to authenticate with Entra ID.

```
Enterprise App vs App Registration:
─────────────────────────────────────

App Registration:                    Enterprise Application:
• Developer-focused                  • Admin-focused
• Define app identity               • Instance in your tenant
• Configure permissions             • Assign users/groups
• Get client ID/secret              • Configure SSO
• API permissions                   • Conditional Access applies

When you register an app, an Enterprise App is auto-created.
```

---

## Service Principals and Managed Identities

This is where Azure fundamentally differs from AWS. In AWS, you use IAM roles. In Azure, you have two options:

### Service Principal
An identity for an application (external to Azure or needs credentials).

```bash
# Create Service Principal with password
az ad sp create-for-rbac \
  --name "my-app-sp" \
  --role "Contributor" \
  --scopes /subscriptions/{subscription-id}/resourceGroups/{rg-name}

# Output:
# {
#   "appId": "xxx",
#   "password": "xxx",  ← You must manage this secret!
#   "tenant": "xxx"
# }
```

### Managed Identity
Azure-managed identity for Azure resources. **No credentials to manage**.

```bash
# System-assigned (created with resource, deleted with resource)
az vm identity assign --name myVM --resource-group myRG

# User-assigned (standalone, can be shared)
az identity create --name myIdentity --resource-group myRG
```

### Decision Matrix: Service Principal vs Managed Identity

| Scenario | Use This | Reason |
|----------|----------|--------|
| VM accessing Key Vault | Managed Identity (System) | No secrets, auto-lifecycle |
| AKS pods accessing Azure services | Managed Identity (User) | Workload identity |
| GitHub Actions deploying to Azure | Workload Identity Federation | No secrets in GitHub |
| Jenkins deploying to Azure | Service Principal | External system needs credentials |
| On-premises app accessing Azure | Service Principal | Can't use MI outside Azure |
| Multiple resources, shared identity | Managed Identity (User) | Centralized identity |
| Terraform running locally | Service Principal | Developer workstation |
| Terraform in Azure DevOps | Managed Identity or Workload Identity | Avoid secrets |

---

## Case Studies: Beginner → Expert

### Beginner: Web App Accessing Storage

**Scenario**: A web app needs to read files from Azure Blob Storage.

**AWS Approach**:
```
EC2 Instance → IAM Role → S3 Bucket Policy
```

**Azure Approach**:
```
App Service → System-assigned Managed Identity → Storage RBAC
```

**Implementation**:

```bash
# 1. Enable managed identity on App Service
az webapp identity assign --name myWebApp --resource-group myRG

# 2. Get the principal ID
principalId=$(az webapp identity show --name myWebApp --resource-group myRG --query principalId -o tsv)

# 3. Assign Storage Blob Data Reader role
az role assignment create \
  --assignee $principalId \
  --role "Storage Blob Data Reader" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{storage}
```

**Application Code** (no credentials!):
```python
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

credential = DefaultAzureCredential()
blob_client = BlobServiceClient(
    account_url="https://mystorageaccount.blob.core.windows.net",
    credential=credential
)
```

**Architecture Decision**:
> *Why Managed Identity over Service Principal?*
> - No secrets to rotate or store
> - Identity lifecycle managed by Azure
> - Simpler security posture (no leaked credentials possible)
> - Follows Zero Trust principle of credential-less access

---

### Intermediate: Multi-Tier App with Shared Identity

**Scenario**: Multiple microservices need access to the same Azure resources (Key Vault, SQL, Storage).

**Problem with System-Assigned**: Each service has its own identity → manage multiple RBAC assignments.

**Solution**: User-Assigned Managed Identity shared across services.

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ API Service  │  │ Worker       │  │ Background   │              │
│  │ (App Service)│  │ (Container)  │  │ Job (Function)│             │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘              │
│         │                 │                 │                       │
│         └─────────────────┼─────────────────┘                       │
│                           │                                         │
│                           ▼                                         │
│              ┌────────────────────────┐                            │
│              │  User-Assigned         │                            │
│              │  Managed Identity      │                            │
│              │  "app-shared-identity" │                            │
│              └────────────┬───────────┘                            │
│                           │                                         │
│         ┌─────────────────┼─────────────────┐                      │
│         │                 │                 │                       │
│         ▼                 ▼                 ▼                       │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │  Key Vault   │  │  SQL         │  │  Storage     │              │
│  │  (Secrets)   │  │  Database    │  │  Account     │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Implementation**:

```bash
# 1. Create user-assigned managed identity
az identity create \
  --name app-shared-identity \
  --resource-group myRG

# Get the identity resource ID and principal ID
identityId=$(az identity show --name app-shared-identity --resource-group myRG --query id -o tsv)
principalId=$(az identity show --name app-shared-identity --resource-group myRG --query principalId -o tsv)

# 2. Assign to App Service
az webapp identity assign \
  --name myApiService \
  --resource-group myRG \
  --identities $identityId

# 3. Assign to Function App
az functionapp identity assign \
  --name myFunctionApp \
  --resource-group myRG \
  --identities $identityId

# 4. Single RBAC assignment covers all services
az role assignment create \
  --assignee $principalId \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{kv}
```

**Architecture Decision**:
> *Why User-Assigned over System-Assigned?*
> - Single point of RBAC management
> - Consistent identity across environments
> - Can pre-configure before deploying resources
> - Survives resource recreation (blue-green deployments)

---

### Expert: Federated Identity for GitHub Actions (No Secrets)

**Scenario**: GitHub Actions pipeline needs to deploy to Azure without storing secrets in GitHub.

**Traditional (Service Principal + Secret)**:
```yaml
# PROBLEM: Secret stored in GitHub
- uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    client-secret: ${{ secrets.AZURE_CLIENT_SECRET }}  # Security risk!
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
```

**Modern (Workload Identity Federation)**:
```yaml
# SOLUTION: No secret needed
- uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    # No client-secret! Uses OIDC token from GitHub
```

**How It Works**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  GitHub Actions                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 1. Workflow requests OIDC token from GitHub                      │   │
│  │    Token contains: repo, branch, environment claims              │   │
│  └────────────────────────────────────┬────────────────────────────┘   │
│                                       │                                  │
│                                       ▼                                  │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 2. Token sent to Entra ID                                        │   │
│  └────────────────────────────────────┬────────────────────────────┘   │
│                                       │                                  │
│                                       ▼                                  │
│  Entra ID                                                               │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 3. Validates token against federated credential config           │   │
│  │    - Issuer: https://token.actions.githubusercontent.com         │   │
│  │    - Subject: repo:org/repo:ref:refs/heads/main                  │   │
│  │ 4. If valid, issues Azure access token                           │   │
│  └────────────────────────────────────┬────────────────────────────┘   │
│                                       │                                  │
│                                       ▼                                  │
│  Azure Resources                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │ 5. Azure access token used to deploy resources                   │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└─────────────────────────────────────────────────────────────────────────┘
```

**Implementation**:

```bash
# 1. Create App Registration
az ad app create --display-name "github-actions-deploy"
appId=$(az ad app list --display-name "github-actions-deploy" --query "[0].appId" -o tsv)

# 2. Create Service Principal
az ad sp create --id $appId

# 3. Add federated credential (trust GitHub's OIDC)
az ad app federated-credential create \
  --id $appId \
  --parameters '{
    "name": "github-main-branch",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:myorg/myrepo:ref:refs/heads/main",
    "description": "GitHub Actions main branch",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# 4. Assign RBAC role
az role assignment create \
  --assignee $appId \
  --role "Contributor" \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}
```

**Architecture Decision**:
> *Why Workload Identity Federation over Service Principal Secrets?*
> - Zero secrets to manage or rotate
> - Secrets can't be leaked (they don't exist)
> - Fine-grained control (specific repo, branch, environment)
> - Follows Zero Trust principle completely
> - Audit trail of which workflows accessed what

---

## Entra ID Licensing Tiers

| Feature | Free | P1 | P2 |
|---------|------|----|----|
| Basic SSO | ✓ | ✓ | ✓ |
| MFA | ✓ | ✓ | ✓ |
| Conditional Access | - | ✓ | ✓ |
| Dynamic Groups | - | ✓ | ✓ |
| Self-service password reset | - | ✓ | ✓ |
| Identity Protection | - | - | ✓ |
| Access Reviews | - | - | ✓ |
| Entitlement Management | - | - | ✓ |
| Privileged Identity Management (PIM) | - | - | ✓ |

---

## Integration with AWS (Hybrid Scenarios)

### Scenario: Users in Entra ID, Resources in Both AWS and Azure

```
┌─────────────────────────────────────────────────────────────────────┐
│                          ENTRA ID                                    │
│                    (Identity Provider)                               │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                         Users                                 │   │
│  └──────────────────────────────┬───────────────────────────────┘   │
└─────────────────────────────────┼───────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
        ┌───────────────────────┐   ┌───────────────────────┐
        │   Azure Resources     │   │   AWS Resources       │
        │   (Native RBAC)       │   │   (IAM Identity Center│
        │                       │   │    + SAML Federation) │
        └───────────────────────┘   └───────────────────────┘
```

**Setup AWS IAM Identity Center with Entra ID**:

1. Configure SAML federation in Entra ID
2. Set up AWS IAM Identity Center with Entra ID as IdP
3. Map Entra ID groups to AWS permission sets
4. Users authenticate via Entra ID, access both clouds

---

## Common Mistakes and How to Avoid Them

| Mistake | Why It's Bad | Better Approach |
|---------|--------------|-----------------|
| Using Service Principal secrets for Azure resources | Secrets can leak, must rotate | Use Managed Identity |
| Over-permissioned Service Principals | Blast radius too large | Least privilege, specific scopes |
| Not using groups | Individual assignments don't scale | Always assign to groups |
| Hardcoding tenant/app IDs | Different per environment | Use environment variables |
| Ignoring sign-in logs | Miss security issues | Set up alerts, review regularly |
| Not using Conditional Access | Anyone with password gets in | Require MFA, device compliance |

---

*Next: [Conditional Access](02-conditional-access.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [License](../LICENSE)*
