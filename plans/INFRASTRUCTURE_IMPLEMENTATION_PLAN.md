# Infrastructure Implementation Plan
## Azure Infrastructure with Terraform

### Executive Summary
This plan details the infrastructure implementation for the document-grounded Q&A application using Terraform to provision and manage Azure resources across multiple environments (dev, qa, prod).

### Infrastructure Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Azure Subscription                        │
├──────────────────────────────────────────────────────────────┤
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐ │
│  │  Dev Resource   │  │  QA Resource    │  │ Prod Resource  │ │
│  │     Group       │  │     Group       │  │    Group       │ │
│  └────────┬────────┘  └────────┬────────┘  └────────┬───────┘ │
│           │                    │                     │         │
│  ┌────────▼────────────────────▼─────────────────────▼───────┐│
│  │                    Shared Services                         ││
│  │  - Azure Key Vault (Secrets Management)                   ││
│  │  - Azure Monitor (Logging & Monitoring)                   ││
│  │  - Azure DevOps (CI/CD Pipelines)                        ││
│  └────────────────────────────────────────────────────────────┘│
└──────────────────────────────────────────────────────────────┘
```

### Phase 1: Terraform Foundation (Week 1)

#### 1.1 Directory Structure
```
infrastructure/
├── environments/
│   ├── dev/
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── qa/
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   └── prod/
│       ├── terraform.tfvars
│       └── backend.tf
├── modules/
│   ├── resource_group/
│   ├── storage/
│   ├── compute/
│   ├── networking/
│   ├── database/
│   ├── security/
│   ├── monitoring/
│   └── ai_services/
├── shared/
│   └── main.tf
├── main.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

#### 1.2 Backend Configuration
```hcl
# backend.tf - State management
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstatedocqa${var.environment}"
    container_name       = "tfstate"
    key                  = "${var.environment}.terraform.tfstate"
    
    # Enable state locking
    use_oidc            = true
    subscription_id     = var.subscription_id
  }
}

# versions.tf - Provider versions
terraform {
  required_version = ">= 1.5.0"
  
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.85.0"
    }
    azuread = {
      source  = "hashicorp/azuread"
      version = "~> 2.47.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6.0"
    }
  }
}
```

### Phase 2: Core Infrastructure Modules (Week 1-2)

#### 2.1 Resource Group Module
```hcl
# modules/resource_group/main.tf
resource "azurerm_resource_group" "main" {
  name     = "rg-docqa-${var.environment}-${var.location_short}"
  location = var.location
  
  tags = merge(
    var.common_tags,
    {
      environment = var.environment
      managed_by  = "terraform"
    }
  )
}

# modules/resource_group/variables.tf
variable "environment" {
  description = "Environment name (dev, qa, prod)"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
  default     = "East US"
}

variable "location_short" {
  description = "Short location code"
  type        = string
  default     = "eus"
}
```

#### 2.2 Networking Module
```hcl
# modules/networking/main.tf
resource "azurerm_virtual_network" "main" {
  name                = "vnet-docqa-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = var.address_space
  
  dns_servers = var.dns_servers
  
  tags = var.tags
}

resource "azurerm_subnet" "app" {
  name                 = "snet-app-${var.environment}"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [cidrsubnet(var.address_space[0], 4, 0)]
  
  service_endpoints = [
    "Microsoft.Storage",
    "Microsoft.KeyVault",
    "Microsoft.Sql"
  ]
}

resource "azurerm_subnet" "data" {
  name                 = "snet-data-${var.environment}"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.main.name
  address_prefixes     = [cidrsubnet(var.address_space[0], 4, 1)]
  
  delegation {
    name = "databricks"
    service_delegation {
      name = "Microsoft.Databricks/workspaces"
      actions = [
        "Microsoft.Network/virtualNetworks/subnets/join/action",
        "Microsoft.Network/virtualNetworks/subnets/prepareNetworkPolicies/action"
      ]
    }
  }
}

resource "azurerm_network_security_group" "main" {
  name                = "nsg-docqa-${var.environment}"
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
    source_address_prefix      = "Internet"
    destination_address_prefix = "*"
  }
  
  tags = var.tags
}
```

### Phase 3: Storage Infrastructure (Week 2)

