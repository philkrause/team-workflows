# GitHub Actions Workflows Documentation

This directory contains reusable GitHub Actions workflows and documentation on how they are used within this project.

## Reusable Terraform Workflow (`terraform.yml`)

The `terraform.yml` workflow is designed to be a **reusable workflow** that can be called by other GitHub Actions workflows in different repositories. It encapsulates the common steps for Terraform linting, security scanning, validation, and planning.

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

To use this reusable workflow in another repository (e.g., your `terraform-monorepo`), you would create a new workflow file (e.g., `.github/workflows/deploy-terraform.yml`) that includes the logic to derive `working_directory`, `environment`, `tfvars_project_name`, and `gcp_project_id` from your branch naming convention. This calling workflow would then pass these values as inputs to `terraform.yml`.

Here's an example of a calling workflow:

```yaml
name: Terraform Reusable Workflow

on:
  workflow_call:
    inputs:
      working_directory:
        description: 'The working directory to run Terraform commands from'
        required: true
        type: string
      environment:
        description: 'The environment to target (e.g., production, staging).'
        required: true
        type: string
      tfvars_project_name:
        description: 'The name of the project in the environments directory (e.g., customer-abc19a39).'
        required: true
        type: string
      gcp_project_id:
        description: 'The target Google Cloud Project ID (e.g., caddi-us-c-abc19a39-prd).'
        required: true
        type: string
    secrets:
      GCP_SA_KEY:
        required: true

jobs:
  terraform:
    runs-on: ubuntu-latest
    env:
      GOOGLE_CLOUD_PROJECT: ${{ inputs.gcp_project_id }}
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
          working_directory: ${{ inputs.working_directory }}
          soft_fail: true # Allow tfsec to fail without failing the job

      - name: Set up TFLint
        uses: terraform-linters/setup-tflint@v3

      - name: Run tflint
        working-directory: ${{ inputs.working_directory }}
        run: tflint --no-color

      - name: Terraform Format Check
        working-directory: ${{ inputs.working_directory }}
        run: terraform fmt -check=true

      - name: Terraform Validate
        working-directory: ${{ inputs.working_directory }}
        run: terraform validate

      - name: Terraform Plan
        working-directory: ${{ inputs.working_directory }}
        run: |
          terraform plan -no-color -var-file=../../environments/${{ inputs.tfvars_project_name }}/${{ inputs.environment }}/terraform.tfvars
```
