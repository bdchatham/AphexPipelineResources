# Data Models

## Overview

AphexPipelineResources uses Tekton's declarative YAML-based data models. This document describes the structure of Task and TriggerTemplate resources provided by this catalog.

## Tekton Task Schema

### Task Metadata

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: string              # Task name (unique within namespace)
  namespace: string         # Namespace (tekton-pipelines for catalog tasks)
  labels:                   # Optional labels
    key: value
spec:
  # Task specification
```

### Task Specification

```yaml
spec:
  description: string       # Human-readable task description
  params:                   # Input parameters
    - name: string
      description: string
      type: string          # string, array, or object
      default: string       # Optional default value
  workspaces:               # Workspace declarations
    - name: string
      description: string
      optional: boolean     # Whether workspace is optional
  steps:                    # Execution steps
    - name: string
      image: string         # Container image
      script: string        # Shell script to execute
      workingDir: string    # Working directory
      securityContext:      # Security settings
        privileged: boolean
      volumeMounts:         # Volume mounts
        - name: string
          mountPath: string
  volumes:                  # Pod volumes
    - name: string
      emptyDir: {}
```

## Task Data Models

### buildah-build Task

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: buildah-build
spec:
  params:
    - name: IMAGE
      type: string
      description: "Container image reference"
    - name: DOCKERFILE
      type: string
      default: "./Dockerfile"
      description: "Path to Dockerfile"
    - name: CONTEXT
      type: string
      default: "."
      description: "Build context directory"
    - name: TLSVERIFY
      type: string
      default: "true"
      description: "Verify registry TLS"
  workspaces:
    - name: source
      description: "Source code workspace"
  steps:
    - name: build-and-push
      image: quay.io/buildah/stable:v1.23.1
      securityContext:
        privileged: true
      # Script omitted for brevity
  volumes:
    - name: varlibcontainers
      emptyDir: {}
```

### python-test Task

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: python-test
spec:
  params:
    - name: test-command
      type: string
      default: "pytest"
      description: "Command to run tests"
  workspaces:
    - name: source
      description: "Source code workspace"
  steps:
    - name: install-deps
      image: python:3.11-slim
      # Script omitted for brevity
    - name: run-tests
      image: python:3.11-slim
      # Script omitted for brevity
```

### kubectl-apply Task

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: kubectl-apply
spec:
  params:
    - name: manifest-dir
      type: string
      description: "Directory containing manifests"
    - name: namespace
      type: string
      default: ""
      description: "Target namespace"
  workspaces:
    - name: source
      description: "Source code workspace"
  steps:
    - name: apply-manifests
      image: bitnami/kubectl:latest
      # Script omitted for brevity
```

## TriggerTemplate Schema

### TriggerTemplate Metadata

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: string
  namespace: string
  labels:
    platform.aphex/template-version: string
    platform.aphex/managed-by: string
spec:
  # Template specification
```

### TriggerTemplate Specification

```yaml
spec:
  params:                   # Input parameters
    - name: string
      description: string
      default: string       # Optional default
  resourcetemplates:        # Resources to create
    - apiVersion: string
      kind: string
      metadata:
        generateName: string
        namespace: string
        labels:
          key: value
      spec:
        # Resource specification
```

## Dispatcher Template Data Model

### run-pipeline-v1 Template

```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: run-pipeline-v1
  labels:
    platform.aphex/template-version: v1
    platform.aphex/managed-by: platform
