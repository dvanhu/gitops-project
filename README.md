# gitops-project

[![CI Pipeline](https://img.shields.io/github/actions/workflow/status/your-org/gitops-project/ci.yml?branch=main&label=CI%20Pipeline&logo=github-actions&logoColor=white)](https://github.com/your-org/gitops-project/actions)
[![Docker Image](https://img.shields.io/docker/pulls/your-dockerhub-username/nginx-app?label=Docker%20Pulls&logo=docker&logoColor=white)](https://hub.docker.com/r/your-dockerhub-username/nginx-app)
[![Docker Image Version](https://img.shields.io/docker/v/your-dockerhub-username/nginx-app?label=Image%20Version&logo=docker)](https://hub.docker.com/r/your-dockerhub-username/nginx-app)

---

## Overview

This repository contains the application source code and the Continuous Integration (CI) pipeline for a production-grade GitOps platform. It is responsible for building and publishing Docker images and propagating new image tags to the GitOps configuration repository, which drives automated deployments via ArgoCD.

This repository forms the **CI layer** of a two-repository GitOps architecture. The corresponding CD layer is maintained in [`gitops-multi-env-deployment`](https://github.com/your-org/gitops-multi-env-deployment).

---

## Repository Structure

```
gitops-project/
├── app-code/
│   ├── index.html          # Application entry point (HTML)
│   └── Dockerfile          # Container image definition
├── .github/
│   └── workflows/
│       └── ci.yml          # GitHub Actions CI pipeline definition
└── README.md
```

---

## CI Architecture

The following diagram illustrates the complete Continuous Integration flow triggered on every push to the `main` branch.

```
  Developer Workstation
  +---------------------+
  | git push → main     |
  +----------+----------+
             |
             v
  GitHub Actions Runner
  +------------------------------------------------------------+
  |                                                            |
  |  Step 1: Checkout Code                                     |
  |          └── actions/checkout@v3                          |
  |                                                            |
  |  Step 2: Authenticate to DockerHub                         |
  |          └── docker/login-action                          |
  |              (DOCKERHUB_USERNAME / DOCKERHUB_TOKEN)        |
  |                                                            |
  |  Step 3: Build Docker Image                                |
  |          └── docker build -t <image>:<sha> ./app-code     |
  |                                                            |
  |  Step 4: Push Docker Image to DockerHub                    |
  |          └── docker push <image>:<sha>                    |
  |                                                            |
  |  Step 5: Update GitOps Repository                          |
  |          └── Clone gitops-multi-env-deployment            |
  |          └── Patch deployment.yaml with new image tag      |
  |          └── Commit and push changes                       |
  |                                                            |
  +------------------------------------------------------------+
             |                           |
             v                           v
  DockerHub Registry           gitops-multi-env-deployment
  +-------------------+        +------------------------------+
  | nginx-app:<sha>   |        | apps/nginx/base/             |
  | nginx-app:latest  |        | deployment.yaml (updated)    |
  +-------------------+        +------------------------------+
                                          |
                                          v
                               ArgoCD detects diff → deploys
```

---

## Pipeline Explanation

The CI pipeline is defined in `.github/workflows/ci.yml` and is triggered on every push to the `main` branch.

### Stage 1 — Code Checkout

The pipeline checks out the repository contents onto the GitHub Actions runner using `actions/checkout@v3`. This makes the application source code and Dockerfile available in the runner environment.

### Stage 2 — DockerHub Authentication

The runner authenticates with DockerHub using credentials stored as GitHub Secrets (`DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN`). This is required before any push operation can succeed.

### Stage 3 — Docker Image Build

The Docker image is built from `app-code/Dockerfile`. The image is tagged with two identifiers: the short Git commit SHA (`GITHUB_SHA`) for traceability, and `latest` for convenience.

```bash
docker build \
  -t your-dockerhub-username/nginx-app:${GITHUB_SHA::7} \
  -t your-dockerhub-username/nginx-app:latest \
  ./app-code
```

### Stage 4 — Docker Image Push

Both tagged images are pushed to the DockerHub registry, making them available for deployment across all environments.

```bash
docker push your-dockerhub-username/nginx-app:${GITHUB_SHA::7}
docker push your-dockerhub-username/nginx-app:latest
```

### Stage 5 — GitOps Repository Update

This is the critical bridge between the CI and CD layers. The pipeline authenticates to GitHub using a Personal Access Token (`GITOPS_PAT`), clones the `gitops-multi-env-deployment` repository, and uses `sed` to update the image tag in the base `deployment.yaml`. It then commits and pushes the change, which ArgoCD detects and uses to trigger a deployment.

```bash
git clone https://${GITOPS_PAT}@github.com/your-org/gitops-multi-env-deployment.git
cd gitops-multi-env-deployment

sed -i "s|image: your-dockerhub-username/nginx-app:.*|image: your-dockerhub-username/nginx-app:${GITHUB_SHA::7}|" \
  apps/nginx/base/deployment.yaml

git config user.email "ci-bot@your-org.com"
git config user.name "CI Bot"
git add apps/nginx/base/deployment.yaml
git commit -m "ci: update image tag to ${GITHUB_SHA::7}"
git push
```

---

## GitHub Actions Workflow Reference

```yaml
# .github/workflows/ci.yml

name: CI Pipeline

on:
  push:
    branches:
      - main

jobs:
  build-and-deploy:
    name: Build, Push, and Update GitOps Repo
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source code
        uses: actions/checkout@v3

      - name: Authenticate to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build Docker image
        run: |
          docker build \
            -t ${{ secrets.DOCKERHUB_USERNAME }}/nginx-app:${GITHUB_SHA::7} \
            -t ${{ secrets.DOCKERHUB_USERNAME }}/nginx-app:latest \
            ./app-code

      - name: Push Docker image to DockerHub
        run: |
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/nginx-app:${GITHUB_SHA::7}
          docker push ${{ secrets.DOCKERHUB_USERNAME }}/nginx-app:latest

      - name: Update image tag in GitOps repository
        run: |
          git clone https://${{ secrets.GITOPS_PAT }}@github.com/your-org/gitops-multi-env-deployment.git
          cd gitops-multi-env-deployment
          sed -i "s|image: .*/nginx-app:.*|image: ${{ secrets.DOCKERHUB_USERNAME }}/nginx-app:${GITHUB_SHA::7}|" \
            apps/nginx/base/deployment.yaml
          git config user.email "ci-bot@your-org.com"
          git config user.name "CI Bot"
          git add apps/nginx/base/deployment.yaml
          git commit -m "ci: update image tag to ${GITHUB_SHA::7}"
          git push
```

---

## GitHub Secrets Configuration

The following secrets must be configured in the repository under **Settings > Secrets and variables > Actions** before the pipeline can execute successfully.

| Secret Name         | Description                                                                 | Required |
|---------------------|-----------------------------------------------------------------------------|----------|
| `DOCKERHUB_USERNAME`| Your DockerHub account username                                             | Yes      |
| `DOCKERHUB_TOKEN`   | DockerHub access token (generated under Account Settings > Security)        | Yes      |
| `GITOPS_PAT`        | GitHub Personal Access Token with `repo` scope for the GitOps repository    | Yes      |

### Generating a DockerHub Access Token

1. Log in to [hub.docker.com](https://hub.docker.com)
2. Navigate to **Account Settings > Security > New Access Token**
3. Assign a descriptive name (e.g., `github-actions-ci`)
4. Copy the token and store it as `DOCKERHUB_TOKEN` in GitHub Secrets

### Generating a GitHub Personal Access Token (PAT)

1. Navigate to **GitHub > Settings > Developer settings > Personal access tokens > Tokens (classic)**
2. Generate a new token with the `repo` scope
3. Store it as `GITOPS_PAT` in GitHub Secrets

---

## Setup Instructions

### Prerequisites

- Docker installed locally (version 20.x or later)
- A DockerHub account with an access token
- A GitHub account with access to both repositories
- `git` installed and configured

### Step 1 — Fork or Clone the Repository

```bash
git clone https://github.com/your-org/gitops-project.git
cd gitops-project
```

### Step 2 — Configure GitHub Secrets

Configure all three secrets as described in the **GitHub Secrets Configuration** section above.

### Step 3 — Update Configuration References

Replace the following placeholders in `.github/workflows/ci.yml` and `app-code/Dockerfile` with your actual values:

- `your-org` — your GitHub organization or username
- `your-dockerhub-username` — your DockerHub username
- `ci-bot@your-org.com` — the commit author email used by the CI bot

### Step 4 — Push to Trigger the Pipeline

```bash
git add .
git commit -m "feat: initial application setup"
git push origin main
```

Navigate to the **Actions** tab in GitHub to monitor the pipeline execution.

---

## Local Development

### Building the Docker Image Locally

```bash
cd app-code
docker build -t nginx-app:local .
```

### Running the Container Locally

```bash
docker run -d -p 8080:80 --name nginx-app-local nginx-app:local
```

The application will be available at `http://localhost:8080`.

### Stopping and Removing the Container

```bash
docker stop nginx-app-local
docker rm nginx-app-local
```

### Verifying the Image

```bash
docker images nginx-app
docker inspect nginx-app:local
```

---

## Application Structure

```
app-code/
├── index.html      # Static HTML served by the Nginx container
└── Dockerfile      # Multi-stage or single-stage container image definition
```

### Dockerfile Reference

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

---

## Related Repository

| Repository | Purpose |
|---|---|
| [`gitops-multi-env-deployment`](https://github.com/dvanhu/gitops-multi-env-deployment) | Kubernetes manifests, Kustomize overlays, and ArgoCD configuration for multi-environment deployment |
