# Infrastructure as Code Best Practices

## Module Design

### Module Structure

```
RECOMMENDED MODULE STRUCTURE:
─────────────────────────────

modules/
├── network/
│   ├── main.tf           # Resource definitions
│   ├── variables.tf      # Input variables with validation
│   ├── outputs.tf        # Output values
│   ├── versions.tf       # Provider version constraints
│   ├── README.md         # Usage documentation
│   └── examples/
│       ├── basic/
│       │   └── main.tf
│       └── complete/
│           └── main.tf
├── compute/
│   └── ...
└── database/
    └── ...

BICEP EQUIVALENT:
modules/
├── network/
│   ├── main.bicep        # Module definition
│   ├── README.md         # Documentation
│   └── examples/
│       └── ...
└── ...
```

### Module Design Principles

```
GOOD MODULE DESIGN:
───────────────────

1. SINGLE RESPONSIBILITY
   ┌────────────────────────────────────────────────────────────────────┐
   │ ✅ GOOD: Module does one thing well                                │
   │    module "vnet" - Creates VNet and subnets                       │
   │    module "nsg"  - Creates NSG with rules                         │
   │    module "vm"   - Creates VM with dependencies                   │
   │                                                                    │
   │ ❌ BAD: Module does everything                                     │
   │    module "infrastructure" - Creates VNet, VMs, DBs, everything   │
   └────────────────────────────────────────────────────────────────────┘

2. EXPLICIT DEPENDENCIES
   ┌────────────────────────────────────────────────────────────────────┐
   │ ✅ GOOD: Dependencies passed as inputs                             │
   │    module "vm" {                                                   │
   │      subnet_id = module.network.subnet_ids["app"]                 │
   │      nsg_id    = module.security.nsg_id                           │
   │    }                                                               │
   │                                                                    │
   │ ❌ BAD: Module creates its own dependencies                        │
   │    module "vm" {                                                   │
   │      # Internally creates VNet, subnet, NSG...                    │
   │    }                                                               │
   └────────────────────────────────────────────────────────────────────┘

3. SENSIBLE DEFAULTS
   ┌────────────────────────────────────────────────────────────────────┐
   │ ✅ GOOD: Works out of box, customizable when needed                │
   │    variable "vm_size" {                                           │
   │      default = "Standard_D2s_v3"  # Reasonable default            │
   │    }                                                               │
   │                                                                    │
   │ ❌ BAD: Requires every parameter                                   │
   │    variable "vm_size" {                                           │
   │      # No default - user must always specify                      │
   │    }                                                               │
   └────────────────────────────────────────────────────────────────────┘

4. VALIDATION
   ┌────────────────────────────────────────────────────────────────────┐
   │ ✅ GOOD: Validate inputs early                                     │
   │    variable "environment" {                                       │
   │      validation {                                                 │
   │        condition = contains(["dev","test","prod"], var.environment)│
   │        error_message = "Must be dev, test, or prod."              │
   │      }                                                             │
   │    }                                                               │
   └────────────────────────────────────────────────────────────────────┘
```

---

## Naming Conventions

### Azure Resource Naming

```hcl
# Terraform - Consistent naming with locals
locals {
  # Format: {resource_type}-{workload}-{environment}-{region}-{instance}
  naming_prefix = "${var.workload}-${var.environment}-${var.location_short}"

  resource_names = {
    resource_group   = "rg-${local.naming_prefix}"
    vnet             = "vnet-${local.naming_prefix}"
    subnet_web       = "snet-web-${local.naming_prefix}"
    subnet_app       = "snet-app-${local.naming_prefix}"
    nsg              = "nsg-${local.naming_prefix}"
    storage_account  = "st${var.workload}${var.environment}${var.location_short}"  # No hyphens
    key_vault        = "kv-${local.naming_prefix}"
    app_service_plan = "asp-${local.naming_prefix}"
    app_service      = "app-${local.naming_prefix}"
    sql_server       = "sql-${local.naming_prefix}"
    aks_cluster      = "aks-${local.naming_prefix}"
  }

  # Location short codes
  location_map = {
    "eastus"        = "eus"
    "eastus2"       = "eus2"
    "westus2"       = "wus2"
    "westeurope"    = "weu"
    "northeurope"   = "neu"
    "southeastasia" = "sea"
  }

  location_short = local.location_map[var.location]
}

# Usage
resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = var.location
}
```

