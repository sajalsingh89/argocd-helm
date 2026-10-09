# argocd-helm

## Overview

This repository contains a Helm chart for deploying the `argocd-learning` web application to Kubernetes using Argo CD.

The Helm chart defines the Kubernetes Deployment and Service. GitHub Actions validates and packages the chart, while Argo CD reads the chart directly from GitHub and deploys it to Kubernetes.

## Repository Structure

```text
argocd-helm/
├── helm/
│   └── argocd-helm/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── .helmignore
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
├── argocd/
│   └── application.yaml
├── .github/
│   └── workflows/
│       └── helm-chart.yaml
└── README.md
```

## Helm Implementation

### 1. Chart.yaml

Defines the Helm chart metadata.

- Chart name: `argocd-helm`
- Chart version: `0.1.0`
- Chart type: `application`

### 2. values.yaml

Defines the default deployment configuration:

- Number of replicas
- Container image repository and tag
- Image pull policy
- Kubernetes image-pull Secret reference
- Service type and ports

The application image is pulled from GitHub Container Registry (GHCR):

```text
ghcr.io/sajalsingh89/argocd-learning:latest
```

The `ghcr-secret` Kubernetes Secret provides authentication for the private image. The actual credentials are not stored in Git.

### 3. templates/deployment.yaml

Defines the Kubernetes Deployment using Helm template expressions.

Helm substitutes values from `values.yaml` to generate the final Kubernetes Deployment manifest.

### 4. templates/service.yaml

Defines a ClusterIP Service that exposes the web application internally within the Kubernetes cluster.

### 5. .helmignore

Excludes unnecessary files from the packaged Helm chart.

## GitHub Actions Workflow

Workflow file:

```text
.github/workflows/helm-chart.yaml
```

The workflow runs on GitHub-hosted runners when relevant chart files change on the `main` branch. It can also be triggered manually.

### Workflow steps

1. Check out the Git repository.
2. Install Helm.
3. Validate the chart using `helm lint`.
4. Render the Kubernetes manifests using `helm template`.
5. Package the chart into a `.tgz` archive.
6. Upload the packaged chart as a GitHub Actions artifact.

The package is generated using:

```bash
helm package ./helm/argocd-helm --destination ./dist
```

The resulting artifact is named similar to:

```text
argocd-helm-0.1.0.tgz
```

**Note:** Argo CD deploys the Helm chart directly from Git. It does not need the `.tgz` artifact produced by GitHub Actions.

## Deploying Through Argo CD

### 1. Argo CD Application configuration

Application manifest:

```text
argocd/application.yaml
```

The relevant configuration is:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-helm
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/sajalsingh89/argocd-helm.git
    targetRevision: main
    path: helm/argocd-helm
    helm:
      releaseName: argocd-helm

  destination:
    server: https://kubernetes.default.svc
    namespace: argocd-helm

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

If the repository is private, configure its Git credentials in Argo CD.

### 2. Prepare the Kubernetes namespace

The following commands assume Argo CD is already installed on Docker Desktop Kubernetes.

```bash
kubectl config use-context docker-desktop

kubectl create namespace argocd-helm
```

Create the `ghcr-secret` image-pull Secret in the `argocd-helm` namespace using a GitHub classic personal access token with `read:packages` permission.

Do not commit the token or Kubernetes Secret to Git.

### 3. Create the Argo CD Application

After committing and pushing the repository files, apply the Application manifest:

```bash
kubectl apply -f argocd/application.yaml
```

This command assumes the repository has been checked out locally. Alternatively, create the Application through the Argo CD UI.

### 4. Verify the deployment

Check the Argo CD Application:

```bash
kubectl get applications -n argocd
```

Check the Kubernetes resources:

```bash
kubectl get deployments,pods,services -n argocd-helm
```

### 5. Access the application

Forward the Service port:

```bash
kubectl port-forward svc/argocd-helm \
  -n argocd-helm 8081:80
```

Open the following URL in your browser:

```text
http://localhost:8081
```

## Deployment Flow

```text
GitHub Repository
       |
       v
GitHub Actions
       |
       +--> Helm lint
       +--> Helm template
       +--> Helm package
       |
       v
Argo CD reads Helm chart from Git
       |
       v
Helm templates rendered by Argo CD
       |
       v
Kubernetes API
       |
       +--> Deployment
       |       |
       |       v
       |      Pods
       |
       +--> ClusterIP Service
```

## Automatic Updates

When Helm chart files are changed and pushed to `main`:

1. GitHub Actions validates the chart and packages it.
2. Argo CD detects changes to the Git repository.
3. Argo CD renders the updated Helm chart.
4. With automated sync enabled, Argo CD applies the updated resources to Kubernetes.

Changing the application image tag in `values.yaml` can trigger a rollout. Publishing a new image under the same `latest` tag alone does not guarantee that Argo CD will deploy it. Immutable image tags
