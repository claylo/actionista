---
name: github-actions
description: This skill should be used when the user asks about "GitHub Actions", "workflow", "CI/CD", ".github/workflows", "actions/checkout", "setup-node", "setup-python", or any GitHub Action by name. Also triggers on YAML files, workflow optimization, action versions, matrix builds, caching, reusable workflows, workflow security, OIDC, secrets management, or runner configuration. Provides comprehensive GitHub Actions expertise with up-to-date action versions.
---

# GitHub Actions Expertise

Comprehensive knowledge for creating, reviewing, and optimizing GitHub Actions workflows with current action versions and best practices.

## Quick Reference

### Action Versions

Consult `index.json` in this skill directory for current versions of 100+ popular actions. The index is updated daily and organized by category.

**Always verify versions** before recommending — use the index or fetch from GitHub releases if needed.

### Core Patterns

| Pattern | When to Use | Reference |
|---------|-------------|-----------|
| Matrix builds | Test across versions/platforms | `patterns/matrix-builds.md` |
| Caching | Speed up dependency installation | `patterns/caching.md` |
| Artifacts | Share files between jobs | `patterns/artifacts.md` |
| Reusable workflows | Share workflows across repos | `patterns/reusable-workflows.md` |
| Concurrency | Prevent duplicate runs | `patterns/concurrency.md` |
| Environments | Deployment gates/approvals | `patterns/environments.md` |
| Composite actions | Create custom actions | `patterns/composite-actions.md` |

### Security Essentials

| Topic | Key Points | Reference |
|-------|------------|-----------|
| Secrets | Never hardcode, use OIDC when possible | `security/secrets.md` |
| Permissions | Principle of least privilege | `security/hardening.md` |
| Supply chain | Pin actions, review dependencies | `security/supply-chain.md` |

## Workflow File Structure

```yaml
name: Workflow Name          # Display name in Actions tab

on:                          # Trigger events
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:         # Manual trigger
    inputs:
      environment:
        description: 'Deploy environment'
        required: true
        default: 'staging'

permissions:                 # GITHUB_TOKEN scope (principle of least privilege)
  contents: read
  packages: write

concurrency:                 # Prevent duplicate runs
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:                         # Workflow-level environment variables
  NODE_VERSION: '20'

jobs:
  job-name:
    runs-on: ubuntu-latest   # Runner selection
    timeout-minutes: 30      # Prevent runaway jobs

    permissions:             # Job-level permission override
      contents: read

    outputs:                 # Outputs for downstream jobs
      version: ${{ steps.extract.outputs.version }}

    steps:
      - uses: actions/checkout@v6
      - name: Step name
        id: step-id          # For referencing outputs
        run: echo "command"
        env:
          SECRET: ${{ secrets.MY_SECRET }}
```

## Essential Workflow Patterns

### Dependency Caching

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'             # Built-in caching for npm/yarn/pnpm
```

For custom caching, see `patterns/caching.md`.

### Matrix Strategy

```yaml
strategy:
  fail-fast: false           # Continue other jobs if one fails
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    node: [18, 20, 22]
    exclude:
      - os: windows-latest
        node: 18
    include:
      - os: ubuntu-latest
        node: 20
        coverage: true       # Additional variable for specific combo
```

### Job Dependencies

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps: [...]

  test:
    needs: build             # Wait for build
    runs-on: ubuntu-latest
    steps: [...]

  deploy:
    needs: [build, test]     # Wait for both
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps: [...]
```

### Conditional Execution

```yaml
- name: Deploy to production
  if: github.event_name == 'push' && github.ref == 'refs/heads/main'
  run: ./deploy.sh

- name: Run on PR only
  if: github.event_name == 'pull_request'
  run: ./pr-checks.sh

- name: Always run (even on failure)
  if: always()
  run: ./cleanup.sh

- name: Run only on failure
  if: failure()
  run: ./notify-failure.sh
```

## Trigger Events

### Common Triggers

```yaml
on:
  push:
    branches: [main, 'release/*']
    paths:
      - 'src/**'
      - '!src/**/*.md'       # Exclude markdown
    tags: ['v*']

  pull_request:
    types: [opened, synchronize, reopened]
    branches: [main]

  workflow_dispatch:          # Manual with inputs
  schedule:
    - cron: '0 0 * * *'      # Daily at midnight UTC

  workflow_call:              # Reusable workflow
    inputs:
      environment:
        type: string
        required: true
    secrets:
      token:
        required: true
```

For complete trigger reference, see `references/triggers.md`.

## Expressions and Contexts

### Common Contexts

| Context | Purpose | Example |
|---------|---------|---------|
| `github` | Event info | `github.event_name`, `github.ref` |
| `env` | Environment vars | `env.NODE_VERSION` |
| `secrets` | Secret values | `secrets.GITHUB_TOKEN` |
| `steps` | Step outputs | `steps.build.outputs.version` |
| `needs` | Job outputs | `needs.build.outputs.artifact` |
| `matrix` | Matrix values | `matrix.os`, `matrix.node` |
| `runner` | Runner info | `runner.os`, `runner.arch` |