```bicep
// Bicep - Consistent naming with variables
var locationShortMap = {
  eastus: 'eus'
  eastus2: 'eus2'
  westus2: 'wus2'
  westeurope: 'weu'
  northeurope: 'neu'
}

var locationShort = locationShortMap[location]
var namingPrefix = '${workload}-${environment}-${locationShort}'

var resourceNames = {
  resourceGroup: 'rg-${namingPrefix}'
  vnet: 'vnet-${namingPrefix}'
  storageAccount: 'st${workload}${environment}${locationShort}'
  keyVault: 'kv-${namingPrefix}'
}
```

### Tagging Strategy

```hcl
# Terraform - Consistent tagging
locals {
  common_tags = {
    Environment     = var.environment
    Project         = var.project_name
    CostCenter      = var.cost_center
    Owner           = var.owner_email
    ManagedBy       = "Terraform"
    Repository      = var.repository_url
    DeploymentDate  = timestamp()
  }

  # Merge common tags with resource-specific tags
  resource_tags = merge(local.common_tags, {
    Component = "networking"
  })
}

# Apply to all resources
resource "azurerm_resource_group" "main" {
  name     = local.resource_names.resource_group
  location = var.location
  tags     = local.common_tags
}

resource "azurerm_virtual_network" "main" {
  name                = local.resource_names.vnet
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  address_space       = var.vnet_address_space
  tags                = local.resource_tags
}
```

---

## State Management

### Remote State Best Practices

```
STATE MANAGEMENT PATTERNS:
──────────────────────────

1. SEPARATE STATE PER ENVIRONMENT
   ┌────────────────────────────────────────────────────────────────────┐
   │                                                                    │
   │  terraform-state-account/                                         │
   │  └── tfstate/                                                     │
   │      ├── dev/                                                     │
   │      │   ├── networking.tfstate                                  │
   │      │   ├── compute.tfstate                                     │
   │      │   └── data.tfstate                                        │
   │      ├── test/                                                    │
   │      │   └── ...                                                  │
   │      └── prod/                                                    │
   │          └── ...                                                  │
   │                                                                    │
   └────────────────────────────────────────────────────────────────────┘

2. STATE FILE SEPARATION
   ┌────────────────────────────────────────────────────────────────────┐
   │                                                                    │
   │  Split by blast radius:                                           │
   │  • networking.tfstate    - VNets, peering (rarely changes)       │
   │  • shared.tfstate        - Key Vault, ACR (shared resources)     │
   │  • compute.tfstate       - VMs, AKS (frequently changes)         │
   │  • data.tfstate          - Databases (sensitive, rarely changes) │
   │                                                                    │
   │  Benefits:                                                        │
   │  • Smaller state = faster operations                              │
   │  • Limited blast radius per deployment                            │
   │  • Different teams can own different states                       │
   │                                                                    │
   └────────────────────────────────────────────────────────────────────┘
```

### Cross-State References

```hcl
# Terraform - Reading from another state file
data "terraform_remote_state" "networking" {
  backend = "azurerm"
  config = {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "${var.environment}/networking.tfstate"
  }
}

# Use outputs from networking state
resource "azurerm_linux_virtual_machine" "app" {
  name                = "vm-app"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location

  network_interface_ids = [azurerm_network_interface.app.id]

  # Reference networking state output
  # (network_interface references subnet from networking state)
}

resource "azurerm_network_interface" "app" {
  name                = "nic-app"
  resource_group_name = azurerm_resource_group.compute.name
  location            = azurerm_resource_group.compute.location

  ip_configuration {
    name                          = "internal"
    subnet_id                     = data.terraform_remote_state.networking.outputs.subnet_ids["app"]
    private_ip_address_allocation = "Dynamic"
  }
}
```

