# Infrastructure as Code Case Studies

## Case Study 1: Enterprise Landing Zone Deployment

### Scenario

**Company**: Fortune 500 manufacturing company
**Challenge**: Migrate 200+ applications from on-premises to Azure with consistent governance
**Goal**: Automated, repeatable infrastructure deployment across multiple subscriptions

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE LANDING ZONE ARCHITECTURE                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  MANAGEMENT GROUP HIERARCHY:                                                │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                     ROOT MANAGEMENT GROUP                            │   │
│  │                                                                       │   │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │   │
│  │  │    Platform     │  │   Landing Zones │  │    Sandbox      │     │   │
│  │  │                 │  │                 │  │                 │     │   │
│  │  │  ┌───────────┐  │  │  ┌───────────┐  │  │  • Dev testing  │     │   │
│  │  │  │ Identity  │  │  │  │   Corp    │  │  │  • No prod     │     │   │
│  │  │  │ (Entra ID)│  │  │  │           │  │  │    connectivity │     │   │
│  │  │  ├───────────┤  │  │  │  ┌─────┐  │  │  │                 │     │   │
│  │  │  │Management │  │  │  │  │ BU1 │  │  │  └─────────────────┘     │   │
│  │  │  │(Monitoring│  │  │  │  │ BU2 │  │  │                          │   │
│  │  │  │ Logging)  │  │  │  │  │ BU3 │  │  │                          │   │
│  │  │  ├───────────┤  │  │  │  └─────┘  │  │                          │   │
│  │  │  │Connectivity│ │  │  ├───────────┤  │                          │   │
│  │  │  │(Hub VNet) │  │  │  │  Online   │  │                          │   │
│  │  │  └───────────┘  │  │  │(Public)   │  │                          │   │
│  │  └─────────────────┘  │  └───────────┘  │                          │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
│  DEPLOYED WITH: Bicep + Azure DevOps                                        │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Decision: Bicep vs Terraform

```
DECISION MATRIX:
────────────────

OPTION A: Terraform
───────────────────
Pros:
• Team has existing Terraform expertise (AWS background)
• Multi-cloud capability for future hybrid scenarios
• Large module ecosystem
• State management gives visibility into drift

Cons:
• External state management complexity
• Additional tool to maintain
• Licensing considerations (HashiCorp BSL)

OPTION B: Bicep (CHOSEN)
────────────────────────
Pros:
• Native Azure integration - no state files
• Immediate access to new Azure features
• Deployment stacks for lifecycle management
• Free, Microsoft-supported
• Better integration with Azure Policy
• Simpler for Azure-only deployments

Cons:
• Azure-only (acceptable for this scenario)
• Smaller ecosystem than Terraform
• Team needs training

DECISION: Bicep

Rationale:
1. Company is "all-in" on Azure - no multi-cloud requirement
2. Deployment stacks feature critical for managing 100+ subscriptions
3. Native integration with Azure Policy for governance
4. Reduced operational complexity (no state management)
5. Microsoft provides ALZ Bicep modules as starting point
```

### Implementation