### Common Functions

```yaml
# String functions
${{ contains(github.event.head_commit.message, '[skip ci]') }}
${{ startsWith(github.ref, 'refs/tags/') }}
${{ format('Hello {0}!', github.actor) }}

# Conditionals
${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}

# JSON parsing
${{ fromJSON(needs.build.outputs.matrix) }}
${{ toJSON(github.event) }}

# Hash for cache keys
${{ hashFiles('**/package-lock.json') }}
```

For complete expression reference, see `references/expressions.md`.

## Security Best Practices

### Minimal Permissions

```yaml
permissions:
  contents: read             # Start minimal

jobs:
  build:
    permissions:
      contents: read

  deploy:
    permissions:
      contents: read
      id-token: write        # For OIDC
      packages: write
```

### Pin Action Versions

```yaml
# Good: Pin to full SHA
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1

# Acceptable: Pin to major version
- uses: actions/checkout@v6

# Bad: Unpinned or floating
- uses: actions/checkout@main
- uses: actions/checkout
```

### OIDC for Cloud Authentication

Prefer OIDC over long-lived credentials. See `security/secrets.md` for setup guides for AWS, GCP, and Azure.

## ASCII Workflow Visualization

When reviewing workflows, generate ASCII flowcharts showing job dependencies:

```
┌─────────────────────────────────────────────────┐
│                   Workflow                       │
└─────────────────────────────────────────────────┘
                        │
                        ▼
                 ┌──────────┐
                 │   lint   │
                 └────┬─────┘
                      │
         ┌────────────┼────────────┐
         │            │            │
         ▼            ▼            ▼
    ┌────────┐   ┌────────┐   ┌────────┐
    │ test-1 │   │ test-2 │   │ test-3 │
    └───┬────┘   └───┬────┘   └───┬────┘
         │            │            │
         └────────────┼────────────┘
                      │
                      ▼
                ┌──────────┐
                │  deploy  │
                └──────────┘
```

Include these visualizations in workflow comments to document complex pipelines.

## Workflow Analysis Checklist

When reviewing workflows, check:

### Versions
- [ ] All actions use current major versions (consult `index.json`)
- [ ] Consider pinning to SHA for security-critical workflows
- [ ] No deprecated action versions

### Performance
- [ ] Dependencies are cached appropriately
- [ ] Parallel jobs where dependencies allow
- [ ] `timeout-minutes` set to prevent runaway jobs
- [ ] `fail-fast: false` if matrix jobs are independent

### Security
- [ ] Minimal `permissions` declared
- [ ] No secrets in logs (`add-mask` for dynamic secrets)
- [ ] OIDC preferred over long-lived credentials
- [ ] Third-party actions reviewed for security

### Reliability
- [ ] `concurrency` configured to prevent duplicate runs
- [ ] Conditional execution (`if:`) used appropriately
- [ ] `continue-on-error` only where truly needed
- [ ] Retry logic for flaky external calls

### Maintainability
- [ ] Descriptive job and step names
- [ ] Reusable workflows for shared logic
- [ ] Environment variables for repeated values
- [ ] Comments for complex logic

## Additional Resources

### Reference Files

For detailed syntax and advanced usage:
- **`references/workflow-syntax.md`** — Complete workflow YAML reference
- **`references/expressions.md`** — Expression syntax, contexts, functions
- **`references/triggers.md`** — All event triggers with examples
- **`references/runners.md`** — Runner selection, self-hosted setup
- **`references/permissions.md`** — GITHUB_TOKEN scopes, OIDC setup
- **`references/troubleshooting.md`** — Common errors and solutions

### Pattern Files

For implementation patterns:
- **`patterns/matrix-builds.md`** — Matrix strategies, include/exclude
- **`patterns/caching.md`** — Cache action, custom keys, restore strategies
- **`patterns/artifacts.md`** — Upload/download, retention, cross-job sharing
- **`patterns/reusable-workflows.md`** — workflow_call, inputs, secrets
- **`patterns/concurrency.md`** — Groups, cancel-in-progress
- **`patterns/environments.md`** — Deployment environments, protection rules
- **`patterns/composite-actions.md`** — Creating custom actions

### Security Files

For security hardening:
- **`security/secrets.md`** — Management, rotation, OIDC setup
- **`security/hardening.md`** — Permissions, pinning, CodeQL
- **`security/supply-chain.md`** — Dependency review, Dependabot

### Example Files

Working templates in `examples/`:
- **`ci-node.yaml`** — Node.js CI with caching and matrix
- **`ci-rust.yaml`** — Rust CI with cargo caching
- **`deploy-aws.yaml`** — AWS deployment with OIDC
- **`release-please.yaml`** — Automated releases

### Actions Index

- **`index.json`** — Current versions of 100+ popular actions (updated daily)
- **`tracked-actions.yaml`** — List of tracked actions by category
