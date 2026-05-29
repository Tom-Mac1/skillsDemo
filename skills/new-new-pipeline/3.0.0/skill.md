---
name: new-new-pipeline
version: 2.0.0
description: >-
  Create an Azure DevOps pipeline that runs Terraform init, fmt, validate, and
  plan stages for a dev environment.
inputs:
  - agent_pool
outputs:
  - newai/outputs
standards:
  - standards/skill-standard.md
prompts:
  template: prompts/main.md
metadata:
  artifact_type: azure-devops
  target_environment: dev
  description: >-
    Create an Azure DevOps pipeline that runs Terraform init, fmt, validate, and
    plan stages for a dev environment.
---

# new-new-pipeline

## Description
Create an Azure DevOps pipeline that runs Terraform init, fmt, validate, and plan stages for a dev environment.

This skill generates azure-devops artifacts for the dev environment.

## Inputs
- agent_pool

## Outputs
- newai/outputs

## Structure
- prompts/main.md: template inputs for generation
- standards/skill-standard.md: generation requirements
- examples/expected-output.txt: reference output
- tests/README.md: validation guidance
