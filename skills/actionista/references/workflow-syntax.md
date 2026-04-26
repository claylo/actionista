# Workflow Syntax Reference

Complete reference for GitHub Actions workflow YAML syntax.

## File Location

Workflows must be in `.github/workflows/` directory with `.yml` or `.yaml` extension.

## Top-Level Keys

```yaml
name: string              # Workflow display name (optional but recommended)
run-name: string          # Dynamic run name with expressions (optional)
on: event | [events]      # Required: trigger events
permissions: object       # GITHUB_TOKEN permissions
env: object               # Workflow-level environment variables
defaults: object          # Default settings for all jobs
concurrency: object       # Concurrency control
jobs: object              # Required: job definitions
```

## `on` - Workflow Triggers

### Single Event

```yaml
on: push
on: pull_request
on: workflow_dispatch
```

### Multiple Events

```yaml
on: [push, pull_request]

on:
  push:
  pull_request:
  workflow_dispatch:
```

### Event Configuration

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
    branches-ignore:
      - 'feature/**'
    paths:
      - 'src/**'
      - '!src/**/*.md'
    paths-ignore:
      - 'docs/**'
    tags:
      - 'v*'
    tags-ignore:
      - 'v*-beta'
```

### Activity Types

```yaml
on:
  pull_request:
    types: [opened, synchronize, reopened, labeled]

  issues:
    types: [opened, labeled, milestoned]

  release:
    types: [published, created, released]
```

### Scheduled Runs

```yaml
on:
  schedule:
    - cron: '0 0 * * *'      # Daily at midnight UTC
    - cron: '0 */6 * * *'    # Every 6 hours
    - cron: '0 9 * * 1-5'    # 9 AM UTC weekdays
```

### Manual Dispatch

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Deployment environment'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      debug:
        description: 'Enable debug mode'
        required: false
        type: boolean
        default: false
```

### Reusable Workflow

```yaml
on:
  workflow_call:
    inputs:
      config-path:
        required: true
        type: string
    secrets:
      token:
        required: true
    outputs:
      result:
        description: "Workflow result"
        value: ${{ jobs.build.outputs.result }}
```

## `permissions`

Control GITHUB_TOKEN scope:

```yaml
permissions: read-all      # Read all scopes
permissions: write-all     # Write all scopes (avoid in production)
permissions: {}            # No permissions

permissions:
  actions: read | write | none
  checks: read | write | none
  contents: read | write | none
  deployments: read | write | none
  id-token: write | none              # For OIDC
  issues: read | write | none
  discussions: read | write | none
  packages: read | write | none
  pages: read | write | none
  pull-requests: read | write | none
  repository-projects: read | write | none
  security-events: read | write | none
  statuses: read | write | none
```

## `env`

Workflow-level environment variables:

```yaml
env:
  NODE_VERSION: '20'
  CI: true
  FORCE_COLOR: 1
```

## `defaults`

Default settings for jobs/steps:

```yaml
defaults:
  run:
    shell: bash
    working-directory: ./src
```

## `concurrency`

Prevent duplicate runs:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
```

## `jobs`

### Basic Job

```yaml
jobs:
  build:
    name: Build Application
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: npm install
```

### Job Keys

```yaml
jobs:
  job-id:
    name: string                     # Display name
    runs-on: runner | [runners]      # Required: runner specification
    permissions: object              # Job-specific permissions
    needs: job-id | [job-ids]        # Job dependencies
    if: expression                   # Conditional execution
    environment: name | object       # Deployment environment
    concurrency: object              # Job-level concurrency
    outputs: object                  # Job outputs
    env: object                      # Job environment variables
    defaults: object                 # Job defaults
    timeout-minutes: number          # Job timeout (default: 360)
    strategy: object                 # Matrix strategy
    continue-on-error: boolean       # Don't fail workflow on job failure
    container: image | object        # Container specification
    services: object                 # Service containers
    steps: [step]                    # Required: job steps
```

### Runner Specification

```yaml
runs-on: ubuntu-latest
runs-on: ubuntu-22.04
runs-on: macos-latest
runs-on: macos-14                    # Apple Silicon
runs-on: windows-latest
runs-on: [self-hosted, linux, x64]
runs-on: ${{ matrix.os }}

# Larger runners (GitHub Teams/Enterprise)
runs-on: ubuntu-latest-4-cores
runs-on: ubuntu-latest-8-cores
```

### Matrix Strategy

```yaml
strategy:
  fail-fast: false
  max-parallel: 4
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [18, 20, 22]
    include:
      - os: ubuntu-latest
        node: 20
        coverage: true
    exclude:
      - os: macos-latest
        node: 18
```

### Job Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    needs: build
    runs-on: ubuntu-latest

  deploy:
    needs: [build, test]
    if: needs.build.result == 'success'
```

### Job Outputs

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.extract.outputs.version }}
    steps:
      - id: extract
        run: echo "version=1.0.0" >> $GITHUB_OUTPUT

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

### Environment

```yaml
jobs:
  deploy:
    environment:
      name: production
      url: https://example.com

  # Or simple form
  deploy:
    environment: production
```

### Container

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: node:20
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
      env:
        NODE_ENV: production
      ports:
        - 80
      volumes:
        - my_docker_volume:/volume_mount
      options: --cpus 1
```

### Services

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379
```

## Steps

### Step Keys

```yaml
steps:
  - id: string              # Step identifier for outputs
    name: string            # Display name
    if: expression          # Conditional execution
    uses: action@version    # Action to run
    with: object            # Action inputs
    run: string             # Shell command
    shell: string           # Shell type
    working-directory: path # Working directory
    env: object             # Step environment
    continue-on-error: bool # Don't fail job on step failure
    timeout-minutes: number # Step timeout
```

### Using Actions

```yaml
- uses: actions/checkout@v6

- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'

- uses: ./.github/actions/my-action    # Local action

- uses: docker://alpine:3.8            # Docker action
```

### Running Commands

```yaml
- run: npm install

- run: |
    npm install
    npm test
    npm build

- name: Multi-line with working directory
  run: |
    cd src
    make build
  working-directory: ./packages
  shell: bash
```

### Shell Options

```yaml
shell: bash                 # Default on Linux/macOS
shell: pwsh                 # PowerShell Core
shell: python               # Python
shell: sh                   # POSIX shell
shell: cmd                  # Windows CMD
shell: powershell           # Windows PowerShell
```

### Step Outputs

```yaml
- id: random
  run: echo "value=$(openssl rand -hex 12)" >> $GITHUB_OUTPUT

- run: echo "Random value is ${{ steps.random.outputs.value }}"
```

### Conditional Steps

```yaml
- run: echo "Only on push"
  if: github.event_name == 'push'

- run: echo "Always runs"
  if: always()

- run: echo "Runs on failure"
  if: failure()

- run: echo "Runs on success"
  if: success()

- run: echo "Runs on cancel"
  if: cancelled()
```

## Complete Example

```yaml
name: CI/CD Pipeline
run-name: Deploy by @${{ github.actor }}

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]
        default: staging

permissions:
  contents: read
  packages: write

env:
  NODE_VERSION: '20'

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 15
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - id: version
        run: echo "value=$(node -p "require('./package.json').version")" >> $GITHUB_OUTPUT

  test:
    needs: build
    runs-on: ubuntu-latest
    strategy:
      matrix:
        node: [18, 20, 22]
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test

  deploy:
    needs: [build, test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment:
      name: ${{ inputs.environment || 'staging' }}
      url: https://${{ inputs.environment || 'staging' }}.example.com
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```
