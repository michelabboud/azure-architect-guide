# Terraform for Azure

## Azure Provider Configuration

### Basic Setup

```hcl
# versions.tf - Pin provider versions
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.45"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.5"
    }
  }

  # Remote state in Azure Storage
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

# provider.tf - Configure providers
provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
  }

  # Optional: Specify subscription
  # subscription_id = "00000000-0000-0000-0000-000000000000"
}

provider "azuread" {
  # Uses same auth as azurerm by default
}
```

### Authentication Methods

```bash
# Method 1: Azure CLI (recommended for development)
az login
terraform plan

# Method 2: Service Principal (CI/CD)
export ARM_CLIENT_ID="00000000-0000-0000-0000-000000000000"
export ARM_CLIENT_SECRET="your-secret"
export ARM_TENANT_ID="00000000-0000-0000-0000-000000000000"
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
terraform plan

# Method 3: Managed Identity (Azure VMs/pipelines)
provider "azurerm" {
  features {}
  use_msi = true
}

# Method 4: OIDC (GitHub Actions, Azure DevOps)
provider "azurerm" {
  features {}
  use_oidc = true
}
```

---

## State Management

### Azure Storage Backend

```bash
# Create storage for state (one-time setup)
az group create --name terraform-state-rg --location eastus

az storage account create \
  --name tfstateaccount \
  --resource-group terraform-state-rg \
  --sku Standard_LRS \
  --encryption-services blob

az storage container create \
  --name tfstate \
  --account-name tfstateaccount

# Enable versioning for state file recovery
az storage blob service-properties update \
  --account-name tfstateaccount \
  --enable-versioning true

# Enable soft delete
az storage blob service-properties update \
  --account-name tfstateaccount \
  --enable-delete-retention true \
  --delete-retention-days 30
```

### State Locking

```hcl
# Azure Storage automatically provides state locking
# No additional configuration needed

# If lock is stuck, break it manually:
# az storage blob lease break \
#   --account-name tfstateaccount \
#   --container-name tfstate \
#   --blob-name prod.terraform.tfstate
```

---

## Module Structure

### Recommended Layout

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── test/
│   │   └── ...
│   └── prod/
│       └── ...
├── modules/
│   ├── network/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   └── ...
│   └── database/
│       └── ...
└── shared/
    └── versions.tf
```

### Module Example

```hcl
# modules/network/variables.tf
variable "resource_group_name" {
  type        = string
  description = "Name of the resource group"
}

variable "location" {
  type        = string
  description = "Azure region"
}

variable "vnet_name" {
  type        = string
  description = "Name of the virtual network"
}

variable "address_space" {
  type        = list(string)
  description = "Address space for the VNet"
  default     = ["10.0.0.0/16"]
}

variable "subnets" {
  type = map(object({
    address_prefix = string
    service_endpoints = optional(list(string), [])
    delegation = optional(object({
      name    = string
      service = string
      actions = list(string)
    }))
  }))
  description = "Map of subnet configurations"
}

variable "tags" {
  type        = map(string)
  description = "Tags to apply to resources"
  default     = {}
}

# modules/network/main.tf
resource "azurerm_virtual_network" "this" {
  name                = var.vnet_name
  resource_group_name = var.resource_group_name
  location            = var.location
  address_space       = var.address_space
  tags                = var.tags
}

resource "azurerm_subnet" "this" {
  for_each = var.subnets

  name                 = each.key
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.this.name
  address_prefixes     = [each.value.address_prefix]
  service_endpoints    = each.value.service_endpoints

  dynamic "delegation" {
    for_each = each.value.delegation != null ? [each.value.delegation] : []
    content {
      name = delegation.value.name
      service_delegation {
        name    = delegation.value.service
        actions = delegation.value.actions
      }
    }
  }
}

# modules/network/outputs.tf
output "vnet_id" {
  value       = azurerm_virtual_network.this.id
  description = "Virtual network ID"
}

output "vnet_name" {
  value       = azurerm_virtual_network.this.name
  description = "Virtual network name"
}

output "subnet_ids" {
  value       = { for k, v in azurerm_subnet.this : k => v.id }
  description = "Map of subnet names to IDs"
}
```

### Using Modules

```hcl
# environments/prod/main.tf
module "network" {
  source = "../../modules/network"

  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  vnet_name           = "prod-vnet"
  address_space       = ["10.0.0.0/16"]

  subnets = {
    web = {
      address_prefix = "10.0.1.0/24"
    }
    app = {
      address_prefix    = "10.0.2.0/24"
      service_endpoints = ["Microsoft.Sql", "Microsoft.Storage"]
    }
    data = {
      address_prefix = "10.0.3.0/24"
    }
    appservice = {
      address_prefix = "10.0.4.0/24"
      delegation = {
        name    = "appservice"
        service = "Microsoft.Web/serverFarms"
        actions = ["Microsoft.Network/virtualNetworks/subnets/action"]
      }
    }
  }

  tags = local.common_tags
}

# Reference module outputs
resource "azurerm_network_security_group" "web" {
  name                = "nsg-web"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
}

