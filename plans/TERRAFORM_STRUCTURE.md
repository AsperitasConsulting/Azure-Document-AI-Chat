# Terraform Project Structure - Detailed Implementation Guide

## Overview

This document provides detailed implementation guidance for the Terraform project structure, including code examples, best practices, and configuration templates.

## Project Organization

### Directory Structure Rationale

```
infrastructure/
├── terraform-storage/        # Isolated project for storage resources
├── terraform-app/           # Isolated project for application resources
└── shared/                  # Shared utilities and scripts
```

**Benefits of Separation:**
- **Reduced Blast Radius**: Changes to storage don't affect app infrastructure
- **Independent Deployment Cycles**: Storage is more stable than app layer
- **Team Ownership**: Different teams can manage different projects
- **State Isolation**: Separate state files prevent accidental modifications

## Terraform Storage Project

### Complete File Structure

```hcl
# terraform-storage/providers.tf
terraform {
  required_version = ">= 1.0"
  
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

# terraform-storage/backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "Terraform-Backend-State-RG"
    storage_account_name = "aspterraformstate"
    container_name       = "tfstate"
    # key is provided at runtime via -backend-config
  }
}

# terraform-storage/variables.tf
variable "environment" {
  description = "Environment name (dev, qa, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "qa", "prod"], var.environment)
    error_message = "Environment must be dev, qa, or prod."
  }
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "eastus"
}

variable "resource_group_name" {
  description = "Name of the resource group"
  type        = string
  default     = ""
}

variable "storage_account_name" {
  description = "Name of the storage account"
  type        = string
  default     = ""
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}

# terraform-storage/locals.tf
locals {
  resource_group_name  = var.resource_group_name != "" ? var.resource_group_name : "doc-qa-rg-${var.environment}"
  storage_account_name = var.storage_account_name != "" ? var.storage_account_name : "aspdocqa${var.environment}"
  
  default_tags = {
    Environment = var.environment
    Project     = "doc-qa"
    ManagedBy   = "Terraform"
    Component   = "storage"
  }
  
  tags = merge(local.default_tags, var.tags)
}

# terraform-storage/main.tf
resource "azurerm_resource_group" "main" {
  name     = local.resource_group_name
  location = var.location
  tags     = local.tags
}

resource "azurerm_storage_account" "main" {
  name                     = local.storage_account_name
  resource_group_name      = azurerm_resource_group.main.name
  location                 = azurerm_resource_group.main.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  
  min_tls_version               = "TLS1_2"
  enable_https_traffic_only     = true
  allow_nested_items_to_be_public = false
  
  blob_properties {
    versioning_enabled = true
    
    delete_retention_policy {
      days = 7
    }
    
    container_delete_retention_policy {
      days = 7
    }
  }
  
  tags = local.tags
}

resource "azurerm_storage_container" "docs" {
  name                  = "docs"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

# terraform-storage/outputs.tf
output "resource_group_name" {
  description = "The name of the resource group"
  value       = azurerm_resource_group.main.name
}

output "resource_group_id" {
  description = "The ID of the resource group"
  value       = azurerm_resource_group.main.id
}

output "storage_account_name" {
  description = "The name of the storage account"
  value       = azurerm_storage_account.main.name
}

output "storage_account_id" {
  description = "The ID of the storage account"
  value       = azurerm_storage_account.main.id
}

output "storage_account_primary_blob_endpoint" {
  description = "The primary blob endpoint"
  value       = azurerm_storage_account.main.primary_blob_endpoint
}

output "storage_container_name" {
  description = "The name of the blob container"
  value       = azurerm_storage_container.docs.name
}

# terraform-storage/environments/dev.tfvars
environment = "dev"
location    = "eastus"

tags = {
  Environment = "Development"
  CostCenter  = "Engineering"
  Owner       = "DevTeam"
}

# terraform-storage/environments/qa.tfvars
environment = "qa"
location    = "eastus"

tags = {
  Environment = "QA"
  CostCenter  = "Quality"
  Owner       = "QATeam"
}

# terraform-storage/environments/prod.tfvars
environment = "prod"
location    = "eastus"

tags = {
  Environment = "Production"
  CostCenter  = "Operations"
  Owner       = "OpsTeam"
  Compliance  = "Required"
}
```

