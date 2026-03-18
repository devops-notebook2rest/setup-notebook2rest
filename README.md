# Build and deploy Notebook2REST to AWS

Converts a repository of Jupyter Notebooks into a production-ready FastAPI application, containerizes it, and deploys it to your AWS EKS (Elastic Kubernetes Service) cluster.

This composite action supports two deployment architectures:
1. **Forking Mode:** Runs directly inside a Jupyter Notebook repository via workflow.
2. **Polling Mode:** Runs from a centralized infrastructure repository, cloning remote notebook repositories on demand.

## How it Works

Under the hood, this Composite Action performs the following steps:
1. Clones the `notebook2rest` execution engine.
2. Runs your notebooks through the generator to generate a FastAPI.
3. Checks your repository for a `n2r_requirements.txt` file to inject your custom dependencies into the build.
4. Authenticates your AWS account.
5. Builds and tags a Docker image containing your new API.
6. Pushes the image to your AWS ECR.
7. Updates your `kubeconfig` and deploys the latest image to your EKS cluster using Helm charts.

## Parameters

Before using this Action, ensure declaring the following parameters:
* `aws_access_key_id` : Your AWS Access Key ID. Pass this securely via `${{ secrets.YOUR_SECRET }}`.
* `aws_secret_access_key` : Your AWS Secret Access Key. Pass this securely via `${{ secrets.YOUR_SECRET }}`.
* `aws_region` : The AWS Region where your ECR and EKS cluster reside (e.g., `eu-north-1`).
* `eks_cluster_name` : The exact name of your target EKS Cluster.
* `ecr_repository` : The name of your ECR repository.
* `notebook_repository_name` : The name of the remote repository folder (leave empty if local).
* `notebook_repository_url` : The Git URL of the remote repository (leave empty if local).

## Injecting Custom Dependencies

If your Jupyter Notebooks rely on specific scientific libraries, you do not need to modify the base engine. 

Simply create a file named **`n2r_requirements.txt`** in the root of your repository and list your packages. The Action will automatically detect this file and merge your custom dependencies into the engine.

## Usage

**Scenario 1: Forking mode**

Create a new file in your repository at `.github/workflows/deploy.yml` and paste the following configuration:

```yaml
name: Deploy Notebook API
on:
  push:
    branches: [ "main" ]
    paths:
      - '**/*.ipynb'

jobs:
  setup-notebook2rest:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout My Notebooks
        uses: actions/checkout@v4
        
      - name: Build and Deploy API
        uses: devops-notebook2rest/setup-notebook2rest@v1
        with:
          aws_access_key_id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws_region: 'eu-north-1'
          eks_cluster_name: 'my-production-cluster'
          ecr_repository: 'my-project/api'
```

**Scenario 2 : Polling mode**

```yaml
name: Poll notebook API

concurrency:
  group: notebook2rest-pipeline
  cancel-in-progress: true

on:
  schedule:
    - cron: "*/15 * * * *"
  workflow_dispatch:

env:
  AWS_REGION: eu-north-1
  EKS_CLUSTER_NAME: notebook2rest_cluster
  AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
  AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  REPOSITORY: notebook2rest/api
  IMAGE_TAG: ${{ github.sha }}-${{ github.run_number }}
  NOTEBOOK_REPOSITORY_NAME: vl-laserfarm # The repository name
  NOTEBOOK_REPOSITORY_URL: https://github.com/devops-notebook2rest/vl-laserfarm.git # The repository url

jobs:
  Poll_notebook_changes:
    runs-on: ubuntu-latest

    outputs:
      should_run: ${{ steps.poll.outputs.should_run }}
      latest_commit: ${{ steps.poll.outputs.latest_commit }}

    steps:
      - name: Poll Notebook Repository
        id: poll
        uses: devops-notebook2rest/check-notebook-changes@v0.32
        with:
          notebook_repository_url: ${{ env.NOTEBOOK_REPOSITORY_URL }}

  Build_and_deploy_N2R_API:
    needs: Poll_notebook_changes
    if: needs.check.outputs.should_run == 'true'
    runs-on: ubuntu-latest

    permissions:
      id-token: write
      contents: read

    steps:
      - name: Build and deploy notebook2rest to AWS
        uses: devops-notebook2rest/setup-notebook2rest@v1
        with:
          aws_access_key_id: ${{ env.AWS_ACCESS_KEY_ID }}
          aws_secret_access_key: ${{ env.AWS_SECRET_ACCESS_KEY }}
          aws_region: ${{ env.AWS_REGION }}
          eks_cluster_name: ${{ env.EKS_CLUSTER_NAME }}
          ecr_repository: ${{ env.REPOSITORY }}
          notebook_repository_name: ${{ env.NOTEBOOK_REPOSITORY_NAME }}
          notebook_repository_url: ${{ env.NOTEBOOK_REPOSITORY_URL }}
```