#### 3.1 Blob Storage Module
```hcl
# modules/storage/main.tf
resource "azurerm_storage_account" "main" {
  name                     = "stdocqa${var.environment}${random_string.storage_suffix.result}"
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = var.account_tier
  account_replication_type = var.replication_type
  
  # Security settings
  min_tls_version                 = "TLS1_2"
  enable_https_traffic_only       = true
  allow_nested_items_to_be_public = false
  
  # Network rules
  network_rules {
    default_action             = "Deny"
    bypass                     = ["AzureServices"]
    virtual_network_subnet_ids = var.allowed_subnet_ids
    ip_rules                   = var.allowed_ip_ranges
  }
  
  # Blob properties
  blob_properties {
    versioning_enabled = true
    
    delete_retention_policy {
      days = var.environment == "prod" ? 30 : 7
    }
    
    container_delete_retention_policy {
      days = var.environment == "prod" ? 30 : 7
    }
  }
  
  # Lifecycle management
  lifecycle_rule {
    enabled = true
    
    blob {
      delete_after_days = var.environment == "prod" ? 365 : 90
      tier_to_cool_after_days = 30
      tier_to_archive_after_days = var.environment == "prod" ? 180 : 60
    }
  }
  
  tags = var.tags
}

# Storage containers
resource "azurerm_storage_container" "documents" {
  name                  = "documents"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "processed" {
  name                  = "processed"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}

resource "azurerm_storage_container" "embeddings" {
  name                  = "embeddings"
  storage_account_name  = azurerm_storage_account.main.name
  container_access_type = "private"
}
```

### Phase 4: Compute Infrastructure (Week 2-3)

#### 4.1 App Service Module
```hcl
# modules/compute/app_service.tf
resource "azurerm_service_plan" "main" {
  name                = "asp-docqa-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  os_type             = "Linux"
  sku_name            = var.sku_name
  
  # Auto-scaling for production
  dynamic "auto_scale_rule" {
    for_each = var.environment == "prod" ? [1] : []
    content {
      min_instances = 2
      max_instances = 10
      
      scale_rule {
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
    }
  }
  
  tags = var.tags
}

resource "azurerm_linux_web_app" "backend" {
  name                = "app-docqa-api-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  service_plan_id     = azurerm_service_plan.main.id
  
  site_config {
    always_on = var.environment == "prod" ? true : false
    
    application_stack {
      node_version = "18-lts"
    }
    
    cors {
      allowed_origins = var.allowed_origins
      support_credentials = true
    }
    
    ip_restriction {
      name       = "AllowFrontend"
      priority   = 100
      action     = "Allow"
      service_tag = "AzureFrontDoor.Backend"
    }
  }
  
  app_settings = {
    "AZURE_STORAGE_CONNECTION_STRING" = azurerm_storage_account.main.primary_connection_string
    "AZURE_SEARCH_ENDPOINT"           = azurerm_search_service.main.endpoint
    "AZURE_OPENAI_ENDPOINT"           = azurerm_cognitive_account.openai.endpoint
    "KEY_VAULT_URI"                   = azurerm_key_vault.main.vault_uri
    "APPLICATION_INSIGHTS_KEY"        = azurerm_application_insights.main.instrumentation_key
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  tags = var.tags
}

# Azure Functions for async processing
resource "azurerm_function_app" "processor" {
  name                       = "func-docqa-processor-${var.environment}"
  location                   = var.location
  resource_group_name        = var.resource_group_name
  app_service_plan_id        = azurerm_service_plan.main.id
  storage_account_name       = azurerm_storage_account.main.name
  storage_account_access_key = azurerm_storage_account.main.primary_access_key
  
  app_settings = {
    "FUNCTIONS_WORKER_RUNTIME"     = "python"
    "PYTHON_VERSION"               = "3.11"
    "AzureWebJobsStorage"          = azurerm_storage_account.main.primary_connection_string
    "AZURE_OPENAI_KEY"            = "@Microsoft.KeyVault(VaultName=${azurerm_key_vault.main.name};SecretName=openai-key)"
    "DOCUMENT_PROCESSING_QUEUE"    = azurerm_storage_queue.processing.name
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  tags = var.tags
}
```

### Phase 5: Database & Search Infrastructure (Week 3)

