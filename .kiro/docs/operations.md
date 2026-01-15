# Operations

## Deployment

### Initial Deployment

Deploy all Tekton tasks to the cluster:

```bash
# From repository root
kubectl apply -k tekton/
```

This creates tasks in the `tekton-pipelines` namespace:
- `buildah-build`
- `python-test`
- `kubectl-apply`

### Updating Tasks

When tasks are modified:

1. Update the task YAML file in `tekton/tasks/`
2. Apply changes:
   ```bash
   kubectl apply -k tekton/
   ```
3. Existing PipelineRuns continue using old task versions
4. New PipelineRuns use updated task definitions

### Versioning Strategy

Tasks are **not** versioned with suffixes (e.g., no `python-test-v2`). Instead:
- Tasks are updated in place
- Breaking changes require communication to product teams
- Product teams update their pipeline references as needed

### Dispatcher Template Deployment

Dispatcher templates are **not** deployed directly. They are:
1. Stored in `templates/dispatchers/`
2. Read by the onboarding controller
3. Materialized into org namespaces when referenced by RepoBindings

## Monitoring

### Task Usage Metrics

Monitor task usage across the platform:

```bash
# Count PipelineRuns using each task
kubectl get pipelineruns -A -o json | \
  jq -r '.items[].spec.pipelineSpec.tasks[].taskRef.name' | \
  sort | uniq -c | sort -rn
```

### Task Execution Success Rate

Monitor task success rates:

```bash
# Get task execution status
kubectl get taskruns -A \
  -l tekton.dev/task=python-test \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.conditions[?(@.type=="Succeeded")].status}{"\n"}{end}'
```

### Resource Consumption

Monitor resource usage by tasks:

```bash
# Check pod resource usage for running tasks
kubectl top pods -n tekton-pipelines -l tekton.dev/task
```

## Alerting

No automated alerting is configured for this repository. Issues are typically discovered through:
- Product team reports of task failures
- Platform team monitoring of PipelineRun success rates
- GitHub Issues for task bugs or feature requests

## Runbooks

### Common Issues

#### Task Not Found

**Symptom**: Pipeline fails with "Task not found" error

**Resolution**:
1. Verify task exists in tekton-pipelines namespace:
   ```bash
   kubectl get task -n tekton-pipelines
   ```
2. If missing, deploy tasks:
   ```bash
   kubectl apply -k tekton/
   ```
3. Verify task name matches reference in pipeline

#### Buildah Build Failures

**Symptom**: `buildah-build` task fails with permission errors

**Resolution**:
1. Verify task has privileged security context:
   ```bash
   kubectl get task buildah-build -n tekton-pipelines -o yaml | grep privileged
   ```
2. Check if cluster has PodSecurityPolicy or PodSecurityStandards blocking privileged pods
3. Verify registry credentials are configured in pipeline ServiceAccount

#### Python Test Dependency Installation Failures

**Symptom**: `python-test` task fails during `install-deps` step

**Resolution**:
1. Verify project has `setup.py` or `pyproject.toml` with `[test]` extras:
   ```bash
   # Check if test extras are defined
   cat setup.py | grep test
   ```
2. Update project to include test dependencies in extras
3. Alternatively, override `test-command` parameter to install dependencies differently

#### kubectl-apply Manifest Not Found

**Symptom**: `kubectl-apply` task fails with "directory does not exist"

**Resolution**:
1. Verify `manifest-dir` parameter points to correct directory in workspace
2. Check if directory exists in source repository
3. Verify workspace is properly mounted and populated

### Troubleshooting

#### Debugging Task Execution

View task execution logs:

```bash
# Get TaskRun logs
kubectl logs -n <namespace> <taskrun-name> --all-containers

# Follow logs in real-time
kubectl logs -n <namespace> <taskrun-name> --all-containers -f
```

#### Inspecting Task Definitions

View current task definition:

```bash
# Get full task YAML
kubectl get task <task-name> -n tekton-pipelines -o yaml

# View task parameters
kubectl get task <task-name> -n tekton-pipelines -o jsonpath='{.spec.params[*].name}'
```

#### Testing Task Changes Locally

Test task changes before deploying:

```bash
# Dry-run apply
kubectl apply -k tekton/ --dry-run=client

# Apply to test namespace first
kubectl apply -k tekton/ -n tekton-pipelines-test
```

## Maintenance

### Regular Updates

**Monthly**:
- Review task image versions for security updates
- Update base images (python, buildah, kubectl)
- Test updated tasks in non-production environment

**Quarterly**:
- Review task usage metrics
- Deprecate unused tasks
- Add new tasks based on product team requests

### Adding New Tasks

To add a new task:

1. Create task YAML in `tekton/tasks/`:
   ```yaml
   apiVersion: tekton.dev/v1beta1
   kind: Task
   metadata:
     name: my-new-task
   spec:
     # Task definition
   ```

2. Add to `tekton/kustomization.yaml`:
   ```yaml
   resources:
     - tasks/my-new-task.yaml
   ```

3. Test deployment:
   ```bash
   kubectl apply -k tekton/ --dry-run=client
   ```

4. Deploy:
   ```bash
   kubectl apply -k tekton/
   ```

5. Document in architecture.md and README

### Updating Dispatcher Templates

To update a dispatcher template:

1. **For non-breaking changes**: Update template in place
2. **For breaking changes**: Create new version (e.g., `run-pipeline-v2.yaml`)
3. Update `templates/dispatchers/README.md` with changes
4. Communicate changes to product teams
5. Onboarding controller will use updated template for new RepoBindings

### Deprecating Tasks

To deprecate a task:

1. Announce deprecation to product teams (6-month notice)
2. Add deprecation notice to task description
3. Monitor usage to ensure no active references
4. Remove from `tekton/kustomization.yaml`
5. Delete task YAML file
6. Remove from cluster:
   ```bash
   kubectl delete task <task-name> -n tekton-pipelines
   ```

**Source**
- `tekton/kustomization.yaml` - Deployment configuration
- `tekton/tasks/` - Task definitions
- `templates/dispatchers/` - Dispatcher templates
- `templates/dispatchers/README.md` - Template guidelines
