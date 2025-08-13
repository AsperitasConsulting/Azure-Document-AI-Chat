# GitHub Actions Workflows - Complete Implementation Guide

## Overview

This document provides complete GitHub Actions workflow implementations for automating Terraform deployments with comprehensive CI/CD practices.

## Workflow Structure

```
.github/
├── workflows/
│   ├── terraform-plan.yml           # Manual trigger for planning
│   ├── terraform-apply.yml          # Manual trigger for applying
│   ├── terraform-destroy.yml        # Manual trigger for destroying
│   ├── terraform-validate.yml       # PR trigger for validation
│   └── terraform-drift-detection.yml # Scheduled drift detection
├── actions/
│   └── terraform-setup/
│       └── action.yml               # Reusable Terraform setup
└── CODEOWNERS                       # Ownership for workflows
```

## Complete Workflow Implementations

### 1. Terraform Plan Workflow

```yaml
# .github/workflows/terraform-plan.yml
name: Terraform Plan

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
        default: 'dev'
        type: choice
        options:
          - dev
          - qa
          - prod
      project:
        description: 'Terraform Project'
        required: true
        type: choice
        options:
          - terraform-storage
          - terraform-app
          - all
      detailed_plan:
        description: 'Show detailed plan output'
        required: false
        default: true
        type: boolean

env:
  TF_IN_AUTOMATION: true
  TF_INPUT: false

jobs:
  setup:
    name: Setup and Validation
    runs-on: ubuntu-latest
    outputs:
      projects: ${{ steps.set-projects.outputs.projects }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        
      - name: Set projects to run
        id: set-projects
        run: |
          if [ "${{ github.event.inputs.project }}" == "all" ]; then
            echo "projects=[\"terraform-storage\", \"terraform-app\"]" >> $GITHUB_OUTPUT
          else
            echo "projects=[\"${{ github.event.inputs.project }}\"]" >> $GITHUB_OUTPUT
          fi

  terraform-plan:
    name: Plan - ${{ matrix.project }}
    needs: setup
    runs-on: ubuntu-latest
    strategy:
      matrix:
        project: ${{ fromJson(needs.setup.outputs.projects) }}
      max-parallel: 2
    
    permissions:
      contents: read
      pull-requests: write
      id-token: write
    
    defaults:
      run:
        working-directory: ./infrastructure/${{ matrix.project }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure Azure credentials
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ github.event.inputs.terraform_version }}
          terraform_wrapper: false
      
      - name: Terraform Format Check
        id: fmt
        run: |
          terraform fmt -check -recursive
        continue-on-error: true
      
      - name: Terraform Init
        id: init
        run: |
          terraform init \
            -backend-config="key=${{ matrix.project }}-${{ github.event.inputs.environment }}.tfstate" \
            -upgrade
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      
      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color
      
      - name: Run tflint
        uses: terraform-linters/setup-tflint@v4
        with:
          tflint_version: latest
      
      - name: Init TFLint
        run: tflint --init
        env:
          GITHUB_TOKEN: ${{ github.token }}
      
      - name: Run TFLint
        run: tflint -f compact
      
      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          working_directory: ./infrastructure/${{ matrix.project }}
          soft_fail: true
      
      - name: Setup Infracost
        uses: infracost/setup-infracost@v2
        with:
          api-key: ${{ secrets.INFRACOST_API_KEY }}
        if: github.event.inputs.environment != 'prod'
      
      - name: Generate Infracost JSON
        run: |
          infracost breakdown \
            --path=. \
            --format=json \
            --out-file=/tmp/infracost.json
        if: github.event.inputs.environment != 'prod'
      
      - name: Terraform Plan
        id: plan
        run: |
          terraform plan \
            -var-file="environments/${{ github.event.inputs.environment }}.tfvars" \
            -out=tfplan \
            -no-color \
            -detailed-exitcode
        continue-on-error: true
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          TF_VAR_openai_api_key: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Terraform Show Plan
        id: show
        if: github.event.inputs.detailed_plan == 'true'
        run: |
          terraform show -no-color tfplan > plan.txt
          echo "## Plan Output" >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          cat plan.txt >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
      
      - name: Upload Plan
        uses: actions/upload-artifact@v3
        with:
          name: tfplan-${{ matrix.project }}-${{ github.event.inputs.environment }}
          path: ./infrastructure/${{ matrix.project }}/tfplan
          retention-days: 7
      
      - name: Create Plan Summary
        if: always()
        run: |
          echo "# Terraform Plan Summary" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          echo "**Project:** ${{ matrix.project }}" >> $GITHUB_STEP_SUMMARY
          echo "**Environment:** ${{ github.event.inputs.environment }}" >> $GITHUB_STEP_SUMMARY
          echo "**Terraform Version:** ${{ github.event.inputs.terraform_version }}" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          
          echo "## Validation Results" >> $GITHUB_STEP_SUMMARY
          echo "| Check | Status |" >> $GITHUB_STEP_SUMMARY
          echo "|-------|--------|" >> $GITHUB_STEP_SUMMARY
          echo "| Format | ${{ steps.fmt.outcome == 'success' && '✅' || '❌' }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Init | ${{ steps.init.outcome == 'success' && '✅' || '❌' }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Validate | ${{ steps.validate.outcome == 'success' && '✅' || '❌' }} |" >> $GITHUB_STEP_SUMMARY
          echo "| Plan | ${{ steps.plan.outcome == 'success' && '✅' || '❌' }} |" >> $GITHUB_STEP_SUMMARY
          
          if [ "${{ steps.plan.outputs.exitcode }}" == "2" ]; then
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "**⚠️ Changes Detected:** Infrastructure changes will be applied" >> $GITHUB_STEP_SUMMARY
          elif [ "${{ steps.plan.outputs.exitcode }}" == "0" ]; then
            echo "" >> $GITHUB_STEP_SUMMARY
            echo "**✅ No Changes:** Infrastructure is up-to-date" >> $GITHUB_STEP_SUMMARY
          fi
```

