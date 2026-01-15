# Overview

## Purpose

AphexPipelineResources provides a catalog of reusable Tekton resources for the Arbiter platform. It contains:
- **Tekton Tasks**: Reusable CI/CD building blocks for common operations
- **Dispatcher Templates**: Thin execution envelopes that convert webhook events into PipelineRuns

This repository serves as the **platform-managed resource catalog** that product teams reference when building their pipelines.

## Archon Integration

This repository is ingested by the **Archon** RAG system, which reads all Markdown files under `.kiro/docs/` to build mental models for sourcing code and architectural information.

Documentation in this repository follows the Archon documentation contract defined in `CLAUDE.md` at the repo root.

## Quick Start

### Using Tekton Tasks

Reference tasks from this catalog in your pipeline:

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
```

### Using Dispatcher Templates

Reference templates in your RepoBinding:

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
  templateRef: run-pipeline-v1  # References dispatcher template
```

## Key Concepts

### Tekton Tasks
Reusable units of work that perform specific operations (build, test, deploy). Tasks are platform-managed and versioned.

### Dispatcher Templates
Thin TriggerTemplates that convert webhook events into PipelineRuns. They contain **no workflow logic** - only parameter transformation and PipelineRun creation.

### Resource Catalog
This repository acts as a centralized catalog. Resources are deployed to the `tekton-pipelines` namespace and referenced by product pipelines.

### Separation of Concerns
- **Platform** (this repo): Provides reusable tasks and dispatcher templates
- **Product Teams**: Compose tasks into pipelines and define workflow logic

## Available Resources

### Tekton Tasks
- `buildah-build` - Build and push container images
- `python-test` - Run Python tests with pytest
- `kubectl-apply` - Apply Kubernetes manifests

### Dispatcher Templates
- `run-pipeline-v1` - Standard webhook-to-PipelineRun dispatcher

## Related Repositories

- **ArbiterPipelineInfrastructure**: Platform infrastructure and controllers
- **AphexCLI**: Command-line tool for managing pipelines
- **AphexPipelineTemplate**: Template for creating new product pipelines

**Source**
- `tekton/` - Tekton Task definitions
- `templates/dispatchers/` - Dispatcher template definitions
- `tekton/kustomization.yaml` - Resource deployment configuration
