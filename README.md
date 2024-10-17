# gh-action-deploy-terraform

Github reusable action to deploy Terraform 


## Usage Terraform plan

## Installation and setup.

```yaml
name: Plan Bicep deployments

on:
  pull_request_target:
    branches:
      - main
    paths:
      - "**/terraform-deployments/**"


  workflow_dispatch:
    inputs:
      environment:
        type: environment
        description: Filter which environment to deploy to

env: 
  working-directory: terraform-deployments

jobs:
  get-environments:
    runs-on: ubuntu-latest
    permissions:
      contents: read # Required for repo checkout
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.head_ref }}
      - name: Install jq
        run: sudo apt-get install -y jq
      - name: Calculate environments
        working-directory: ${{ env.working-directory }}
        id: get-environments
        run: |
          echo "Current directory:"
          pwd
          echo "Contents of the directory:"
          echo "Environments:"
          environments=$(for dir in */; do echo "\"${dir%/}\""; done | jq -c -s .)
          echo "environments=$environments" >> "$GITHUB_OUTPUT"
          echo $environments
    outputs:
      environments: ${{ steps.get-environments.outputs.environments }}
          
        
  plan-terraform-parallel:
    name: "[${{ matrix.environment }}]Plan Terraform deployments"
    needs: get-environments
    strategy:
      fail-fast: false
      matrix:
        environment: ${{ fromJSON(needs.get-environments.outputs.environments) }}
    runs-on: ubuntu-latest
    if: needs.get-environments.outputs.environments != '[]'
    permissions:
      id-token: write # Required for the OIDC Login
      contents: read # Required for repo checkout
      pull-requests: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.head_ref }}

      - name: Azure login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ vars.APP_ID }}
          tenant-id: ${{ vars.TENANT_ID }}
          subscription-id: ${{ vars.SUBSCRIPTION_ID }}

      - name: Plan Terraform deployments
        uses: alflig/gh-action-deploy-terraform@main
        with:
          tf-storage-account-name: ${{ vars.TF_STORAGE_ACCOUNT_NAME }}
          tf-apply: false
          environment: ${{ matrix.environment }}
          workdir: ${{ env.working-directory }}
          APP_ID: ${{ vars.APP_ID }}
          TENANT_ID: ${{ vars.TENANT_ID }}
          SUBSCRIPTION_ID: ${{ vars.SUBSCRIPTION_ID }}
```

## Usage terraform apply

```yaml
name: Apply Terraform deployments
on:
  schedule:
    - cron: 0 23 * * *
  push:
    branches:
      - main
    paths:
      - "**/terraform-deployments/**"

env: 
  working-directory: terraform-deployments

jobs:
  get-environments:
    runs-on: ubuntu-latest
    permissions:
      contents: read # Required for repo checkout
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.head_ref }}
      - name: Install jq
        run: sudo apt-get install -y jq
      - name: Calculate environments
        working-directory: ${{ env.working-directory }}
        id: get-environments
        run: |
          echo "Current directory:"
          pwd
          echo "Contents of the directory:"
          echo "Environments:"
          environments=$(for dir in */; do echo "\"${dir%/}\""; done | jq -c -s .)
          echo "environments=$environments" >> "$GITHUB_OUTPUT"
          echo $environments
    outputs:
      environments: ${{ steps.get-environments.outputs.environments }}
          
        
  apply-terraform-parallel:
    name: "[${{ matrix.environment }}]Apply Terraform deployments"
    needs: get-environments
    strategy:
      fail-fast: false
      matrix:
        environment: ${{ fromJSON(needs.get-environments.outputs.environments) }}
    runs-on: ubuntu-latest
    if: needs.get-environments.outputs.environments != '[]'
    permissions:
      id-token: write # Required for the OIDC Login
      contents: read # Required for repo checkout
      pull-requests: write
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          ref: ${{ github.head_ref }}

      - name: Azure login via OIDC
        uses: azure/login@v2
        with:
          client-id: ${{ vars.APP_ID }}
          tenant-id: ${{ vars.TENANT_ID }}
          subscription-id: ${{ vars.SUBSCRIPTION_ID }}

      - name: Apply Terraform deployments
        uses: alflig/gh-action-deploy-terraform@main
        with:
          tf-storage-account-name: ${{ vars.TF_STORAGE_ACCOUNT_NAME }}
          tf-apply: true
          environment: ${{ matrix.environment }}
          workdir: ${{ env.working-directory }}
          APP_ID: ${{ vars.APP_ID }}
          TENANT_ID: ${{ vars.TENANT_ID }}
          SUBSCRIPTION_ID: ${{ vars.SUBSCRIPTION_ID }}
```
