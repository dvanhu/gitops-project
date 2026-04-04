# GitOps Project

A **GitOps CI/CD project** that automates the full lifecycle of a containerized web application — from source code to running Kubernetes deployment — using GitHub Actions for continuous integration and ArgoCD for continuous delivery.

---

## Overview

This project separates application code from deployment concerns:

- **`app-code/`** — the application source and Dockerfile
- **`.github/workflows/`** — the CI pipeline that builds and pushes Docker images
- A **separate GitOps config repository** (or a dedicated path) that ArgoCD watches for Kubernetes manifest changes

---

## Architecture

```
Developer pushes to main
        │
        ▼
GitHub Actions (CI)
  ├── Build Docker image (HTML app)
  └── Push image to registry
        │
        ▼
Update Kubernetes manifests (image tag)
        │
        ▼
ArgoCD detects manifest change
        │
        ▼
Kubernetes cluster synced & updated
```

---

## Project Structure

```
gitops-project/
├── .github/
│   └── workflows/
│       └── ci.yml            # GitHub Actions CI pipeline
└── app-code/
    ├── index.html            # Web application (HTML frontend)
    └── Dockerfile            # Container image definition
```

**Language breakdown:** HTML 62% · Dockerfile 38%

---

## Tech Stack

| Layer | Tool |
|---|---|
| Application | HTML (static web app) |
| Containerization | Docker |
| CI Pipeline | GitHub Actions |
| GitOps / CD | ArgoCD |
| Orchestration | Kubernetes |

---

## Getting Started

### Prerequisites

- Docker installed
- A Kubernetes cluster with ArgoCD
- DockerHub (or any container registry) account

### Run Locally with Docker

```bash
git clone https://github.com/dvanhu/gitops-project.git
cd gitops-project/app-code

# Build the image
docker build -t gitops-app .

# Run the container
docker run -p 8080:80 gitops-app
```

Open `http://localhost:8080` in your browser.

---

## CI Pipeline (GitHub Actions)

The workflow in `.github/workflows/ci.yml` triggers on every push to `main`:

```
1. Checkout source code
2. Login to container registry
3. Build Docker image from app-code/Dockerfile
4. Tag image (e.g., with Git commit SHA)
5. Push image to registry
6. Update Kubernetes manifest with new image tag
```

### Required GitHub Secrets

| Secret | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Your DockerHub username |
| `DOCKERHUB_TOKEN` | DockerHub access token |

---

## Kubernetes Deployment (via ArgoCD)

After the CI pipeline pushes a new image, ArgoCD picks up the manifest update and syncs the cluster automatically.

### Register the app with ArgoCD

```bash
argocd app create gitops-app \
  --repo https://github.com/dvanhu/gitops-project.git \
  --path . \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated
```

ArgoCD will then keep the cluster in sync with whatever is declared in the repository.

---

## Key Concepts Demonstrated

- **GitOps workflow** — Git is the only mechanism to trigger deployments
- **Immutable images** — every build produces a uniquely tagged image
- **Separation of concerns** — app code and deployment config are clearly divided
- **Automated delivery** — zero manual steps from commit to running pod
- **Lightweight app** — simple HTML + Dockerfile, focus is entirely on the pipeline