```bicep
// main.bicep - Landing Zone deployment
targetScope = 'managementGroup'

@description('Management group ID for the landing zone')
param managementGroupId string

@description('Subscription ID for the landing zone')
param subscriptionId string

@description('Landing zone configuration')
param config object = {
  businessUnit: 'finance'
  environment: 'prod'
  location: 'eastus'
  networkConfig: {
    addressSpace: '10.50.0.0/16'
    subnets: {
      app: '10.50.1.0/24'
      data: '10.50.2.0/24'
      mgmt: '10.50.3.0/24'
    }
  }
}

// Deploy policy assignments at management group level
module policyAssignments 'modules/policy-assignments.bicep' = {
  name: 'policy-${config.businessUnit}-${config.environment}'
  scope: managementGroup(managementGroupId)
  params: {
    environment: config.environment
    businessUnit: config.businessUnit
  }
}

// Deploy landing zone resources at subscription level
module landingZone 'modules/landing-zone.bicep' = {
  name: 'lz-${config.businessUnit}-${config.environment}'
  scope: subscription(subscriptionId)
  params: {
    config: config
    hubVnetId: '/subscriptions/xxx/resourceGroups/connectivity-rg/providers/Microsoft.Network/virtualNetworks/hub-vnet'
  }
}

// modules/landing-zone.bicep
targetScope = 'subscription'

param config object
param hubVnetId string

// Resource group
resource rg 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: 'rg-${config.businessUnit}-${config.environment}'
  location: config.location
  tags: {
    BusinessUnit: config.businessUnit
    Environment: config.environment
    ManagedBy: 'Bicep'
  }
}

// Spoke VNet
module network 'network.bicep' = {
  name: 'network'
  scope: rg
  params: {
    vnetName: 'vnet-${config.businessUnit}-${config.environment}'
    addressSpace: config.networkConfig.addressSpace
    subnets: config.networkConfig.subnets
    location: config.location
  }
}

// VNet Peering to Hub
module peering 'peering.bicep' = {
  name: 'peering'
  scope: rg
  params: {
    spokeVnetId: network.outputs.vnetId
    hubVnetId: hubVnetId
  }
}

// Key Vault for application secrets
module keyVault 'keyvault.bicep' = {
  name: 'keyvault'
  scope: rg
  params: {
    name: 'kv-${config.businessUnit}-${config.environment}'
    location: config.location
  }
}

// Log Analytics workspace (or connect to central)
module monitoring 'monitoring.bicep' = {
  name: 'monitoring'
  scope: rg
  params: {
    workspaceName: 'log-${config.businessUnit}-${config.environment}'
    location: config.location
  }
}
```

### Pipeline Configuration

```yaml
# azure-pipelines-landing-zone.yml
trigger: none  # Manual trigger for new landing zones

parameters:
  - name: businessUnit
    displayName: 'Business Unit'
    type: string
    values:
      - finance
      - hr
      - engineering
      - sales

  - name: environment
    displayName: 'Environment'
    type: string
    values:
      - dev
      - test
      - prod

  - name: subscriptionId
    displayName: 'Target Subscription ID'
    type: string

stages:
  - stage: Validate
    jobs:
      - job: ValidateBicep
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: AzureCLI@2
            displayName: 'Validate Bicep'
            inputs:
              azureSubscription: 'Platform-Connection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az bicep build --file main.bicep
                az deployment mg validate \
                  --management-group-id "LandingZones" \
                  --location eastus \
                  --template-file main.bicep \
                  --parameters subscriptionId=${{ parameters.subscriptionId }} \
                  --parameters config='{"businessUnit":"${{ parameters.businessUnit }}","environment":"${{ parameters.environment }}"}'

  - stage: WhatIf
    dependsOn: Validate
    jobs:
      - job: WhatIf
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: AzureCLI@2
            displayName: 'What-If Analysis'
            inputs:
              azureSubscription: 'Platform-Connection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az deployment mg what-if \
                  --management-group-id "LandingZones" \
                  --location eastus \
                  --template-file main.bicep \
                  --parameters subscriptionId=${{ parameters.subscriptionId }} \
                  --parameters config='{"businessUnit":"${{ parameters.businessUnit }}","environment":"${{ parameters.environment }}"}'

  - stage: Deploy
    dependsOn: WhatIf
    jobs:
      - deployment: DeployLandingZone
        environment: 'landing-zones'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self
                - task: AzureCLI@2
                  displayName: 'Deploy Landing Zone'
                  inputs:
                    azureSubscription: 'Platform-Connection'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      az deployment mg create \
                        --management-group-id "LandingZones" \
                        --location eastus \
                        --template-file main.bicep \
                        --parameters subscriptionId=${{ parameters.subscriptionId }} \
                        --parameters config='{"businessUnit":"${{ parameters.businessUnit }}","environment":"${{ parameters.environment }}"}'
```

### Results

```
OUTCOMES:
─────────

DEPLOYMENT METRICS:
• 150 landing zones deployed in 6 months
• Average deployment time: 15 minutes (vs 2 weeks manual)
• 100% policy compliance at deployment
• Zero drift - Bicep deployments are declarative

GOVERNANCE:
• Consistent naming across all subscriptions
• Mandatory tags enforced via policy
• Network segmentation standardized
• Central logging enabled by default

TEAM EFFICIENCY:
• Platform team: 4 engineers managing 150+ subscriptions
• Application teams: Self-service via pipeline
• No manual Azure portal configurations

LESSONS LEARNED:
• Start with ALZ Bicep modules, customize as needed
• Use deployment stacks for cleanup/lifecycle
• Pipeline approvals critical for production
• Document parameter choices in ADO wiki
```

