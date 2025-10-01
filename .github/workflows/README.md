# GitHub Actions Workflows Documentation

This directory contains reusable GitHub Actions workflows and documentation on how they are used within this project.

## Reusable Terraform Workflow (`terraform.yaml`)

The `terraform.yaml` workflow is designed to be a **reusable workflow** that can be called by other GitHub Actions workflows in different repositories. It encapsulates the common steps for Terraform linting, security scanning, validation, and planning.

### Inputs

When calling this workflow, the following inputs are required:

*   `working_directory`:
    *   Description: The working directory to run Terraform commands from (e.g., `infrastructure/terraform/modules/customer-project`).
    *   Required: `true`
    *   Type: `string`
*   `environment`:
    *   Description: The environment to target (e.g., `production`, `staging`).
    *   Required: `true`
    *   Type: `string`
*   `tfvars_project_name`:
    *   Description: The name of the project in the environments directory (e.g., `customer-abc19a39`).
    *   Required: `true`
    *   Type: `string`
*   `gcp_project_id`:
    *   Description: The target Google Cloud Project ID (e.g., `caddi-us-c-abc19a39-prd`).
    *   Required: `true`
    *   Type: `string`

### Secrets

*   `GCP_SA_KEY`:
    *   Description: Google Cloud Platform service account key in JSON format, required for authentication.
    *   Required: `true`

### How to Call the Reusable Workflow

To use this reusable workflow in another repository (e.g., your `terraform-monorepo`), you would create a new workflow file (e.g., `.github/workflows/deploy-terraform.yaml`) that includes the logic to derive `working_directory`, `environment`, `tfvars_project_name`, and `gcp_project_id` from your branch naming convention. This calling workflow would then pass these values as inputs to `terraform.yaml`.

Here's an example of a calling workflow:

```yaml
name: Deploy Terraform Infrastructure

on:
  push:
    branches:
      - 'customer-*-*-*'
      - 'shared-*-*'
  pull_request:
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
      gcp_project_id: ${{ steps.extract.outputs.gcp_project_id }}
    steps:
      - name: Extract branch info
        id: extract
        run: |
          BRANCH_NAME="${{ github.head_ref || github.ref_name }}"
          echo "Branch Name: $BRANCH_NAME"

          TFVARS_PROJECT_NAME=""
          ENVIRONMENT=""
          WORKING_DIRECTORY=""
          GCP_PROJECT_ID=""

          if [[ $BRANCH_NAME =~ ^customer-([a-zA-Z0-9-]+)-([a-zA-Z0-9-]+)-([a-zA-Z0-9-]+)$ ]]; then
            CUSTOMER_ID="${BASH_REMATCH[1]}"
            ENVIRONMENT="${BASH_REMATCH[2]}"
            TFVARS_PROJECT_NAME="customer-$CUSTOMER_ID"
            WORKING_DIRECTORY="terraform/modules/customer-project"
            GCP_PROJECT_ID="caddi-us-c-${CUSTOMER_ID}-${ENVIRONMENT}"
            echo "Customer Project Detected"
          elif [[ $BRANCH_NAME =~ ^shared-([a-zA-Z0-9-]+)-(.*)$ ]]; then # Updated regex for shared branches
            ENVIRONMENT="${BASH_REMATCH[1]}"
            TFVARS_PROJECT_NAME="shared-infrastructure"
            WORKING_DIRECTORY="terraform/modules/shared-infrastructure"
            GCP_PROJECT_ID="caddi-us-inf-${ENVIRONMENT}"
            echo "Shared Infrastructure Project Detected"
          else
            echo "Error: Branch name does not match expected pattern."
            exit 1
          fi

          echo "tfvars_project_name=$TFVARS_PROJECT_NAME" >> "$GITHUB_OUTPUT"
          echo "environment=$ENVIRONMENT" >> "$GITHUB_OUTPUT"
          echo "working_directory=$WORKING_DIRECTORY" >> "$GITHUB_OUTPUT"
          echo "gcp_project_id=$GCP_PROJECT_ID" >> "$GITHUB_OUTPUT"

  call-terraform-workflow:
    needs: extract-branch-info
    uses: YOUR_GITHUB_USERNAME/team-workflows/.github/workflows/terraform.yaml@main
    with:
      working_directory: ${{ needs.extract-branch-info.outputs.working_directory }}
      environment: ${{ needs.extract-branch-info.outputs.environment }}
      tfvars_project_name: ${{ needs.extract-branch-info.outputs.tfvars_project_name }}
      gcp_project_id: ${{ needs.extract-branch-info.outputs.gcp_project_id }}
    secrets:
      GCP_SA_KEY: ${{ secrets.GCP_SA_KEY }}
```