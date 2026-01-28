# IaC Quick Reference

## Bicep vs Terraform Syntax Comparison

### Resource Definition

```
STORAGE ACCOUNT:
────────────────

BICEP:
┌────────────────────────────────────────────────────────────────────────────┐
│ resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = { │
│   name: storageAccountName                                                  │
│   location: location                                                        │
│   sku: {                                                                    │
│     name: 'Standard_LRS'                                                   │
│   }                                                                         │
│   kind: 'StorageV2'                                                        │
│   properties: {                                                             │
│     minimumTlsVersion: 'TLS1_2'                                            │
│     allowBlobPublicAccess: false                                           │
│   }                                                                         │
│ }                                                                           │
└────────────────────────────────────────────────────────────────────────────┘

TERRAFORM:
┌────────────────────────────────────────────────────────────────────────────┐
│ resource "azurerm_storage_account" "example" {                              │
│   name                     = var.storage_account_name                       │
│   resource_group_name      = azurerm_resource_group.example.name           │
│   location                 = azurerm_resource_group.example.location       │
│   account_tier             = "Standard"                                     │
│   account_replication_type = "LRS"                                         │
│   min_tls_version          = "TLS1_2"                                      │
│   allow_nested_items_to_be_public = false                                  │
│ }                                                                           │
└────────────────────────────────────────────────────────────────────────────┘
```

### Variables/Parameters

```
BICEP (Parameters):
───────────────────
// main.bicep
@description('Location for all resources')
param location string = resourceGroup().location

@description('Environment name')
@allowed(['dev', 'test', 'prod'])
param environment string

@secure()
param adminPassword string

// parameters.json (optional)
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "environment": { "value": "prod" }
  }
}


TERRAFORM (Variables):
──────────────────────
// variables.tf
variable "location" {
  description = "Location for all resources"
  type        = string
  default     = "eastus"
}

variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["dev", "test", "prod"], var.environment)
    error_message = "Environment must be dev, test, or prod."
  }
}

variable "admin_password" {
  description = "Admin password"
  type        = string
  sensitive   = true
}

// terraform.tfvars
environment = "prod"
```

### Outputs

```
BICEP:
──────
output storageAccountId string = storageAccount.id
output storageAccountName string = storageAccount.name
output primaryEndpoint string = storageAccount.properties.primaryEndpoints.blob


TERRAFORM:
──────────
output "storage_account_id" {
  value       = azurerm_storage_account.example.id
  description = "The ID of the storage account"
}

output "storage_account_name" {
  value = azurerm_storage_account.example.name
}

output "primary_endpoint" {
  value = azurerm_storage_account.example.primary_blob_endpoint
}
```

### Modules

```
BICEP:
──────
// modules/storage.bicep
param name string
param location string

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: name
  location: location
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

output id string = storageAccount.id

// main.bicep
module storage 'modules/storage.bicep' = {
  name: 'storageDeployment'
  params: {
    name: 'mystorageaccount'
    location: location
  }
}

output storageId string = storage.outputs.id


TERRAFORM:
──────────
// modules/storage/main.tf
variable "name" { type = string }
variable "location" { type = string }
variable "resource_group_name" { type = string }

resource "azurerm_storage_account" "this" {
  name                     = var.name
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

output "id" { value = azurerm_storage_account.this.id }

// main.tf
module "storage" {
  source              = "./modules/storage"
  name                = "mystorageaccount"
  location            = var.location
  resource_group_name = azurerm_resource_group.example.name
}

output "storage_id" {
  value = module.storage.id
}
```

---

## Common Resources

### Virtual Network

```bicep
// BICEP
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'myVNet'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
    subnets: [
      {
        name: 'web'
        properties: {
          addressPrefix: '10.0.1.0/24'
        }
      }
      {
        name: 'app'
        properties: {
          addressPrefix: '10.0.2.0/24'
        }
      }
    ]
  }
}
```

```hcl
# TERRAFORM
resource "azurerm_virtual_network" "example" {
  name                = "myVNet"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "web" {
  name                 = "web"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.0.1.0/24"]
}

resource "azurerm_subnet" "app" {
  name                 = "app"
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = ["10.0.2.0/24"]
}
```

### App Service

```bicep
// BICEP
resource appServicePlan 'Microsoft.Web/serverfarms@2023-01-01' = {
  name: 'myAppPlan'
  location: location
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
  }
  kind: 'linux'
  properties: {
    reserved: true
  }
}

resource webApp 'Microsoft.Web/sites@2023-01-01' = {
  name: 'myWebApp${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    serverFarmId: appServicePlan.id
    siteConfig: {
      linuxFxVersion: 'DOTNETCORE|8.0'
      alwaysOn: true
    }
    httpsOnly: true
  }
  identity: {
    type: 'SystemAssigned'
  }
}
```

```hcl
# TERRAFORM
resource "azurerm_service_plan" "example" {
  name                = "myAppPlan"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  os_type             = "Linux"
  sku_name            = "P1v3"
}

resource "azurerm_linux_web_app" "example" {
  name                = "mywebapp${random_string.suffix.result}"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  service_plan_id     = azurerm_service_plan.example.id
  https_only          = true

  site_config {
    always_on = true
    application_stack {
      dotnet_version = "8.0"
    }
  }

  identity {
    type = "SystemAssigned"
  }
}
```

### Azure SQL