---

## Case Study 2: Multi-Environment Application Infrastructure

### Scenario

**Company**: SaaS startup with 50 engineers
**Challenge**: Inconsistent environments causing "works on my machine" issues
**Goal**: Identical infrastructure across dev, staging, and production

### Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                 MULTI-ENVIRONMENT APPLICATION INFRASTRUCTURE                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ENVIRONMENT CONFIGURATION (same code, different parameters):               │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                          SHARED MODULES                              │   │
│  │                                                                       │   │
│  │  modules/                                                            │   │
│  │  ├── aks/           # Kubernetes cluster                            │   │
│  │  ├── postgresql/    # Database with replicas                        │   │
│  │  ├── redis/         # Caching layer                                 │   │
│  │  ├── storage/       # Blob storage                                  │   │
│  │  └── monitoring/    # App Insights + Log Analytics                  │   │
│  │                                                                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                          │                                                  │
│          ┌───────────────┼───────────────┐                                 │
│          │               │               │                                 │
│          ▼               ▼               ▼                                 │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                       │
│  │     DEV      │ │   STAGING    │ │  PRODUCTION  │                       │
│  │              │ │              │ │              │                       │
│  │ AKS: 2 nodes │ │ AKS: 3 nodes │ │ AKS: 6 nodes │                       │
│  │ PG: Basic    │ │ PG: Standard │ │ PG: Premium  │                       │
│  │ Redis: Basic │ │ Redis: Std   │ │ Redis: Prem  │                       │
│  │              │ │              │ │              │                       │
│  │ Cost: ~$500  │ │ Cost: ~$1500 │ │ Cost: ~$5000 │                       │
│  └──────────────┘ └──────────────┘ └──────────────┘                       │
│                                                                              │
│  DEPLOYED WITH: Terraform + GitHub Actions                                  │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Decision: Workspaces vs Directory Structure

```
DECISION MATRIX:
────────────────

OPTION A: Terraform Workspaces
──────────────────────────────
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

Pros:
• Single codebase
• Easy workspace switching
• Built-in feature

Cons:
• Same backend for all environments (risk)
• Can accidentally apply to wrong workspace
• Harder to have env-specific backends

OPTION B: Directory Structure (CHOSEN)
──────────────────────────────────────
environments/
├── dev/
│   ├── main.tf
│   ├── terraform.tfvars
│   └── backend.tf
├── staging/
│   └── ...
└── prod/
    └── ...

Pros:
• Explicit separation
• Different backends per environment
• Harder to make mistakes
• Clear audit trail

Cons:
• Some code duplication
• More files to manage

DECISION: Directory Structure

Rationale:
1. Production isolation is critical
2. Different state backends per environment
3. Approval workflows differ by environment
4. Team can work on different environments simultaneously
```

### Implementation

```hcl
# environments/prod/main.tf
terraform {
  required_version = ">= 1.5.0"

  backend "azurerm" {
    resource_group_name  = "terraform-state-prod-rg"
    storage_account_name = "tfstateprod"
    container_name       = "tfstate"
    key                  = "app.terraform.tfstate"
  }
}

locals {
  environment = "prod"
  location    = "eastus"
  project     = "myapp"

  common_tags = {
    Environment = local.environment
    Project     = local.project
    ManagedBy   = "Terraform"
    Repository  = "github.com/company/infrastructure"
  }
}

# Resource Group
resource "azurerm_resource_group" "main" {
  name     = "rg-${local.project}-${local.environment}"
  location = local.location
  tags     = local.common_tags
}

# AKS Cluster
module "aks" {
  source = "../../modules/aks"

  resource_group_name = azurerm_resource_group.main.name
  location            = local.location
  cluster_name        = "aks-${local.project}-${local.environment}"

  # Production sizing
  node_count         = var.aks_node_count         # 6
  node_vm_size       = var.aks_node_vm_size       # Standard_D4s_v3
  enable_auto_scaling = true
  min_node_count     = 3
  max_node_count     = 12

  # Production features
  enable_azure_policy    = true
  enable_oms_agent       = true
  log_analytics_workspace_id = module.monitoring.workspace_id

  tags = local.common_tags
}

# PostgreSQL
module "postgresql" {
  source = "../../modules/postgresql"

  resource_group_name = azurerm_resource_group.main.name
  location            = local.location
  server_name         = "psql-${local.project}-${local.environment}"

  # Production sizing
  sku_name   = var.postgresql_sku   # GP_Gen5_4
  storage_mb = var.postgresql_storage # 102400

  # High availability
  geo_redundant_backup = true
  ha_mode             = "ZoneRedundant"

  # Security
  ssl_enforcement_enabled = true
  public_network_access   = false
  subnet_id               = module.network.database_subnet_id

  tags = local.common_tags
}

# Redis Cache
module "redis" {
  source = "../../modules/redis"

  resource_group_name = azurerm_resource_group.main.name
  location            = local.location
  name                = "redis-${local.project}-${local.environment}"

  # Production sizing
  capacity = var.redis_capacity  # 2
  family   = "P"                 # Premium
  sku_name = "Premium"

  # Production features
  enable_non_ssl_port = false
  minimum_tls_version = "1.2"
  subnet_id           = module.network.cache_subnet_id
  zones               = ["1", "2"]

  tags = local.common_tags
}

# Monitoring
module "monitoring" {
  source = "../../modules/monitoring"

  resource_group_name = azurerm_resource_group.main.name
  location            = local.location
  workspace_name      = "log-${local.project}-${local.environment}"
  app_insights_name   = "appi-${local.project}-${local.environment}"

  # Production retention
  retention_in_days = 90

  tags = local.common_tags
}
```