```bicep
// Bicep - Cross-deployment references
// Reference existing resources from other deployments

resource existingVnet 'Microsoft.Network/virtualNetworks@2023-05-01' existing = {
  name: 'vnet-prod-eus'
  scope: resourceGroup('networking-prod-rg')
}

resource existingSubnet 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' existing = {
  parent: existingVnet
  name: 'snet-app'
}

// Use in new deployment
resource nic 'Microsoft.Network/networkInterfaces@2023-05-01' = {
  name: 'nic-app'
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'internal'
        properties: {
          subnet: {
            id: existingSubnet.id
          }
        }
      }
    ]
  }
}
```

---

## Testing IaC

### Terraform Testing

```hcl
# tests/network_test.tftest.hcl (Terraform 1.6+)
variables {
  environment = "test"
  location    = "eastus"
  workload    = "unittest"
}

run "creates_vnet_with_correct_address_space" {
  command = plan

  assert {
    condition     = azurerm_virtual_network.main.address_space[0] == "10.0.0.0/16"
    error_message = "VNet address space incorrect"
  }
}

run "creates_required_subnets" {
  command = plan

  assert {
    condition     = length(azurerm_subnet.this) == 3
    error_message = "Expected 3 subnets"
  }
}

run "applies_correct_tags" {
  command = plan

  assert {
    condition     = azurerm_resource_group.main.tags["Environment"] == "test"
    error_message = "Environment tag not set correctly"
  }
}
```

```bash
# Run Terraform tests
terraform test

# Run specific test file
terraform test -filter=tests/network_test.tftest.hcl

# Verbose output
terraform test -verbose
```

### Bicep Testing with PSRule

```powershell
# Install PSRule for Azure
Install-Module -Name PSRule.Rules.Azure -Scope CurrentUser

# Create test configuration
# ps-rule.yaml
@"
configuration:
  AZURE_DEPLOYMENT_NONSENSITIVE_PARAMETER_NAMES:
    - environment
    - location
    - workload

rule:
  baseline:
    include:
      - Azure.Resource.UseTags
      - Azure.KeyVault.SoftDelete
      - Azure.Storage.MinTLS
      - Azure.AKS.NetworkPolicy
"@ | Out-File -FilePath ps-rule.yaml

# Run validation
Assert-PSRule -InputPath ./infra -Module PSRule.Rules.Azure -OutputFormat Detailed
```

### Integration Testing

```bash
#!/bin/bash
# integration-test.sh

set -e

ENVIRONMENT="test"
TIMESTAMP=$(date +%Y%m%d%H%M%S)
RESOURCE_GROUP="rg-integration-test-${TIMESTAMP}"

echo "=== Creating test infrastructure ==="
terraform init
terraform apply -auto-approve \
  -var="environment=${ENVIRONMENT}" \
  -var="resource_group_name=${RESOURCE_GROUP}"

echo "=== Running integration tests ==="

# Test 1: Verify VNet exists
VNET_ID=$(az network vnet show \
  --resource-group "${RESOURCE_GROUP}" \
  --name "vnet-${ENVIRONMENT}" \
  --query id -o tsv)

if [ -z "$VNET_ID" ]; then
  echo "FAIL: VNet not created"
  exit 1
fi
echo "PASS: VNet created"

# Test 2: Verify subnet connectivity
SUBNET_COUNT=$(az network vnet subnet list \
  --resource-group "${RESOURCE_GROUP}" \
  --vnet-name "vnet-${ENVIRONMENT}" \
  --query "length(@)")

if [ "$SUBNET_COUNT" -lt 3 ]; then
  echo "FAIL: Expected at least 3 subnets, got ${SUBNET_COUNT}"
  exit 1
fi
echo "PASS: Subnets created correctly"

# Test 3: Verify NSG rules
NSG_RULES=$(az network nsg rule list \
  --resource-group "${RESOURCE_GROUP}" \
  --nsg-name "nsg-${ENVIRONMENT}" \
  --query "length(@)")

if [ "$NSG_RULES" -lt 1 ]; then
  echo "FAIL: No NSG rules found"
  exit 1
fi
echo "PASS: NSG rules configured"

echo "=== Cleaning up test infrastructure ==="
terraform destroy -auto-approve \
  -var="environment=${ENVIRONMENT}" \
  -var="resource_group_name=${RESOURCE_GROUP}"

echo "=== All integration tests passed ==="
```

