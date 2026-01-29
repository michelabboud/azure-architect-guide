# Azure Governance Deep Dive

## Introduction

Governance in Azure is the art of enabling innovation while maintaining control. It's the guardrails that keep your organization moving fast without driving off a cliff. Done right, governance is invisible to developers yet provides complete visibility to leadership.

> *"The best governance is like a good referee—you barely notice them until something goes wrong."*

## AWS to Azure Governance Mapping

| AWS Service | Azure Equivalent | Key Differences |
|-------------|------------------|-----------------|
| Service Control Policies (SCPs) | Azure Policy | Azure Policy can also remediate non-compliant resources |
| AWS Organizations | Management Groups | Deeper policy inheritance, up to 6 levels |
| AWS Config | Azure Policy + Resource Graph | Built-in compliance, no additional service needed |
| Cost Explorer | Cost Management | Native FinOps features, deeper integration |
| AWS Budgets | Azure Budgets | Similar capabilities, better M365 integration |
| Resource Groups (tags) | Resource Groups + Tags | Tags propagate differently, policy-enforced |
| AWS Control Tower | Landing Zone Accelerator | More prescriptive, built-in security baselines |

---

## Management Groups and Subscription Organization

Management Groups are the backbone of Azure governance. They provide a hierarchical structure for managing access, policies, and compliance across multiple subscriptions.

### Management Group Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MANAGEMENT GROUP ARCHITECTURE                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Tenant Root Group (/)                                                     │
│   │                                                                         │
│   └── Contoso (Top-level MG)                ← Enterprise-wide policies      │
│       │                                                                     │
│       ├── Platform                          ← Central IT managed            │
│       │   │                                                                 │
│       │   ├── Identity                      ← AD DS, DNS                    │
│       │   │   └── [sub-identity-prod]                                       │
│       │   │                                                                 │
│       │   ├── Management                    ← Monitoring, automation        │
│       │   │   └── [sub-management-prod]                                     │
│       │   │                                                                 │
│       │   └── Connectivity                  ← Hub networking                │
│       │       └── [sub-connectivity-prod]                                   │
│       │                                                                     │
│       ├── Landing Zones                     ← Workload subscriptions        │
│       │   │                                                                 │
│       │   ├── Corp                          ← Internal applications         │
│       │   │   ├── [sub-hr-prod]                                             │
│       │   │   ├── [sub-finance-prod]                                        │
│       │   │   └── [sub-erp-prod]                                            │
│       │   │                                                                 │
│       │   └── Online                        ← Internet-facing apps          │
│       │       ├── [sub-web-prod]                                            │
│       │       └── [sub-api-prod]                                            │
│       │                                                                     │
│       ├── Sandbox                           ← Experimentation               │
│       │   └── [sub-sandbox-001]             ← Limited connectivity          │
│       │                                                                     │
│       └── Decommissioned                    ← Retired subscriptions         │
│           └── [sub-legacy-001]              ← Read-only, scheduled delete   │
│                                                                             │
│   POLICY INHERITANCE:                                                       │
│   ────────────────────                                                      │
│   Tenant Root → Contoso → Platform → Specific MG → Subscription → RG        │
│                                                                             │
│   Policies applied at parent AUTOMATICALLY apply to all children            │
│   Cannot be overridden at lower levels (unlike AWS SCPs which can be)       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Creating Management Group Structure

**Azure CLI:**

```bash
# Create the top-level management group
az account management-group create \
  --name "Contoso" \
  --display-name "Contoso Enterprise"

# Create Platform management groups
az account management-group create \
  --name "Platform" \
  --display-name "Platform" \
  --parent "Contoso"

az account management-group create \
  --name "Identity" \
  --display-name "Identity" \
  --parent "Platform"

az account management-group create \
  --name "Management" \
  --display-name "Management" \
  --parent "Platform"

az account management-group create \
  --name "Connectivity" \
  --display-name "Connectivity" \
  --parent "Platform"

# Create Landing Zone management groups
az account management-group create \
  --name "LandingZones" \
  --display-name "Landing Zones" \
  --parent "Contoso"

az account management-group create \
  --name "Corp" \
  --display-name "Corp" \
  --parent "LandingZones"

az account management-group create \
  --name "Online" \
  --display-name "Online" \
  --parent "LandingZones"

# Create Sandbox
az account management-group create \
  --name "Sandbox" \
  --display-name "Sandbox" \
  --parent "Contoso"

# Move subscription to management group
az account management-group subscription add \
  --name "Corp" \
  --subscription "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

**Bicep (Infrastructure as Code):**

```bicep
targetScope = 'tenant'

// Top-level management group
resource contosoMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'Contoso'
  properties: {
    displayName: 'Contoso Enterprise'
  }
}

// Platform management group
resource platformMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'Platform'
  properties: {
    displayName: 'Platform'
    details: {
      parent: {
        id: contosoMg.id
      }
    }
  }
}

// Identity management group
resource identityMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'Identity'
  properties: {
    displayName: 'Identity'
    details: {
      parent: {
        id: platformMg.id
      }
    }
  }
}

// Management management group
resource managementMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'Management'
  properties: {
    displayName: 'Management'
    details: {
      parent: {
        id: platformMg.id
      }
    }
  }
}

// Connectivity management group
resource connectivityMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'Connectivity'
  properties: {
    displayName: 'Connectivity'
    details: {
      parent: {
        id: platformMg.id
      }
    }
  }
}

