# GitHub Actions Workflows Documentation

This directory contains GitHub Actions workflows and documentation on how they are used within this project.

## Branching Model for Terraform Deployments

To ensure consistent and automated deployments of Terraform configurations, a specific branching model is followed to determine the target project, environment, and working directory.

### 1. Customer-Specific Projects

For Terraform configurations related to individual customer projects, the branch names must follow this format:

`customer-<customer_id>-<environment>-<jira_ticket_id>`

*   `<customer_id>`: This segment identifies the unique customer (e.g., `abc19a39`). This will be used to derive the `tfvars_project_name`.
*   `<environment>`: This segment specifies the deployment environment (e.g., `staging`, `production`). This will be used to derive the `environment`.
*   `<jira_ticket_id>`: This segment is for the associated Jira ticket (e.g., `JIRA-123`). This part of the branch name is not used for deriving `tfvars_project_name` or `environment`.

**Example Branch Names:**
*   `customer-abc19a39-staging-JIRA-456`
*   `customer-abc19a39-production-JIRA-789`

**Derived Values:**
*   `tfvars_project_name`: `customer-<customer_id>` (e.g., `customer-abc19a39`)
*   `environment`: `<environment>` (e.g., `staging`)
*   `working_directory`: `infrastructure/terraform/modules/customer-project`

### 2. Shared Infrastructure Projects

For Terraform configurations related to shared infrastructure components, the branch names must follow this format:

`shared-<environment>-<jira_ticket_id>`

*   `<environment>`: This segment specifies the deployment environment (e.g., `dev`, `staging`, `production`). This will be used to derive the `environment`.
*   `<jira_ticket_id>`: This segment is for the associated Jira ticket (e.g., `JIRA-123`). This part of the branch name is not used for deriving `tfvars_project_name` or `environment`.

**Example Branch Names:**
*   `shared-dev-JIRA-101`
*   `shared-staging-JIRA-102`

**Derived Values:**
*   `tfvars_project_name`: `shared-infrastructure` (fixed value)
*   `environment`: `<environment>` (e.g., `dev`)
*   `working_directory`: `infrastructure/terraform/modules/shared-infrastructure`

## Workflow (`terraform.yml`)

The `terraform.yml` workflow is a self-contained workflow that automatically extracts the `tfvars_project_name`, `environment`, and `working_directory` based on the branch name using bash scripting and regular expressions. This workflow is triggered on pushes to branches matching the defined branching model. It then proceeds to run `tfsec`, `tflint`, `terraform fmt`, `terraform validate`, and `terraform plan`.

### Example from `terraform.yml`:

```yaml
name: Terraform Reusable Workflow

on:
  push:
    branches:
      - 'customer-*-*-*'
      - 'shared-*-*'

jobs:
  extract-branch-info:
    runs-on: ubuntu-latest
    outputs:
      tfvars_project_name: ${{ steps.extract.outputs.tfvars_project_name }}
      environment: ${{ steps.extract.outputs.environment }}
      working_directory: ${{ steps.extract.outputs.working_directory }}
    steps:
      - name: Extract branch info
        id: extract
        run: |
          BRANCH_NAME="${{ github.head_ref || github.ref_name }}"
          echo "Branch Name: $BRANCH_NAME"

          TFVARS_PROJECT_NAME=""
          ENVIRONMENT=""
          WORKING_DIRECTORY=""

          if [[ $BRANCH_NAME =~ ^customer-([a-zA-Z0-9-]+)-([a-zA-Z0-9-]+)-([a-zA-Z0-9-]+)$ ]]; then
            TFVARS_PROJECT_NAME="customer-${BASH_REMATCH[1]}"
            ENVIRONMENT="${BASH_REMATCH[2]}"
            WORKING_DIRECTORY="infrastructure/terraform/modules/customer-project"
            echo "Customer Project Detected"
          elif [[ $BRANCH_NAME =~ ^shared-([a-zA-Z0-9-]+)-([a-zA-Z0-9-]+)$ ]]; then
            TFVARS_PROJECT_NAME="shared-infrastructure"
            ENVIRONMENT="${BASH_REMATCH[1]}"
            WORKING_DIRECTORY="infrastructure/terraform/modules/shared-infrastructure"
            echo "Shared Infrastructure Project Detected"
          else
            echo "Error: Branch name does not match expected pattern."
            exit 1
          fi

          echo "tfvars_project_name=$TFVARS_PROJECT_NAME" >> "$GITHUB_OUTPUT"
          echo "environment=$ENVIRONMENT" >> "$GITHUB_OUTPUT"
          echo "working_directory=$WORKING_DIRECTORY" >> "$GITHUB_OUTPUT"

  terraform:
    needs: extract-branch-info
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.x.x

      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          working_directory: ${{ needs.extract-branch-info.outputs.working_directory }}
          soft_fail: true

      - name: Run tflint
        uses: terraform-linters/tflint-action@v2.0.0
        with:
          working_directory: ${{ needs.extract-branch-info.outputs.working_directory }}
          tflint_version: v0.50.0

      - name: Terraform Format Check
        working-directory: ${{ needs.extract-branch-info.outputs.working_directory }}
        run: terraform fmt -check=true

      - name: Terraform Validate
        working-directory: ${{ needs.extract-branch-info.outputs.working_directory }}
        run: terraform validate

      - name: Terraform Plan
        working-directory: ${{ needs.extract-branch-info.outputs.working_directory }}
        run: |
          terraform plan -no-color -var-file=../../environments/${{ needs.extract-branch-info.outputs.tfvars_project_name }}/${{ needs.extract-branch-info.outputs.environment }}/terraform.tfvars
```
