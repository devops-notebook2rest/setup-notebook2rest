# Build and deploy Notebook2REST to AWS

Converts a repository of Jupyter Notebooks into a production-ready FastAPI application, containerizes it, and deploys it to your AWS EKS (Elastic Kubernetes Service) cluster.

## How it Works

Under the hood, this Composite Action performs the following steps:
1. Clones the `notebook2rest` execution engine.
2. Runs your notebooks through the generator to generate a FastAPI.
3. Authenticates your AWS account.
4. Builds and tags a Docker image containing your new API.
5. Pushes the image to your AWS ECR.
6. Updates your `kubeconfig` and deploys the latest image to your EKS cluster using Helm charts.

## Parameters

Before using this Action, ensure declaring the following parameters:
* `aws_access_key_id` : Your AWS Access Key ID. Pass this securely via `${{ secrets.YOUR_SECRET }}`.
* `aws_secret_access_key` : Your AWS Secret Access Key. Pass this securely via `${{ secrets.YOUR_SECRET }}`.
* `aws_region` : The AWS Region where your ECR and EKS cluster reside (e.g., `eu-north-1`).
* `eks_cluster_name` : The exact name of your target EKS Cluster.
* `ecr_repository` : The name of your ECR repository

## Usage

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
