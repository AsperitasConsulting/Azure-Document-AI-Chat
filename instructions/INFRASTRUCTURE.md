Create Terraform code to provision the following Azure resources for a “doc-qa” application:

- Resource Group: doc-qa-rg (region: East US)
- Azure Storage Account: aspdevdocqa (Standard LRS)
  - Blob container: docs
- Azure Linux Function App: doc-qa-api
  - On a Consumption plan
  - Runs Python 3.11
  - Storage account for function runtime files
  - Application setting: OPENAI_API_KEY
- App Service Plan for the Function App
- Configure Terraform to use an existing Azure storage account backend with the following specifics:
    - resource_group_name  = "Terraform-Backend-State-RG"
    - storage_account_name = "aspterraformstate"
    - container_name       = "tfstate"
    - key                  = (will be different for each Terraform project and environment)
- I expect multiple environments. dev today, but I expect to add qa and prod later
- Use best practices for making arguments variables. Record values expected to change between environments in .tfvars files. 
- Separate Terraform projects for the storage account and linux app reduce the blast radius of infrasructure code.
- Place all Terraform projects in folder infrastructure
- Add init output and any other temporary files to the .gitignore so I don't check them in
- terraform is installed locally and Azure credentials are in the environment variables below. perform an init and plan locally after it's written
    - AZURE_CLIENT_ID
    - AZURE_CLIENT_SECRET
    - AZURE_SUBSCRIPTION_ID
    - AZURE_TENANT_ID

Create GitHub workflows to run Terraform infrastructure
- Workflows are executed  on dispatch
- Provide users option to specify the Terraform version (Defaulting to 1.12.2)
- Provide users option to select the environment deployed (default: dev)
- Provide user to have an apply executed, but make the plan without an apply the default
- Assume the following secrets will be provided for AZURE credentials:
    - OPENAI_API_KEY 
    - AZURE_CLIENT_ID
    - AZURE_CLIENT_SECRET
    - AZURE_SUBSCRIPTION_ID
    - AZURE_TENANT_ID