# Technical Specifications - Azure Document AI Chat Infrastructure

## Module Specifications

### Storage Module (`modules/storage`)

#### Purpose
Provision and configure Azure Storage Account with blob containers for document storage.

#### Input Variables
```hcl
variable "resource_group_name" {
  description = "Name of the resource group"
  type        = string
}

variable "location" {
  description = "Azure region for resources"
  type        = string
}

variable "storage_account_name" {
  description = "Name of the storage account"
  type        = string
}

variable "environment" {
  description = "Environment name (dev, qa, prod)"
  type        = string
}

variable "replication_type" {
  description = "Storage replication type"
  type        = string
  default     = "LRS"
}

variable "containers" {
  description = "List of blob containers to create"
  type        = list(string)
  default     = ["docs"]
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
}
```

#### Outputs
```hcl
output "storage_account_id" {
  description = "Storage account resource ID"
  value       = azurerm_storage_account.main.id
}

output "primary_connection_string" {
  description = "Primary connection string"
  value       = azurerm_storage_account.main.primary_connection_string
  sensitive   = true
}

output "primary_blob_endpoint" {
  description = "Primary blob endpoint"
  value       = azurerm_storage_account.main.primary_blob_endpoint
}

output "container_names" {
  description = "Created container names"
  value       = values(azurerm_storage_container.containers)[*].name
}
```

### Function App Module (`modules/function-app`)

#### Purpose
Deploy Azure Function App with Python runtime for document processing API.

#### Input Variables
```hcl
variable "resource_group_name" {
  description = "Name of the resource group"
  type        = string
}

variable "location" {
  description = "Azure region for resources"
  type        = string
}

variable "function_app_name" {
  description = "Name of the function app"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "storage_account_name" {
  description = "Storage account for function runtime"
  type        = string
}

variable "storage_account_key" {
  description = "Storage account access key"
  type        = string
  sensitive   = true
}

variable "app_service_plan_id" {
  description = "App Service Plan ID (optional for consumption plan)"
  type        = string
  default     = null
}

variable "runtime_version" {
  description = "Python runtime version"
  type        = string
  default     = "3.11"
}

variable "app_settings" {
  description = "Application settings"
  type        = map(string)
  default     = {}
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
}
```

#### Outputs
```hcl
output "function_app_id" {
  description = "Function app resource ID"
  value       = azurerm_linux_function_app.main.id
}

output "default_hostname" {
  description = "Default hostname"
  value       = azurerm_linux_function_app.main.default_hostname
}

output "outbound_ip_addresses" {
  description = "Outbound IP addresses"
  value       = split(",", azurerm_linux_function_app.main.outbound_ip_addresses)
}

output "identity_principal_id" {
  description = "Managed identity principal ID"
  value       = azurerm_linux_function_app.main.identity[0].principal_id
}
```

## GitHub Actions Workflow Specifications

### Main Deployment Workflow

