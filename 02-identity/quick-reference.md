# Identity Quick Reference

## One-Page Cheat Sheet

### Entra ID vs AWS IAM

| Concept | AWS | Azure | Command |
|---------|-----|-------|---------|
| Create User | `aws iam create-user` | `az ad user create` | Entra ID |
| List Users | `aws iam list-users` | `az ad user list` | Entra ID |
| Create Group | `aws iam create-group` | `az ad group create` | Entra ID |
| Service Identity | IAM Role + Trust Policy | Managed Identity | No credentials |
| App Identity | IAM User + Access Keys | Service Principal | App Registration |
| Get Current Identity | `aws sts get-caller-identity` | `az ad signed-in-user show` | Entra ID |

### Identity Types

```
┌────────────────────────────────────────────────────────────────────────┐
│                         ENTRA ID IDENTITIES                            │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  USERS                           WORKLOAD IDENTITIES                   │
│  ─────                           ──────────────────                    │
│  ┌─────────────┐                 ┌─────────────────────────────────┐   │
│  │ Member User │                 │ Service Principal               │   │
│  │ (Internal)  │                 │ • App Registration creates one  │   │
│  └─────────────┘                 │ • Has credentials (secret/cert) │   │
│                                  └─────────────────────────────────┘   │
│  ┌─────────────┐                 ┌─────────────────────────────────┐   │
│  │ Guest User  │                 │ Managed Identity                │   │
│  │ (External)  │                 │ • System-assigned (tied to resource)│
│  └─────────────┘                 │ • User-assigned (standalone)    │   │
│                                  │ • NO credentials to manage      │   │
│                                  └─────────────────────────────────┘   │
└────────────────────────────────────────────────────────────────────────┘
```

### When to Use What

| Scenario | Use This | Why |
|----------|----------|-----|
| Azure VM accessing Key Vault | System-assigned Managed Identity | Automatic lifecycle, no secrets |
| Multiple resources sharing identity | User-assigned Managed Identity | Reusable, centralized |
| Third-party app integration | Service Principal | External apps need credentials |
| DevOps pipeline (Azure DevOps) | Service Principal / Workload Identity | Pipeline authentication |
| DevOps pipeline (GitHub Actions) | Workload Identity Federation | No secrets, OIDC tokens |
| Cross-tenant access | Service Principal + Guest | B2B collaboration |

---

## Conditional Access Quick Reference

### Policy Structure

```
┌──────────────────────────────────────────────────────────────────────┐
│                     CONDITIONAL ACCESS POLICY                         │
├──────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  ASSIGNMENTS (WHO + WHAT)                                            │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐       │
│  │ Users & Groups  │  │  Cloud Apps     │  │  Conditions     │       │
│  │ • Include       │  │  • All apps     │  │  • Locations    │       │
│  │ • Exclude       │  │  • Select apps  │  │  • Devices      │       │
│  └─────────────────┘  └─────────────────┘  │  • Risk levels  │       │
│                                            │  • Client apps  │       │
│                                            └─────────────────┘       │
│                              ↓                                        │
│  ACCESS CONTROLS (THEN WHAT)                                         │
│  ┌─────────────────────────────────────────────────────────────┐     │
│  │ Grant                           │ Session                    │     │
│  │ • Block access                  │ • App enforced restrictions│     │
│  │ • Grant access                  │ • Sign-in frequency        │     │
│  │   - Require MFA                 │ • Persistent browser       │     │
│  │   - Require compliant device    │ • Disable resilience defaults   │
│  │   - Require Hybrid Azure AD Join│                            │     │
│  │   - Require approved client app │                            │     │
│  └─────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────┘
```

### Common Policies (Copy-Paste Ready)

**1. Require MFA for All Users (Baseline)**
```
Assignment:
  Users: All users
  Exclude: Break-glass accounts
  Apps: All cloud apps
Conditions:
  (none - applies everywhere)
Grant:
  Require MFA
```

**2. Block Legacy Authentication**
```
Assignment:
  Users: All users
  Apps: All cloud apps
Conditions:
  Client apps: Exchange ActiveSync, Other clients
Grant:
  Block access
```

**3. Require Compliant Device for Sensitive Apps**
```
Assignment:
  Users: All users
  Apps: Office 365, Azure Management
Conditions:
  Device platforms: iOS, Android, Windows, macOS
Grant:
  Require device to be marked as compliant
```

**4. Block Access from Untrusted Locations**
```
Assignment:
  Users: All users
  Apps: All cloud apps
Conditions:
  Locations:
    Include: All locations
    Exclude: Named locations (office IPs)
Grant:
  Block access
  OR
  Require MFA
```

---

## RBAC Quick Reference

### Built-in Roles (Most Used)

| Role | Scope | Use Case |
|------|-------|----------|
| Owner | Any | Full control + role assignment |
| Contributor | Any | Full control, no role assignment |
| Reader | Any | Read-only access |
| User Access Administrator | Any | Manage role assignments only |
| Security Admin | Subscription | Security Center management |
| Network Contributor | Resource Group | Network management |
| Storage Blob Data Contributor | Storage Account | Blob read/write (data plane) |
| Key Vault Secrets User | Key Vault | Read secrets only |
| Virtual Machine Contributor | Resource Group | VM management |

### Role Assignment Commands

```bash
# List role assignments
az role assignment list --assignee user@domain.com

# Assign role to user
az role assignment create \
  --assignee user@domain.com \
  --role "Contributor" \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg-name}

# Assign role to Managed Identity
az role assignment create \
  --assignee {managed-identity-principal-id} \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/{sub-id}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{sa}

# Assign role to Service Principal
az role assignment create \
  --assignee {service-principal-app-id} \
  --role "Reader" \
  --scope /subscriptions/{sub-id}
```

