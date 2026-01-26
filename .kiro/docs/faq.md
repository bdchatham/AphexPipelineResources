# FAQ

## General Questions

### What is this repository for?
AphexPipelineResources is a catalog of reusable Tekton resources for the Aphex platform. It provides:
- Tekton Tasks for common CI/CD operations (build, test, deploy)
- Dispatcher Templates that convert webhook events into PipelineRuns

Product teams reference these resources when building their pipelines.

### How does this fit into the larger system?
This repository is part of the Aphex platform's CI/CD infrastructure:
- **Platform Team** (this repo): Provides reusable tasks and dispatcher templates
- **Product Teams**: Compose tasks into pipelines and define workflow logic
- **Onboarding Controller**: Materializes dispatcher templates into org namespaces
- **AphexCLI**: Manages pipeline deployments that reference these resources

## Development Questions

### How do I set up my development environment?
1. Install kubectl: `brew install kubectl`
2. Configure kubeconfig for your cluster
3. Clone this repository
4. Verify access: `kubectl get tasks -n tekton-pipelines`

### How do I test task changes?
```bash
# Test locally with dry-run
kubectl apply -k tekton/ --dry-run=client

# Deploy to test namespace
kubectl apply -k tekton/ -n tekton-pipelines-test

# Create test PipelineRun
kubectl create -f test-pipelinerun.yaml
```

### How do I add a new task?
1. Create task YAML in `tekton/tasks/`
2. Add to `tekton/kustomization.yaml`
3. Test deployment: `kubectl apply -k tekton/ --dry-run=client`
4. Deploy: `kubectl apply -k tekton/`
5. Document in architecture.md and api.md

## Operational Questions

### How do I deploy changes?
```bash
# Deploy all tasks
kubectl apply -k tekton/

# Verify deployment
kubectl get tasks -n tekton-pipelines
```

Existing PipelineRuns continue using old task versions. New PipelineRuns use updated definitions.

### What should I do if a task fails?
1. Check TaskRun logs: `kubectl logs -n <namespace> <taskrun-name> --all-containers`
2. Verify task parameters are correct
3. Check workspace is properly mounted
4. Review task definition: `kubectl get task <task-name> -n tekton-pipelines -o yaml`
5. See operations.md for specific troubleshooting guides

### How do I update a dispatcher template?
1. **Non-breaking changes**: Update template YAML in place
2. **Breaking changes**: Create new version (e.g., `run-pipeline-v2.yaml`)
3. Update `templates/dispatchers/README.md`
4. Communicate changes to product teams
5. Onboarding controller uses updated template for new RepoBindings

### How are tasks versioned?
Tasks are **not** versioned with suffixes. They are updated in place. Breaking changes require:
- Communication to product teams
- Coordination for pipeline updates
- Deprecation period if removing functionality

## Usage Questions

### How do I reference a task in my pipeline?
Use Tekton's cluster resolver:

```yaml
taskRef:
  resolver: cluster
  params:
    - name: name
      value: python-test
    - name: namespace
      value: tekton-pipelines
```

### How do I reference a dispatcher template?
In your RepoBinding:

```yaml
apiVersion: aphex.io/v1alpha1
kind: RepoBinding
spec:
  templateRef: run-pipeline-v1
```

### Can I customize task behavior?
Yes, through parameters:

```yaml
params:
  - name: test-command
    value: "pytest --cov=src tests/"
```

Tasks are designed to be parameterized for flexibility.

### What workspaces do tasks require?
All tasks require a `source` workspace containing the repository code:

```yaml
workspaces:
  - name: source
    workspace: shared-workspace
```

### Why does buildah-build require privileged mode?
Buildah needs privileged mode to:
- Create container images
- Mount filesystems
- Manage container storage

This is a Buildah requirement, not a task design choice.

### How do I use the argocd-deployment task?
The `argocd-deployment` task creates ArgoCD Applications for GitOps deployment:

```yaml
- name: deploy-to-argocd
  taskRef:
    resolver: cluster
    params:
      - name: name
        value: argocd-deployment
      - name: namespace
        value: tekton-pipelines
  params:
    - name: app-name
      value: my-app
    - name: app-project
      value: my-project
    - name: repo-url
      value: https://github.com/org/repo.git
    - name: repo-revision
      value: $(params.git-revision)
    - name: manifest-path
      value: manifests/
    - name: target-namespace
      value: my-app-namespace
```

Your pipeline ServiceAccount must have the `argocd-application-deployer` ClusterRole binding.

**Source**
- `tekton/tasks/argocd-deployment.yaml`

## Archon-Specific Questions

### How is this repository ingested by Archon?
Archon reads all Markdown files under `.kiro/docs/` from this public GitHub repository. Documentation follows the contract defined in `CLAUDE.md`.

### How do I update documentation?
Update the relevant files under `.kiro/docs/` and ensure changes are grounded in code. Include "Source" references to relevant files.

### What documentation standards apply?
- Ground all statements in actual code
- Use RAG-friendly structure (400-800 token sections)
- Include "Source" references
- Maintain consistent terminology
- See `CLAUDE.md` for complete standards

**Source**
- `tekton/tasks/` - Task implementations
- `templates/dispatchers/` - Dispatcher templates
- `tekton/kustomization.yaml` - Deployment configuration
- `CLAUDE.md` - Documentation contract
- `.kiro/steering/archon-docs.md` - Documentation steering