spec:
  params:
    # Webhook parameters
    - name: git-url
      description: "Repository clone URL"
    - name: git-revision
      description: "Git commit SHA or branch"
    - name: repo-full-name
      description: "Repository name (org/repo)"
    - name: event-type
      description: "Webhook event type"
    - name: event-id
      description: "Unique event identifier"
    
    # Pipeline targeting
    - name: pipeline-name
      description: "Name of pipeline to run"
    - name: pipeline-namespace
      description: "Namespace where pipeline exists"
    
    # Execution parameters
    - name: execution-role
      description: "Execution profile (standard/elevated)"
      default: standard
    - name: org-name
      description: "Organization name"
      default: ""
    - name: triggered-at
      description: "Webhook timestamp"
      default: ""
  
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: "$(tt.params.pipeline-name)-"
        namespace: "$(tt.params.pipeline-namespace)"
        labels:
          platform.aphex/triggered: "true"
          platform.aphex/event-type: "$(tt.params.event-type)"
          platform.aphex/event-id: "$(tt.params.event-id)"
      spec:
        pipelineRef:
          resolver: cluster
          params:
            - name: name
              value: "$(tt.params.pipeline-name)"
            - name: namespace
              value: "$(tt.params.pipeline-namespace)"
        params:
          - name: git-url
            value: "$(tt.params.git-url)"
          - name: git-revision
            value: "$(tt.params.git-revision)"
          - name: repo-full-name
            value: "$(tt.params.repo-full-name)"
          - name: event-type
            value: "$(tt.params.event-type)"
          - name: event-id
            value: "$(tt.params.event-id)"
          - name: triggered-at
            value: "$(tt.params.triggered-at)"
          - name: org-name
            value: "$(tt.params.org-name)"
        serviceAccountName: pipeline-runner
        timeout: 1h
```

## Data Flow

### Task Execution Flow

1. **PipelineRun Creation**: Pipeline creates TaskRun referencing catalog task
2. **Parameter Binding**: TaskRun parameters bound to Task parameters
3. **Workspace Mounting**: Workspaces mounted into task pod
4. **Step Execution**: Steps execute sequentially in task pod
5. **Result Collection**: Task results captured and returned to PipelineRun

### Dispatcher Template Flow

1. **Webhook Event**: GitHub webhook triggers EventListener
2. **TriggerBinding**: Extracts parameters from webhook payload
3. **TriggerTemplate**: Receives parameters and creates PipelineRun
4. **PipelineRun Execution**: Pipeline executes with canonical parameters
5. **Status Update**: PipelineRun status tracked via labels

## Validation Rules

### Task Parameter Validation

- **IMAGE** (buildah-build): Must be valid container image reference (registry/org/image:tag)
- **DOCKERFILE** (buildah-build): Must be relative path within workspace
- **CONTEXT** (buildah-build): Must be relative path within workspace
- **test-command** (python-test): Must be valid shell command
- **manifest-dir** (kubectl-apply): Must be relative path within workspace
- **namespace** (kubectl-apply): Must be valid Kubernetes namespace name (DNS label)

### Template Parameter Validation

- **git-url**: Must be valid Git URL (https:// or git@)
- **git-revision**: Must be valid Git ref (SHA, branch, or tag)
- **repo-full-name**: Must match pattern `org/repo`
- **pipeline-name**: Must be valid Kubernetes resource name
- **pipeline-namespace**: Must be valid Kubernetes namespace name
- **execution-role**: Must be `standard` or `elevated`

### Workspace Validation

- **source workspace**: Must be provided for all tasks
- **Workspace path**: Must be absolute path starting with `/`
- **Workspace volume**: Must be backed by PVC, emptyDir, or configMap

## Kustomize Configuration

### kustomization.yaml Structure

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:                  # List of resources to deploy
  - tasks/python-test.yaml
  - tasks/buildah-build.yaml
  - tasks/kubectl-apply.yaml

namespace: tekton-pipelines  # Target namespace for all resources
```

**Source**
- `tekton/tasks/buildah-build.yaml` - Buildah task schema
- `tekton/tasks/python-test.yaml` - Python test task schema
- `tekton/tasks/kubectl-apply.yaml` - kubectl apply task schema
- `templates/dispatchers/run-pipeline-v1.yaml` - Dispatcher template schema
- `tekton/kustomization.yaml` - Kustomize configuration
