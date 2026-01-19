# bymv-gitops-atlantis

A GitOps infrastructure automation project using **Atlantis** to enable Terraform workflows through GitHub Pull Requests.

## Overview

This project uses [Atlantis](https://www.runatlantis.io/) to automate Terraform deployments via GitHub PR workflows. Atlantis acts as a middleware between GitHub and your Terraform infrastructure, allowing teams to review and approve infrastructure changes as code through pull requests.

## Project Structure

```
.
├── helm/                          # Kubernetes Helm chart for Atlantis deployment
│   ├── Chart.yaml                 # Helm chart metadata (Atlantis v4.15.0)
│   ├── requirements.yaml           # Helm dependencies (Atlantis v5.25.0)
│   ├── values.yaml                # Atlantis configuration values
│   └── charts/                     # Versioned chart dependencies
├── terraform.tfvars               # Terraform variables (stored as Kubernetes secret)◊
└── README.md                       # This file
```

## Key Components

### Atlantis Helm Chart (`helm/`)

This chart deploys Atlantis to Kubernetes with the following configuration:

- **GitHub App Integration**: Configured with GitHub App ID `2654766` (slug: `bymv-atlantis`)
- **Repository**: Targets `github.com/MoralesV93/bymv-frontend-pbrand`
- **Terraform Version**: 1.3.0 (configurable via `ATLANTIS_DEFAULT_TF_VERSION`)
- **PR Requirements**: Approvals, mergeability, and no divergence required before applying

### Sensitive Data Management

**terraform.tfvars** is intentionally **not versioned** in this repository. Instead, it is:
- Stored as a Kubernetes secret (`atlantis-tf-vars`)
- Mounted into the Atlantis pod at `/atlantis-data/tfvars/terraform.tfvars`
- Referenced by the `TF_VAR_FILE` environment variable

This approach prevents sensitive infrastructure data (AWS credentials, resource IDs, etc.) from being exposed in version control while allowing Atlantis to access them at runtime.

### AWS Credentials

AWS credentials are stored separately as a Kubernetes secret (`atlantis-aws-cred`) and injected into the Atlantis container environment.

## Setup & Prerequisites

### 1. GitHub App Configuration

To use Atlantis with GitHub, you need to set up a GitHub App with webhook integration:

1. **Create a GitHub App**:
   - Go to your GitHub organization → Settings → Developer settings → GitHub Apps
   - Create a new GitHub App with the following permissions:
     - **Repository Permissions**: Read & write on `Contents`, `Pull Requests`, `Issues`, `Checks`
     - **Webhooks**: Subscribe to `Pull Request`, `Push`, and `Issue Comment` events

2. **Configure Webhook**:
   - Set the webhook URL to your Atlantis instance: `http://your-atlantis-domain/events`
   - The webhook must be accessible from GitHub's IP ranges
   - Atlantis will automatically respond to PR events and post plan/apply comments

3. **Install the GitHub App**:
   - Install the app on your organization or specific repositories
   - Grant access to the target repository: `github.com/MoralesV93/bymv-frontend-pbrand`

### 2. Kubernetes Deployment

Secrets must be created before deploying the Helm chart:

```bash
# Create Kubernetes secrets
kubectl create secret generic atlantis-tf-vars --from-file=terraform.tfvars=./terraform.tfvars
kubectl create secret generic atlantis-aws-cred --from-literal=AWS_ACCESS_KEY_ID=xxx --from-literal=AWS_SECRET_ACCESS_KEY=xxx
kubectl create secret generic atlantis-vcs --from-literal=github-token=ghp_xxx
```

Deploy Atlantis:

```bash
helm repo add runatlantis https://runatlantis.github.io/helm-charts
helm dependency update helm/
helm install atlantis helm/
```

## Workflow

1. **Developer** creates a pull request with Terraform changes
2. **GitHub App webhook** notifies Atlantis of the PR
3. **Atlantis** runs `terraform plan` and posts results as a PR comment
4. **Reviewers** examine the plan and add a comment: `atlantis apply`
5. **Atlantis** validates PR requirements (approvals, mergeable, no divergence)
6. **Atlantis** runs `terraform apply` if all requirements are met
7. **PR is merged** with infrastructure changes deployed

## Versioning

Currently, chart dependencies are versioned and stored in `helm/charts/`. In future iterations, this will be managed differently (e.g., via Helm repositories or package registries).

## Environment Variables

Key Atlantis environment variables configured in `values.yaml`:

- `ATLANTIS_DEFAULT_TF_VERSION`: Default Terraform version (1.3.0)
- `TF_VAR_FILE`: Path to Terraform variables file inside the container
- `ATLANTIS_DEFAULT_TF_VERSION`: Terraform version for plans/applies