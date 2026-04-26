# Concurrency

Control parallel execution of workflows and jobs.

## Basic Concurrency

Cancel older runs when new run starts:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## Concurrency Group

Jobs/workflows with the same group run serially:

```yaml
concurrency:
  group: deploy-production
```

Only one workflow with this group runs at a time.

## Group Patterns

### Per-Branch

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

Separate concurrency per branch.

### Per-PR

```yaml
concurrency:
  group: pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

Cancel previous runs on same PR.

### Per-Environment

```yaml
concurrency:
  group: deploy-${{ github.event.inputs.environment }}
  cancel-in-progress: false  # Queue deployments
```

### Global

```yaml
concurrency:
  group: global-deploy
  cancel-in-progress: false
```

Only one deployment at a time across all branches.

## Workflow-Level vs Job-Level

### Workflow-Level

```yaml
name: CI
on: push

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
  test:
    runs-on: ubuntu-latest
```

Entire workflow respects concurrency.

### Job-Level

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    # No concurrency - runs in parallel

  deploy:
    runs-on: ubuntu-latest
    concurrency:
      group: deploy-production
      cancel-in-progress: false
```

Only specific jobs are serialized.

## Cancel-in-Progress

### `true` - Cancel Previous

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

New run cancels any running/pending runs in same group.

### `false` - Queue (Default)

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

New runs wait for existing runs to complete.

**Behavior:**
- Maximum one running + one pending per group
- New pending runs replace older pending runs
- Running jobs complete before next starts

## Common Patterns

### CI with Auto-Cancel

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: npm test
```

### Deployment Queue

```yaml
name: Deploy

on:
  push:
    branches: [main]

concurrency:
  group: production-deploy
  cancel-in-progress: false  # Never cancel deployments

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy.sh
```

### PR Preview Environments

```yaml
concurrency:
  group: preview-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  deploy-preview:
    runs-on: ubuntu-latest
    steps:
      - run: ./deploy-preview.sh
```

### Conditional Cancel

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  # Only cancel PRs, not main branch
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}
```

### Release Protection

```yaml
name: Release

on:
  release:
    types: [published]

concurrency:
  group: release-${{ github.event.release.tag_name }}
  cancel-in-progress: false  # Never cancel releases
```

## Handling Cancellation

### Cleanup on Cancel

```yaml
steps:
  - run: ./start-resources.sh

  - run: npm test

  - name: Cleanup
    if: cancelled()
    run: ./cleanup-resources.sh
```

### Always Cleanup

```yaml
steps:
  - run: ./setup.sh

  - run: npm test

  - name: Cleanup
    if: always()  # Runs on success, failure, and cancel
    run: ./cleanup.sh
```

## Dynamic Groups

### Based on Input

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]

concurrency:
  group: deploy-${{ inputs.environment }}
```

### Based on Event

```yaml
concurrency:
  group: >-
    ${{ github.workflow }}-
    ${{ github.event_name == 'pull_request' && github.event.pull_request.number || github.ref }}
  cancel-in-progress: true
```

## Debugging

### Check Concurrency Status

Look in workflow run UI:
- "Waiting" - queued behind another run
- "Cancelled" - superseded by newer run

### Workflow Annotations

GitHub shows annotations when runs are:
- Waiting due to concurrency
- Cancelled due to concurrency

## Best Practices

### 1. Always Set Concurrency for CI

```yaml
concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true
```

Saves runner minutes on rapid pushes.

### 2. Never Cancel Deployments

```yaml
concurrency:
  group: deploy-production
  cancel-in-progress: false
```

Interrupted deployments can leave bad state.

### 3. Use Meaningful Group Names

```yaml
# Good: Clear and specific
group: deploy-production-aws

# Bad: Too vague
group: deploy
```

### 4. Include Workflow Name

```yaml
group: ${{ github.workflow }}-${{ github.ref }}
```

Prevents conflicts between different workflows.

### 5. Protect Main Branch

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: ${{ github.ref != 'refs/heads/main' }}
```

Only cancel PR runs, not main branch runs.

## Limitations

- Only one pending run per group (newer bumps older)
- Can't set priority between runs
- Can't pause/resume runs
- Maximum run time still applies while waiting