## Terraform App Project

### Complete File Structure

```hcl
# terraform-app/providers.tf
terraform {
  required_version = ">= 1.0"
  
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

# terraform-app/backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "Terraform-Backend-State-RG"
    storage_account_name = "aspterraformstate"
    container_name       = "tfstate"
    # key is provided at runtime via -backend-config
  }
}

# terraform-app/data.tf
# Reference the storage infrastructure
data "azurerm_resource_group" "storage" {
  name = "doc-qa-rg-${var.environment}"
}

data "azurerm_storage_account" "main" {
  name                = "aspdocqa${var.environment}"
  resource_group_name = data.azurerm_resource_group.storage.name
}

# terraform-app/variables.tf
variable "environment" {
  description = "Environment name (dev, qa, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "qa", "prod"], var.environment)
    error_message = "Environment must be dev, qa, or prod."
  }
}

variable "location" {
  description = "Azure region for resources"
  type        = string
  default     = "eastus"
}

variable "function_app_name" {
  description = "Name of the function app"
  type        = string
  default     = ""
}

variable "app_service_plan_name" {
  description = "Name of the app service plan"
  type        = string
  default     = ""
}

variable "openai_api_key" {
  description = "OpenAI API Key"
  type        = string
  sensitive   = true
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}

# terraform-app/locals.tf
locals {
  function_app_name      = var.function_app_name != "" ? var.function_app_name : "doc-qa-api-${var.environment}"
  app_service_plan_name  = var.app_service_plan_name != "" ? var.app_service_plan_name : "doc-qa-plan-${var.environment}"
  storage_account_name   = "docqafunc${var.environment}"
  
  default_tags = {
    Environment = var.environment
    Project     = "doc-qa"
    ManagedBy   = "Terraform"
    Component   = "application"
  }
  
  tags = merge(local.default_tags, var.tags)
}

# terraform-app/main.tf
# Function App Storage Account
resource "azurerm_storage_account" "function" {
  name                     = local.storage_account_name
  resource_group_name      = data.azurerm_resource_group.storage.name
  location                 = data.azurerm_resource_group.storage.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  
  min_tls_version           = "TLS1_2"
  enable_https_traffic_only = true
  
  tags = local.tags
}

# App Service Plan (Consumption)
resource "azurerm_service_plan" "main" {
  name                = local.app_service_plan_name
  resource_group_name = data.azurerm_resource_group.storage.name
  location            = data.azurerm_resource_group.storage.location
  os_type             = "Linux"
  sku_name            = "Y1"  # Consumption plan
  
  tags = local.tags
}

# Application Insights
resource "azurerm_application_insights" "main" {
  name                = "${local.function_app_name}-insights"
  resource_group_name = data.azurerm_resource_group.storage.name
  location            = data.azurerm_resource_group.storage.location
  application_type    = "other"
  
  tags = local.tags
}

# Linux Function App
resource "azurerm_linux_function_app" "main" {
  name                = local.function_app_name
  resource_group_name = data.azurerm_resource_group.storage.name
  location            = data.azurerm_resource_group.storage.location
  
  storage_account_name       = azurerm_storage_account.function.name
  storage_account_access_key = azurerm_storage_account.function.primary_access_key
  service_plan_id            = azurerm_service_plan.main.id
  
  site_config {
    application_stack {
      python_version = "3.11"
    }
    
    application_insights_connection_string = azurerm_application_insights.main.connection_string
    application_insights_key              = azurerm_application_insights.main.instrumentation_key
    
    cors {
      allowed_origins = ["https://portal.azure.com"]
    }
  }
  
  app_settings = {
    "OPENAI_API_KEY"                       = var.openai_api_key
    "STORAGE_ACCOUNT_NAME"                 = data.azurerm_storage_account.main.name
    "STORAGE_CONTAINER_NAME"               = "docs"
    "AzureWebJobsFeatureFlags"            = "EnableWorkerIndexing"
    "FUNCTIONS_WORKER_RUNTIME"            = "python"
    "WEBSITE_RUN_FROM_PACKAGE"            = "1"
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  tags = local.tags
}

# Grant Function App access to Storage Account
resource "azurerm_role_assignment" "function_storage_access" {
  scope                = data.azurerm_storage_account.main.id
  role_definition_name = "Storage Blob Data Contributor"
  principal_id         = azurerm_linux_function_app.main.identity[0].principal_id
}

# terraform-app/outputs.tf
output "function_app_name" {
  description = "The name of the function app"
  value       = azurerm_linux_function_app.main.name
}

output "function_app_default_hostname" {
  description = "The default hostname of the function app"
  value       = azurerm_linux_function_app.main.default_hostname
}

output "function_app_id" {
  description = "The ID of the function app"
  value       = azurerm_linux_function_app.main.id
}

output "app_service_plan_id" {
  description = "The ID of the app service plan"
  value       = azurerm_service_plan.main.id
}

output "application_insights_id" {
  description = "The ID of Application Insights"
  value       = azurerm_application_insights.main.id
}

output "application_insights_instrumentation_key" {
  description = "The instrumentation key for Application Insights"
  value       = azurerm_application_insights.main.instrumentation_key
  sensitive   = true
}

# terraform-app/environments/dev.tfvars
environment = "dev"
location    = "eastus"

tags = {
  Environment = "Development"
  CostCenter  = "Engineering"
  Owner       = "DevTeam"
}

# terraform-app/environments/qa.tfvars
environment = "qa"
location    = "eastus"

tags = {
  Environment = "QA"
  CostCenter  = "Quality"
  Owner       = "QATeam"
}

# terraform-app/environments/prod.tfvars
environment = "prod"
location    = "eastus"

tags = {
  Environment = "Production"
  CostCenter  = "Operations"
  Owner       = "OpsTeam"
  Compliance  = "Required"
  SLA         = "99.9"
}
```