#### 5.1 Azure AI Search Module
```hcl
# modules/ai_services/search.tf
resource "azurerm_search_service" "main" {
  name                = "srch-docqa-${var.environment}-${random_string.search_suffix.result}"
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = var.search_sku
  replica_count       = var.environment == "prod" ? 2 : 1
  partition_count     = var.environment == "prod" ? 2 : 1
  
  public_network_access_enabled = false
  
  identity {
    type = "SystemAssigned"
  }
  
  tags = var.tags
}

# Private endpoint for search service
resource "azurerm_private_endpoint" "search" {
  name                = "pe-search-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  subnet_id           = var.subnet_id
  
  private_service_connection {
    name                           = "psc-search-${var.environment}"
    private_connection_resource_id = azurerm_search_service.main.id
    subresource_names              = ["searchService"]
    is_manual_connection           = false
  }
  
  tags = var.tags
}
```

#### 5.2 Cosmos DB Module
```hcl
# modules/database/cosmosdb.tf
resource "azurerm_cosmosdb_account" "main" {
  name                = "cosmos-docqa-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"
  
  consistency_policy {
    consistency_level       = "Session"
    max_interval_in_seconds = 10
    max_staleness_prefix    = 200
  }
  
  # Multi-region for production
  dynamic "geo_location" {
    for_each = var.environment == "prod" ? var.geo_locations : [var.primary_location]
    content {
      location          = geo_location.value.location
      failover_priority = geo_location.value.priority
      zone_redundant    = var.environment == "prod" ? true : false
    }
  }
  
  # Security
  is_virtual_network_filter_enabled = true
  ip_range_filter                   = join(",", var.allowed_ip_ranges)
  
  virtual_network_rule {
    id = var.subnet_id
  }
  
  # Backup
  backup {
    type                = "Continuous"
    redundancy         = var.environment == "prod" ? "Geo" : "Local"
    retention_in_hours = var.environment == "prod" ? 720 : 168
  }
  
  tags = var.tags
}

resource "azurerm_cosmosdb_sql_database" "main" {
  name                = "docqa-db"
  resource_group_name = var.resource_group_name
  account_name        = azurerm_cosmosdb_account.main.name
  
  autoscale_settings {
    max_throughput = var.environment == "prod" ? 4000 : 1000
  }
}

# Collections
resource "azurerm_cosmosdb_sql_container" "documents" {
  name                = "documents"
  resource_group_name = var.resource_group_name
  account_name        = azurerm_cosmosdb_account.main.name
  database_name       = azurerm_cosmosdb_sql_database.main.name
  partition_key_path  = "/documentId"
  
  autoscale_settings {
    max_throughput = var.environment == "prod" ? 4000 : 1000
  }
  
  indexing_policy {
    indexing_mode = "consistent"
    
    included_path {
      path = "/*"
    }
    
    excluded_path {
      path = "/embeddings/*"
    }
  }
}
```

### Phase 6: AI Services Infrastructure (Week 3-4)

#### 6.1 Azure OpenAI Service
```hcl
# modules/ai_services/openai.tf
resource "azurerm_cognitive_account" "openai" {
  name                = "oai-docqa-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  kind                = "OpenAI"
  sku_name            = "S0"
  
  custom_subdomain_name = "docqa-${var.environment}"
  
  network_acls {
    default_action = "Deny"
    
    virtual_network_rules {
      subnet_id = var.subnet_id
    }
    
    ip_rules = var.allowed_ip_ranges
  }
  
  identity {
    type = "SystemAssigned"
  }
  
  tags = var.tags
}

# OpenAI Deployments
resource "azurerm_cognitive_deployment" "gpt4" {
  name                 = "gpt-4"
  cognitive_account_id = azurerm_cognitive_account.openai.id
  
  model {
    format  = "OpenAI"
    name    = "gpt-4"
    version = "0613"
  }
  
  scale {
    type     = "Standard"
    capacity = var.environment == "prod" ? 120 : 20
  }
}

resource "azurerm_cognitive_deployment" "embeddings" {
  name                 = "text-embedding-ada-002"
  cognitive_account_id = azurerm_cognitive_account.openai.id
  
  model {
    format  = "OpenAI"
    name    = "text-embedding-ada-002"
    version = "2"
  }
  
  scale {
    type     = "Standard"
    capacity = var.environment == "prod" ? 240 : 60
  }
}
```

### Phase 7: Security Infrastructure (Week 4)

