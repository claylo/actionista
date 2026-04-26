# Supply Chain Security

Protect your workflow from supply chain attacks.

## The Threat

Supply chain attacks target:
- GitHub Actions (malicious updates)
- Dependencies (compromised packages)
- Build tools (modified compilers)
- Container images (trojaned images)

## Pin Actions to SHA

### Why SHA Pinning

Tags can be moved. A v1 tag today might point to different code tomorrow.

```yaml
# Tag can change
- uses: actions/checkout@v6

# SHA is immutable
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11
```

### How to Pin

1. Find the SHA from releases:
   ```bash
   git ls-remote https://github.com/actions/checkout refs/tags/v4.1.1
   ```

2. Add comment with version:
   ```yaml
   - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11  # v4.1.1
   ```

### Automation

Use Dependabot or Renovate to update pinned SHAs:

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    commit-message:
      prefix: "ci"
```

## Dependency Review

### Enable for PRs

```yaml
name: Dependency Review

on: pull_request

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/dependency-review-action@v4
        with:
          fail-on-severity: moderate
          deny-licenses: GPL-3.0, AGPL-3.0
          allow-ghsas: GHSA-xxxx-xxxx-xxxx  # Known safe
```

### Configuration Options

```yaml
- uses: actions/dependency-review-action@v4
  with:
    # Fail thresholds
    fail-on-severity: critical  # low, moderate, high, critical
    fail-on-scopes: runtime     # development, runtime, unknown

    # License blocking
    deny-licenses: GPL-3.0
    allow-licenses: MIT, Apache-2.0

    # Vulnerability exceptions
    allow-ghsas: GHSA-1234-5678-9abc

    # Package blocking
    deny-packages: npm:malicious-package
```

## Lock Dependencies

### Package Lock Files

Always commit lock files:
- `package-lock.json` (npm)
- `yarn.lock` (yarn)
- `pnpm-lock.yaml` (pnpm)
- `Cargo.lock` (Rust)
- `go.sum` (Go)
- `poetry.lock` (Python)

```yaml
# Use ci/frozen install commands
- run: npm ci           # Not npm install
- run: yarn --frozen-lockfile
- run: pnpm install --frozen-lockfile
```

### Verify Integrity

```yaml
- run: npm ci
  env:
    npm_config_ignore_scripts: true  # Don't run install scripts
```

## Container Security

### Pin Image Digests

```yaml
# Tag can change
container: node:20

# Digest is immutable
container: node@sha256:abc123...

# In Dockerfile
FROM node@sha256:abc123...
```

### Get Digest

```bash
docker pull node:20
docker inspect --format='{{index .RepoDigests 0}}' node:20
```

### Scan Images

```yaml
- uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'my-image:latest'
    format: 'sarif'
    output: 'trivy-results.sarif'

- uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: 'trivy-results.sarif'
```

## SLSA Provenance

### Generate Provenance

```yaml
- uses: slsa-framework/slsa-github-generator/.github/workflows/generator_generic_slsa3.yml@v1.9.0
  with:
    base64-subjects: ${{ needs.build.outputs.digest }}
```

### Verify Provenance

```bash
slsa-verifier verify-artifact my-artifact \
  --provenance-path provenance.intoto.jsonl \
  --source-uri github.com/owner/repo
```

## Sigstore Signing

### Sign Artifacts

```yaml
- uses: sigstore/cosign-installer@v3

- name: Sign container
  run: cosign sign ${{ env.IMAGE }}
  env:
    COSIGN_EXPERIMENTAL: 1

- name: Verify signature
  run: cosign verify ${{ env.IMAGE }}
```

## Dependabot

### Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  # GitHub Actions
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    groups:
      actions:
        patterns:
          - "*"

  # npm
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    ignore:
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]

  # Docker
  - package-ecosystem: docker
    directory: /
    schedule:
      interval: weekly
```

### Auto-Merge Safe Updates

```yaml
name: Dependabot Auto-Merge

on: pull_request

permissions:
  contents: write
  pull-requests: write

jobs:
  auto-merge:
    if: github.actor == 'dependabot[bot]'
    runs-on: ubuntu-latest
    steps:
      - uses: dependabot/fetch-metadata@v2
        id: metadata

      - if: steps.metadata.outputs.update-type == 'version-update:semver-patch'
        run: gh pr merge --auto --merge "$PR_URL"
        env:
          PR_URL: ${{ github.event.pull_request.html_url }}
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Reproducible Builds

### Lock Everything

```yaml
jobs:
  build:
    runs-on: ubuntu-22.04  # Specific version
    steps:
      - uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11

      - uses: actions/setup-node@v4
        with:
          node-version-file: '.nvmrc'  # Version from file

      - run: npm ci  # Locked dependencies

      - run: npm run build
        env:
          SOURCE_DATE_EPOCH: 0  # Reproducible timestamps
```

### Verify Reproducibility

```yaml
- name: Build twice and compare
  run: |
    npm run build
    mv dist dist1
    npm run build
    diff -r dist dist1
```

## Secrets in Dependencies

### Detect Leaked Secrets

```yaml
- uses: trufflesecurity/trufflehog@main
  with:
    path: ./
    base: ${{ github.event.repository.default_branch }}
    head: HEAD
```

### Git Guardian

```yaml
- uses: GitGuardian/ggshield-action@v1
  env:
    GITHUB_PUSH_BEFORE_SHA: ${{ github.event.before }}
    GITHUB_PUSH_BASE_SHA: ${{ github.event.base }}
    GITHUB_DEFAULT_BRANCH: ${{ github.event.repository.default_branch }}
    GITGUARDIAN_API_KEY: ${{ secrets.GITGUARDIAN_API_KEY }}
```

## Security Checklist

### Actions
- [ ] All actions pinned to SHA
- [ ] Dependabot updates configured
- [ ] Third-party actions reviewed

### Dependencies
- [ ] Lock files committed
- [ ] `npm ci` / frozen installs used
- [ ] Dependency review on PRs
- [ ] No known vulnerabilities

### Containers
- [ ] Images pinned to digest
- [ ] Base images scanned
- [ ] Minimal base images used

### Builds
- [ ] Reproducible builds enabled
- [ ] Build provenance generated
- [ ] Artifacts signed

### Monitoring
- [ ] Dependabot alerts enabled
- [ ] Secret scanning enabled
- [ ] Code scanning enabled
- [ ] Audit logs reviewed