resource "azurerm_subnet_network_security_group_association" "web" {
  subnet_id                 = module.network.subnet_ids["web"]
  network_security_group_id = azurerm_network_security_group.web.id
}
```

---

## Data Sources

```hcl
# Get existing resources
data "azurerm_client_config" "current" {}

data "azurerm_subscription" "current" {}

data "azurerm_resource_group" "existing" {
  name = "existing-rg"
}

data "azurerm_key_vault" "existing" {
  name                = "my-keyvault"
  resource_group_name = "keyvault-rg"
}

data "azurerm_key_vault_secret" "db_password" {
  name         = "db-admin-password"
  key_vault_id = data.azurerm_key_vault.existing.id
}

# Use data sources
resource "azurerm_mssql_server" "example" {
  name                         = "sql-server"
  resource_group_name          = data.azurerm_resource_group.existing.name
  location                     = data.azurerm_resource_group.existing.location
  administrator_login          = "sqladmin"
  administrator_login_password = data.azurerm_key_vault_secret.db_password.value
}

# Query Azure AD
data "azuread_group" "admins" {
  display_name = "Cloud Admins"
}

resource "azurerm_role_assignment" "admin_contributor" {
  scope                = azurerm_resource_group.main.id
  role_definition_name = "Contributor"
  principal_id         = data.azuread_group.admins.object_id
}
```

---

## Import Existing Resources

```bash
# Import existing resource into Terraform state
terraform import azurerm_resource_group.example /subscriptions/{sub}/resourceGroups/myRG

terraform import azurerm_storage_account.example /subscriptions/{sub}/resourceGroups/myRG/providers/Microsoft.Storage/storageAccounts/mystorageaccount

# Then add matching resource block to your .tf file
```

```hcl
# Terraform 1.5+ import block (declarative)
import {
  to = azurerm_resource_group.example
  id = "/subscriptions/{sub}/resourceGroups/myRG"
}

resource "azurerm_resource_group" "example" {
  name     = "myRG"
  location = "eastus"
}

# Generate config from import
# terraform plan -generate-config-out=generated.tf
```

---

## Common Patterns

### Conditional Resources

```hcl
variable "enable_diagnostics" {
  type    = bool
  default = true
}

resource "azurerm_monitor_diagnostic_setting" "example" {
  count = var.enable_diagnostics ? 1 : 0

  name                       = "diag-settings"
  target_resource_id         = azurerm_storage_account.example.id
  log_analytics_workspace_id = azurerm_log_analytics_workspace.example.id

  metric {
    category = "AllMetrics"
    enabled  = true
  }
}

# Reference conditional resource
output "diagnostic_setting_id" {
  value = var.enable_diagnostics ? azurerm_monitor_diagnostic_setting.example[0].id : null
}
```

### Dynamic Blocks

```hcl
variable "nsg_rules" {
  type = list(object({
    name                       = string
    priority                   = number
    direction                  = string
    access                     = string
    protocol                   = string
    source_port_range          = string
    destination_port_range     = string
    source_address_prefix      = string
    destination_address_prefix = string
  }))
}

resource "azurerm_network_security_group" "example" {
  name                = "nsg-example"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location

  dynamic "security_rule" {
    for_each = var.nsg_rules
    content {
      name                       = security_rule.value.name
      priority                   = security_rule.value.priority
      direction                  = security_rule.value.direction
      access                     = security_rule.value.access
      protocol                   = security_rule.value.protocol
      source_port_range          = security_rule.value.source_port_range
      destination_port_range     = security_rule.value.destination_port_range
      source_address_prefix      = security_rule.value.source_address_prefix
      destination_address_prefix = security_rule.value.destination_address_prefix
    }
  }
}
```

### Lifecycle Rules

```hcl
resource "azurerm_virtual_machine" "example" {
  name                = "vm-example"
  # ...

  lifecycle {
    # Prevent destruction
    prevent_destroy = true

    # Ignore changes (e.g., managed by autoscaler)
    ignore_changes = [
      tags["LastUpdated"],
    ]

    # Create new before destroying old
    create_before_destroy = true

    # Custom preconditions
    precondition {
      condition     = var.vm_size != "Standard_B1s" || var.environment != "prod"
      error_message = "Cannot use B1s in production."
    }
  }
}
```

---

## Commands Reference

```bash
# Initialize
terraform init
terraform init -upgrade  # Upgrade providers

# Format and validate
terraform fmt -recursive
terraform validate

# Plan
terraform plan
terraform plan -out=tfplan
terraform plan -var-file=prod.tfvars
terraform plan -target=module.network  # Plan specific resource

# Apply
terraform apply
terraform apply tfplan
terraform apply -auto-approve  # CI/CD only!

# Destroy
terraform destroy
terraform destroy -target=azurerm_storage_account.example

# State management
terraform state list
terraform state show azurerm_resource_group.example
terraform state mv azurerm_storage_account.old azurerm_storage_account.new
terraform state rm azurerm_storage_account.example  # Remove without destroying

# Workspaces (for multi-env)
terraform workspace list
terraform workspace new dev
terraform workspace select prod

# Output
terraform output
terraform output storage_account_id
```

---

*Next: [Best Practices](03-best-practices.md)* | *Back to [Bicep](01-bicep.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