```hcl
# environments/prod/terraform.tfvars
# Production-specific values

aks_node_count    = 6
aks_node_vm_size  = "Standard_D4s_v3"

postgresql_sku     = "GP_Gen5_4"
postgresql_storage = 102400

redis_capacity = 2
```

```hcl
# environments/dev/terraform.tfvars
# Development-specific values (cost-optimized)

aks_node_count    = 2
aks_node_vm_size  = "Standard_B2s"

postgresql_sku     = "B_Gen5_1"
postgresql_storage = 5120

redis_capacity = 0  # Basic tier
```

### GitHub Actions Pipeline

```yaml
# .github/workflows/infrastructure.yml
name: Infrastructure

on:
  push:
    branches: [main]
    paths: ['environments/**', 'modules/**']
  pull_request:
    branches: [main]
    paths: ['environments/**', 'modules/**']
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        type: choice
        options:
          - dev
          - staging
          - prod
      action:
        description: 'Action to perform'
        required: true
        type: choice
        options:
          - plan
          - apply
          - destroy

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      dev: ${{ steps.changes.outputs.dev }}
      staging: ${{ steps.changes.outputs.staging }}
      prod: ${{ steps.changes.outputs.prod }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: changes
        with:
          filters: |
            dev:
              - 'environments/dev/**'
              - 'modules/**'
            staging:
              - 'environments/staging/**'
              - 'modules/**'
            prod:
              - 'environments/prod/**'
              - 'modules/**'

  plan-dev:
    needs: detect-changes
    if: needs.detect-changes.outputs.dev == 'true' || github.event.inputs.environment == 'dev'
    uses: ./.github/workflows/terraform-plan.yml
    with:
      environment: dev
    secrets: inherit

  plan-staging:
    needs: detect-changes
    if: needs.detect-changes.outputs.staging == 'true' || github.event.inputs.environment == 'staging'
    uses: ./.github/workflows/terraform-plan.yml
    with:
      environment: staging
    secrets: inherit

  plan-prod:
    needs: detect-changes
    if: needs.detect-changes.outputs.prod == 'true' || github.event.inputs.environment == 'prod'
    uses: ./.github/workflows/terraform-plan.yml
    with:
      environment: prod
    secrets: inherit

  apply-dev:
    needs: plan-dev
    if: github.ref == 'refs/heads/main' && (needs.detect-changes.outputs.dev == 'true' || github.event.inputs.environment == 'dev')
    uses: ./.github/workflows/terraform-apply.yml
    with:
      environment: dev
    secrets: inherit

  apply-staging:
    needs: [plan-staging, apply-dev]
    if: github.ref == 'refs/heads/main' && (needs.detect-changes.outputs.staging == 'true' || github.event.inputs.environment == 'staging')
    uses: ./.github/workflows/terraform-apply.yml
    with:
      environment: staging
    secrets: inherit

  apply-prod:
    needs: [plan-prod, apply-staging]
    if: github.ref == 'refs/heads/main' && (needs.detect-changes.outputs.prod == 'true' || github.event.inputs.environment == 'prod')
    uses: ./.github/workflows/terraform-apply.yml
    with:
      environment: prod
    secrets: inherit
```

