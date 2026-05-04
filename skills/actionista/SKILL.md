---
description: >-
  Creates, debugs, and optimizes GitHub Actions workflow YAML files. Recommends current action versions with SHA pinning from a daily-updated index of 260+ actions. Configures matrix builds, dependency caching, reusable workflows, OIDC authentication, secrets management, and runner selection. Includes a workflow-analyzer agent for reviewing existing workflows. Use when the user asks about GitHub Actions, workflows, CI/CD, .github/workflows, any action by name (actions/checkout, setup-node, setup-python, etc.), or workflow analysis.
license: MIT
metadata:
  repository: https://github.com/claylo/actionista
  version: 1.2.6
name: actionista
---

<SUBAGENT-STOP>
If you were dispatched as a subagent to execute a specific task, skip this skill.
</SUBAGENT-STOP>

# GitHub Actions Expertise

Comprehensive knowledge for creating, debugging, and optimizing GitHub Actions workflows with current action versions and best practices.

## Agents

This skill includes specialized agents in the `agents/` directory.

| Agent | When to Use |
|-------|-------------|
| `workflow-analyzer` | Any request to review, check, audit, optimize, or analyze existing workflow files |

**When a user requests workflow analysis, review, or optimization of existing workflows, ALWAYS dispatch the `workflow-analyzer` agent. Do not perform the analysis inline.**

This skill (SKILL.md) handles workflow **creation** and reference. The agent handles workflow **review**.

## Environment

!`command -v actionlint 2>/dev/null && echo "actionlint: installed" || echo "actionlint: not installed"`
!`command -v act 2>/dev/null && echo "act: installed" || echo "act: not installed"`

If either tool is missing, suggest the user install them:
- `brew install actionlint` — workflow linter that catches syntax errors, type mismatches, and expression problems offline
- `brew install act` — runs workflows locally in Docker without pushing

## Quick Reference

### Action Versions

Consult `actions-index.json` in this skill directory for current versions of 264 popular actions. The index is updated daily and organized by category.

**Do not read the entire file.** Use `jq` to query only what you need.

### Using the Index

The index has a top-level `_metadata` object and an `actions` object keyed by action name (e.g., `actions/checkout`).

Each action entry has these fields:

| Field | Always Present | Description |
|-------|:--------------:|-------------|
| `latest` | ✓ | Latest major version tag (e.g., `v6`) |
| `latestFull` | ✓ | Full semver version (e.g., `v6.0.2`) |
| `sha` | ✓ | Commit SHA for the latest release — use this for SHA pinning |
| `description` | ✓ | Short description of the action |
| `category` | ✓ | One of: `core`, `setup-languages`, `build-tools`, `testing`, `linting`, `release`, `security`, `docker`, `cloud-aws`, `cloud-azure`, `cloud-gcp`, `infrastructure`, `kubernetes`, `notifications`, `git-operations`, `package-managers`, `documentation`, `mobile`, `ai-assistants`, `utilities` |
| `inputs` | mostly | Array of accepted input names |
| `deprecated` | sometimes | Array of deprecated major versions |
| `migrations` | sometimes | Breaking changes between major versions, including added/removed inputs |

#### jq Examples

**Get SHA pin + version comment for an action:**
```bash
jq -r '.actions["actions/cache"] | "- uses: actions/cache@\(.sha)  # \(.latestFull)"' actions-index.json
# Output: - uses: actions/cache@27d5ce7f107fe9357f9df03efb73ab90386fccae  # v5.0.5
```

**Get latest version of an action:**
```bash
jq -r '.actions["actions/checkout"].latestFull' actions-index.json
```

**List all actions in a category:**
```bash
jq -r '[.actions | to_entries[] | select(.value.category == "setup-languages")] | sort_by(.key) | .[] | "\(.key) \(.value.latestFull)"' actions-index.json
```

**Check if a version is deprecated:**
```bash
jq '.actions["actions/checkout"].deprecated // []' actions-index.json
```

**Get migration notes between versions:**
```bash
jq '.actions["8398a7/action-slack"].migrations' actions-index.json
```

**Always verify versions** before recommending — use the index or fetch from GitHub releases if needed.

### Core Patterns

| Pattern | When to Use | Reference |
|---------|-------------|-----------|
| Matrix builds | Test across versions/platforms | `references/patterns-matrix-builds.md` |
| Caching | Speed up dependency installation | `references/patterns-caching.md` |
| Artifacts | Share files between jobs | `references/patterns-artifacts.md` |
| Reusable workflows | Share workflows across repos | `references/patterns-reusable-workflows.md` |
| Concurrency | Prevent duplicate runs | `references/patterns-concurrency.md` |
| Environments | Deployment gates/approvals | `references/patterns-environments.md` |
| Composite actions | Create custom actions | `references/patterns-composite-actions.md` |

