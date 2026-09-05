# GitOps Continuous Deployment with Argo CD

This directory contains the declarative **Argo CD Application** manifest (`argocd-app.yaml`).

## GitOps Architecture

1. **Jenkins (CI)** builds versioned Docker images, pushes them to Docker Hub (`onkar2003/wanderlust-backend:vX.Y.Z`), updates `kubernetes/*.yaml` manifests, and commits/pushes back to GitHub.
2. **Argo CD (CD Operator)** runs inside the Kubernetes cluster in the `argocd` namespace.
3. Argo CD continuously monitors `https://github.com/onkar1502/Wanderlust-Mega-Project.git` under path `kubernetes/`.
4. When Git changes are detected (e.g. image tag updates or manual replica count adjustments), Argo CD automatically reconciles and syncs the live Kubernetes cluster state.

## Deploying Argo CD Application

To apply the Argo CD application definition into your Kubernetes cluster:

```bash
kubectl apply -f GitOps/argocd-app.yaml
```
