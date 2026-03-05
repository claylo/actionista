# Security Hardening

Secure your GitHub Actions workflows.

## Principle of Least Privilege

### Minimal Permissions

```yaml
permissions:
  contents: read  # Start minimal

jobs:
  build:
    permissions:
      contents: read

  deploy:
    permissions:
      contents: read
      id-token: write  # Only when needed
```

### No Permissions When Possible

```yaml
permissions: {}  # No GITHUB_TOKEN access
```

## Pin Action Versions

### Best: Full SHA

```yaml
# Most secure - immutable
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
```

### Acceptable: Major Version

```yaml
# Receives security updates
- uses: actions/checkout@v6
```

### Avoid: Branch or Latest

```yaml
# Dangerous - can change anytime
- uses: actions/checkout@main
- uses: actions/checkout@latest
```

### Finding SHA

```bash
# Get SHA for a tag
git ls-remote https://github.com/actions/checkout refs/tags/v4.1.1

# Or from releases page
# github.com/actions/checkout/releases
```

## Secure Workflow Triggers

### Avoid `pull_request_target`

Runs with write access on external PRs:

```yaml
# DANGEROUS for public repos
on:
  pull_request_target:

# SAFER: Use pull_request
on:
  pull_request:
```

If you must use `pull_request_target`:

```yaml
on:
  pull_request_target:
    types: [labeled]

jobs:
  build:
    if: contains(github.event.pull_request.labels.*.name, 'safe-to-build')
    runs-on: ubuntu-latest
    steps:
      # Only checkout specific files, not untrusted code
      - uses: actions/checkout@v6
        with:
          ref: ${{ github.event.pull_request.head.sha }}
          sparse-checkout: |
            .github
            docs
```

### Validate Inputs

```yaml
on:
  workflow_dispatch:
    inputs:
      environment:
        type: choice
        options: [staging, production]  # Limited options

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      # Still validate
      - if: inputs.environment != 'staging' && inputs.environment != 'production'
        run: exit 1
```

## Prevent Script Injection

### Bad: Direct Interpolation

```yaml
# VULNERABLE to injection
- run: echo "Hello ${{ github.event.issue.title }}"
```

If title is `"; rm -rf / #`, command becomes:
```bash
echo "Hello "; rm -rf / #"
```

### Good: Use Environment Variables

```yaml
# SAFE
- run: echo "Hello $TITLE"
  env:
    TITLE: ${{ github.event.issue.title }}
```

### Dangerous Contexts

Always use env vars for:
- `github.event.issue.title`
- `github.event.issue.body`
- `github.event.pull_request.title`
- `github.event.pull_request.body`
- `github.event.comment.body`
- `github.event.head_commit.message`
- `github.event.commits[*].message`

## Restrict GITHUB_TOKEN

### Default for Public Repos

```yaml
# Explicitly set minimal permissions
permissions:
  contents: read
```

### Job-Specific Permissions

```yaml
jobs:
  build:
    permissions:
      contents: read

  publish:
    permissions:
      contents: read
      packages: write
```

## Secure Secrets Handling

### Never Log Secrets

```yaml
# BAD
- run: echo ${{ secrets.API_KEY }}
- run: printenv | grep API

# GOOD
- run: ./script.sh
  env:
    API_KEY: ${{ secrets.API_KEY }}
```

### Mask Dynamic Values

```yaml
- run: |
    TOKEN=$(generate-token)
    echo "::add-mask::$TOKEN"
    export TOKEN
```

## Third-Party Actions

### Audit Before Using

1. Check repository stars and activity
2. Read source code
3. Verify no suspicious behavior
4. Check for security advisories

### Prefer Official Actions

```yaml
# Prefer GitHub-maintained
- uses: actions/checkout@v6
- uses: actions/setup-node@v4

# Over third-party equivalents
- uses: random-user/checkout-action@v1  # Avoid if possible
```

### Fork and Pin

For critical third-party actions:

1. Fork to your organization
2. Audit the code
3. Pin to specific SHA
4. Keep updated with upstream

```yaml
- uses: your-org/forked-action@abc123def
```

## Self-Hosted Runners

### Never for Public Repos

Self-hosted runners on public repos allow arbitrary code execution.

### Isolate by Trust Level

- Production deployment: Dedicated runners
- CI builds: Separate runner group
- External contributors: GitHub-hosted only

### Use Ephemeral Runners

```bash
./config.sh --ephemeral
```

Runner cleans up after each job.

### Network Isolation

- Limit network access
- Use private networks
- Firewall rules

## Code Scanning

### Enable CodeQL

```yaml
name: CodeQL

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'

jobs:
  analyze:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v6
      - uses: github/codeql-action/init@v3
        with:
          languages: javascript
      - uses: github/codeql-action/analyze@v3
```

### Secret Scanning

Enable in repository settings:
- Secret scanning
- Push protection
- Secret scanning for partner patterns

## Dependency Review

```yaml
name: Dependency Review

on: pull_request

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: moderate
          deny-licenses: GPL-3.0
```

## Workflow Permissions

### Repository Settings

Settings → Actions → General:
- Fork pull request workflows from outside collaborators: Require approval
- Workflow permissions: Read repository contents only

### CODEOWNERS

```
# .github/CODEOWNERS
.github/workflows/ @security-team
```

Require review for workflow changes.

## Audit Logging

Monitor Actions usage:
- Settings → Logs → Security log
- Filter by `action:workflow`
- Review failed authentications

## Security Checklist

### Permissions
- [ ] Minimal `permissions` declared
- [ ] No `write-all` permissions
- [ ] Job-level permissions where needed

### Actions
- [ ] Pin to SHA or major version
- [ ] No `@main` or `@latest` references
- [ ] Third-party actions audited

### Secrets
- [ ] Use OIDC over long-lived credentials
- [ ] Environment secrets for sensitive envs
- [ ] No secrets in logs or commands

### Inputs
- [ ] Untrusted input in env vars, not inline
- [ ] Input validation for workflow_dispatch
- [ ] No script injection vulnerabilities

### Infrastructure
- [ ] No self-hosted on public repos
- [ ] Runners isolated by trust level
- [ ] Network access restricted

### Monitoring
- [ ] CodeQL enabled
- [ ] Dependency review enabled
- [ ] Secret scanning enabled
- [ ] Audit logs reviewed