## Shared Resources

### Initialization Scripts

```bash
# shared/scripts/init.sh
#!/bin/bash
set -e

PROJECT=$1
ENVIRONMENT=$2

if [ -z "$PROJECT" ] || [ -z "$ENVIRONMENT" ]; then
    echo "Usage: ./init.sh <project> <environment>"
    echo "Example: ./init.sh terraform-storage dev"
    exit 1
fi

cd "../$PROJECT"

terraform init \
    -backend-config="key=${PROJECT}-${ENVIRONMENT}.tfstate" \
    -reconfigure

echo "Terraform initialized for $PROJECT in $ENVIRONMENT environment"

# shared/scripts/plan.sh
#!/bin/bash
set -e

PROJECT=$1
ENVIRONMENT=$2

if [ -z "$PROJECT" ] || [ -z "$ENVIRONMENT" ]; then
    echo "Usage: ./plan.sh <project> <environment>"
    exit 1
fi

cd "../$PROJECT"

terraform plan \
    -var-file="environments/${ENVIRONMENT}.tfvars" \
    -out="${PROJECT}-${ENVIRONMENT}.tfplan"

echo "Terraform plan created for $PROJECT in $ENVIRONMENT environment"

# shared/scripts/apply.sh
#!/bin/bash
set -e

PROJECT=$1
ENVIRONMENT=$2

if [ -z "$PROJECT" ] || [ -z "$ENVIRONMENT" ]; then
    echo "Usage: ./apply.sh <project> <environment>"
    exit 1
fi

cd "../$PROJECT"

if [ ! -f "${PROJECT}-${ENVIRONMENT}.tfplan" ]; then
    echo "No plan file found. Run plan.sh first."
    exit 1
fi

terraform apply "${PROJECT}-${ENVIRONMENT}.tfplan"

echo "Terraform apply completed for $PROJECT in $ENVIRONMENT environment"
```

### Makefile for Convenience