### Results

```
OUTCOMES:
─────────

CONSISTENCY:
• 100% environment parity (same Terraform, different vars)
• Zero "staging works, prod doesn't" issues
• Developers can spin up identical environments

DEPLOYMENT SPEED:
• PR → Prod: 30 minutes (automated)
• Manual rollback available via git revert
• Environment provisioning: 20 minutes

COST VISIBILITY:
• Dev:     $500/month  (auto-shutdown enabled)
• Staging: $1,500/month
• Prod:    $5,000/month
• Clear cost per environment

SECURITY:
• Separate state files per environment
• Production requires additional approval
• No cross-environment access possible
```

---

## Case Study 3: Brownfield Migration with Import

### Scenario

**Company**: Healthcare provider with existing Azure resources
**Challenge**: 500+ resources created manually over 3 years, need IaC governance
**Goal**: Import existing resources into Terraform without disruption

### Migration Strategy

```
BROWNFIELD MIGRATION APPROACH:
──────────────────────────────

PHASE 1: DISCOVERY (Week 1-2)
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  • Export resource list with Azure Resource Graph                     │
│  • Identify resource dependencies                                     │
│  • Document current state and configurations                          │
│  • Categorize: Critical / Important / Standard                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

PHASE 2: PARALLEL DEVELOPMENT (Week 3-6)
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  ┌──────────────────────┐    ┌──────────────────────┐                │
│  │   EXISTING AZURE     │    │   TERRAFORM CODE     │                │
│  │                      │    │                      │                │
│  │  (Still operational) │◄──▶│  (Written to match)  │                │
│  │                      │    │                      │                │
│  └──────────────────────┘    └──────────────────────┘                │
│                                                                        │
│  • Write Terraform to match existing resources exactly                │
│  • Use terraform plan to verify - should show NO changes             │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

PHASE 3: IMPORT (Week 7-8)
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  1. terraform import (per resource)                                   │
│  2. terraform plan (verify no changes)                                │
│  3. terraform apply (confirm state matches)                           │
│  4. Repeat for all resources                                          │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘

PHASE 4: GOVERNANCE (Week 9+)
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  • All changes via Terraform only                                     │
│  • Portal access read-only for operations team                        │
│  • Drift detection via scheduled terraform plan                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Implementation

```bash
# Step 1: Discovery - Export all resources
az graph query -q "
  Resources
  | project name, type, resourceGroup, subscriptionId, location, tags
  | order by type asc
" -o json > resources.json

# Count by type
az graph query -q "
  Resources
  | summarize count() by type
  | order by count_ desc
" -o table
```

```hcl
# Step 2: Generate import blocks (Terraform 1.5+)
# imports.tf

import {
  to = azurerm_resource_group.production
  id = "/subscriptions/xxx/resourceGroups/production-rg"
}

import {
  to = azurerm_virtual_network.main
  id = "/subscriptions/xxx/resourceGroups/production-rg/providers/Microsoft.Network/virtualNetworks/main-vnet"
}

import {
  to = azurerm_subnet.app
  id = "/subscriptions/xxx/resourceGroups/production-rg/providers/Microsoft.Network/virtualNetworks/main-vnet/subnets/app-subnet"
}

import {
  to = azurerm_key_vault.main
  id = "/subscriptions/xxx/resourceGroups/production-rg/providers/Microsoft.KeyVault/vaults/prod-keyvault"
}

# ... 500+ more imports
```

```bash
# Step 3: Generate initial configuration
terraform plan -generate-config-out=generated.tf

# Review and organize generated code
# Split into logical files: networking.tf, compute.tf, etc.
```

```hcl
# Step 4: Refine generated code (example)
# generated.tf (auto-generated - needs cleanup)
resource "azurerm_virtual_network" "main" {
  address_space       = ["10.0.0.0/16"]
  location            = "eastus"
  name                = "main-vnet"
  resource_group_name = "production-rg"
  tags                = {}  # Empty in portal
}

