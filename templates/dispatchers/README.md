# Dispatcher Templates

## Overview

Dispatcher templates are **thin execution envelopes** that convert webhook events into PipelineRuns. They contain no workflow logic - only parameter transformation and PipelineRun creation.

## Philosophy

Templates are **not** pipelines. They are dispatchers that:
- ✅ Select the execution namespace
- ✅ Select the pipeline to run
- ✅ Inject canonical parameters
- ✅ Apply execution profiles

Templates do **not**:
- ❌ Clone repositories
- ❌ Build artifacts
- ❌ Run tests
- ❌ Deploy software
- ❌ Contain workflow logic

All workflow logic lives in product-owned pipelines.

## Available Templates

### run-pipeline-v1

**Version**: v1  
**Status**: Stable  
**Contract**: [Pipeline Contract v1](../../PIPELINE_CONTRACT.md)

The standard dispatcher template for webhook-triggered pipelines.

**What it does**:
1. Receives webhook parameters from TriggerBinding
2. Creates a PipelineRun in the target namespace
3. Injects canonical parameters per Pipeline Contract v1
4. Applies execution profile settings
5. Sets labels for traceability

**What it doesn't do**:
- No git operations
- No build logic
- No deployment logic
- No conditional execution
- No workflow composition

**Usage**:

```yaml
apiVersion: aphex.io/v1alpha1
kind: RepoBinding
metadata:
  name: my-repo-binding
spec:
  aphexOrg: my-org
  repoOrg: bdchatham
  repoName: my-repo
  pipelineName: my-pipeline
  templateRef: run-pipeline-v1  # References this template
  executionProfile: standard
```

**Parameters**:

| Parameter | Source | Description |
|-----------|--------|-------------|
| `git-url` | TriggerBinding | Repository clone URL |
| `git-revision` | TriggerBinding | Commit SHA |
| `repo-full-name` | TriggerBinding | org/repo |
| `event-type` | TriggerBinding | push, pull_request, etc. |
| `event-id` | TriggerBinding | Unique event ID |
| `pipeline-name` | RepoBinding | Target pipeline name |
| `pipeline-namespace` | RepoBinding | Target namespace |
| `execution-role` | RepoBinding | standard or elevated |
| `org-name` | Organization | Aphex org name |
| `triggered-at` | Platform | ISO8601 timestamp |

**Output**: PipelineRun in target namespace with canonical parameters

## Template Lifecycle

### Creation
Templates are created by the platform team and stored in this repository.

### Materialization
When a RepoBinding references a template, the onboarding controller:
1. Reads the template from this catalog
2. Creates a TriggerTemplate in the org namespace
3. Wires it to a Trigger for the specific repository

### Versioning
Templates are versioned (v1, v2, etc.). Breaking changes require a new version.

### Deprecation
Old template versions remain supported for 6 months after a new version is released.

## Adding New Templates

To add a new template:

1. **Create the template YAML** in this directory
2. **Follow naming convention**: `<purpose>-<version>.yaml`
3. **Add labels**:
   ```yaml
   labels:
     platform.aphex/template-version: v1
     platform.aphex/managed-by: platform
   ```
4. **Document in this README**
5. **Update controller** to recognize the new template
6. **Test** with a sample RepoBinding

## Template Design Guidelines

### Keep Templates Thin
Templates should be < 100 lines. If longer, you're doing too much.

### No Workflow Logic
If you're adding `when` conditions, task composition, or branching logic, stop. That belongs in pipelines.

### Stable Parameters
Only inject parameters from the Pipeline Contract. Don't invent new ones.

### Execution Profiles
Apply execution profiles (SA, timeout, resources) but don't embed workflow decisions.

### Labels for Traceability
Always add labels for event tracking and debugging.

## See Also

- [Pipeline Contract](../../PIPELINE_CONTRACT.md) - Parameter interface specification
- [Execution Profiles](../profiles/) - Standard and elevated profiles
- [Example Pipelines](../../examples/) - Reference implementations