### 2. Terraform Apply Workflow

```yaml
# .github/workflows/terraform-apply.yml
name: Terraform Apply

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
        default: 'dev'
        type: choice
        options:
          - dev
          - qa
          - prod
      project:
        description: 'Terraform Project'
        required: true
        type: choice
        options:
          - terraform-storage
          - terraform-app
      apply:
        description: 'Apply changes (false = plan only)'
        required: true
        default: false
        type: boolean
      auto_approve:
        description: 'Auto approve apply (prod requires manual approval)'
        required: false
        default: false
        type: boolean

env:
  TF_IN_AUTOMATION: true
  TF_INPUT: false

jobs:
  terraform-apply:
    name: Apply - ${{ github.event.inputs.project }}
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}
    
    permissions:
      contents: read
      pull-requests: write
      id-token: write
      issues: write
    
    defaults:
      run:
        working-directory: ./infrastructure/${{ github.event.inputs.project }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Validate Production Deployment
        if: github.event.inputs.environment == 'prod'
        run: |
          echo "🔒 Production deployment requested"
          echo "Ensure all approvals are in place"
          if [ "${{ github.event.inputs.auto_approve }}" == "true" ]; then
            echo "❌ Auto-approve is not allowed for production"
            exit 1
          fi
      
      - name: Configure Azure credentials
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ github.event.inputs.terraform_version }}
          terraform_wrapper: false
      
      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="key=${{ github.event.inputs.project }}-${{ github.event.inputs.environment }}.tfstate" \
            -upgrade
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      
      - name: Download Plan Artifact
        id: download
        uses: actions/download-artifact@v3
        with:
          name: tfplan-${{ github.event.inputs.project }}-${{ github.event.inputs.environment }}
          path: ./infrastructure/${{ github.event.inputs.project }}
        continue-on-error: true
      
      - name: Terraform Plan
        if: steps.download.outcome != 'success'
        run: |
          echo "No saved plan found. Creating new plan..."
          terraform plan \
            -var-file="environments/${{ github.event.inputs.environment }}.tfvars" \
            -out=tfplan \
            -no-color
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          TF_VAR_openai_api_key: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Show Plan Summary
        run: |
          echo "## Plan Summary" >> $GITHUB_STEP_SUMMARY
          terraform show -no-color tfplan | head -100 >> $GITHUB_STEP_SUMMARY
      
      - name: Manual Approval Check
        if: github.event.inputs.apply == 'true' && github.event.inputs.auto_approve == 'false'
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: ${{ vars.TERRAFORM_APPROVERS }}
          minimum-approvals: 1
          issue-title: "Terraform Apply Approval Required"
          issue-body: |
            ## Terraform Apply Approval Required
            
            **Project:** ${{ github.event.inputs.project }}
            **Environment:** ${{ github.event.inputs.environment }}
            **Triggered by:** ${{ github.actor }}
            
            Please review the plan and approve to proceed with apply.
      
      - name: Terraform Apply
        if: github.event.inputs.apply == 'true'
        run: |
          if [ "${{ github.event.inputs.auto_approve }}" == "true" ]; then
            terraform apply tfplan
          else
            terraform apply tfplan
          fi
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          TF_VAR_openai_api_key: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Terraform Output
        if: github.event.inputs.apply == 'true'
        run: |
          echo "## Terraform Outputs" >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
          terraform output -json | jq . >> $GITHUB_STEP_SUMMARY
          echo '```' >> $GITHUB_STEP_SUMMARY
      
      - name: Create Issue on Failure
        if: failure() && github.event.inputs.apply == 'true'
        uses: actions/create-issue@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          title: "Terraform Apply Failed - ${{ github.event.inputs.environment }}"
          body: |
            ## Terraform Apply Failed
            
            **Project:** ${{ github.event.inputs.project }}
            **Environment:** ${{ github.event.inputs.environment }}
            **Workflow Run:** ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}
            **Triggered by:** ${{ github.actor }}
            
            Please investigate the failure and retry.
          labels: infrastructure, incident
