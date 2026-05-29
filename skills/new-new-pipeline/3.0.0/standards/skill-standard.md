# Azure DevOps Terraform Pipeline — Standards & Best Practices

## 1. Naming Conventions

| Artifact | Convention | Example |
|---|---|---|
| Pipeline YAML file | `azure-pipelines-<env>.yml` | `azure-pipelines-dev.yml` |
| Stage names | PascalCase, descriptive | `Validate`, `Plan`, `Apply` |
| Job names | PascalCase, action-oriented | `TerraformValidate`, `TerraformPlan` |
| Variable groups | `terraform-<env>-secrets` | `terraform-dev-secrets` |
| Published artifacts | `terraform-plan-<env>-$(Build.BuildId)` | `terraform-plan-dev-20240101.1` |
| Artifact output path | `newai/outputs` | `newai/outputs/tfplan` |
| Terraform state key | `<project>/<env>/terraform.tfstate` | `myapp/dev/terraform.tfstate` |
| Pipeline display name | `$(Date:yyyyMMdd).$(Rev:r)-<env>-terraform-pipeline` | `20240101.1-dev-terraform-pipeline` |

## 2. Security Requirements

- **No hard-coded secrets**: All credentials (ARM_CLIENT_ID, ARM_CLIENT_SECRET, ARM_SUBSCRIPTION_ID, ARM_TENANT_ID, backend storage keys) must be stored in an Azure DevOps variable group linked to Azure Key Vault.
- **Least privilege service principal**: The service principal used for Terraform must have only the minimum required RBAC roles in the dev subscription (e.g., `Contributor` scoped to the dev resource group, not the entire subscription).
- **Secret masking**: Mark all sensitive variables as `secret: true` in variable group definitions to ensure they are masked in logs.
- **Branch protection**: Pipeline triggers should require pull request review before merging to `main`; direct pushes to `main` should be restricted.
- **Artifact integrity**: Published `tfplan` artifacts must not be manually editable; use pipeline artifact locking where supported.
- **No `terraform apply` in PR pipelines**: The `plan` stage is the terminal stage for dev PR pipelines. Apply steps must be gated separately.
- **Agent pool hygiene**: Use Microsoft-hosted agents or hardened self-hosted agents. Self-hosted agents must not persist Terraform state or credentials between runs.

## 3. Azure DevOps–Specific Best Practices

### Pipeline Structure
- Use **multi-stage YAML pipelines** (not classic release pipelines) for all Terraform workflows.
- Separate `Validate` and `Plan` into distinct stages to enable granular retry and reporting.
- Use `dependsOn` to enforce stage ordering explicitly.
- Always set `timeoutInMinutes` on jobs (recommended: `30` for init/fmt/validate/plan).

### Terraform Task Usage
- Prefer the `TerraformInstaller@1` and `TerraformTaskV4@4` (or `TerraformCLI`) tasks from the marketplace for standardized Terraform execution.
- Pin Terraform versions explicitly via `TF_VERSION` variable — never use `latest`.
- Run `terraform fmt -check` (not `terraform fmt`) in CI to detect but not auto-correct formatting issues.

### Backend Configuration
- Use Azure Blob Storage as the Terraform remote backend for all environments.
- Use a dedicated storage account per environment (`tfstatedev`, `tfstateprod`) with:
  - Soft delete enabled.
  - Versioning enabled.
  - Private endpoint or service endpoint restricted access.
  - Storage account key accessed via Key Vault reference in the variable group.

### Variable Management
- Use a `dev.tfvars` file committed to the repository for non-sensitive dev environment variable values.
- Use variable group references for all sensitive values.
- Separate variable groups by environment: `terraform-dev-secrets`, `terraform-prod-secrets`.

### Artifact Publishing
- Always publish the Terraform plan binary (`tfplan`) AND a human-readable plan output (`tfplan.txt`) as pipeline artifacts.
- Use `PublishPipelineArtifact@1` (not the legacy `PublishBuildArtifacts@1`).
- Artifact retention: Set a minimum retention of **30 days** for dev plan artifacts.

### Logging and Observability
- Use `##[group]` and `##[endgroup]` log decorators in script steps for readable pipeline logs.
- Redirect `terraform plan` output to both stdout and `tfplan.txt` simultaneously using `tee`.
- Add `displayName` to every step for clarity in the pipeline run UI.

### Dev Environment Specifics
- Dev pipelines should run on every push to feature branches and `dev` branch.
- Do not require manual approval gates in dev (reserve for staging/prod).
- Use `continueOnError: false` for all Terraform steps to fail fast.
- Tag all pipeline runs with `Environment: dev` for cost and audit traceability.