### Security Essentials

| Topic | Key Points | Reference |
|-------|------------|-----------|
| Secrets | Never hardcode, use OIDC when possible | `references/security-secrets.md` |
| Permissions | Principle of least privilege | `references/security-hardening.md` |
| Supply chain | Pin actions, review dependencies | `references/security-supply-chain.md` |

## Workflow Essentials

Every workflow should include these often-missed fields:

```yaml
permissions:
  contents: read

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 30
```

Full JSON Schema for workflow validation: `references/github-action.schema.json`

## Essential Workflow Patterns

### Dependency Caching

```yaml
- uses: actions/setup-node@48b55a011bda9f5d6aeb4c2d9c7362e8dae4041e  # v6.4.0
  with:
    node-version: '22'
    cache: 'npm'
```

For custom caching, see `references/patterns-caching.md`.

### Matrix Strategy

```yaml
strategy:
  fail-fast: false
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    node: [18, 20, 22]
    exclude:
      - os: windows-latest
        node: 18
    include:
      - os: ubuntu-latest
        node: 20
        coverage: true
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
    steps: [...]

  deploy:
    needs: [build, test]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps: [...]
```

For conditional execution patterns (`if:`, `always()`, `failure()`), trigger event configuration, and expression syntax, see `references/triggers.md` and `references/expressions.md`.

## Security Best Practices

### Minimal Permissions

```yaml
permissions:
  contents: read

jobs:
  build:
    permissions:
      contents: read

  deploy:
    permissions:
      contents: read
      id-token: write
      packages: write
```

### Pin Action Versions

**NEVER fabricate a SHA.** The `actions-index.json` file is right here in this skill directory with the correct full 40-character SHA for every tracked action. Use `jq` to read it. There is no excuse for generating, guessing, or completing a partial SHA from memory — LLMs will confidently produce hex strings that look correct but point at nonexistent commits. A fabricated SHA defeats the entire purpose of pinning and is a security-critical failure. Always read the real SHA from the index.

```yaml
# Good: Pin to full SHA (read from actions-index.json)
- uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd  # v6.0.2

# Acceptable: Pin to major version
- uses: actions/checkout@v6

# Bad: Unpinned or floating
- uses: actions/checkout@main
- uses: actions/checkout
```

### OIDC for Cloud Authentication

Prefer OIDC over long-lived credentials. See `references/security-secrets.md` for setup guides for AWS, GCP, and Azure.

## Validating Workflows

Before pushing workflow changes, verify correctness:

1. **Lint with actionlint** — catches syntax errors, type mismatches, and expression problems that YAML parsers miss:
   ```bash
   actionlint .github/workflows/*.yml
   ```

2. **Test locally with `act`** — runs workflows in Docker without pushing:
   ```bash
   act push --job build
   act pull_request -e event.json
   ```

3. **Check workflow run logs** — after pushing, verify in the Actions tab:
   - Confirm expected jobs triggered
   - Check for deprecation warnings in logs
   - Verify caching is hitting (`cache hit: true`)

4. **Validate secrets access** — workflows fail silently when secrets are missing. Verify required secrets exist in repo/org settings before relying on them.

## Workflow Analysis

For workflow review, auditing, and optimization of existing workflows, dispatch the `workflow-analyzer` agent. See the [Agents](#agents) section above.

## Additional Resources

All reference files are in `references/`:

| File | Topic |
|------|-------|
| `workflow-syntax.md` | Complete workflow YAML reference |
| `expressions.md` | Expression syntax, contexts, functions |
| `triggers.md` | All event triggers with examples |
| `runners.md` | Runner selection, self-hosted setup |
| `permissions.md` | GITHUB_TOKEN scopes, OIDC setup |
| `troubleshooting.md` | Common errors and solutions |
| `patterns-matrix-builds.md` | Matrix strategies, include/exclude |
| `patterns-caching.md` | Cache action, custom keys, restore strategies |
| `patterns-artifacts.md` | Upload/download, retention, cross-job sharing |
| `patterns-reusable-workflows.md` | workflow_call, inputs, secrets |
| `patterns-concurrency.md` | Groups, cancel-in-progress |
| `patterns-environments.md` | Deployment environments, protection rules |
| `patterns-composite-actions.md` | Creating custom actions |
| `security-secrets.md` | Management, rotation, OIDC setup |
| `security-hardening.md` | Permissions, pinning, CodeQL |
| `security-supply-chain.md` | Dependency review, Dependabot |

Working templates in `examples/`: `ci-node.yaml`, `ci-rust.yaml`, `deploy-aws.yaml`, `release-please.yaml`

- **`actions-index.json`** — Current versions of 260+ popular actions (updated daily)
- **`tracked-actions.yaml`** — List of tracked actions by category
