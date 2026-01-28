# Chapter 06: Infrastructure as Code

## Overview

Infrastructure as Code (IaC) enables you to define, deploy, and manage Azure resources using declarative configuration files. This chapter covers Bicep (Azure's native IaC language), Terraform, and ARM templates.

## AWS to Azure IaC Mapping

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       IaC TOOLS: AWS vs AZURE                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  AWS                                   AZURE                                │
│  ───                                   ─────                                │
│                                                                              │
│  CloudFormation          ──────────▶   ARM Templates / Bicep                │
│  (Native IaC)                          (Native IaC)                         │
│                                                                              │
│  AWS CDK                 ──────────▶   Azure Developer CLI (azd)           │
│  (Abstraction layer)                   Bicep + deployment scripts           │
│                                                                              │
│  Terraform               ──────────▶   Terraform                            │
│  (Multi-cloud)                         (Same tool, Azure provider)          │
│                                                                              │
│  SAM (Serverless)        ──────────▶   Bicep + Azure Functions             │
│                                                                              │
│  KEY DIFFERENCES:                                                           │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  CloudFormation:                  Bicep:                             │   │
│  │  • JSON/YAML                      • Purpose-built language           │   │
│  │  • Verbose syntax                 • Concise, readable                │   │
│  │  • Nested stacks                  • Modules                          │   │
│  │  • Change sets                    • What-if deployment               │   │
│  │                                                                       │   │
│  │  CFN Resources:                   ARM Resources:                     │   │
│  │  • AWS::EC2::Instance            • Microsoft.Compute/virtualMachines│   │
│  │  • AWS::S3::Bucket               • Microsoft.Storage/storageAccounts│   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## IaC Options Comparison

```
AZURE IaC OPTIONS:
──────────────────

┌────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  BICEP (Recommended for Azure-only)                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Pros:                              Cons:                             │   │
│  │ ✓ Clean, readable syntax           ✗ Azure-only                     │   │
│  │ ✓ First-class Azure support        ✗ Newer (less community content)│   │
│  │ ✓ Best VS Code tooling             ✗ Learning new language          │   │
│  │ ✓ Native what-if                                                    │   │
│  │ ✓ Compiles to ARM                                                   │   │
│  │ ✓ No state file to manage                                           │   │
│  │                                                                       │   │
│  │ Best for: Azure-focused teams, new Azure projects                   │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  TERRAFORM (Recommended for multi-cloud)                                   │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Pros:                              Cons:                             │   │
│  │ ✓ Multi-cloud (AWS, GCP, Azure)   ✗ State file management           │   │
│  │ ✓ Huge community/modules          ✗ Provider version issues         │   │
│  │ ✓ Mature ecosystem                ✗ Learning HCL                    │   │
│  │ ✓ Terraform Cloud available       ✗ Third-party dependency          │   │
│  │ ✓ Many existing skills                                              │   │
│  │                                                                       │   │
│  │ Best for: Multi-cloud, existing Terraform teams                     │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  ARM TEMPLATES (Legacy, avoid for new projects)                            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │ Pros:                              Cons:                             │   │
│  │ ✓ Full Azure API coverage         ✗ Verbose JSON                    │   │
│  │ ✓ Direct portal export            ✗ Hard to read/maintain           │   │
│  │ ✓ No compilation step             ✗ Limited reuse                   │   │
│  │                                   ✗ Poor tooling                    │   │
│  │                                                                       │   │
│  │ Best for: Only if required by existing systems                      │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└────────────────────────────────────────────────────────────────────────────┘
```

## Chapter Contents

### Quick Reference
- [Quick Reference](quick-reference.md) - Bicep vs Terraform syntax comparison

### Deep Dive Topics
1. [Bicep Fundamentals](01-bicep.md) - Azure's native IaC language
2. [Terraform for Azure](02-terraform.md) - Multi-cloud IaC with Azure provider
3. [Best Practices](03-best-practices.md) - Modules, testing, and CI/CD
4. [Case Studies](case-studies.md) - Real-world IaC implementations

## Getting Started

### Bicep Quick Start

```bash
# Install Bicep CLI
az bicep install

# Create first Bicep file (main.bicep)
cat > main.bicep << 'EOF'
param location string = resourceGroup().location
param storageAccountName string = 'st${uniqueString(resourceGroup().id)}'

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
}

output storageAccountId string = storageAccount.id
EOF

# Deploy
az deployment group create \
  --resource-group myRG \
  --template-file main.bicep

# Preview changes (what-if)
az deployment group what-if \
  --resource-group myRG \
  --template-file main.bicep
```

### Terraform Quick Start

```bash
# Install Terraform (if not installed)
# brew install terraform (macOS) or download from terraform.io

# Create Terraform configuration (main.tf)
cat > main.tf << 'EOF'
terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "example" {
  name     = "example-rg"
  location = "East US"
}

resource "azurerm_storage_account" "example" {
  name                     = "stexample${random_string.suffix.result}"
  resource_group_name      = azurerm_resource_group.example.name
  location                 = azurerm_resource_group.example.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}

output "storage_account_id" {
  value = azurerm_storage_account.example.id
}
EOF

# Initialize and deploy
terraform init
terraform plan
terraform apply
```

## Learning Path

```
RECOMMENDED STUDY ORDER:
────────────────────────

Week 1: Bicep Fundamentals
├── Read: Quick Reference (syntax comparison)
├── Do: Install Bicep CLI and VS Code extension
├── Do: Deploy storage account with Bicep
└── Practice: Create VNet with subnets

Week 2: Advanced Bicep
├── Read: Bicep deep dive
├── Do: Create reusable modules
├── Do: Deploy multi-resource solution
└── Practice: Hub-spoke network with modules

Week 3: Terraform for Azure
├── Read: Terraform deep dive
├── Do: Set up Terraform state in Azure Storage
├── Do: Deploy same resources with Terraform
└── Practice: Compare Bicep vs Terraform

Week 4: Production Patterns
├── Read: Best practices
├── Do: Set up CI/CD pipeline
├── Do: Implement testing
├── Review: Case studies
└── Practice: Full environment deployment
```

## Key Concepts

### Declarative vs Imperative

```
DECLARATIVE (Bicep, Terraform, ARM):
────────────────────────────────────
"I want a storage account with these properties"

resource storageAccount 'Microsoft.Storage/storageAccounts@2023-01-01' = {
  name: 'mystorageaccount'
  location: 'eastus'
  sku: { name: 'Standard_LRS' }
  kind: 'StorageV2'
}

The tool figures out:
• Does it exist? → Update if needed
• Doesn't exist? → Create it
• Extra resources? → (Depends on tool/mode)


IMPERATIVE (Azure CLI scripts):
───────────────────────────────
"Execute these commands in order"

az storage account create \
  --name mystorageaccount \
  --resource-group myRG \
  --location eastus \
  --sku Standard_LRS

You must handle:
• Check if exists first
• Create or update logic
• Error handling
• Idempotency
```

### State Management

```
BICEP/ARM: No State File
────────────────────────
• Azure IS the state
• Compares desired vs actual in Azure
• Each deployment is independent
• Can cause drift if deployed differently

TERRAFORM: State File Required
──────────────────────────────
• State file tracks what Terraform manages
• Must be stored securely (Azure Storage recommended)
• Enables resource tracking across deployments
• State locking prevents concurrent modifications

RECOMMENDATION:
• Bicep: Use deployment stacks (preview) for state-like behavior
• Terraform: Store state in Azure Storage with locking
```

---

*Next: [Quick Reference](quick-reference.md)* | *Back to [Chapter 05: AI Platform](../05-ai-platform/README.md)*

---

*Author: Michel Abboud | AI-Assisted Content | [APACHE 2.0 License](../LICENSE)*
