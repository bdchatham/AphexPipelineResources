# Tekton Tasks

This directory contains reusable Tekton tasks for the Arbiter platform.

## Available Tasks

### argocd-deployment

Creates or updates an ArgoCD Application resource to deploy applications using GitOps.

**Parameters:**
- `app-name` (required): Name of the ArgoCD Application
- `repo-url` (required): Git repository URL containing the manifests
- `repo-revision` (default: "main"): Git revision (branch, tag, or commit SHA)
- `manifest-path` (required): Path within the repository containing Kubernetes manifests
- `target-namespace` (required): Target namespace where the application will be deployed
- `argocd-namespace` (default: "argocd"): Namespace where ArgoCD is installed
- `auto-sync` (default: "true"): Enable automatic sync when Git repository changes
- `prune` (default: "true"): Enable pruning of resources no longer in Git
- `self-heal` (default: "true"): Enable self-healing when cluster state drifts from Git
- `create-namespace` (default: "true"): Create target namespace if it doesn't exist

**RBAC Requirements:**

To use this task, your pipeline's ServiceAccount needs permission to create ArgoCD Applications. The platform provides a shared ClusterRole for this:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: my-pipeline-argocd-deployer
  namespace: argocd
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: argocd-application-deployer
subjects:
  - kind: ServiceAccount
    name: pipeline-runner
    namespace: my-pipeline-namespace
```

**Example Usage:**

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: deploy-my-app
spec:
  params:
    - name: git-url
      type: string
    - name: git-revision
      type: string
      default: main
  
  tasks:
    - name: deploy-to-argocd
      taskRef:
        name: argocd-deployment
      params:
        - name: app-name
          value: my-application
        - name: repo-url
          value: $(params.git-url)
        - name: repo-revision
          value: $(params.git-revision)
        - name: manifest-path
          value: manifests/production
        - name: target-namespace
          value: my-app-prod
```

### buildah-build

Builds container images using Buildah.

### kubectl-apply

Applies Kubernetes manifests using kubectl.

### python-test

Runs Python tests using pytest.

## Deployment

These tasks are deployed to the `tekton-pipelines` namespace via Kustomize:

```bash
kubectl apply -k tekton/
```

The platform automatically deploys these tasks via ArgoCD.