```bicep
// BICEP
resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
  name: 'sql-${uniqueString(resourceGroup().id)}'
  location: location
  properties: {
    administratorLogin: 'sqladmin'
    administratorLoginPassword: sqlPassword
    minimalTlsVersion: '1.2'
  }
}

resource sqlDatabase 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
  parent: sqlServer
  name: 'mydb'
  location: location
  sku: {
    name: 'GP_S_Gen5_2'
    tier: 'GeneralPurpose'
  }
  properties: {
    collation: 'SQL_Latin1_General_CP1_CI_AS'
    maxSizeBytes: 34359738368 // 32GB
  }
}
```

```hcl
# TERRAFORM
resource "azurerm_mssql_server" "example" {
  name                         = "sql-${random_string.suffix.result}"
  resource_group_name          = azurerm_resource_group.example.name
  location                     = azurerm_resource_group.example.location
  version                      = "12.0"
  administrator_login          = "sqladmin"
  administrator_login_password = var.sql_password
  minimum_tls_version          = "1.2"
}

resource "azurerm_mssql_database" "example" {
  name           = "mydb"
  server_id      = azurerm_mssql_server.example.id
  collation      = "SQL_Latin1_General_CP1_CI_AS"
  max_size_gb    = 32
  sku_name       = "GP_S_Gen5_2"
}
```

---

## CLI Commands

### Bicep Commands

```bash
# Install/upgrade Bicep
az bicep install
az bicep upgrade

# Build (compile to ARM)
az bicep build --file main.bicep

# Decompile ARM to Bicep
az bicep decompile --file template.json

# Deploy to resource group
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @parameters.json \
  --parameters environment=prod

# Preview changes (what-if)
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep

# Deploy to subscription (for resource groups, policies)
az deployment sub create \
  --location eastus \
  --template-file main.bicep

# Validate template
az deployment group validate \
  --resource-group myRG \
  --template-file main.bicep

# Delete deployment (not resources!)
az deployment group delete \
  --resource-group myRG \
  --name myDeployment
```

### Terraform Commands

```bash
# Initialize (download providers)
terraform init

# Format code
terraform fmt -recursive

# Validate configuration
terraform validate

# Plan (preview changes)
terraform plan -out=tfplan

# Apply changes
terraform apply tfplan
# or interactively:
terraform apply

# Show current state
terraform show

# List resources in state
terraform state list

# Destroy all resources
terraform destroy

# Import existing resource
terraform import azurerm_resource_group.example /subscriptions/{sub}/resourceGroups/myRG

# Taint resource (force recreation)
terraform taint azurerm_storage_account.example

# Remove from state (without deleting)
terraform state rm azurerm_storage_account.example

# Refresh state from Azure
terraform refresh
```

---

## Loops and Conditions

### Loops

```bicep
// BICEP - Loop with array
param storageAccounts array = [
  { name: 'storage1', sku: 'Standard_LRS' }
  { name: 'storage2', sku: 'Standard_GRS' }
]

resource accounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for account in storageAccounts: {
  name: account.name
  location: location
  sku: { name: account.sku }
  kind: 'StorageV2'
}]

// Loop with index
resource subnets 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = [for (subnet, i) in subnetConfigs: {
  name: 'subnet-${i}'
  // ...
}]
```

```hcl
# TERRAFORM - for_each with map
variable "storage_accounts" {
  default = {
    storage1 = { sku = "Standard_LRS" }
    storage2 = { sku = "Standard_GRS" }
  }
}

resource "azurerm_storage_account" "accounts" {
  for_each = var.storage_accounts

  name                     = each.key
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = split("_", each.value.sku)[1]
}

# count with list
variable "subnet_names" {
  default = ["web", "app", "data"]
}

resource "azurerm_subnet" "subnets" {
  count = length(var.subnet_names)

  name                 = var.subnet_names[count.index]
  resource_group_name  = azurerm_resource_group.example.name
  virtual_network_name = azurerm_virtual_network.example.name
  address_prefixes     = [cidrsubnet("10.0.0.0/16", 8, count.index)]
}
```

### Conditions

```bicep
// BICEP
param deployDiagnostics bool = true

resource diagnostics 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = if (deployDiagnostics) {
  name: 'diag-settings'
  scope: storageAccount
  properties: {
    // ...
  }
}

// Conditional property
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'
  }
  kind: 'StorageV2'
}
```

```hcl
# TERRAFORM
variable "deploy_diagnostics" {
  type    = bool
  default = true
}

resource "azurerm_monitor_diagnostic_setting" "example" {
  count = var.deploy_diagnostics ? 1 : 0

  name               = "diag-settings"
  target_resource_id = azurerm_storage_account.example.id
  # ...
}

# Conditional expression
resource "azurerm_storage_account" "example" {
  name                     = var.storage_account_name
  account_replication_type = var.environment == "prod" ? "GRS" : "LRS"
  # ...
}
```

---

## State Management (Terraform)

```hcl
# Backend configuration for Azure Storage
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstateaccount"
    container_name       = "tfstate"
    key                  = "prod.terraform.tfstate"
  }
}

# Create the storage account for state (run once manually or with bootstrap)
# az storage account create --name tfstateaccount --resource-group terraform-state-rg
# az storage container create --name tfstate --account-name tfstateaccount
```

```bash
# Initialize with backend config
terraform init \
  -backend-config="storage_account_name=tfstateaccount" \
  -backend-config="container_name=tfstate" \
  -backend-config="key=prod.terraform.tfstate" \
  -backend-config="resource_group_name=terraform-state-rg"
```

---

*Next: [Bicep Fundamentals](01-bicep.md)* | *Back to [Chapter Overview](README.md)*