// Landing Zones management group
resource landingZonesMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'LandingZones'
  properties: {
    displayName: 'Landing Zones'
    details: {
      parent: {
        id: contosoMg.id
      }
    }
  }
}

// Corp management group
resource corpMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'Corp'
  properties: {
    displayName: 'Corp'
    details: {
      parent: {
        id: landingZonesMg.id
      }
    }
  }
}

// Online management group
resource onlineMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'Online'
  properties: {
    displayName: 'Online'
    details: {
      parent: {
        id: landingZonesMg.id
      }
    }
  }
}

// Sandbox management group
resource sandboxMg 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'Sandbox'
  properties: {
    displayName: 'Sandbox'
    details: {
      parent: {
        id: contosoMg.id
      }
    }
  }
}
```

### Subscription Strategy Patterns

| Pattern | Description | Best For |
|---------|-------------|----------|
| **Application-centric** | One subscription per application | Large enterprises with many apps |
| **Environment-based** | Dev/Test/Prod subscriptions | Smaller organizations |
| **Business unit** | Subscriptions per department | Decentralized organizations |
| **Hybrid** | Platform + App-specific subscriptions | Most enterprise scenarios |

---

## Azure Policy Management

Azure Policy is the enforcement engine for governance. It evaluates resources against business rules and can prevent, audit, or remediate non-compliant configurations.

### Policy Effects Explained

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         AZURE POLICY EFFECTS                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   PREVENTIVE EFFECTS (Block bad configurations)                             │
│   ──────────────────────────────────────────────                            │
│                                                                             │
│   Deny         → Blocks resource creation/modification                      │
│                  "You cannot create a storage account without HTTPS"        │
│                                                                             │
│   DenyAction   → Blocks specific actions (e.g., delete)                     │
│                  "You cannot delete resources with this tag"                │
│                                                                             │
│   DETECTIVE EFFECTS (Find non-compliance)                                   │
│   ────────────────────────────────────────                                  │
│                                                                             │
│   Audit        → Logs non-compliant resources                               │
│                  "Flag VMs without backup enabled"                          │
│                                                                             │
│   AuditIfNotExists → Checks for related resources                           │
│                      "Audit VMs without diagnostic settings"                │
│                                                                             │
│   CORRECTIVE EFFECTS (Auto-remediate)                                       │
│   ────────────────────────────────────                                      │
│                                                                             │
│   DeployIfNotExists → Creates missing related resources                     │
│                       "Deploy diagnostic settings to all VMs"               │
│                                                                             │
│   Modify       → Changes existing resource properties                       │
│                  "Add required tags to resources"                           │
│                                                                             │
│   Append       → Adds properties during creation                            │
│                  "Append security rules to NSGs"                            │
│                                                                             │
│   INFORMATIONAL EFFECTS                                                     │
│   ──────────────────────                                                    │
│                                                                             │
│   Disabled     → Policy exists but not evaluated                            │
│                  Useful for testing or temporary bypass                     │
│                                                                             │
│   Manual       → Requires manual attestation                                │
│                  "Confirm SOC 2 compliance annually"                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Essential Policy Examples

**1. Require Tags on All Resources:**

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "anyOf": [
        {
          "field": "tags['Environment']",
          "exists": "false"
        },
        {
          "field": "tags['CostCenter']",
          "exists": "false"
        },
        {
          "field": "tags['Owner']",
          "exists": "false"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {}
}
```

