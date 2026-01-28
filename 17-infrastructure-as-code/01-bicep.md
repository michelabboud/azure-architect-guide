# Bicep Fundamentals

## What is Bicep?

Bicep is Azure's domain-specific language (DSL) for deploying Azure resources. It's a transparent abstraction over ARM templates, providing a cleaner syntax while compiling to the same ARM JSON that Azure Resource Manager processes.

### Bicep vs ARM Templates

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      BICEP vs ARM TEMPLATES                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ARM TEMPLATE (JSON):                  BICEP:                               │
│  ───────────────────                   ──────                               │
│                                                                              │
│  {                                     param location string = 'eastus'     │
│    "$schema": "...",                   param name string                    │
│    "parameters": {                                                          │
│      "location": {                     resource storage 'Microsoft....' = { │
│        "type": "string",                 name: name                         │
│        "defaultValue": "eastus"          location: location                 │
│      },                                  sku: { name: 'Standard_LRS' }      │
│      "name": {                           kind: 'StorageV2'                  │
│        "type": "string"                }                                    │
│      }                                                                      │
│    },                                  output id string = storage.id        │
│    "resources": [{                                                          │
│      "type": "Microsoft...",                                               │
│      "apiVersion": "...",                                                   │
│      "name": "[parameters('name')]",                                       │
│      "location": "[parameters('location')]",                               │
│      "sku": {                                                              │
│        "name": "Standard_LRS"                                              │
│      },                                                                     │
│      "kind": "StorageV2"                                                   │
│    }],                                                                      │
│    "outputs": {                                                             │
│      "id": {                                                               │
│        "type": "string",                                                   │
│        "value": "[resourceId('Microsoft...', parameters('name'))]"        │
│      }                                                                      │
│    }                                                                        │
│  }                                                                          │
│                                                                              │
│  60+ lines                             10 lines                            │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Bicep Structure

### Basic Syntax

```bicep
// Parameters - inputs to your template
@description('The Azure region for resources')
param location string = resourceGroup().location

@description('Environment name')
@allowed(['dev', 'test', 'prod'])
param environment string

@secure()
@description('SQL admin password')
param sqlPassword string

@minLength(3)
@maxLength(24)
param storageAccountName string

// Variables - computed values
var prefix = 'myapp-${environment}'
var tags = {
  Environment: environment
  ManagedBy: 'Bicep'
  CostCenter: 'IT'
}

// Resources - Azure resources to deploy
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  tags: tags
  sku: {
    name: environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}

// Outputs - values to return after deployment
output storageAccountId string = storageAccount.id
output primaryEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

### Resource Syntax Deep Dive

```bicep
// RESOURCE DECLARATION:
// resource <symbolic-name> '<type>@<api-version>' = {
//   name: '<resource-name>'
//   location: '<location>'
//   properties: { ... }
// }

// Symbolic name: Used only within Bicep (not deployed to Azure)
// Type: Full resource provider path
// API version: Determines available properties

resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'myVNet'
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: ['10.0.0.0/16']
    }
  }
}

// Child resources - two approaches:

// Option 1: Nested (preferred for simple cases)
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'myVNet'
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.0.0.0/16'] }
    subnets: [
      {
        name: 'web'
        properties: { addressPrefix: '10.0.1.0/24' }
      }
    ]
  }
}

// Option 2: Separate resource with parent
resource vnet 'Microsoft.Network/virtualNetworks@2023-05-01' = {
  name: 'myVNet'
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.0.0.0/16'] }
  }
}

resource subnet 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = {
  parent: vnet  // Reference to parent
  name: 'web'
  properties: {
    addressPrefix: '10.0.1.0/24'
  }
}
```

---

## Modules

### Creating Modules

```bicep
// modules/storage.bicep
@description('Storage account name')
param name string

@description('Location')
param location string

@description('SKU name')
@allowed(['Standard_LRS', 'Standard_GRS', 'Standard_ZRS'])
param sku string = 'Standard_LRS'

