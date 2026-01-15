# Aphex Pipeline Resources

A catalog of reusable Tekton resources for the Arbiter platform.

## Overview

This repository provides:
- **Tekton Tasks**: Reusable CI/CD building blocks (build, test, deploy)
- **Dispatcher Templates**: Thin execution envelopes that convert webhook events into PipelineRuns

Product teams reference these resources when building their pipelines.

## Available Resources

### Tekton Tasks

- **buildah-build**: Build and push container images with Buildah
- **python-test**: Run Python tests with pytest
- **kubectl-apply**: Apply Kubernetes manifests with kubectl

### Dispatcher Templates

- **run-pipeline-v1**: Standard webhook-to-PipelineRun dispatcher

## Quick Start

### Using Tasks in Pipelines

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: my-pipeline
spec:
  tasks:
    - name: test
      taskRef:
        resolver: cluster
        params:
          - name: name
            value: python-test
          - name: namespace
            value: tekton-pipelines
      workspaces:
        - name: source
          workspace: shared-workspace
```

### Using Dispatcher Templates

```yaml
apiVersion: arbiter.io/v1alpha1
kind: RepoBinding
metadata:
  name: my-repo-binding
spec:
  aphexOrg: my-org
  repoOrg: github-org
  repoName: my-repo
  pipelineName: my-pipeline
  templateRef: run-pipeline-v1
```

## Deployment

Deploy all tasks to your cluster:

```bash
kubectl apply -k tekton/
```

This creates tasks in the `tekton-pipelines` namespace.

## Documentation

Complete documentation is available in `.kiro/docs/`:
- [Overview](.kiro/docs/overview.md) - Purpose and key concepts
- [Architecture](.kiro/docs/architecture.md) - System design and components
- [Operations](.kiro/docs/operations.md) - Deployment and troubleshooting
- [API](.kiro/docs/api.md) - Task and template interfaces
- [Data Models](.kiro/docs/data-models.md) - Resource schemas
- [FAQ](.kiro/docs/faq.md) - Common questions and answers

## Adding New Tasks

1. Create task YAML in `tekton/tasks/`
2. Add to `tekton/kustomization.yaml`
3. Test: `kubectl apply -k tekton/ --dry-run=client`
4. Deploy: `kubectl apply -k tekton/`
5. Document in `.kiro/docs/`

## Philosophy

### Tasks are Reusable
Tasks are generic and parameterized for multiple use cases. They contain no product-specific logic.

### Dispatchers are Thin
Dispatcher templates contain **no workflow logic** - only parameter transformation and PipelineRun creation. All workflow logic lives in product-owned pipelines.

### Platform Ownership
Resources are managed by the platform team and versioned. Product teams reference them via cluster resolver.

## Related Repositories

- **ArbiterPipelineInfrastructure**: Platform infrastructure and controllers
- **AphexCLI**: Command-line tool for managing pipelines
- **AphexPipelineTemplate**: Template for creating new product pipelines

## Archon Integration

This repository participates in the **Archon** RAG system. Documentation under `.kiro/docs/` is ingested to provide context for automated agents and engineers.

See `CLAUDE.md` for the complete documentation contract.

## License

MIT
