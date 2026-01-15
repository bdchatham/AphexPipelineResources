# API

## Overview

AphexPipelineResources provides a **declarative API** through Tekton Task and TriggerTemplate resources. Product teams interact with these resources by referencing them in their pipelines and RepoBindings.

This document describes the task parameter interfaces and dispatcher template contracts.

## Tekton Task APIs

### buildah-build

Build and push container images using Buildah.

**Task Reference:**
```yaml
taskRef:
  resolver: cluster
  params:
    - name: name
      value: buildah-build
    - name: namespace
      value: tekton-pipelines
```

**Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `IMAGE` | string | Yes | - | Container image reference (e.g., `registry.io/org/image:tag`) |
| `DOCKERFILE` | string | No | `./Dockerfile` | Path to Dockerfile relative to workspace |
| `CONTEXT` | string | No | `.` | Build context directory relative to workspace |
| `TLSVERIFY` | string | No | `true` | Verify TLS on registry endpoint |

**Workspaces:**

| Workspace | Description | Required |
|-----------|-------------|----------|
| `source` | Source code containing Dockerfile and build context | Yes |

**Example Usage:**
```yaml
- name: build-image
  taskRef:
    resolver: cluster
    params:
      - name: name
        value: buildah-build
      - name: namespace
        value: tekton-pipelines
  params:
    - name: IMAGE
      value: "registry.io/myorg/myapp:$(params.git-revision)"
    - name: DOCKERFILE
      value: "./docker/Dockerfile"
    - name: CONTEXT
      value: "."
  workspaces:
    - name: source
      workspace: shared-workspace
```

### python-test

Run Python tests using pytest.

**Task Reference:**
```yaml
taskRef:
  resolver: cluster
  params:
    - name: name
      value: python-test
    - name: namespace
      value: tekton-pipelines
```

**Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `test-command` | string | No | `pytest` | Command to execute tests |

**Workspaces:**

| Workspace | Description | Required |
|-----------|-------------|----------|
| `source` | Source code containing Python project with setup.py or pyproject.toml | Yes |

**Example Usage:**
```yaml
- name: run-tests
  taskRef:
    resolver: cluster
    params:
      - name: name
        value: python-test
      - name: namespace
        value: tekton-pipelines
  params:
    - name: test-command
      value: "pytest --cov=src tests/"
  workspaces:
    - name: source
      workspace: shared-workspace
```

### kubectl-apply

Apply Kubernetes manifests using kubectl.

**Task Reference:**
```yaml
taskRef:
  resolver: cluster
  params:
    - name: name
      value: kubectl-apply
    - name: namespace
      value: tekton-pipelines
```

**Parameters:**

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `manifest-dir` | string | Yes | - | Directory containing Kubernetes manifests |
| `namespace` | string | No | `""` | Target namespace (uses manifest namespace if not specified) |

**Workspaces:**

| Workspace | Description | Required |
|-----------|-------------|----------|
| `source` | Source code containing Kubernetes manifests | Yes |

**Example Usage:**
```yaml
- name: deploy-manifests
  taskRef:
    resolver: cluster
    params:
      - name: name
        value: kubectl-apply
      - name: namespace
        value: tekton-pipelines
  params:
    - name: manifest-dir
      value: "./k8s/overlays/production"
    - name: namespace
      value: "my-app-prod"
  workspaces:
    - name: source
      workspace: shared-workspace
```

## Dispatcher Template API

### run-pipeline-v1

Standard dispatcher template for webhook-triggered pipelines.

**Template Reference:**
```yaml
apiVersion: arbiter.io/v1alpha1
kind: RepoBinding
spec:
  templateRef: run-pipeline-v1
```

**Input Parameters:**

The template receives parameters from the TriggerBinding and RepoBinding:

| Parameter | Source | Type | Description |
|-----------|--------|------|-------------|
| `git-url` | TriggerBinding | string | Repository clone URL |
| `git-revision` | TriggerBinding | string | Git commit SHA or branch |
| `repo-full-name` | TriggerBinding | string | Repository name as `org/repo` |
| `event-type` | TriggerBinding | string | Webhook event type (push, pull_request, etc.) |
| `event-id` | TriggerBinding | string | Unique event identifier |
| `pipeline-name` | RepoBinding | string | Target pipeline name |
| `pipeline-namespace` | RepoBinding | string | Target namespace |
| `execution-role` | RepoBinding | string | Execution profile (standard/elevated) |
| `org-name` | Organization | string | Aphex organization name |
| `triggered-at` | Platform | string | ISO8601 timestamp |