### RBAC vs Data Plane Permissions

```
MANAGEMENT PLANE (RBAC):              DATA PLANE (Resource-specific):
────────────────────────              ──────────────────────────────

"Contributor" on Storage Account      "Storage Blob Data Contributor"
        ↓                                      ↓
Can create/delete containers          Can read/write blob DATA
CANNOT read blob data                 CANNOT create containers

Example: DevOps needs both:
• Contributor: Deploy infrastructure
• Storage Blob Data Contributor: Deploy application data
```

---

## Managed Identity Quick Reference

### System-Assigned vs User-Assigned

| Feature | System-Assigned | User-Assigned |
|---------|-----------------|---------------|
| Created | With resource | Standalone |
| Lifecycle | Tied to resource | Independent |
| Sharing | One resource only | Multiple resources |
| Use case | Single resource needs identity | Shared identity pattern |

### Enable Managed Identity

```bash
# System-assigned on VM
az vm identity assign \
  --name myVM \
  --resource-group myRG

# User-assigned (create then assign)
az identity create \
  --name myIdentity \
  --resource-group myRG

az vm identity assign \
  --name myVM \
  --resource-group myRG \
  --identities /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.ManagedIdentity/userAssignedIdentities/myIdentity
```

### Access Azure Resources with Managed Identity

```python
# Python - No credentials needed!
from azure.identity import DefaultAzureCredential
from azure.storage.blob import BlobServiceClient

credential = DefaultAzureCredential()  # Uses Managed Identity automatically
blob_service = BlobServiceClient(
    account_url="https://mystorageaccount.blob.core.windows.net",
    credential=credential
)
```

```csharp
// C# - No credentials needed!
var credential = new DefaultAzureCredential();
var blobServiceClient = new BlobServiceClient(
    new Uri("https://mystorageaccount.blob.core.windows.net"),
    credential);
```

---

## Governance Quick Reference

### Access Reviews

```bash
# Create access review (via Graph API or Portal)
# Typical settings:
- Scope: Group membership OR Application access OR Role assignment
- Reviewers: Managers, Group owners, or Self-review
- Frequency: Monthly, Quarterly, Annually
- Auto-apply results: Yes/No
- If no response: No change OR Remove access
```

### Entitlement Management

```
Access Package Structure:
┌─────────────────────────────────────────────────────────────────────┐
│                        ACCESS PACKAGE                                │
├─────────────────────────────────────────────────────────────────────┤
│  Name: "Project Alpha Access"                                        │
│                                                                      │
│  RESOURCES INCLUDED:                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐              │
│  │ Azure AD     │  │ SharePoint   │  │ Azure        │              │
│  │ Group        │  │ Site         │  │ Subscription │              │
│  │ (Member)     │  │ (Contribute) │  │ (Reader)     │              │
│  └──────────────┘  └──────────────┘  └──────────────┘              │
│                                                                      │
│  POLICIES:                                                          │
│  • Who can request: Users in directory + specific external orgs     │
│  • Approval: Manager → Project Owner (2-stage)                      │
│  • Duration: 90 days, renewable                                     │
│  • Access review: Quarterly                                         │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Troubleshooting Quick Reference

### Sign-in Failures

| Error | Likely Cause | Fix |
|-------|--------------|-----|
| AADSTS50105 | User not assigned to app | Assign user/group to enterprise app |
| AADSTS50076 | MFA required | Complete MFA or check CA policy |
| AADSTS53003 | Blocked by Conditional Access | Review CA policy conditions |
| AADSTS50034 | User not found | Check user exists in tenant |
| AADSTS700016 | App not found | Check app ID / tenant |
| AADSTS7000215 | Invalid client secret | Rotate secret |

### Diagnostic Commands

```bash
# Check user's group memberships
az ad user get-member-groups --id user@domain.com

# Check service principal permissions
az ad sp show --id {app-id}

# List app role assignments
az ad app show --id {app-id} --query "appRoles"

# Check Conditional Access policies (requires Graph)
az rest --method GET \
  --uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies"
```

### Sign-in Logs (KQL)

```kusto
// Failed sign-ins in last 24 hours
SigninLogs
| where TimeGenerated > ago(24h)
| where ResultType != 0
| project TimeGenerated, UserPrincipalName, AppDisplayName,
          ResultType, ResultDescription, ConditionalAccessStatus
| order by TimeGenerated desc
```

---

## Decision Tree: Choosing Identity Type

```
                        ┌─────────────────────────┐
                        │ Who/What needs access?  │
                        └───────────┬─────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
              ▼                     ▼                     ▼
        ┌──────────┐         ┌──────────┐         ┌──────────┐
        │  Human   │         │  Azure   │         │ External │
        │          │         │ Resource │         │   App    │
        └────┬─────┘         └────┬─────┘         └────┬─────┘
             │                    │                    │
             ▼                    ▼                    ▼
        ┌──────────┐         ┌──────────┐         ┌──────────┐
        │ Entra ID │         │ Managed  │         │ Service  │
        │   User   │         │ Identity │         │Principal │
        └──────────┘         └────┬─────┘         └──────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
              ┌──────────┐               ┌──────────┐
              │ Single   │               │ Multiple │
              │ Resource │               │ Resources│
              └────┬─────┘               └────┬─────┘
                   │                          │
                   ▼                          ▼
              ┌──────────┐               ┌──────────┐
              │ System-  │               │  User-   │
              │ Assigned │               │ Assigned │
              └──────────┘               └──────────┘
```

---

*Deep Dive: [Entra ID](01-entra-id.md)* | *Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