```makefile
# infrastructure/Makefile
.PHONY: help init-storage init-app plan-storage plan-app apply-storage apply-app

ENVIRONMENT ?= dev

help:
	@echo "Available targets:"
	@echo "  init-storage    - Initialize terraform-storage project"
	@echo "  init-app        - Initialize terraform-app project"
	@echo "  plan-storage    - Plan terraform-storage changes"
	@echo "  plan-app        - Plan terraform-app changes"
	@echo "  apply-storage   - Apply terraform-storage changes"
	@echo "  apply-app       - Apply terraform-app changes"
	@echo ""
	@echo "Set ENVIRONMENT variable (default: dev)"
	@echo "Example: make plan-storage ENVIRONMENT=prod"

init-storage:
	@cd shared/scripts && ./init.sh terraform-storage $(ENVIRONMENT)

init-app:
	@cd shared/scripts && ./init.sh terraform-app $(ENVIRONMENT)

plan-storage:
	@cd shared/scripts && ./plan.sh terraform-storage $(ENVIRONMENT)

plan-app:
	@cd shared/scripts && ./plan.sh terraform-app $(ENVIRONMENT)

apply-storage:
	@cd shared/scripts && ./apply.sh terraform-storage $(ENVIRONMENT)

apply-app:
	@cd shared/scripts && ./apply.sh terraform-app $(ENVIRONMENT)

fmt:
	@terraform fmt -recursive .

validate-storage:
	@cd terraform-storage && terraform validate

validate-app:
	@cd terraform-app && terraform validate

clean:
	@find . -type f -name "*.tfplan" -delete
	@find . -type f -name "*.tfstate*" -delete
	@find . -type d -name ".terraform" -exec rm -rf {} +
```

## .gitignore Configuration

```gitignore
# infrastructure/.gitignore

# Terraform files
*.tfstate
*.tfstate.*
*.tfplan
.terraform/
.terraform.lock.hcl

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# CLI configuration files
.terraformrc
terraform.rc

# Environment variables
.env
*.env

# IDE files
.idea/
.vscode/
*.swp
*.swo
*~

# OS files
.DS_Store
Thumbs.db

# Crash log files
crash.log
crash.*.log

# Sensitive variable files
*.auto.tfvars
terraform.tfvars

# Ignore any .pem files
*.pem
*.key
*.crt
```

## Deployment Order

### Initial Setup Sequence

1. **Deploy Storage Infrastructure First**
   ```bash
   cd infrastructure/terraform-storage
   terraform init -backend-config="key=storage-dev.tfstate"
   terraform plan -var-file="environments/dev.tfvars"
   terraform apply -var-file="environments/dev.tfvars"
   ```

2. **Deploy Application Infrastructure**
   ```bash
   cd infrastructure/terraform-app
   terraform init -backend-config="key=app-dev.tfstate"
   export TF_VAR_openai_api_key="your-api-key"
   terraform plan -var-file="environments/dev.tfvars"
   terraform apply -var-file="environments/dev.tfvars"
   ```

### Multi-Environment Deployment

```bash
# Deploy to all environments
for env in dev qa prod; do
    echo "Deploying storage to $env"
    cd infrastructure/terraform-storage
    terraform init -backend-config="key=storage-${env}.tfstate"
    terraform apply -var-file="environments/${env}.tfvars" -auto-approve
    
    echo "Deploying app to $env"
    cd ../terraform-app
    terraform init -backend-config="key=app-${env}.tfstate"
    terraform apply -var-file="environments/${env}.tfvars" -auto-approve
    cd ..
done
```

## Best Practices Implementation

### 1. State Management
- Separate state files per project and environment
- Remote backend with locking enabled
- Regular state backups

### 2. Security
- Sensitive variables marked with `sensitive = true`
- Secrets passed via environment variables
- Managed identities for Azure resource access

### 3. Naming Conventions
- Consistent naming: `{resource}-{app}-{env}`
- Resource tags for organization
- Descriptive variable names

### 4. Module Design
- Single responsibility per module
- Clear input/output contracts
- Version pinning for stability

### 5. Documentation
- README in each project directory
- Inline comments for complex logic
- Variable descriptions

This structure provides a solid foundation for managing infrastructure as code with proper separation of concerns, environment management, and deployment automation.