---

## CI/CD Pipelines

### GitHub Actions Pipeline

```yaml
# .github/workflows/terraform.yml
name: 'Terraform'

on:
  push:
    branches: [ main ]
    paths:
      - 'terraform/**'
  pull_request:
    branches: [ main ]
    paths:
      - 'terraform/**'

permissions:
  id-token: write   # OIDC auth
  contents: read
  pull-requests: write

env:
  ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
  ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
  ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
  ARM_USE_OIDC: true
  TF_VERSION: '1.6.0'

jobs:
  validate:
    name: 'Validate'
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Format Check
      run: terraform fmt -check -recursive
      working-directory: terraform

    - name: Terraform Init
      run: terraform init -backend=false
      working-directory: terraform

    - name: Terraform Validate
      run: terraform validate
      working-directory: terraform

  security-scan:
    name: 'Security Scan'
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: tfsec
      uses: aquasecurity/tfsec-action@v1.0.0
      with:
        working_directory: terraform

    - name: Checkov
      uses: bridgecrewio/checkov-action@v12
      with:
        directory: terraform
        framework: terraform

  plan:
    name: 'Plan'
    runs-on: ubuntu-latest
    needs: [validate, security-scan]
    if: github.event_name == 'pull_request'

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Init
      run: terraform init
      working-directory: terraform/environments/prod

    - name: Terraform Plan
      id: plan
      run: |
        terraform plan -no-color -out=tfplan 2>&1 | tee plan_output.txt
        echo "plan_output<<EOF" >> $GITHUB_OUTPUT
        cat plan_output.txt >> $GITHUB_OUTPUT
        echo "EOF" >> $GITHUB_OUTPUT
      working-directory: terraform/environments/prod

    - name: Post Plan to PR
      uses: actions/github-script@v7
      with:
        script: |
          const output = `#### Terraform Plan 📖

          <details><summary>Show Plan</summary>

          \`\`\`terraform
          ${{ steps.plan.outputs.plan_output }}
          \`\`\`

          </details>`;

          github.rest.issues.createComment({
            issue_number: context.issue.number,
            owner: context.repo.owner,
            repo: context.repo.repo,
            body: output
          });

  apply:
    name: 'Apply'
    runs-on: ubuntu-latest
    needs: [validate, security-scan]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production

    steps:
    - name: Checkout
      uses: actions/checkout@v4

    - name: Azure Login
      uses: azure/login@v1
      with:
        client-id: ${{ secrets.AZURE_CLIENT_ID }}
        tenant-id: ${{ secrets.AZURE_TENANT_ID }}
        subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Init
      run: terraform init
      working-directory: terraform/environments/prod

    - name: Terraform Apply
      run: terraform apply -auto-approve
      working-directory: terraform/environments/prod
```

### Azure DevOps Pipeline

```yaml
# azure-pipelines.yml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - terraform/**