```

### 3. Terraform Validate Workflow (PR Trigger)

```yaml
# .github/workflows/terraform-validate.yml
name: Terraform Validate

on:
  pull_request:
    paths:
      - 'infrastructure/**'
      - '.github/workflows/terraform-*.yml'
  push:
    branches:
      - main
    paths:
      - 'infrastructure/**'

env:
  TF_IN_AUTOMATION: true
  TF_INPUT: false

jobs:
  detect-changes:
    name: Detect Changes
    runs-on: ubuntu-latest
    outputs:
      storage-changed: ${{ steps.filter.outputs.storage }}
      app-changed: ${{ steps.filter.outputs.app }}
    steps:
      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2
        id: filter
        with:
          filters: |
            storage:
              - 'infrastructure/terraform-storage/**'
            app:
              - 'infrastructure/terraform-app/**'

  validate-storage:
    name: Validate Storage Project
    needs: detect-changes
    if: needs.detect-changes.outputs.storage-changed == 'true'
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        environment: [dev, qa, prod]
    
    defaults:
      run:
        working-directory: ./infrastructure/terraform-storage
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.12.2
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
      
      - name: Terraform Init
        run: |
          terraform init -backend=false
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
      
      - name: Run TFLint
        run: |
          tflint --init
          tflint
      
      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          working_directory: ./infrastructure/terraform-storage
      
      - name: Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ./infrastructure/terraform-storage
          framework: terraform
          soft_fail: true

  validate-app:
    name: Validate App Project
    needs: detect-changes
    if: needs.detect-changes.outputs.app-changed == 'true'
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        environment: [dev, qa, prod]
    
    defaults:
      run:
        working-directory: ./infrastructure/terraform-app
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.12.2
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
      
      - name: Terraform Init
        run: |
          terraform init -backend=false
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: TFLint
        uses: terraform-linters/setup-tflint@v4
      
      - name: Run TFLint
        run: |
          tflint --init
          tflint
      
      - name: tfsec
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          working_directory: ./infrastructure/terraform-app
      
      - name: Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: ./infrastructure/terraform-app
          framework: terraform
          soft_fail: true

  pr-comment:
    name: PR Comment
    needs: [validate-storage, validate-app]
    if: always() && github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    
    steps:
      - name: Comment PR
        uses: actions/github-script@v6
        with:
          script: |
            const output = `## Terraform Validation Results
            
            | Project | Status |
            |---------|--------|
            | Storage | ${{ needs.validate-storage.result == 'success' && '✅ Passed' || needs.validate-storage.result == 'skipped' && '⏭️ Skipped' || '❌ Failed' }} |
            | App | ${{ needs.validate-app.result == 'success' && '✅ Passed' || needs.validate-app.result == 'skipped' && '⏭️ Skipped' || '❌ Failed' }} |
            
            <details>
            <summary>Workflow Details</summary>
            
            - **Workflow Run:** [View Results](${context.serverUrl}/${context.repo.owner}/${context.repo.repo}/actions/runs/${context.runId})
            - **Commit:** ${context.sha.substring(0, 7)}
            - **Triggered by:** ${context.actor}
            
            </details>`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
```

### 4. Terraform Destroy Workflow

```yaml
# .github/workflows/terraform-destroy.yml
name: Terraform Destroy

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
      project:
        description: 'Terraform Project'
        required: true
        type: choice
        options:
          - terraform-app
          - terraform-storage
      confirm_destroy:
        description: 'Type DESTROY to confirm'
        required: true
        type: string