**Output:**

Creates a PipelineRun with these characteristics:

```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: {pipeline-name}-
  namespace: {pipeline-namespace}
  labels:
    platform.arbiter.io/triggered: "true"
    platform.arbiter.io/event-type: {event-type}
    platform.arbiter.io/event-id: {event-id}
spec:
  pipelineRef:
    resolver: cluster
    params:
      - name: name
        value: {pipeline-name}
      - name: namespace
        value: {pipeline-namespace}
  params:
    - name: git-url
      value: {git-url}
    - name: git-revision
      value: {git-revision}
    - name: repo-full-name
      value: {repo-full-name}
    - name: event-type
      value: {event-type}
    - name: event-id
      value: {event-id}
    - name: triggered-at
      value: {triggered-at}
    - name: org-name
      value: {org-name}
  serviceAccountName: pipeline-runner
  timeout: 1h
```

**Pipeline Contract:**

Pipelines referenced by this template MUST accept these parameters:

- `git-url` (string): Repository clone URL
- `git-revision` (string): Commit SHA or branch
- `repo-full-name` (string): Repository name as `org/repo`
- `event-type` (string): Webhook event type
- `event-id` (string): Unique event identifier
- `triggered-at` (string): ISO8601 timestamp
- `org-name` (string): Organization name

## Authentication

Tasks run with the ServiceAccount specified in the PipelineRun. Common patterns:

- **Default**: `pipeline-runner` ServiceAccount in pipeline namespace
- **Elevated**: `pipeline-runner-elevated` ServiceAccount with additional permissions
- **Custom**: Product-specific ServiceAccount for specialized access

ServiceAccounts must have:
- Permissions to create pods in the namespace
- Registry credentials for pushing images (buildah-build)
- Kubernetes API permissions for applying manifests (kubectl-apply)

## Error Handling

### Task Failures

Tasks fail with non-zero exit codes. Common failure scenarios:

**buildah-build:**
- Registry authentication failure → Check ServiceAccount imagePullSecrets
- Dockerfile not found → Verify DOCKERFILE parameter path
- Build failure → Check build logs for compilation errors

**python-test:**
- Dependency installation failure → Verify setup.py or pyproject.toml
- Test failures → Check test logs for assertion errors
- Import errors → Verify package structure and dependencies

**kubectl-apply:**
- Manifest directory not found → Verify manifest-dir parameter
- Permission denied → Check ServiceAccount RBAC permissions
- Invalid manifest → Validate YAML syntax

### Dispatcher Template Failures

Template failures occur during PipelineRun creation:

- Pipeline not found → Verify pipeline exists in target namespace
- Parameter validation failure → Check parameter types and values
- ServiceAccount not found → Verify ServiceAccount exists in namespace

## Integration Examples

### Complete Pipeline Using All Tasks

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: full-ci-cd
spec:
  params:
    - name: git-url
    - name: git-revision
    - name: image-name
  workspaces:
    - name: shared-workspace
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
    
    - name: build
      runAfter: [test]
      taskRef:
        resolver: cluster
        params:
          - name: name
            value: buildah-build
          - name: namespace
            value: tekton-pipelines
      params:
        - name: IMAGE
          value: "$(params.image-name):$(params.git-revision)"
      workspaces:
        - name: source
          workspace: shared-workspace
    
    - name: deploy
      runAfter: [build]
      taskRef:
        resolver: cluster
        params:
          - name: name
            value: kubectl-apply
          - name: namespace
            value: tekton-pipelines
      params:
        - name: manifest-dir
          value: "./k8s"
        - name: namespace
          value: "production"
      workspaces:
        - name: source
          workspace: shared-workspace
```

### RepoBinding with Dispatcher Template

```yaml
apiVersion: arbiter.io/v1alpha1
kind: RepoBinding
metadata:
  name: my-app-binding
  namespace: platform-system
spec:
  aphexOrg: my-org
  repoOrg: github-org
  repoName: my-app
  pipelineName: full-ci-cd
  templateRef: run-pipeline-v1
```

**Source**
- `tekton/tasks/buildah-build.yaml` - Buildah task definition
- `tekton/tasks/python-test.yaml` - Python test task definition
- `tekton/tasks/kubectl-apply.yaml` - kubectl apply task definition
- `templates/dispatchers/run-pipeline-v1.yaml` - Dispatcher template definition
- `templates/dispatchers/README.md` - Template contract and guidelines