#### 7.1 Key Vault Module
```hcl
# modules/security/keyvault.tf
resource "azurerm_key_vault" "main" {
  name                        = "kv-docqa-${var.environment}-${random_string.kv_suffix.result}"
  location                    = var.location
  resource_group_name         = var.resource_group_name
  enabled_for_disk_encryption = true
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  soft_delete_retention_days  = var.environment == "prod" ? 90 : 7
  purge_protection_enabled    = var.environment == "prod" ? true : false
  sku_name                    = "standard"
  
  network_acls {
    default_action             = "Deny"
    bypass                     = "AzureServices"
    virtual_network_subnet_ids = [var.subnet_id]
    ip_rules                   = var.allowed_ip_ranges
  }
  
  tags = var.tags
}

# Access policies
resource "azurerm_key_vault_access_policy" "app_service" {
  key_vault_id = azurerm_key_vault.main.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = azurerm_linux_web_app.backend.identity[0].principal_id
  
  secret_permissions = [
    "Get",
    "List"
  ]
}

# Secrets
resource "azurerm_key_vault_secret" "openai_key" {
  name         = "openai-key"
  value        = azurerm_cognitive_account.openai.primary_access_key
  key_vault_id = azurerm_key_vault.main.id
}

resource "azurerm_key_vault_secret" "storage_connection" {
  name         = "storage-connection-string"
  value        = azurerm_storage_account.main.primary_connection_string
  key_vault_id = azurerm_key_vault.main.id
}

resource "azurerm_key_vault_secret" "search_key" {
  name         = "search-admin-key"
  value        = azurerm_search_service.main.primary_key
  key_vault_id = azurerm_key_vault.main.id
}
```

#### 7.2 Azure AD B2C Configuration
```hcl
# modules/security/adb2c.tf
resource "azuread_application" "main" {
  display_name = "docqa-app-${var.environment}"
  
  web {
    homepage_url  = "https://docqa-${var.environment}.azurewebsites.net"
    redirect_uris = [
      "https://docqa-${var.environment}.azurewebsites.net/auth/callback",
      "http://localhost:3000/auth/callback"
    ]
    
    implicit_grant {
      access_token_issuance_enabled = true
      id_token_issuance_enabled     = true
    }
  }
  
  required_resource_access {
    resource_app_id = "00000003-0000-0000-c000-000000000000" # Microsoft Graph
    
    resource_access {
      id   = "e1fe6dd8-ba31-4d61-89e7-88639da4683d" # User.Read
      type = "Scope"
    }
  }
}

resource "azuread_service_principal" "main" {
  application_id = azuread_application.main.application_id
}

resource "azuread_application_password" "main" {
  application_object_id = azuread_application.main.object_id
  display_name          = "terraform-managed"
  end_date_relative     = "8760h" # 1 year
}
```

### Phase 8: Monitoring & Observability (Week 4-5)

#### 8.1 Application Insights Module
```hcl
# modules/monitoring/main.tf
resource "azurerm_log_analytics_workspace" "main" {
  name                = "log-docqa-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  sku                 = "PerGB2018"
  retention_in_days   = var.environment == "prod" ? 90 : 30
  
  tags = var.tags
}

resource "azurerm_application_insights" "main" {
  name                = "appi-docqa-${var.environment}"
  location            = var.location
  resource_group_name = var.resource_group_name
  workspace_id        = azurerm_log_analytics_workspace.main.id
  application_type    = "web"
  
  retention_in_days = var.environment == "prod" ? 90 : 30
  
  tags = var.tags
}

# Alerts
resource "azurerm_monitor_metric_alert" "high_response_time" {
  name                = "alert-high-response-time-${var.environment}"
  resource_group_name = var.resource_group_name
  scopes              = [azurerm_application_insights.main.id]
  description         = "Alert when response time exceeds threshold"
  severity            = 2
  frequency           = "PT5M"
  window_size         = "PT15M"
  
  criteria {
    metric_namespace = "Microsoft.Insights/components"
    metric_name      = "requests/duration"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = var.environment == "prod" ? 2000 : 5000
  }
  
  action {
    action_group_id = azurerm_monitor_action_group.main.id
  }
  
  tags = var.tags
}

resource "azurerm_monitor_action_group" "main" {
  name                = "ag-docqa-${var.environment}"
  resource_group_name = var.resource_group_name
  short_name          = "docqa"
  
  email_receiver {
    name          = "sendtodevops"
    email_address = var.alert_email
  }
  
  webhook_receiver {
    name        = "slack-webhook"
    service_uri = var.slack_webhook_url
  }
  
  tags = var.tags
}
```