```yaml
name: Deploy Infrastructure

on:
  workflow_dispatch:
    inputs:
      terraform_version:
        description: 'Terraform Version'
        required: false
        default: '1.12.2'
        type: string
      environment:
        description: 'Target Environment'
        required: true
        type: choice
        options:
          - dev
          - qa
          - prod
        default: 'dev'
      module:
        description: 'Module to Deploy'
        required: true
        type: choice
        options:
          - all
          - storage
          - function-app
        default: 'all'
      apply_changes:
        description: 'Apply Changes (false = plan only)'
        required: true
        type: boolean
        default: false

env:
  TERRAFORM_VERSION: ${{ inputs.terraform_version }}
  ENVIRONMENT: ${{ inputs.environment }}
  MODULE: ${{ inputs.module }}
  APPLY: ${{ inputs.apply_changes }}

jobs:
  terraform-plan:
    name: Terraform Plan
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}
          
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: |
            {
              "clientId": "${{ secrets.AZURE_CLIENT_ID }}",
              "clientSecret": "${{ secrets.AZURE_CLIENT_SECRET }}",
              "subscriptionId": "${{ secrets.AZURE_SUBSCRIPTION_ID }}",
              "tenantId": "${{ secrets.AZURE_TENANT_ID }}"
            }
            
      - name: Terraform Init
        working-directory: ./infrastructure/environments/${{ env.ENVIRONMENT }}
        run: |
          terraform init \
            -backend-config="resource_group_name=Terraform-Backend-State-RG" \
            -backend-config="storage_account_name=aspterraformstate" \
            -backend-config="container_name=tfstate" \
            -backend-config="key=docqa-${{ env.MODULE }}-${{ env.ENVIRONMENT }}.tfstate"
            
      - name: Terraform Validate
        working-directory: ./infrastructure/environments/${{ env.ENVIRONMENT }}
        run: terraform validate
        
      - name: Terraform Plan
        working-directory: ./infrastructure/environments/${{ env.ENVIRONMENT }}
        run: |
          terraform plan \
            -var="openai_api_key=${{ secrets.OPENAI_API_KEY }}" \
            -out=tfplan
            
      - name: Upload Plan
        uses: actions/upload-artifact@v3
        with:
          name: terraform-plan-${{ env.ENVIRONMENT }}
          path: ./infrastructure/environments/${{ env.ENVIRONMENT }}/tfplan
          
  terraform-apply:
    name: Terraform Apply
    needs: terraform-plan
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    if: ${{ inputs.apply_changes == true }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        
      - name: Download Plan
        uses: actions/download-artifact@v3
        with:
          name: terraform-plan-${{ env.ENVIRONMENT }}
          path: ./infrastructure/environments/${{ env.ENVIRONMENT }}
          
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}
          
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: |
            {
              "clientId": "${{ secrets.AZURE_CLIENT_ID }}",
              "clientSecret": "${{ secrets.AZURE_CLIENT_SECRET }}",
              "subscriptionId": "${{ secrets.AZURE_SUBSCRIPTION_ID }}",
              "tenantId": "${{ secrets.AZURE_TENANT_ID }}"
            }
            
      - name: Terraform Init
        working-directory: ./infrastructure/environments/${{ env.ENVIRONMENT }}
        run: |
          terraform init \
            -backend-config="resource_group_name=Terraform-Backend-State-RG" \
            -backend-config="storage_account_name=aspterraformstate" \
            -backend-config="container_name=tfstate" \
            -backend-config="key=docqa-${{ env.MODULE }}-${{ env.ENVIRONMENT }}.tfstate"
            
      - name: Terraform Apply
        working-directory: ./infrastructure/environments/${{ env.ENVIRONMENT }}
        run: terraform apply tfplan
        
      - name: Generate Output Summary
        working-directory: ./infrastructure/environments/${{ env.ENVIRONMENT }}
        run: |
          echo "## Deployment Summary" >> $GITHUB_STEP_SUMMARY
          echo "Environment: ${{ env.ENVIRONMENT }}" >> $GITHUB_STEP_SUMMARY
          echo "Module: ${{ env.MODULE }}" >> $GITHUB_STEP_SUMMARY
          echo "### Outputs" >> $GITHUB_STEP_SUMMARY
          terraform output -json | jq -r 'to_entries[] | "- **\(.key)**: \(.value.value)"' >> $GITHUB_STEP_SUMMARY
```

### Destroy Workflow

```yaml
name: Destroy Infrastructure

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target Environment'
        required: true
        type: choice
        options:
          - dev
          - qa
          - prod
      confirm_destroy:
        description: 'Type DESTROY to confirm'
        required: true
        type: string

jobs:
  destroy:
    name: Terraform Destroy
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    if: ${{ inputs.confirm_destroy == 'DESTROY' }}
    
    steps:
      - name: Checkout Code
        uses: actions/checkout@v4
        
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.12.2'
          
      - name: Azure Login
        uses: azure/login@v1
        with:
          creds: |
            {
              "clientId": "${{ secrets.AZURE_CLIENT_ID }}",
              "clientSecret": "${{ secrets.AZURE_CLIENT_SECRET }}",
              "subscriptionId": "${{ secrets.AZURE_SUBSCRIPTION_ID }}",
              "tenantId": "${{ secrets.AZURE_TENANT_ID }}"
            }
            
      - name: Terraform Init
        working-directory: ./infrastructure/environments/${{ inputs.environment }}
        run: |
          terraform init \
            -backend-config="resource_group_name=Terraform-Backend-State-RG" \
            -backend-config="storage_account_name=aspterraformstate" \
            -backend-config="container_name=tfstate" \
            -backend-config="key=docqa-all-${{ inputs.environment }}.tfstate"
            
      - name: Terraform Destroy
        working-directory: ./infrastructure/environments/${{ inputs.environment }}
        run: |
          terraform destroy \
            -var="openai_api_key=${{ secrets.OPENAI_API_KEY }}" \
            -auto-approve
```