# networking.tf (cleaned up)
resource "azurerm_virtual_network" "main" {
  name                = "main-vnet"
  resource_group_name = azurerm_resource_group.production.name
  location            = azurerm_resource_group.production.location
  address_space       = ["10.0.0.0/16"]

  tags = local.common_tags  # Add proper tags
}
```

```bash
# Step 5: Import and verify
terraform init
terraform plan  # Should show only tag additions, no destructive changes

# Output:
# Plan: 0 to add, 15 to change (tags only), 0 to destroy.
```

### Automation Script

```bash
#!/bin/bash
# import-resources.sh

set -e

RESOURCE_GROUP="production-rg"
STATE_FILE="terraform.tfstate"

# Function to import a resource
import_resource() {
  local tf_address=$1
  local azure_id=$2

  echo "Importing: $tf_address"

  if terraform state show "$tf_address" &>/dev/null; then
    echo "  Already imported, skipping"
    return 0
  fi

  if terraform import "$tf_address" "$azure_id"; then
    echo "  Success"
  else
    echo "  FAILED - manual review needed"
    echo "$tf_address,$azure_id" >> failed_imports.csv
  fi
}

# Import resources by type
echo "=== Importing Resource Groups ==="
import_resource "azurerm_resource_group.production" \
  "/subscriptions/$SUB_ID/resourceGroups/production-rg"

echo "=== Importing Virtual Networks ==="
import_resource "azurerm_virtual_network.main" \
  "/subscriptions/$SUB_ID/resourceGroups/production-rg/providers/Microsoft.Network/virtualNetworks/main-vnet"

echo "=== Importing Subnets ==="
for subnet in app data management; do
  import_resource "azurerm_subnet.$subnet" \
    "/subscriptions/$SUB_ID/resourceGroups/production-rg/providers/Microsoft.Network/virtualNetworks/main-vnet/subnets/$subnet-subnet"
done

# ... continue for all resource types

echo "=== Import Complete ==="
echo "Verifying state..."
terraform plan -detailed-exitcode

if [ $? -eq 0 ]; then
  echo "SUCCESS: No changes detected"
elif [ $? -eq 2 ]; then
  echo "WARNING: Changes detected - review plan output"
else
  echo "ERROR: Terraform plan failed"
  exit 1
fi
```

### Results

```
MIGRATION OUTCOMES:
───────────────────

RESOURCES MIGRATED:
• 523 resources imported successfully
• 12 resources required manual intervention
• 0 production disruptions

TIMELINE:
• Discovery: 2 weeks
• Code development: 4 weeks
• Import process: 2 weeks
• Total: 8 weeks

BEFORE / AFTER:
┌────────────────────────────────────────────────────────────────────────┐
│ BEFORE:                          AFTER:                                │
│ • Portal-based changes           • Git-based changes                  │
│ • No audit trail                 • Full git history                   │
│ • Inconsistent naming            • Standardized naming                │
│ • Unknown drift                  • Daily drift detection              │
│ • 4 people with portal access    • 2 people can approve PRs          │
└────────────────────────────────────────────────────────────────────────┘

CHALLENGES OVERCOME:
• Secret handling: Used data sources for existing Key Vault secrets
• Circular dependencies: Imported in correct order
• Large state file: Split into multiple state files by domain
• Team training: 2-day Terraform workshop for all engineers
```

---

## Key Takeaways

```
IaC IMPLEMENTATION PRINCIPLES:
──────────────────────────────

1. CHOOSE THE RIGHT TOOL
   • Azure-only? Consider Bicep for native integration
   • Multi-cloud or existing expertise? Terraform
   • Enterprise governance? Both can work with proper design

2. STATE MANAGEMENT IS CRITICAL
   • Terraform: Remote state with locking
   • Bicep: Deployment stacks for lifecycle management
   • Both: Separate state per environment

3. MODULE DESIGN MATTERS
   • Single responsibility
   • Explicit dependencies
   • Sensible defaults
   • Comprehensive testing

4. CI/CD IS NON-NEGOTIABLE
   • Automated validation on every PR
   • Security scanning (tfsec, Checkov)
   • Plan output in PR comments
   • Approval gates for production

5. BROWNFIELD IS POSSIBLE
   • Discovery first
   • Write code to match existing
   • Import methodically
   • Verify with plan before proceeding
```

---

*Back to [Chapter Overview](README.md)* | *Next Chapter: [FinOps & Cost Management](../07-finops/README.md)*