### Phase 9: CI/CD Pipeline for Infrastructure (Week 5)

#### 9.1 Azure DevOps Pipeline
```yaml
# azure-pipelines-infrastructure.yml
trigger:
  branches:
    include:
      - main
      - develop
  paths:
    include:
      - infrastructure/*

variables:
  - group: terraform-vars
  - name: terraformVersion
    value: '1.5.0'

stages:
  - stage: Validate
    jobs:
      - job: TerraformValidate
        pool:
          vmImage: 'ubuntu-latest'
        steps:
          - task: TerraformInstaller@0
            inputs:
              terraformVersion: $(terraformVersion)
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Init'
            inputs:
              provider: 'azurerm'
              command: 'init'
              workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure'
              backendServiceArm: 'Azure-Service-Connection'
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Validate'
            inputs:
              provider: 'azurerm'
              command: 'validate'
              workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure'
          
          - task: TerraformTaskV4@4
            displayName: 'Terraform Plan'
            inputs:
              provider: 'azurerm'
              command: 'plan'
              workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure'
              environmentServiceNameAzureRM: 'Azure-Service-Connection'
              commandOptions: '-out=tfplan'

  - stage: DeployDev
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/develop'))
    jobs:
      - deployment: DeployInfrastructureDev
        environment: 'development'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformTaskV4@4
                  displayName: 'Terraform Apply - Dev'
                  inputs:
                    provider: 'azurerm'
                    command: 'apply'
                    workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure/environments/dev'
                    environmentServiceNameAzureRM: 'Azure-Service-Connection'
                    commandOptions: '-auto-approve'

  - stage: DeployQA
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployInfrastructureQA
        environment: 'qa'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            deploy:
              steps:
                - task: TerraformTaskV4@4
                  displayName: 'Terraform Apply - QA'
                  inputs:
                    provider: 'azurerm'
                    command: 'apply'
                    workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure/environments/qa'
                    environmentServiceNameAzureRM: 'Azure-Service-Connection'
                    commandOptions: '-auto-approve'

  - stage: DeployProd
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: DeployInfrastructureProd
        environment: 'production'
        pool:
          vmImage: 'ubuntu-latest'
        strategy:
          runOnce:
            preDeploy:
              steps:
                - task: ManualValidation@0
                  timeoutInMinutes: 60
                  inputs:
                    notifyUsers: '[devops-team@company.com]'
                    instructions: 'Please validate the infrastructure changes before production deployment'
            deploy:
              steps:
                - task: TerraformTaskV4@4
                  displayName: 'Terraform Apply - Prod'
                  inputs:
                    provider: 'azurerm'
                    command: 'apply'
                    workingDirectory: '$(System.DefaultWorkingDirectory)/infrastructure/environments/prod'
                    environmentServiceNameAzureRM: 'Azure-Service-Connection'
                    commandOptions: '-auto-approve'
```

### Phase 10: Cost Optimization (Ongoing)

#### 10.1 Cost Management Configuration
```hcl
# modules/cost_management/main.tf
resource "azurerm_consumption_budget_resource_group" "main" {
  name              = "budget-docqa-${var.environment}"
  resource_group_id = var.resource_group_id
  
  amount     = var.monthly_budget
  time_grain = "Monthly"
  
  time_period {
    start_date = formatdate("YYYY-MM-01'T'00:00:00Z", timestamp())
  }
  
  notification {
    enabled        = true
    threshold      = 80
    operator       = "GreaterThan"
    threshold_type = "Actual"
    
    contact_emails = var.budget_alert_emails
  }
  
  notification {
    enabled        = true
    threshold      = 100
    operator       = "GreaterThan"
    threshold_type = "Forecasted"
    
    contact_emails = var.budget_alert_emails
  }
}

# Auto-shutdown for non-production
resource "azurerm_dev_test_global_vm_shutdown_schedule" "main" {
  count = var.environment != "prod" ? 1 : 0
  
  virtual_machine_id = azurerm_linux_virtual_machine.main[0].id
  location           = var.location
  enabled            = true
  
  daily_recurrence_time = "1900"
  timezone              = "Eastern Standard Time"
  
  notification_settings {
    enabled = false
  }
  
  tags = var.tags
}
```