**2. Allowed Locations (Data Sovereignty):**

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "location",
          "notIn": "[parameters('allowedLocations')]"
        },
        {
          "field": "location",
          "notEquals": "global"
        }
      ]
    },
    "then": {
      "effect": "deny"
    }
  },
  "parameters": {
    "allowedLocations": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed locations",
        "description": "The list of locations that can be specified when deploying resources."
      },
      "defaultValue": ["eastus", "westus2", "centralus"]
    }
  }
}
```

**3. Deny Public IP Addresses:**

```json
{
  "mode": "All",
  "policyRule": {
    "if": {
      "field": "type",
      "equals": "Microsoft.Network/publicIPAddresses"
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

**4. Deploy Diagnostic Settings (DeployIfNotExists):**

```bicep
targetScope = 'managementGroup'

resource policyDefinition 'Microsoft.Authorization/policyDefinitions@2021-06-01' = {
  name: 'deploy-diagnostics-keyvault'
  properties: {
    displayName: 'Deploy Diagnostic Settings for Key Vault'
    description: 'Deploys diagnostic settings for Key Vault to a Log Analytics workspace'
    policyType: 'Custom'
    mode: 'Indexed'
    parameters: {
      logAnalyticsWorkspaceId: {
        type: 'String'
        metadata: {
          displayName: 'Log Analytics Workspace ID'
          description: 'The resource ID of the Log Analytics workspace'
        }
      }
    }
    policyRule: {
      if: {
        field: 'type'
        equals: 'Microsoft.KeyVault/vaults'
      }
      then: {
        effect: 'deployIfNotExists'
        details: {
          type: 'Microsoft.Insights/diagnosticSettings'
          name: 'setByPolicy'
          existenceCondition: {
            allOf: [
              {
                field: 'Microsoft.Insights/diagnosticSettings/logs.enabled'
                equals: 'true'
              }
              {
                field: 'Microsoft.Insights/diagnosticSettings/workspaceId'
                equals: '[parameters(\'logAnalyticsWorkspaceId\')]'
              }
            ]
          }
          roleDefinitionIds: [
            '/providers/Microsoft.Authorization/roleDefinitions/749f88d5-cbae-40b8-bcfc-e573ddc772fa' // Monitoring Contributor
            '/providers/Microsoft.Authorization/roleDefinitions/92aaf0da-9dab-42b6-94a3-d43ce8d16293' // Log Analytics Contributor
          ]
          deployment: {
            properties: {
              mode: 'incremental'
              template: {
                '$schema': 'https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#'
                contentVersion: '1.0.0.0'
                parameters: {
                  resourceName: {
                    type: 'string'
                  }
                  logAnalyticsWorkspaceId: {
                    type: 'string'
                  }
                  location: {
                    type: 'string'
                  }
                }
                resources: [
                  {
                    type: 'Microsoft.KeyVault/vaults/providers/diagnosticSettings'
                    apiVersion: '2021-05-01-preview'
                    name: '[concat(parameters(\'resourceName\'), \'/Microsoft.Insights/setByPolicy\')]'
                    location: '[parameters(\'location\')]'
                    properties: {
                      workspaceId: '[parameters(\'logAnalyticsWorkspaceId\')]'
                      logs: [
                        {
                          category: 'AuditEvent'
                          enabled: true
                        }
                        {
                          category: 'AzurePolicyEvaluationDetails'
                          enabled: true
                        }
                      ]
                      metrics: [
                        {
                          category: 'AllMetrics'
                          enabled: true
                        }
                      ]
                    }
                  }
                ]
              }
              parameters: {
                resourceName: {
                  value: '[field(\'name\')]'
                }
                logAnalyticsWorkspaceId: {
                  value: '[parameters(\'logAnalyticsWorkspaceId\')]'
                }
                location: {
                  value: '[field(\'location\')]'
                }
              }
            }
          }
        }
      }
    }
  }
}
```

### Policy Initiatives (Policy Sets)

Group related policies into initiatives for easier management:

```json
{
  "name": "security-baseline",
  "displayName": "Security Baseline Initiative",
  "description": "Foundational security policies for all landing zones",
  "policyDefinitions": [
    {
      "policyDefinitionReferenceId": "RequireHTTPS",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/404c3081-a854-4457-ae30-26a93ef643f9",
      "parameters": {}
    },
    {
      "policyDefinitionReferenceId": "DenyPublicIP",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/custom-deny-public-ip",
      "parameters": {}
    },
    {
      "policyDefinitionReferenceId": "RequireEncryption",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/0a914e76-4921-4c19-b460-a2d36003525a",
      "parameters": {}
    },
    {
      "policyDefinitionReferenceId": "RequireTags",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/custom-require-tags",
      "parameters": {}
    }
  ]
}
```

### Assigning Policies via CLI

```bash
# Assign a built-in policy
az policy assignment create \
  --name "allowed-locations" \
  --display-name "Allowed Locations" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c" \
  --scope "/providers/Microsoft.Management/managementGroups/LandingZones" \
  --params '{"listOfAllowedLocations": {"value": ["eastus", "westus2"]}}'

# Assign a policy initiative
az policy assignment create \
  --name "security-baseline" \
  --display-name "Security Baseline" \
  --policy-set-definition "security-baseline" \
  --scope "/providers/Microsoft.Management/managementGroups/Contoso" \
  --mi-system-assigned \
  --location "eastus"

# Create remediation task for DeployIfNotExists policy
az policy remediation create \
  --name "remediate-diagnostic-settings" \
  --policy-assignment "deploy-diagnostics" \
  --resource-group "myRG"
```

---

## Tagging Strategies

Tags are metadata attached to resources that enable cost management, automation, and organization. A well-designed tagging strategy is essential for governance at scale.

### Tagging Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         TAGGING STRATEGY                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   REQUIRED TAGS (Enforced by Policy)                                        │
│   ──────────────────────────────────                                        │
│                                                                             │
│   Tag Name        │ Purpose                  │ Example Values               │
│   ────────────────┼──────────────────────────┼───────────────────────────── │
│   Environment     │ Lifecycle stage          │ prod, staging, dev, test     │
│   CostCenter      │ Billing attribution      │ CC-12345, IT-001             │
│   Owner           │ Responsible person/team  │ john.doe@contoso.com         │
│   Application     │ Workload identification  │ CRM, ERP, WebStore           │
│   BusinessUnit    │ Department               │ Finance, HR, Engineering     │
│                                                                             │
│   RECOMMENDED TAGS (Encouraged but not enforced)                            │
│   ──────────────────────────────────────────────                            │
│                                                                             │
│   Tag Name        │ Purpose                  │ Example Values               │
│   ────────────────┼──────────────────────────┼───────────────────────────── │
│   DataClass       │ Data sensitivity         │ Public, Internal, Confid     │
│   Criticality     │ Business importance      │ High, Medium, Low            │
│   CreatedBy       │ Who created resource     │ Terraform, Manual, Pipeline  │
│   CreatedDate     │ When created             │ 2024-01-15                   │
│   Project         │ Project code             │ PRJ-2024-001                 │
│                                                                             │
│   OPERATIONAL TAGS (System/automation use)                                  │
│   ─────────────────────────────────────────                                 │
│                                                                             │
│   Tag Name        │ Purpose                  │ Example Values               │
│   ────────────────┼──────────────────────────┼───────────────────────────── │
│   AutoShutdown    │ Schedule VM shutdown     │ true, false                  │
│   MaintenanceWin  │ Patching window          │ Sunday-02:00-06:00           │
│   BackupPolicy    │ Backup configuration     │ Daily, Weekly, None          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Tag Inheritance Behavior

Unlike AWS, Azure tags do NOT automatically inherit from parent scopes:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    TAG INHERITANCE (OR LACK THEREOF)                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   AWS BEHAVIOR:                      AZURE BEHAVIOR:                        │
│   ─────────────                      ───────────────                        │
│                                                                             │
│   Account Tags                       Subscription Tags                      │
│   └── Environment: Prod              └── Environment: Prod                  │
│       │                                  │                                  │
│       └── Resource (inherits)            └── Resource (does NOT inherit)    │
│           Environment: Prod                  Environment: <missing>         │
│                                                                             │
│   SOLUTION: Use Azure Policy to inherit tags from Resource Group            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Policy for Tag Inheritance

```json
{
  "mode": "Indexed",
  "policyRule": {
    "if": {
      "allOf": [
        {
          "field": "[concat('tags[', parameters('tagName'), ']')]",
          "exists": "false"
        },
        {
          "value": "[resourceGroup().tags[parameters('tagName')]]",
          "notEquals": ""
        }
      ]
    },
    "then": {
      "effect": "modify",
      "details": {
        "roleDefinitionIds": [
          "/providers/microsoft.authorization/roleDefinitions/b24988ac-6180-42a0-ab88-20f7382dd24c"
        ],
        "operations": [
          {
            "operation": "add",
            "field": "[concat('tags[', parameters('tagName'), ']')]",
            "value": "[resourceGroup().tags[parameters('tagName')]]"
          }
        ]
      }
    }
  },
  "parameters": {
    "tagName": {
      "type": "String",
      "metadata": {
        "displayName": "Tag Name",
        "description": "Name of the tag to inherit from resource group"
      }
    }
  }
}
```

### Bulk Tag Operations

**PowerShell: Tag All Resources in Subscription:**

```powershell
# Get all resources without required tags and add them
$requiredTags = @{
    "Environment" = "prod"
    "CostCenter" = "CC-12345"
    "Owner" = "platform-team@contoso.com"
}

Get-AzResource | ForEach-Object {
    $resource = $_
    $tags = $resource.Tags

    if ($null -eq $tags) {
        $tags = @{}
    }

    $needsUpdate = $false
    foreach ($key in $requiredTags.Keys) {
        if (-not $tags.ContainsKey($key)) {
            $tags[$key] = $requiredTags[$key]
            $needsUpdate = $true
        }
    }

    if ($needsUpdate) {
        Set-AzResource -ResourceId $resource.ResourceId -Tag $tags -Force
        Write-Host "Updated tags on $($resource.Name)"
    }
}
```

**Azure CLI: Query Untagged Resources:**

```bash
# Find resources missing required tags
az graph query -q "
Resources
| where tags['Environment'] == '' or isnull(tags['Environment'])
| project name, type, resourceGroup, subscriptionId
| order by type asc
" --output table

# Count resources by tag value
az graph query -q "
Resources
| where isnotnull(tags['Environment'])
| summarize count() by tostring(tags['Environment'])
" --output table
```

---

## Cost Management Integration

Cost Management is critical for governance—without cost visibility, cloud spending can spiral quickly.

### Cost Management Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COST MANAGEMENT ARCHITECTURE                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                        ┌─────────────────────────┐                          │
│                        │    Azure Cost           │                          │
│                        │    Management           │                          │
│                        └───────────┬─────────────┘                          │
│                                    │                                        │
│         ┌──────────────────────────┼──────────────────────────┐             │
│         │                          │                          │             │
│         ▼                          ▼                          ▼             │
│   ┌───────────────┐        ┌───────────────┐        ┌───────────────┐       │
│   │ Cost Analysis │        │   Budgets     │        │   Exports     │       │
│   │               │        │               │        │               │       │
│   │ • View costs  │        │ • Thresholds  │        │ • CSV/Parquet │       │
│   │ • Group by    │        │ • Alerts      │        │ • Storage     │       │
│   │   - Tag       │        │ • Actions     │        │ • Power BI    │       │
│   │   - Resource  │        │               │        │               │       │
│   │   - Service   │        │               │        │               │       │
│   └───────────────┘        └───────────────┘        └───────────────┘       │
│                                    │                                        │
│                                    ▼                                        │
│                        ┌─────────────────────────┐                          │
│                        │    Action Groups        │                          │
│                        │                         │                          │
│                        │ • Email alerts          │                          │
│                        │ • SMS notifications     │                          │
│                        │ • Logic App triggers    │                          │
│                        │ • Azure Function        │                          │
│                        │ • Webhook               │                          │
│                        └─────────────────────────┘                          │
│                                                                             │
│   HIERARCHY:                                                                │
│   ──────────                                                                │
│   Management Group → Subscription → Resource Group → Resource               │
│   Costs roll up through hierarchy for consolidated views                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Budget Configuration

**Azure CLI:**

```bash
# Create a budget at subscription level
az consumption budget create \
  --budget-name "Monthly-Budget-Prod" \
  --amount 50000 \
  --category Cost \
  --time-grain Monthly \
  --start-date "2024-01-01" \
  --end-date "2025-12-31" \
  --resource-group "rg-finance" \
  --subscription "prod-subscription-id"

# Create budget with notifications
az consumption budget create \
  --budget-name "Critical-Workload-Budget" \
  --amount 10000 \
  --category Cost \
  --time-grain Monthly \
  --start-date "2024-01-01" \
  --end-date "2025-12-31" \
  --notifications '{
    "Actual_GreaterThan_80_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 80,
      "contactEmails": ["finance@contoso.com", "ops@contoso.com"],
      "contactRoles": ["Owner", "Contributor"],
      "thresholdType": "Actual"
    },
    "Forecasted_GreaterThan_100_Percent": {
      "enabled": true,
      "operator": "GreaterThan",
      "threshold": 100,
      "contactEmails": ["finance@contoso.com", "cfo@contoso.com"],
      "thresholdType": "Forecasted"
    }
  }'
```

**Bicep:**

```bicep
resource budget 'Microsoft.Consumption/budgets@2023-05-01' = {
  name: 'monthly-budget'
  properties: {
    category: 'Cost'
    amount: 50000
    timeGrain: 'Monthly'
    timePeriod: {
      startDate: '2024-01-01'
      endDate: '2025-12-31'
    }
    filter: {
      dimensions: {
        name: 'ResourceGroupName'
        operator: 'In'
        values: [
          'rg-prod-*'
        ]
      }
    }
    notifications: {
      actual80Percent: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 80
        thresholdType: 'Actual'
        contactEmails: [
          'ops@contoso.com'
        ]
        contactRoles: [
          'Owner'
        ]
      }
      forecasted100Percent: {
        enabled: true
        operator: 'GreaterThan'
        threshold: 100
        thresholdType: 'Forecasted'
        contactEmails: [
          'finance@contoso.com'
        ]
      }
    }
  }
}
```

### Cost Allocation Rules

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COST ALLOCATION STRATEGIES                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   1. TAG-BASED ALLOCATION                                                   │
│   ───────────────────────                                                   │
│   Resources tagged → Costs grouped by tag → Charge to cost center           │
│                                                                             │
│   CostCenter: CC-Finance  ────► All costs rolled to Finance team            │
│   CostCenter: CC-HR       ────► All costs rolled to HR team                 │
│                                                                             │
│   2. SHARED SERVICES ALLOCATION                                             │
│   ──────────────────────────────                                            │
│   Hub networking, monitoring → Split across landing zones                   │
│                                                                             │
│   Hub VNet ($10K/month)                                                     │
│   ├── Corp LZ (60% traffic) ────► $6K charged to Corp                       │
│   ├── Online LZ (35% traffic) ──► $3.5K charged to Online                   │
│   └── Sandbox (5% traffic) ─────► $0.5K charged to Sandbox                  │
│                                                                             │
│   3. SHOWBACK vs CHARGEBACK                                                 │
│   ─────────────────────────                                                 │
│   Showback: "Here's what you used" (informational)                          │
│   Chargeback: "Here's your invoice" (actual billing)                        │
│                                                                             │
│   Most orgs start with showback → move to chargeback as FinOps matures      │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Cost Analysis Queries (KQL via Resource Graph)

```bash
# Get costs by resource group for last 30 days
az graph query -q "
ResourceContainers
| where type == 'microsoft.resources/subscriptions/resourcegroups'
| project resourceGroup=name, subscriptionId
| join kind=inner (
    Resources
    | summarize count() by resourceGroup, subscriptionId
) on resourceGroup, subscriptionId
| project resourceGroup, resourceCount=count_
| order by resourceCount desc
"

# Find expensive resources
az graph query -q "
Resources
| where type in ('microsoft.compute/virtualmachines', 'microsoft.sql/servers/databases', 'microsoft.storage/storageaccounts')
| extend sku = properties.hardwareProfile.vmSize
| project name, type, sku, resourceGroup, location
| order by type asc
"
```

---

## Security Baseline

A security baseline defines the minimum security controls that must be in place for all resources.

### Security Baseline Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    AZURE SECURITY BASELINE                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   IDENTITY                                                                  │
│   ────────                                                                  │
│   □ MFA enforced for all users                                              │
│   □ Conditional Access policies configured                                  │
│   □ PIM enabled for admin roles                                             │
│   □ Service principals have minimal permissions                             │
│   □ Managed Identities used (no stored credentials)                         │
│                                                                             │
│   NETWORK                                                                   │
│   ───────                                                                   │
│   □ No public IPs without justification                                     │
│   □ NSGs on all subnets                                                     │
│   □ Private Endpoints for PaaS services                                     │
│   □ Azure Firewall or NVA for egress filtering                              │
│   □ DDoS Protection on public workloads                                     │
│                                                                             │
│   DATA                                                                      │
│   ────                                                                      │
│   □ Encryption at rest (default enabled)                                    │
│   □ Encryption in transit (TLS 1.2+)                                        │
│   □ Customer-managed keys for sensitive data                                │
│   □ Soft delete enabled on storage/Key Vault                                │
│   □ Data classification and labeling                                        │
│                                                                             │
│   COMPUTE                                                                   │
│   ───────                                                                   │
│   □ Defender for Cloud enabled                                              │
│   □ Vulnerability scanning configured                                       │
│   □ Update management enabled                                               │
│   □ Just-in-time VM access configured                                       │
│   □ Antimalware deployed                                                    │
│                                                                             │
│   LOGGING                                                                   │
│   ───────                                                                   │
│   □ Diagnostic settings on all resources                                    │
│   □ Activity logs sent to Log Analytics                                     │
│   □ Retention period: 90+ days                                              │
│   □ Sentinel enabled for SIEM                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Security Baseline Policy Initiative

```json
{
  "name": "security-baseline-initiative",
  "displayName": "Enterprise Security Baseline",
  "description": "Enforces minimum security controls across all landing zones",
  "metadata": {
    "category": "Security"
  },
  "policyDefinitions": [
    {
      "policyDefinitionReferenceId": "StorageHttpsOnly",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/404c3081-a854-4457-ae30-26a93ef643f9"
    },
    {
      "policyDefinitionReferenceId": "SqlServerTdeEnabled",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/86a912f6-9a06-4e26-b447-11b16ba8659f"
    },
    {
      "policyDefinitionReferenceId": "KeyVaultPurgeProtection",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/0b60c0b2-2dc2-4e1c-b5c9-abbed971de53"
    },
    {
      "policyDefinitionReferenceId": "KeyVaultSoftDelete",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/1e66c121-a66a-4b1f-9b83-0fd99bf0fc2d"
    },
    {
      "policyDefinitionReferenceId": "AuditPublicNetworkAccess",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/b2982f36-99f2-4db5-8eff-283140c09693"
    },
    {
      "policyDefinitionReferenceId": "DenyClassicResources",
      "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/6c112d4e-5bc7-47ae-a041-ea2d9dccd749"
    }
  ]
}
```

### Defender for Cloud Integration

**Enable Defender Plans via CLI:**

```bash
# Enable Defender for Cloud on subscription
az security pricing create \
  --name "VirtualMachines" \
  --tier "Standard"

az security pricing create \
  --name "SqlServers" \
  --tier "Standard"

az security pricing create \
  --name "StorageAccounts" \
  --tier "Standard"

az security pricing create \
  --name "KeyVaults" \
  --tier "Standard"

az security pricing create \
  --name "Containers" \
  --tier "Standard"

# Get current security score
az security secure-score list --output table
```

**Bicep: Enable Defender for Cloud:**

```bicep
resource defenderVMs 'Microsoft.Security/pricings@2023-01-01' = {
  name: 'VirtualMachines'
  properties: {
    pricingTier: 'Standard'
    subPlan: 'P2'
  }
}

resource defenderSQL 'Microsoft.Security/pricings@2023-01-01' = {
  name: 'SqlServers'
  properties: {
    pricingTier: 'Standard'
  }
}

resource defenderStorage 'Microsoft.Security/pricings@2023-01-01' = {
  name: 'StorageAccounts'
  properties: {
    pricingTier: 'Standard'
    subPlan: 'DefenderForStorageV2'
  }
}

resource defenderKeyVault 'Microsoft.Security/pricings@2023-01-01' = {
  name: 'KeyVaults'
  properties: {
    pricingTier: 'Standard'
  }
}
```

---

## Compliance Management

Compliance ensures your Azure environment meets regulatory requirements (GDPR, HIPAA, SOC 2, etc.).

### Compliance Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    COMPLIANCE MANAGEMENT                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                     ┌──────────────────────────────┐                        │
│                     │    REGULATORY REQUIREMENTS   │                        │
│                     │                              │                        │
│                     │  • GDPR     • SOC 2         │                         │
│                     │  • HIPAA    • PCI DSS       │                         │
│                     │  • ISO 27001 • FedRAMP      │                         │
│                     └──────────────┬───────────────┘                        │
│                                    │                                        │
│                                    ▼                                        │
│   ┌────────────────────────────────────────────────────────────────────┐    │
│   │                  AZURE COMPLIANCE TOOLS                            │    │
│   │                                                                    │    │
│   │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │    │
│   │   │ Regulatory      │  │  Azure Policy   │  │   Microsoft     │    │    │
│   │   │ Compliance      │  │  Compliance     │  │   Purview       │    │    │
│   │   │ Dashboard       │  │  (Built-in      │  │   (Data         │    │    │
│   │   │                 │  │   initiatives)  │  │    Governance)  │    │    │
│   │   │ • Audit reports │  │ • NIST 800-53   │  │ • Data catalog  │    │    │
│   │   │ • Certifications│  │ • CIS           │  │ • Sensitivity   │    │    │
│   │   │ • Shared resp.  │  │ • Azure CIS     │  │   labels        │    │    │
│   │   │                 │  │ • ISO 27001     │  │ • DLP policies  │    │    │
│   │   └─────────────────┘  └─────────────────┘  └─────────────────┘    │    │
│   │                                                                    │    │
│   │   ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │    │
│   │   │ Defender for    │  │  Azure          │  │   Export &      │    │    │
│   │   │ Cloud           │  │  Blueprints     │  │   Reporting     │    │    │
│   │   │                 │  │  (deprecated    │  │                 │    │    │
│   │   │ • Secure score  │  │   but still     │  │ • Compliance    │    │    │
│   │   │ • Recommends    │  │   used)         │  │   reports       │    │    │
│   │   │ • Compliance    │  │                 │  │ • Audit logs    │    │    │
│   │   │   dashboard     │  │                 │  │ • Evidence      │    │    │
│   │   └─────────────────┘  └─────────────────┘  └─────────────────┘    │    │
│   │                                                                    │    │
│   └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│   WORKFLOW:                                                                 │
│   ─────────                                                                 │
│   1. Map regulations to Azure Policy initiatives                            │
│   2. Assign initiatives to management groups                                │
│   3. Monitor compliance through dashboards                                  │
│   4. Remediate non-compliant resources                                      │
│   5. Generate audit evidence                                                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Built-in Compliance Initiatives

| Initiative | Description | Controls |
|------------|-------------|----------|
| Azure Security Benchmark | Microsoft's cloud security baseline | 200+ controls |
| CIS Microsoft Azure Foundations | Center for Internet Security benchmark | 100+ controls |
| NIST SP 800-53 Rev. 5 | US Federal security standard | 300+ controls |
| ISO 27001:2013 | International security standard | 150+ controls |
| HIPAA HITRUST 9.2 | Healthcare data protection | 200+ controls |
| PCI DSS 3.2.1 | Payment card security | 150+ controls |
| SOC 2 Type 2 | Service organization controls | 100+ controls |

### Assigning Compliance Initiative

```bash
# Assign Azure Security Benchmark
az policy assignment create \
  --name "azure-security-benchmark" \
  --display-name "Azure Security Benchmark" \
  --policy-set-definition "/providers/Microsoft.Authorization/policySetDefinitions/1f3afdf9-d0c9-4c3d-847f-89da613e70a8" \
  --scope "/providers/Microsoft.Management/managementGroups/Contoso" \
  --enforcement-mode "Default"

# Assign NIST SP 800-53 Rev 5
az policy assignment create \
  --name "nist-800-53-r5" \
  --display-name "NIST SP 800-53 Rev. 5" \
  --policy-set-definition "/providers/Microsoft.Authorization/policySetDefinitions/179d1daa-458f-4e47-8086-2a68d0d6c38f" \
  --scope "/providers/Microsoft.Management/managementGroups/Contoso" \
  --enforcement-mode "DoNotEnforce"  # Audit-only initially
```

### Compliance Reporting

**Export Compliance State:**

```powershell
# Get compliance state for all policies
$complianceResults = Get-AzPolicyState -SubscriptionId $subscriptionId |
    Where-Object { $_.ComplianceState -eq "NonCompliant" } |
    Select-Object PolicyAssignmentName, PolicyDefinitionName, ResourceId, ComplianceState

# Export to CSV
$complianceResults | Export-Csv -Path "compliance-report.csv" -NoTypeInformation

# Get compliance summary
$summary = Get-AzPolicyStateSummary -SubscriptionId $subscriptionId
Write-Host "Compliant: $($summary.Results.NonCompliantResources)"
Write-Host "Non-Compliant: $($summary.Results.NonCompliantPolicies)"
```

---

## Resource Locks

Resource Locks prevent accidental deletion or modification of critical resources.

### Lock Types

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         RESOURCE LOCKS                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   LOCK TYPES:                                                               │
│   ───────────                                                               │
│                                                                             │
│   CanNotDelete (Delete Lock)                                                │
│   ──────────────────────────                                                │
│   • Resource CAN be read and modified                                       │
│   • Resource CANNOT be deleted                                              │
│   • Use for: Production databases, critical storage accounts                │
│                                                                             │
│   ReadOnly (Read-Only Lock)                                                 │
│   ─────────────────────────                                                 │
│   • Resource CAN be read                                                    │
│   • Resource CANNOT be modified or deleted                                  │
│   • Use for: Compliance evidence, archived resources                        │
│   • WARNING: May break some operations (e.g., storage account keys)         │
│                                                                             │
│   LOCK INHERITANCE:                                                         │
│   ─────────────────                                                         │
│                                                                             │
│   Subscription Lock                                                         │
│   └── Applies to ALL resource groups and resources                          │
│                                                                             │
│   Resource Group Lock                                                       │
│   └── Applies to ALL resources in that RG                                   │
│                                                                             │
│   Resource Lock                                                             │
│   └── Applies to THAT resource only                                         │
│                                                                             │
│   PERMISSIONS:                                                              │
│   ────────────                                                              │
│   Creating/deleting locks requires:                                         │
│   • Microsoft.Authorization/locks/* (Owner role has this)                   │
│   • Even resource owners can't delete locked resources without lock access  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Implementing Resource Locks

**Azure CLI:**

```bash
# Create a delete lock on a resource group
az lock create \
  --name "CannotDeleteProdRG" \
  --resource-group "rg-prod-critical" \
  --lock-type CanNotDelete \
  --notes "Production resource group - do not delete"

# Create a read-only lock on a specific resource
az lock create \
  --name "ReadOnlyLock" \
  --resource-group "rg-prod-critical" \
  --resource-name "stprodcriticalstorage" \
  --resource-type "Microsoft.Storage/storageAccounts" \
  --lock-type ReadOnly \
  --notes "Archive storage - read only"

# List all locks in a subscription
az lock list --output table

# Delete a lock (requires proper permissions)
az lock delete \
  --name "CannotDeleteProdRG" \
  --resource-group "rg-prod-critical"
```

**Bicep:**

```bicep
// Delete lock on resource group (deploy at RG scope)
resource rgLock 'Microsoft.Authorization/locks@2020-05-01' = {
  name: 'CannotDeleteProdRG'
  properties: {
    level: 'CanNotDelete'
    notes: 'Production resource group - do not delete'
  }
}

// Delete lock on specific resource
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' existing = {
  name: 'stprodcritical'
}

resource storageLock 'Microsoft.Authorization/locks@2020-05-01' = {
  name: 'CannotDeleteStorage'
  scope: storageAccount
  properties: {
    level: 'CanNotDelete'
    notes: 'Critical production storage - do not delete'
  }
}
```

**PowerShell:**

```powershell
# Create lock on all production resource groups
Get-AzResourceGroup | Where-Object { $_.ResourceGroupName -like "*-prod-*" } | ForEach-Object {
    $lockName = "CannotDelete-$($_.ResourceGroupName)"

    $existingLock = Get-AzResourceLock -ResourceGroupName $_.ResourceGroupName -LockName $lockName -ErrorAction SilentlyContinue

    if (-not $existingLock) {
        New-AzResourceLock `
            -LockName $lockName `
            -LockLevel CanNotDelete `
            -LockNotes "Production resource group - protected by policy" `
            -ResourceGroupName $_.ResourceGroupName `
            -Force

        Write-Host "Created lock on $($_.ResourceGroupName)"
    }
}
```

### Lock Best Practices

| Scenario | Lock Type | Level |
|----------|-----------|-------|
| Production databases | CanNotDelete | Resource |
| Production VNets | CanNotDelete | Resource |
| Hub networking | CanNotDelete | Resource Group |
| Compliance archives | ReadOnly | Resource |
| Management subscriptions | CanNotDelete | Subscription |
| Audit logs storage | ReadOnly | Resource |

---

## Governance Maturity Model

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    GOVERNANCE MATURITY LEVELS                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   LEVEL 1: INITIAL                                                          │
│   ────────────────                                                          │
│   □ No management group hierarchy                                           │
│   □ Subscriptions created ad-hoc                                            │
│   □ No tagging standards                                                    │
│   □ No Azure Policy                                                         │
│   □ Cost management not configured                                          │
│   Status: Chaos. Cloud spending unknown. Security posture unknown.          │
│                                                                             │
│   LEVEL 2: DEVELOPING                                                       │
│   ───────────────────                                                       │
│   □ Basic management groups exist                                           │
│   □ Some tagging in place (not enforced)                                    │
│   □ Audit-only policies assigned                                            │
│   □ Budget alerts configured                                                │
│   □ Defender for Cloud enabled (free tier)                                  │
│   Status: Visibility improving. Still reactive.                             │
│                                                                             │
│   LEVEL 3: DEFINED                                                          │
│   ─────────────────                                                         │
│   □ Full management group hierarchy                                         │
│   □ Tagging enforced via policy                                             │
│   □ Security baseline policies enforced                                     │
│   □ Compliance initiatives assigned                                         │
│   □ Resource locks on critical resources                                    │
│   □ Cost allocation by cost center                                          │
│   Status: Proactive governance. Compliance measurable.                      │
│                                                                             │
│   LEVEL 4: MANAGED                                                          │
│   ─────────────────                                                         │
│   □ Subscription vending automated                                          │
│   □ Policy remediation automated                                            │
│   □ Compliance reports generated automatically                              │
│   □ FinOps practices in place (showback/chargeback)                         │
│   □ Security posture continuously monitored                                 │
│   Status: Governance is automated. Teams self-service.                      │
│                                                                             │
│   LEVEL 5: OPTIMIZING                                                       │
│   ────────────────────                                                      │
│   □ AI-driven cost optimization                                             │
│   □ Predictive compliance                                                   │
│   □ GitOps for policy management                                            │
│   □ Continuous compliance validation                                        │
│   □ Zero-touch governance                                                   │
│   Status: Governance enables innovation. Security and cost optimized.       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Governance Assessment Checklist

```
SELF-ASSESSMENT: Where is your organization?

STRUCTURE
□ Do you have a management group hierarchy? (Level 2+)
□ Is the hierarchy aligned with CAF recommendations? (Level 3+)
□ Is subscription creation automated? (Level 4+)

POLICY
□ Are audit policies assigned? (Level 2+)
□ Are deny policies enforced? (Level 3+)
□ Is policy remediation automated? (Level 4+)

TAGGING
□ Do you have a tagging standard? (Level 2+)
□ Are required tags enforced by policy? (Level 3+)
□ Is tag inheritance configured? (Level 3+)

COST
□ Are budgets configured with alerts? (Level 2+)
□ Can you attribute costs to business units? (Level 3+)
□ Do you have chargeback processes? (Level 4+)

SECURITY
□ Is Defender for Cloud enabled? (Level 2+)
□ Are security baseline policies enforced? (Level 3+)
□ Is compliance continuously monitored? (Level 4+)

COMPLIANCE
□ Are compliance initiatives assigned? (Level 3+)
□ Can you generate audit reports? (Level 3+)
□ Is compliance reporting automated? (Level 4+)
```

---

## Quick Reference: Governance Commands

### Policy Commands

```bash
# List all policy assignments
az policy assignment list --scope "/providers/Microsoft.Management/managementGroups/Contoso"

# Get compliance summary
az policy state summarize --management-group "Contoso"

# Trigger policy evaluation
az policy state trigger-scan --resource-group "myRG"

# Create remediation task
az policy remediation create \
  --name "remediate-tags" \
  --policy-assignment "require-tags" \
  --resource-group "myRG"
```

### Cost Management Commands

```bash
# Query costs for current month
az consumption usage list \
  --start-date "2024-01-01" \
  --end-date "2024-01-31" \
  --query "[].{Name:instanceName, Cost:pretaxCost}" \
  --output table

# List budgets
az consumption budget list --output table

# Get cost forecast
az costmanagement forecast show \
  --scope "/subscriptions/{subscription-id}" \
  --type "ActualCost" \
  --timeframe "MonthToDate"
```

### Lock Commands

```bash
# List all locks
az lock list --output table

# Create lock
az lock create --name "ProdLock" --resource-group "rg-prod" --lock-type CanNotDelete

# Delete lock
az lock delete --name "ProdLock" --resource-group "rg-prod"
```

---

*Continue to [Quick Reference](quick-reference.md)*

*Back to [Landing Zones](02-landing-zones.md)*

*Back to [Chapter Overview](README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