@description('Tags')
param tags object = {}

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: name
  location: location
  tags: tags
  sku: {
    name: sku
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
    supportsHttpsTrafficOnly: true
  }
}

// Expose what consumers need
output id string = storageAccount.id
output name string = storageAccount.name
output primaryEndpoint string = storageAccount.properties.primaryEndpoints.blob
```

### Using Modules

```bicep
// main.bicep
param location string = resourceGroup().location
param environment string

// Local module
module storage 'modules/storage.bicep' = {
  name: 'storageDeployment'  // Deployment name (for tracking)
  params: {
    name: 'st${uniqueString(resourceGroup().id)}'
    location: location
    sku: environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'
    tags: {
      Environment: environment
    }
  }
}

// Use module output
output storageId string = storage.outputs.id

// Module from registry (public)
module appService 'br/public:app/app-service:1.0.1' = {
  name: 'appServiceDeployment'
  params: {
    name: 'myapp-${environment}'
    location: location
  }
}

// Module from private registry
module customModule 'br:myregistry.azurecr.io/bicep/modules/storage:v1' = {
  name: 'customStorageDeployment'
  params: {
    name: 'customstorage'
    location: location
  }
}

// Module deployed to different scope
module resourceGroup 'modules/resourceGroup.bicep' = {
  name: 'rgDeployment'
  scope: subscription()  // Deploy at subscription level
  params: {
    name: 'my-new-rg'
    location: location
  }
}
```

---

## Loops and Conditions

### Loops

```bicep
// Loop over array
param storageAccounts array = [
  { name: 'storage1', sku: 'Standard_LRS' }
  { name: 'storage2', sku: 'Standard_GRS' }
  { name: 'storage3', sku: 'Standard_ZRS' }
]

resource accounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for account in storageAccounts: {
  name: account.name
  location: location
  sku: { name: account.sku }
  kind: 'StorageV2'
}]

// Loop with index
resource subnets 'Microsoft.Network/virtualNetworks/subnets@2023-05-01' = [for (subnet, index) in subnetConfigs: {
  parent: vnet
  name: subnet.name
  properties: {
    addressPrefix: cidrSubnet('10.0.0.0/16', 8, index)
  }
}]

// Loop with range
resource nics 'Microsoft.Network/networkInterfaces@2023-05-01' = [for i in range(0, vmCount): {
  name: 'nic-${i}'
  location: location
  properties: {
    ipConfigurations: [
      {
        name: 'ipconfig1'
        properties: {
          subnet: { id: subnet.id }
        }
      }
    ]
  }
}]

// Filtered loop
var prodStorageAccounts = filter(storageAccounts, account => account.sku == 'Standard_GRS')

// Loop with modules
module storageModules 'modules/storage.bicep' = [for account in storageAccounts: {
  name: 'storage-${account.name}'
  params: {
    name: account.name
    location: location
    sku: account.sku
  }
}]
```

### Conditions

```bicep
// Conditional resource
param deployDiagnostics bool = true

resource diagnosticSettings 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = if (deployDiagnostics) {
  name: 'diag-settings'
  scope: storageAccount
  properties: {
    workspaceId: logAnalyticsWorkspace.id
    logs: [
      {
        category: 'StorageRead'
        enabled: true
      }
    ]
  }
}

// Conditional property value
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: environment == 'prod' ? 'Standard_GRS' : 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    // Conditional nested object
    networkAcls: environment == 'prod' ? {
      defaultAction: 'Deny'
      bypass: 'AzureServices'
    } : {
      defaultAction: 'Allow'
    }
  }
}

// Combine loop and condition
resource storageAccounts 'Microsoft.Storage/storageAccounts@2023-01-01' = [for account in storageAccountConfigs: if (account.deploy) {
  name: account.name
  location: location
  sku: { name: account.sku }
  kind: 'StorageV2'
}]
```

---

## Deployment Scopes

```bicep
// RESOURCE GROUP (default)
// main.bicep - deployed to a resource group
resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  // ...
}