### Environment-Specific Configurations

#### Development Environment
```hcl
# environments/dev/terraform.tfvars
environment = "dev"
location    = "East US"

# Minimal resources for development
app_service_sku = "B1"
search_sku      = "basic"
storage_replication = "LRS"

# Relaxed security for development
allowed_ip_ranges = ["0.0.0.0/0"]
enable_public_access = true

# Cost optimization
auto_shutdown_enabled = true
monthly_budget = 500
```

#### QA Environment
```hcl
# environments/qa/terraform.tfvars
environment = "qa"
location    = "East US"

# Moderate resources for testing
app_service_sku = "S1"
search_sku      = "standard"
storage_replication = "GRS"

# Moderate security
allowed_ip_ranges = ["corporate_ip_range"]
enable_public_access = false

# Cost control
auto_shutdown_enabled = true
monthly_budget = 1500
```

#### Production Environment
```hcl
# environments/prod/terraform.tfvars
environment = "prod"
location    = "East US"

# Production-grade resources
app_service_sku = "P2v3"
search_sku      = "standard2"
storage_replication = "RAGRS"

# Maximum security
allowed_ip_ranges = ["corporate_ip_range", "cdn_ip_range"]
enable_public_access = false

# High availability
enable_zone_redundancy = true
backup_retention_days = 30
geo_replicated = true

# Monitoring
monthly_budget = 5000
alert_severity = "critical"
```

### Deployment Commands

```bash
# Initialize Terraform
cd infrastructure/environments/${ENVIRONMENT}
terraform init

# Plan changes
terraform plan -var-file="${ENVIRONMENT}.tfvars" -out=tfplan

# Apply changes
terraform apply tfplan

# Destroy resources (careful!)
terraform destroy -var-file="${ENVIRONMENT}.tfvars"
```

### Security Best Practices

1. **State Management**
   - Store Terraform state in Azure Storage with encryption
   - Enable state locking using blob leases
   - Implement state backup strategy

2. **Secret Management**
   - Never commit secrets to version control
   - Use Azure Key Vault for all secrets
   - Rotate keys regularly

3. **Network Security**
   - Implement private endpoints for all services
   - Use Network Security Groups restrictively
   - Enable Azure Firewall for production

4. **Identity Management**
   - Use Managed Identities wherever possible
   - Implement RBAC with least privilege
   - Enable MFA for all admin accounts

### Cost Optimization Strategies

1. **Resource Sizing**
   - Start small and scale based on metrics
   - Use auto-scaling for production
   - Implement scheduled scaling for non-production

2. **Storage Optimization**
   - Use lifecycle policies for blob storage
   - Archive old data automatically
   - Compress data where possible

3. **Reserved Capacity**
   - Purchase reserved instances for production
   - Use spot instances for batch processing
   - Leverage Azure Hybrid Benefit

### Monitoring & Maintenance

1. **Health Checks**
   - Implement automated health checks
   - Monitor resource utilization
   - Set up availability alerts

2. **Backup Strategy**
   - Daily backups for production
   - Test restore procedures regularly
   - Maintain backup documentation

3. **Update Management**
   - Schedule regular Terraform updates
   - Review and update provider versions
   - Implement change management process

### Disaster Recovery Plan

1. **RPO/RTO Targets**
   - Production: RPO = 1 hour, RTO = 4 hours
   - QA: RPO = 24 hours, RTO = 8 hours
   - Dev: RPO = 24 hours, RTO = 24 hours

2. **Backup Procedures**
   - Automated daily backups
   - Cross-region replication for production
   - Regular restore testing

3. **Failover Process**
   - Document failover procedures
   - Automate where possible
   - Regular DR drills

### Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Infrastructure Provisioning Time | < 30 minutes | Terraform apply duration |
| Infrastructure Cost | Within budget ±10% | Azure Cost Management |
| Security Compliance | 100% | Azure Policy compliance |
| Availability | > 99.9% | Azure Monitor |
| Deployment Success Rate | > 95% | Azure DevOps metrics |

### Next Steps

1. Review and approve infrastructure plan
2. Set up Azure subscription and permissions
3. Initialize Terraform backend storage
4. Create service principals for automation
5. Begin Phase 1 implementation
6. Schedule infrastructure review meetings