## Environment Configuration Files

### Development Environment (`environments/dev/terraform.tfvars`)

```hcl
# Environment Configuration
environment = "dev"
location    = "eastus"

# Resource Group
resource_group_name = "doc-qa-rg-dev"

# Storage Account
storage_account_name     = "aspdevdocqadev"
storage_replication_type = "LRS"
storage_containers      = ["docs", "temp", "logs"]

# Function App
function_app_name   = "doc-qa-api-dev"
app_service_plan_sku = "Y1"
python_version      = "3.11"

# Networking
allowed_ip_ranges = [
  "0.0.0.0/0" # Open in dev for testing
]

# Tags
tags = {
  Environment        = "Development"
  Project           = "DocQA"
  ManagedBy         = "Terraform"
  CostCenter        = "Engineering"
  DataClassification = "Internal"
}

# Monitoring
enable_application_insights = true
log_retention_days         = 30
```

### QA Environment (`environments/qa/terraform.tfvars`)

```hcl
# Environment Configuration
environment = "qa"
location    = "eastus"

# Resource Group
resource_group_name = "doc-qa-rg-qa"

# Storage Account
storage_account_name     = "aspdevdocqaqa"
storage_replication_type = "LRS"
storage_containers      = ["docs", "temp", "logs", "test-data"]

# Function App
function_app_name   = "doc-qa-api-qa"
app_service_plan_sku = "Y1"
python_version      = "3.11"

# Networking
allowed_ip_ranges = [
  "10.0.0.0/8",     # Corporate network
  "172.16.0.0/12"   # VPN range
]

# Tags
tags = {
  Environment        = "QA"
  Project           = "DocQA"
  ManagedBy         = "Terraform"
  CostCenter        = "Engineering"
  DataClassification = "Internal"
}

# Monitoring
enable_application_insights = true
log_retention_days         = 60
```

### Production Environment (`environments/prod/terraform.tfvars`)

```hcl
# Environment Configuration
environment = "prod"
location    = "eastus"

# Resource Group
resource_group_name = "doc-qa-rg-prod"

# Storage Account
storage_account_name     = "aspdevdocqaprod"
storage_replication_type = "GRS"
storage_containers      = ["docs", "archive", "logs", "backup"]

# Function App
function_app_name   = "doc-qa-api-prod"
app_service_plan_sku = "EP1"
python_version      = "3.11"

# Networking
allowed_ip_ranges = [
  "10.0.0.0/8"      # Corporate network only
]

# High Availability
enable_zone_redundancy = true
minimum_instance_count = 2
maximum_instance_count = 10

# Tags
tags = {
  Environment        = "Production"
  Project           = "DocQA"
  ManagedBy         = "Terraform"
  CostCenter        = "Operations"
  DataClassification = "Confidential"
  Compliance        = "SOC2"
}

# Monitoring
enable_application_insights = true
log_retention_days         = 90
enable_diagnostics         = true
metrics_retention_days     = 365
```

## Security Configuration

### Key Vault Integration

```hcl
resource "azurerm_key_vault" "main" {
  name                = "kv-docqa-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  tenant_id          = data.azurerm_client_config.current.tenant_id
  sku_name           = "standard"

  access_policy {
    tenant_id = data.azurerm_client_config.current.tenant_id
    object_id = azurerm_linux_function_app.main.identity[0].principal_id

    secret_permissions = [
      "Get",
      "List"
    ]
  }

  network_acls {
    default_action = "Deny"
    bypass        = "AzureServices"
    
    ip_rules = var.allowed_ip_ranges
  }

  tags = var.tags
}

resource "azurerm_key_vault_secret" "openai_key" {
  name         = "openai-api-key"
  value        = var.openai_api_key
  key_vault_id = azurerm_key_vault.main.id
}
```

### Network Security Group