pr:
  branches:
    include:
      - main
  paths:
    include:
      - terraform/**

variables:
  - group: terraform-secrets
  - name: tfVersion
    value: '1.6.0'
  - name: workingDirectory
    value: 'terraform/environments/prod'

stages:
  - stage: Validate
    displayName: 'Validate'
    jobs:
      - job: Validate
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(tfVersion)

          - script: terraform fmt -check -recursive
            displayName: 'Terraform Format'
            workingDirectory: terraform

          - script: terraform init -backend=false
            displayName: 'Terraform Init'
            workingDirectory: $(workingDirectory)

          - script: terraform validate
            displayName: 'Terraform Validate'
            workingDirectory: $(workingDirectory)

  - stage: SecurityScan
    displayName: 'Security Scan'
    dependsOn: Validate
    jobs:
      - job: Scan
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - script: |
              pip install checkov
              checkov -d terraform --framework terraform
            displayName: 'Checkov Scan'

  - stage: Plan
    displayName: 'Plan'
    dependsOn: SecurityScan
    condition: and(succeeded(), eq(variables['Build.Reason'], 'PullRequest'))
    jobs:
      - job: Plan
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: TerraformInstaller@1
            inputs:
              terraformVersion: $(tfVersion)

          - task: AzureCLI@2
            displayName: 'Terraform Init & Plan'
            inputs:
              azureSubscription: 'Production'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                export ARM_CLIENT_ID=$(ARM_CLIENT_ID)
                export ARM_CLIENT_SECRET=$(ARM_CLIENT_SECRET)
                export ARM_TENANT_ID=$(ARM_TENANT_ID)
                export ARM_SUBSCRIPTION_ID=$(ARM_SUBSCRIPTION_ID)

                cd $(workingDirectory)
                terraform init
                terraform plan -out=tfplan

          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: '$(workingDirectory)/tfplan'
              artifact: 'tfplan'

  - stage: Apply
    displayName: 'Apply'
    dependsOn: SecurityScan
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: Apply
        pool:
          vmImage: 'ubuntu-latest'
        environment: 'production'
        strategy:
          runOnce:
            deploy:
              steps:
                - checkout: self

                - task: TerraformInstaller@1
                  inputs:
                    terraformVersion: $(tfVersion)

                - task: AzureCLI@2
                  displayName: 'Terraform Apply'
                  inputs:
                    azureSubscription: 'Production'
                    scriptType: 'bash'
                    scriptLocation: 'inlineScript'
                    inlineScript: |
                      export ARM_CLIENT_ID=$(ARM_CLIENT_ID)
                      export ARM_CLIENT_SECRET=$(ARM_CLIENT_SECRET)
                      export ARM_TENANT_ID=$(ARM_TENANT_ID)
                      export ARM_SUBSCRIPTION_ID=$(ARM_SUBSCRIPTION_ID)

                      cd $(workingDirectory)
                      terraform init
                      terraform apply -auto-approve
```

---

## Security Best Practices

### Secrets Management

```
SECRETS MANAGEMENT:
───────────────────

NEVER DO THIS:
┌────────────────────────────────────────────────────────────────────────┐
│ ❌ Hardcoded secrets in code                                           │
│    password = "SuperSecret123!"                                        │
│                                                                        │
│ ❌ Secrets in terraform.tfvars committed to git                        │
│    db_password = "production-password"                                 │
│                                                                        │
│ ❌ Secrets in environment variables in pipeline YAML                   │
│    env:                                                                │
│      DB_PASSWORD: "password123"                                        │
└────────────────────────────────────────────────────────────────────────┘

DO THIS INSTEAD:
┌────────────────────────────────────────────────────────────────────────┐
│ ✅ Azure Key Vault data source                                         │
│    data "azurerm_key_vault_secret" "db_password" {                    │
│      name         = "db-admin-password"                               │
│      key_vault_id = data.azurerm_key_vault.main.id                    │
│    }                                                                   │
│                                                                        │
│ ✅ Pipeline secret variables                                           │
│    - Use Azure DevOps variable groups with Key Vault integration      │
│    - Use GitHub secrets                                                │
│                                                                        │
│ ✅ Managed Identity for authentication                                 │
│    - No credentials to manage                                          │
│    - Automatic rotation                                                │
└────────────────────────────────────────────────────────────────────────┘
```

```hcl
# Terraform - Secure secrets handling
data "azurerm_key_vault" "main" {
  name                = "kv-secrets-prod"
  resource_group_name = "security-prod-rg"
}

data "azurerm_key_vault_secret" "db_admin_password" {
  name         = "sql-admin-password"
  key_vault_id = data.azurerm_key_vault.main.id
}

resource "azurerm_mssql_server" "main" {
  name                         = "sql-prod-eus"
  resource_group_name          = azurerm_resource_group.main.name
  location                     = azurerm_resource_group.main.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = data.azurerm_key_vault_secret.db_admin_password.value

  # Mark as sensitive in state
  lifecycle {
    ignore_changes = [administrator_login_password]
  }
}

# Sensitive outputs
output "db_connection_string" {
  value     = "Server=${azurerm_mssql_server.main.fully_qualified_domain_name}..."
  sensitive = true
}
```

### Provider Authentication

```hcl
# Terraform - OIDC Authentication (recommended for CI/CD)
provider "azurerm" {
  features {}

  use_oidc        = true
  client_id       = var.client_id       # From env: ARM_CLIENT_ID
  tenant_id       = var.tenant_id       # From env: ARM_TENANT_ID
  subscription_id = var.subscription_id # From env: ARM_SUBSCRIPTION_ID

  # No client_secret needed!
}

# For GitHub Actions OIDC
# 1. Create App Registration in Entra ID
# 2. Add Federated Credential for GitHub
# 3. Configure in workflow:
#    - azure/login action with OIDC
#    - Set ARM_USE_OIDC=true
```

---

## Cost Optimization in IaC

### Right-sizing Resources

```hcl
# Terraform - Environment-based sizing
variable "environment" {
  type = string
}

locals {
  # Size resources based on environment
  vm_sizes = {
    dev  = "Standard_B2s"
    test = "Standard_D2s_v3"
    prod = "Standard_D4s_v3"
  }

  db_sku = {
    dev  = { tier = "Basic", capacity = 5 }
    test = { tier = "Standard", capacity = 10 }
    prod = { tier = "Premium", capacity = 125 }
  }

  app_service_sku = {
    dev  = { tier = "Basic", size = "B1" }
    test = { tier = "Standard", size = "S1" }
    prod = { tier = "PremiumV3", size = "P1v3" }
  }
}

resource "azurerm_linux_virtual_machine" "app" {
  name                = "vm-app-${var.environment}"
  size                = local.vm_sizes[var.environment]
  # ...
}

resource "azurerm_mssql_database" "main" {
  name      = "sqldb-${var.environment}"
  sku_name  = local.db_sku[var.environment].tier
  max_size_gb = local.db_sku[var.environment].capacity
  # ...
}
```

### Auto-shutdown for Non-Production

```hcl
# Terraform - Auto-shutdown for dev/test VMs
resource "azurerm_dev_test_global_vm_shutdown_schedule" "auto_shutdown" {
  count = var.environment != "prod" ? 1 : 0

  virtual_machine_id = azurerm_linux_virtual_machine.app.id
  location           = azurerm_resource_group.main.location
  enabled            = true

  daily_recurrence_time = "1900"  # 7 PM
  timezone              = "Eastern Standard Time"

  notification_settings {
    enabled         = true
    email           = var.notification_email
    time_in_minutes = 30
  }
}
```

---

## Disaster Recovery

### Multi-Region Deployment

```hcl
# Terraform - Multi-region deployment pattern
variable "regions" {
  type = map(object({
    location      = string
    is_primary    = bool
    address_space = list(string)
  }))
  default = {
    primary = {
      location      = "eastus"
      is_primary    = true
      address_space = ["10.0.0.0/16"]
    }
    secondary = {
      location      = "westus2"
      is_primary    = false
      address_space = ["10.1.0.0/16"]
    }
  }
}

# Create resources in each region
resource "azurerm_resource_group" "regional" {
  for_each = var.regions

  name     = "rg-${var.workload}-${each.value.location}"
  location = each.value.location
  tags     = local.common_tags
}

resource "azurerm_virtual_network" "regional" {
  for_each = var.regions

  name                = "vnet-${var.workload}-${each.value.location}"
  resource_group_name = azurerm_resource_group.regional[each.key].name
  location            = each.value.location
  address_space       = each.value.address_space
  tags                = local.common_tags
}

# VNet peering between regions
resource "azurerm_virtual_network_peering" "primary_to_secondary" {
  name                      = "peer-primary-to-secondary"
  resource_group_name       = azurerm_resource_group.regional["primary"].name
  virtual_network_name      = azurerm_virtual_network.regional["primary"].name
  remote_virtual_network_id = azurerm_virtual_network.regional["secondary"].id
  allow_forwarded_traffic   = true
}

resource "azurerm_virtual_network_peering" "secondary_to_primary" {
  name                      = "peer-secondary-to-primary"
  resource_group_name       = azurerm_resource_group.regional["secondary"].name
  virtual_network_name      = azurerm_virtual_network.regional["secondary"].name
  remote_virtual_network_id = azurerm_virtual_network.regional["primary"].id
  allow_forwarded_traffic   = true
}
```

---

*Next: [Case Studies](case-studies.md)* | *Back to [Terraform](02-terraform.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