env:
  TF_IN_AUTOMATION: true
  TF_INPUT: false

jobs:
  validate-destroy:
    name: Validate Destroy Request
    runs-on: ubuntu-latest
    steps:
      - name: Validate Confirmation
        run: |
          if [ "${{ github.event.inputs.confirm_destroy }}" != "DESTROY" ]; then
            echo "❌ Confirmation failed. You must type DESTROY to proceed."
            exit 1
          fi
      
      - name: Check Environment
        if: github.event.inputs.environment == 'prod'
        run: |
          echo "⚠️ WARNING: Production destroy requested!"
          echo "This action will permanently delete production resources."

  terraform-destroy:
    name: Destroy - ${{ github.event.inputs.project }}
    needs: validate-destroy
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment }}-destroy
    
    defaults:
      run:
        working-directory: ./infrastructure/${{ github.event.inputs.project }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure Azure credentials
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ github.event.inputs.terraform_version }}
      
      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="key=${{ github.event.inputs.project }}-${{ github.event.inputs.environment }}.tfstate"
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      
      - name: Terraform Plan Destroy
        run: |
          terraform plan \
            -destroy \
            -var-file="environments/${{ github.event.inputs.environment }}.tfvars" \
            -out=destroy.tfplan
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          TF_VAR_openai_api_key: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Manual Approval
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: ${{ vars.TERRAFORM_DESTROY_APPROVERS }}
          minimum-approvals: 2
          issue-title: "⚠️ Terraform Destroy Approval Required"
          issue-body: |
            ## ⚠️ Terraform Destroy Approval Required
            
            **Project:** ${{ github.event.inputs.project }}
            **Environment:** ${{ github.event.inputs.environment }}
            **Triggered by:** ${{ github.actor }}
            
            This action will **permanently destroy** all resources in the project.
            
            Please review carefully before approving.
      
      - name: Terraform Destroy
        run: terraform apply destroy.tfplan
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          TF_VAR_openai_api_key: ${{ secrets.OPENAI_API_KEY }}
```

### 5. Drift Detection Workflow

```yaml
# .github/workflows/terraform-drift-detection.yml
name: Terraform Drift Detection

on:
  schedule:
    # Run daily at 2 AM UTC
    - cron: '0 2 * * *'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to check'
        required: false
        default: 'all'
        type: choice
        options:
          - all
          - dev
          - qa
          - prod

env:
  TF_IN_AUTOMATION: true
  TF_INPUT: false

jobs:
  detect-drift:
    name: Detect Drift - ${{ matrix.environment }} - ${{ matrix.project }}
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        environment: ${{ github.event.inputs.environment == 'all' && fromJson('["dev", "qa", "prod"]') || fromJson(format('["{0}"]', github.event.inputs.environment)) }}
        project: [terraform-storage, terraform-app]
    
    defaults:
      run:
        working-directory: ./infrastructure/${{ matrix.project }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Configure Azure credentials
        uses: azure/login@v1
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.12.2
      
      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="key=${{ matrix.project }}-${{ matrix.environment }}.tfstate"
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
      
      - name: Terraform Plan
        id: plan
        run: |
          terraform plan \
            -var-file="environments/${{ matrix.environment }}.tfvars" \
            -detailed-exitcode \
            -no-color \
            -out=tfplan
        continue-on-error: true
        env:
          ARM_CLIENT_ID: ${{ secrets.AZURE_CLIENT_ID }}
          ARM_CLIENT_SECRET: ${{ secrets.AZURE_CLIENT_SECRET }}
          ARM_SUBSCRIPTION_ID: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
          ARM_TENANT_ID: ${{ secrets.AZURE_TENANT_ID }}
          TF_VAR_openai_api_key: ${{ secrets.OPENAI_API_KEY }}
      
      - name: Check for Drift
        if: steps.plan.outputs.exitcode == '2'
        run: |
          echo "⚠️ Drift detected in ${{ matrix.project }} - ${{ matrix.environment }}"
          echo "DRIFT_DETECTED=true" >> $GITHUB_ENV
      
      - name: Create Issue for Drift
        if: env.DRIFT_DETECTED == 'true'
        uses: actions/create-issue@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          title: "Drift Detected - ${{ matrix.project }} - ${{ matrix.environment }}"
          body: |
            ## Infrastructure Drift Detected
            
            **Project:** ${{ matrix.project }}
            **Environment:** ${{ matrix.environment }}
            **Detection Time:** ${{ github.event.repository.updated_at }}
            
            Terraform has detected configuration drift in the infrastructure.
            
            ### Next Steps
            1. Review the drift in the [workflow run](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})
            2. Determine if the drift is intentional
            3. Either update Terraform configuration or apply to restore desired state
            
            ### Workflow Run
            [View Details](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})
          labels: drift, infrastructure, ${{ matrix.environment }}