// Deploy: az deployment group create --resource-group myRG --template-file main.bicep


// SUBSCRIPTION SCOPE
// subscription.bicep
targetScope = 'subscription'

param location string

resource resourceGroup 'Microsoft.Resources/resourceGroups@2023-07-01' = {
  name: 'my-new-rg'
  location: location
}

// Deploy resources into the new RG
module storage 'modules/storage.bicep' = {
  scope: resourceGroup  // Target the new RG
  name: 'storageDeployment'
  params: {
    name: 'mystorage'
    location: location
  }
}

// Deploy: az deployment sub create --location eastus --template-file subscription.bicep


// MANAGEMENT GROUP SCOPE
// mg.bicep
targetScope = 'managementGroup'

resource policy 'Microsoft.Authorization/policyDefinitions@2021-06-01' = {
  name: 'require-tags'
  properties: {
    policyType: 'Custom'
    mode: 'Indexed'
    // ...
  }
}

// Deploy: az deployment mg create --management-group-id myMG --location eastus --template-file mg.bicep


// TENANT SCOPE
// tenant.bicep
targetScope = 'tenant'

resource managementGroup 'Microsoft.Management/managementGroups@2021-04-01' = {
  name: 'myNewMG'
  properties: {
    displayName: 'My New Management Group'
  }
}

// Deploy: az deployment tenant create --location eastus --template-file tenant.bicep
```

---

## Deployment Commands

```bash
# Validate template
az deployment group validate \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters environment=dev

# What-if (preview changes)
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters environment=dev

# Deploy
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters environment=dev \
  --name myDeployment-$(date +%Y%m%d%H%M%S)

# Deploy with parameter file
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @parameters.prod.json

# Deploy with inline + file parameters
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep \
  --parameters @parameters.json \
  --parameters environment=prod vmSize=Standard_D4s_v3

# Get deployment output
az deployment group show \
  --resource-group myRG \
  --name myDeployment \
  --query properties.outputs.storageId.value -o tsv
```

---

## Best Practices

```
BICEP BEST PRACTICES:
─────────────────────

1. USE MODULES FOR REUSE
   ┌─────────────────────────────────────────────────────────────────────┐
   │ • Create modules for each resource type                             │
   │ • Keep modules focused (single responsibility)                      │
   │ • Version modules in registry for enterprise use                   │
   └─────────────────────────────────────────────────────────────────────┘

2. USE PARAMETER FILES PER ENVIRONMENT
   ┌─────────────────────────────────────────────────────────────────────┐
   │ main.bicep                                                          │
   │ parameters.dev.json                                                 │
   │ parameters.test.json                                                │
   │ parameters.prod.json                                                │
   └─────────────────────────────────────────────────────────────────────┘

3. ALWAYS USE WHAT-IF BEFORE DEPLOYING
   ┌─────────────────────────────────────────────────────────────────────┐
   │ az deployment group what-if --template-file main.bicep             │
   │                                                                      │
   │ Review changes before applying to catch mistakes                   │
   └─────────────────────────────────────────────────────────────────────┘

4. USE MEANINGFUL SYMBOLIC NAMES
   ┌─────────────────────────────────────────────────────────────────────┐
   │ // Good                                                              │
   │ resource productionStorageAccount 'Microsoft.Storage...' = { }     │
   │                                                                      │
   │ // Bad                                                               │
   │ resource sa1 'Microsoft.Storage...' = { }                          │
   └─────────────────────────────────────────────────────────────────────┘

5. ADD DESCRIPTIONS TO PARAMETERS
   ┌─────────────────────────────────────────────────────────────────────┐
   │ @description('The Azure region where resources will be deployed')  │
   │ param location string                                               │
   └─────────────────────────────────────────────────────────────────────┘
```

---

*Next: [Terraform for Azure](02-terraform.md)* | *Back to [Quick Reference](quick-reference.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
