# Architecture

## System Design

AphexPipelineResources is a **resource catalog repository** that provides reusable Tekton components for the Arbiter platform. It follows a clear separation between platform-managed resources (this repo) and product-owned pipelines (product repos).

### Design Principles

1. **Reusability**: Tasks are generic and parameterized for multiple use cases
2. **Thin Dispatchers**: Templates contain no workflow logic, only parameter transformation
3. **Platform Ownership**: Resources are managed by the platform team and versioned
4. **Cluster Resolution**: Resources are referenced via Tekton's cluster resolver

## Components

### Tekton Tasks

Reusable CI/CD building blocks deployed to the `tekton-pipelines` namespace.

#### buildah-build
Builds and pushes container images using Buildah.

**Parameters:**
- `IMAGE`: Container image reference (e.g., `registry.io/org/image:tag`)
- `DOCKERFILE`: Path to Dockerfile (default: `./Dockerfile`)
- `CONTEXT`: Build context directory (default: `.`)
- `TLSVERIFY`: Verify registry TLS (default: `true`)

**Workspaces:**
- `source`: Source code workspace containing Dockerfile and context

**Behavior:**
- Builds OCI-format images with no cache
- Pushes to specified registry
- Requires privileged security context for Buildah

#### python-test
Runs Python tests using pytest.

**Parameters:**
- `test-command`: Command to execute tests (default: `pytest`)

**Workspaces:**
- `source`: Source code workspace containing Python project

**Behavior:**
- Installs dependencies from `setup.py` or `pyproject.toml` (editable install with `[test]` extras)
- Runs specified test command
- Uses Python 3.11-slim image

#### kubectl-apply
Applies Kubernetes manifests using kubectl.

**Parameters:**
- `manifest-dir`: Directory containing Kubernetes manifests
- `namespace`: Target namespace (optional, uses manifest namespace if not specified)

**Workspaces:**
- `source`: Source code workspace containing manifests

**Behavior:**
- Detects and uses Kustomize if `kustomization.yaml` exists
- Falls back to direct `kubectl apply -f` for plain manifests
- Validates directory existence before applying

### Dispatcher Templates

Thin TriggerTemplates that convert webhook events into PipelineRuns.

#### run-pipeline-v1
Standard dispatcher template for webhook-triggered pipelines.

**Version**: v1  
**Status**: Stable  
**Contract**: Pipeline Contract v1

**Input Parameters:**
- `git-url`: Repository clone URL (from TriggerBinding)
- `git-revision`: Commit SHA or branch (from TriggerBinding)
- `repo-full-name`: Repository name as `org/repo` (from TriggerBinding)
- `event-type`: Webhook event type (from TriggerBinding)
- `event-id`: Unique event identifier (from TriggerBinding)
- `pipeline-name`: Target pipeline name (from RepoBinding)
- `pipeline-namespace`: Target namespace (from RepoBinding)
- `execution-role`: Execution profile - `standard` or `elevated` (from RepoBinding)
- `org-name`: Aphex organization name (from Organization)
- `triggered-at`: ISO8601 timestamp (from platform)

**Output:**
Creates a PipelineRun in the target namespace with:
- Generated name: `{pipeline-name}-{random}`
- Labels for traceability (event-type, event-id)
- Canonical parameters per Pipeline Contract v1
- ServiceAccount: `pipeline-runner`
- Timeout: 1 hour

**What it does:**
- Receives webhook parameters from TriggerBinding
- Creates PipelineRun in target namespace
- Injects canonical parameters
- Applies execution profile settings
- Sets labels for traceability

**What it doesn't do:**
- No git operations
- No build logic
- No deployment logic
- No conditional execution
- No workflow composition

## Technology Stack

- **Tekton Pipelines**: CI/CD framework (v1beta1 and v1 APIs)
- **Buildah**: Container image building (v1.23.1)
- **Python**: Testing runtime (3.11-slim)
- **kubectl**: Kubernetes manifest application (bitnami/kubectl:latest)
- **Kustomize**: Kubernetes resource management (built into kubectl)

## Architectural Patterns

### Cluster Resolver Pattern
Tasks are referenced using Tekton's cluster resolver, allowing pipelines to reference tasks by name and namespace without embedding full task definitions.

```yaml
taskRef:
  resolver: cluster
  params:
    - name: name
      value: python-test
    - name: namespace
      value: tekton-pipelines
```

### Thin Dispatcher Pattern
Dispatcher templates are intentionally minimal (< 100 lines) and contain no workflow logic. All workflow decisions live in product-owned pipelines.

### Workspace Pattern
Tasks use Tekton workspaces for sharing data between steps and tasks, following Tekton best practices for data flow.

### Parameterization Pattern
Tasks are highly parameterized with sensible defaults, allowing reuse across different contexts while maintaining flexibility.

## Dependencies

### Upstream Dependencies
- **Tekton Pipelines**: Core Tekton installation in cluster
- **Container Registry**: For pushing built images (buildah-build)
- **Kubernetes API**: For applying manifests (kubectl-apply)

### Downstream Dependencies
- **Product Pipelines**: Reference tasks from this catalog
- **RepoBindings**: Reference dispatcher templates
- **Onboarding Controller**: Materializes dispatcher templates into org namespaces

## Deployment Model

Resources are deployed using Kustomize to the `tekton-pipelines` namespace:

```bash
kubectl apply -k tekton/
```

This creates:
- Task: `buildah-build`
- Task: `python-test`
- Task: `kubectl-apply`

Dispatcher templates are stored in `templates/dispatchers/` and materialized by the onboarding controller when referenced by RepoBindings.

**Source**
- `tekton/tasks/` - Tekton Task definitions
- `tekton/kustomization.yaml` - Kustomize configuration
- `templates/dispatchers/` - Dispatcher template definitions
- `templates/dispatchers/README.md` - Dispatcher philosophy and guidelines