```

## Reusable Actions

### Terraform Setup Action

```yaml
# .github/actions/terraform-setup/action.yml
name: 'Setup Terraform Environment'
description: 'Setup Terraform with Azure authentication'

inputs:
  terraform_version:
    description: 'Terraform version to install'
    required: false
    default: '1.12.2'
  working_directory:
    description: 'Working directory for Terraform'
    required: true
  azure_client_id:
    description: 'Azure Client ID'
    required: true
  azure_client_secret:
    description: 'Azure Client Secret'
    required: true
  azure_subscription_id:
    description: 'Azure Subscription ID'
    required: true
  azure_tenant_id:
    description: 'Azure Tenant ID'
    required: true

runs:
  using: 'composite'
  steps:
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v3
      with:
        terraform_version: ${{ inputs.terraform_version }}
    
    - name: Azure Login
      uses: azure/login@v1
      with:
        creds: |
          {
            "clientId": "${{ inputs.azure_client_id }}",
            "clientSecret": "${{ inputs.azure_client_secret }}",
            "subscriptionId": "${{ inputs.azure_subscription_id }}",
            "tenantId": "${{ inputs.azure_tenant_id }}"
          }
    
    - name: Verify Azure Connection
      shell: bash
      run: |
        az account show
        az group list --output table
```

## GitHub Environment Configuration

### Required Environments

1. **dev** - Development environment
   - No protection rules
   - Automatic deployments allowed

2. **qa** - QA environment
   - Required reviewers: QA team
   - Deployment branches: main, release/*

3. **prod** - Production environment
   - Required reviewers: 2 from ops team
   - Deployment branches: main only
   - Wait timer: 5 minutes

4. **prod-destroy** - Production destroy environment
   - Required reviewers: 3 from leadership
   - Manual approval required

### Environment Secrets

Each environment should have:
```
AZURE_CLIENT_ID
AZURE_CLIENT_SECRET
AZURE_SUBSCRIPTION_ID
AZURE_TENANT_ID
OPENAI_API_KEY
```

### Repository Variables

```
TERRAFORM_APPROVERS=user1,user2,user3
TERRAFORM_DESTROY_APPROVERS=admin1,admin2
INFRACOST_API_KEY=your-infracost-key
```

## CODEOWNERS File

```
# .github/CODEOWNERS
# Infrastructure ownership
/infrastructure/terraform-storage/ @infrastructure-team
/infrastructure/terraform-app/ @platform-team
/.github/workflows/terraform-*.yml @devops-team

# Environment-specific ownership
/infrastructure/*/environments/prod.tfvars @leadership-team
/infrastructure/*/environments/qa.tfvars @qa-team
```

## Branch Protection Rules

### Main Branch Protection

- Require pull request reviews (2 approvals)
- Dismiss stale pull request approvals
- Require review from CODEOWNERS
- Require status checks:
  - terraform-validate
  - terraform-fmt
  - tflint
  - tfsec
- Require branches to be up to date
- Include administrators
- Restrict who can push to matching branches

## Notification Setup

### Slack Integration

```yaml
# Add to any workflow for Slack notifications
- name: Notify Slack
  if: always()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: |
      Workflow: ${{ github.workflow }}
      Job: ${{ github.job }}
      Status: ${{ job.status }}
      Environment: ${{ github.event.inputs.environment }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
    channel: '#infrastructure'
```

## Best Practices

1. **Always use version pinning** for Terraform and providers
2. **Never commit sensitive data** - use secrets and environment variables
3. **Implement approval gates** for production changes
4. **Use reusable workflows** to reduce duplication
5. **Enable drift detection** to catch manual changes
6. **Set up proper RBAC** in Azure for service principals
7. **Use separate service principals** per environment
8. **Implement cost alerts** via Infracost
9. **Regular security scanning** with tfsec and Checkov
10. **Maintain documentation** in workflow files

This comprehensive GitHub Actions setup provides robust CI/CD for Terraform infrastructure with proper security, validation, and approval processes.