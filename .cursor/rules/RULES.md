# Workflow Authoring Rules for This Project

## Workflow Naming
- Use `terraform-*.yaml` for infrastructure workflows
- Use `security-*.yaml` for scanning jobs

## Input/Output Standards
- All reusable workflows must define `inputs:` clearly
- Outputs should be documented in `README.md`

## Permissions
- Always set `permissions:` explicitly in every job
- Avoid using `permissions: write-all`

## Secrets
- Use `${{ secrets.<name> }}` — never hardcode anything
- GitHub OIDC should be used for cloud access

## Workflow Triggers
- Only use `on: workflow_call` or `on: push` — avoid cron for now

## Linting
- All workflows must pass `actionlint` before merging
