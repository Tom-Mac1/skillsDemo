# Azure DevOps Terraform Pipeline Generation

## Objective
Generate a production-ready Azure DevOps YAML pipeline definition that executes the core Terraform workflow stages: **init**, **fmt**, **validate**, and **plan** for the **dev** target environment.

## Input Parameters
- **Agent Pool**: `{{agent_pool}}` — The Azure DevOps agent pool to use for all pipeline jobs.

## Pipeline Requirements

### Trigger Configuration
- Configure the pipeline to trigger on changes to the `main` or `dev` branch.
- Include a `paths` filter so the pipeline only triggers when Terraform files (`*.tf`, `*.tfvars`) are modified.
- Add a `pull_request` trigger for branch policy enforcement.

### Parameters Block
- The pipeline YAML **must** declare a top-level `parameters:` block containing at minimum:
  ```
  parameters:
    - name: agentPool
      type: string
      default: {{agent_pool}}
  ```
- The agent pool for **every job** in the pipeline **must** be set using `name: ${{ parameters.agentPool }}` under the `pool:` key. Do **not** hardcode any pool name string (such as `test`, `Default`, or any other literal) anywhere in the YAML. The `{{agent_pool}}` input value must flow through this parameter and be referenced exclusively as `${{ parameters.agentPool }}`.

### Variables
- Define pipeline-level variables for:
  - `TF_VERSION`: Terraform version to install (default: `1.7.5`).
  - `WORKING_DIRECTORY`: Root directory containing Terraform configuration files (default: `$(System.DefaultWorkingDirectory)/terraform`).
  - `ENVIRONMENT`: Set to `dev`.
- Reference sensitive values (e.g., backend storage account key, ARM credentials) from an Azure DevOps variable group named `terraform-dev-secrets`. Do NOT hard-code secrets.

### Stages and Jobs

#### Stage 1: `Validate`
- **Job**: `TerraformValidate`
- **Pool**: Must use `name: ${{ parameters.agentPool }}` — no hardcoded pool name.
- **Steps**:
  1. **Install Terraform**: Use the `TerraformInstaller@1` task to install the version specified by `TF_VERSION`.
  2. **Terraform Init**: Run `terraform init` with backend configuration pointing to the dev Azure Storage Account backend. Use variables from the variable group for `storage_account_name`, `container_name`, `key`, and `resource_group_name`.
  3. **Terraform Format Check**: Run `terraform fmt -check -recursive` to enforce formatting standards. Fail the pipeline if any files are not properly formatted.
  4. **Terraform Validate**: Run `terraform validate` to check configuration syntax and internal consistency.

#### Stage 2: `Plan`
- **Job**: `TerraformPlan`
- **Pool**: Must use `name: ${{ parameters.agentPool }}` — no hardcoded pool name.
- **Depends On**: `Validate` stage must succeed.
- **Steps**:
  1. **Install Terraform**: Re-install the same Terraform version (`TF_VERSION`).
  2. **Terraform Init**: Repeat `terraform init` with the same backend configuration.
  3. **Terraform Plan**: Run `terraform plan -out=tfplan -var-file=dev.tfvars` from `$(WORKING_DIRECTORY)`. ARM credentials (`ARM_CLIENT_ID`, `ARM_CLIENT_SECRET`, `ARM_SUBSCRIPTION_ID`, `ARM_TENANT_ID`) must be injected as environment variables from the variable group.
  4. **Generate Human-Readable Plan**: Run `terraform show -no-color tfplan > tfplan.txt` from `$(WORKING_DIRECTORY)` to produce a plain-text representation of the plan. Both `tfplan` and `tfplan.txt` will be located in `$(WORKING_DIRECTORY)` after this step.
  5. **Stage Artifacts for Publishing**: Add an explicit `CopyFiles@2` task (or equivalent shell `cp`/`mkdir` commands) that:
     - Creates the staging directory `$(Build.ArtifactStagingDirectory)/newai/outputs`.
     - Copies **both** `$(WORKING_DIRECTORY)/tfplan` and `$(WORKING_DIRECTORY)/tfplan.txt` into `$(Build.ArtifactStagingDirectory)/newai/outputs`.
     This step **must** appear after the `terraform show` step and before the publish step, ensuring both files are guaranteed to be co-located in the staging directory.
  6. **Publish Plan Artifact**: Use `PublishPipelineArtifact@1` to publish the staged artifacts with the following **exact** settings:
     - `targetPath` **must** be set to `$(Build.ArtifactStagingDirectory)/newai/outputs` — do **not** set `targetPath` to `$(WORKING_DIRECTORY)` or any other path.
     - `artifact` must be set to `terraform-plan-dev-$(Build.BuildId)`.
     - This task must only run after the staging copy step completes successfully.

### Output Artifacts
- All pipeline artifacts must be published to the `newai/outputs` path. The `targetPath` in `PublishPipelineArtifact@1` must resolve to a directory containing only the staged `tfplan` and `tfplan.txt` files — never the entire `$(WORKING_DIRECTORY)`.
- Publish both the `tfplan` binary and the human-readable `tfplan.txt` (generated via `terraform show -no-color tfplan > tfplan.txt`). Both files must be explicitly copied into the staging directory before publishing; do not rely on them being implicitly available.
- Artifact name convention: `terraform-plan-dev-$(Build.BuildId)`.

### Security and Compliance
- All secrets must come from the `terraform-dev-secrets` variable group linked to Azure Key Vault.
- No plaintext secrets in the YAML file.
- Enable `condition: succeeded()` guards on plan steps to prevent partial execution.
- Add a `timeoutInMinutes: 30` cap on all jobs.

### Notifications
- Add a pipeline-level `name` using the format: `$(Date:yyyyMMdd).$(Rev:r)-dev-terraform-pipeline`.

## Output
Produce a complete, valid `azure-pipelines.yml` file saved to the `newai/outputs` directory. The file must be ready to commit to a repository and run without modification beyond filling in variable group values.