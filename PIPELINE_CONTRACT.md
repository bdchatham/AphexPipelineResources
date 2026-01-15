# Pipeline Contract v1

## Overview

The Pipeline Contract defines the canonical parameter interface between the Aphex platform and product team pipelines. This contract is stable, versioned, and guarantees backward compatibility.

**Current Version**: `v1`

## Purpose

When the platform triggers a pipeline via webhook, it provides a standard set of parameters. Product teams write Tekton Pipelines that accept these parameters, enabling:
- Consistent webhook-to-pipeline integration
- Multi-repo workspace support (future)
- Platform-managed execution profiles
- Event traceability and debugging

## Required Parameters

All pipelines triggered by the platform **must** accept these parameters:

| Parameter | Type | Description |
|-----------|------|-------------|
| `git-url` | string | Clone URL of the repository |
| `git-revision` | string | Commit SHA or branch ref |
| `repo-full-name` | array | List of repository names (org/repo). Supports multi-repo workspaces. |
| `event-type` | string | Webhook event type (push, pull_request, etc.) |
| `event-id` | string | Unique identifier for this webhook event |

## Optional Parameters

Pipelines **may** use or ignore these platform-injected parameters:

| Parameter | Type | Description | Default |
|-----------|------|-------------|---------|
| `execution-profile` | string | Execution profile name (standard, elevated) | `standard` |
| `triggered-at` | string | ISO8601 timestamp when webhook was received | `""` |
| `org-name` | string | Aphex organization name | `""` |

## Example Pipeline

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: my-pipeline
  namespace: my-pipeline-ns
spec:
  params:
    # Required parameters
    - name: git-url
      type: string
      description: Repository clone URL
    - name: git-revision
      type: string
      description: Git commit SHA or branch
    - name: repo-full-name
      type: array
      description: List of repository names (org/repo)
    - name: event-type
      type: string
      description: Webhook event type
    - name: event-id
      type: string
      description: Unique event identifier
    
    # Optional parameters
    - name: execution-profile
      type: string
      default: standard
      description: Execution profile (standard/elevated)
    - name: triggered-at
      type: string
      default: ""
      description: Webhook timestamp
    - name: org-name
      type: string
      default: ""
      description: Organization name
  
  tasks:
    - name: clone
      taskRef:
        name: git-clone
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)
      workspaces:
        - name: output
          workspace: source
    
    - name: build
      runAfter: [clone]
      taskRef:
        name: build-task
      params:
        - name: event-id
          value: $(params.event-id)
      workspaces:
        - name: source
          workspace: source
  
  workspaces:
    - name: source
      description: Workspace for source code
```

## Multi-Repo Support

The `repo-full-name` parameter is an array to support future multi-repo workspaces:

### Single Repo (Current)
```yaml
params:
  - name: repo-full-name
    value: ["bdchatham/archon-agent"]
```

### Multi-Repo Workspace (Future)
```yaml
params:
  - name: repo-full-name
    value: 
      - "bdchatham/archon-agent"
      - "bdchatham/archon-shared"
      - "bdchatham/archon-config"
```

When multi-repo support is enabled, the platform will:
1. Clone all repositories into the workspace
2. Provide array indexing: `$(params.repo-full-name[0])`, `$(params.repo-full-name[1])`, etc.
3. Maintain consistent directory structure

## Contract Stability Guarantees

The platform guarantees:

✅ **Required parameters will always be provided**  
✅ **Parameter names and types will not change within a contract version**  
✅ **New optional parameters may be added without breaking existing pipelines**  
✅ **Contract version changes (v1 → v2) will be announced and supported in parallel**

## Versioning

Contract versions follow semantic versioning:
- **v1** - Current stable version
- **v2** - Future version (when breaking changes are needed)

When a new version is released:
- Old version remains supported for 6 months minimum
- Migration guide provided
- Templates updated to new version
- Pipelines can opt-in to new version via RepoBinding

## Validation

Pipelines are validated against the contract when:
- RepoBinding is created
- Pipeline is updated
- Template is applied

Validation checks:
- All required parameters are declared
- Parameter types match contract
- No conflicting parameter names

## See Also

- [Dispatcher Templates](./templates/dispatchers/) - Platform-provided templates that implement this contract
- [Example Pipelines](./examples/) - Reference implementations
- [Execution Profiles](./profiles/) - Standard and elevated profile definitions