```hcl
resource "azurerm_network_security_group" "function_app" {
  name                = "nsg-${var.function_app_name}"
  location            = var.location
  resource_group_name = var.resource_group_name

  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefixes    = var.allowed_ip_ranges
    destination_address_prefix = "*"
  }

  security_rule {
    name                       = "DenyAllInbound"
    priority                   = 200
    direction                  = "Inbound"
    access                     = "Deny"
    protocol                   = "*"
    source_port_range          = "*"
    destination_port_range     = "*"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = var.tags
}
```

## Monitoring Configuration

### Application Insights

```hcl
resource "azurerm_application_insights" "main" {
  name                = "ai-${var.function_app_name}"
  location            = var.location
  resource_group_name = var.resource_group_name
  application_type    = "web"
  retention_in_days   = var.log_retention_days

  tags = var.tags
}

resource "azurerm_monitor_diagnostic_setting" "function_app" {
  name               = "diag-${var.function_app_name}"
  target_resource_id = azurerm_linux_function_app.main.id

  log_analytics_workspace_id = azurerm_log_analytics_workspace.main.id

  log {
    category = "FunctionAppLogs"
    enabled  = true

    retention_policy {
      enabled = true
      days    = var.log_retention_days
    }
  }

  metric {
    category = "AllMetrics"
    enabled  = true

    retention_policy {
      enabled = true
      days    = var.metrics_retention_days
    }
  }
}
```

### Alerts Configuration

```hcl
resource "azurerm_monitor_metric_alert" "function_app_errors" {
  name                = "alert-${var.function_app_name}-errors"
  resource_group_name = var.resource_group_name
  scopes              = [azurerm_linux_function_app.main.id]
  description         = "Alert when function app errors exceed threshold"

  criteria {
    metric_namespace = "Microsoft.Web/sites"
    metric_name      = "FunctionExecutionErrors"
    aggregation      = "Total"
    operator         = "GreaterThan"
    threshold        = 10
  }

  window_size        = "PT5M"
  frequency          = "PT1M"
  severity           = 2
  auto_mitigate      = true

  action {
    action_group_id = azurerm_monitor_action_group.main.id
  }

  tags = var.tags
}
```

## Cost Optimization

### Budget Configuration

```hcl
resource "azurerm_consumption_budget_resource_group" "main" {
  name              = "budget-${var.resource_group_name}"
  resource_group_id = azurerm_resource_group.main.id

  amount     = var.monthly_budget
  time_grain = "Monthly"

  time_period {
    start_date = formatdate("YYYY-MM-01'T'00:00:00Z", timestamp())
  }

  notification {
    enabled   = true
    threshold = 80
    operator  = "GreaterThan"

    contact_emails = var.budget_alert_emails
  }

  notification {
    enabled   = true
    threshold = 100
    operator  = "GreaterThan"

    contact_emails = var.budget_alert_emails
  }
}
```

### Auto-scaling Configuration

```hcl
resource "azurerm_monitor_autoscale_setting" "function_app" {
  count               = var.environment == "prod" ? 1 : 0
  name                = "autoscale-${var.function_app_name}"
  resource_group_name = var.resource_group_name
  location            = var.location
  target_resource_id  = azurerm_service_plan.main.id

  profile {
    name = "default"

    capacity {
      default = 2
      minimum = var.minimum_instance_count
      maximum = var.maximum_instance_count
    }

    rule {
      metric_trigger {
        metric_name        = "CpuPercentage"
        metric_resource_id = azurerm_service_plan.main.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "GreaterThan"
        threshold          = 70
      }

      scale_action {
        direction = "Increase"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT5M"
      }
    }

    rule {
      metric_trigger {
        metric_name        = "CpuPercentage"
        metric_resource_id = azurerm_service_plan.main.id
        time_grain         = "PT1M"
        statistic          = "Average"
        time_window        = "PT5M"
        time_aggregation   = "Average"
        operator           = "LessThan"
        threshold          = 30
      }

      scale_action {
        direction = "Decrease"
        type      = "ChangeCount"
        value     = "1"
        cooldown  = "PT10M"
      }
    }
  }

  notification {
    email {
      send_to_subscription_administrator    = true
      send_to_subscription_co_administrators = true
      custom_emails                         = var.scaling_alert_emails
    }
  }
}
```

---
*Document Version: 1.0*  
*Last Updated: 2025-01-12*