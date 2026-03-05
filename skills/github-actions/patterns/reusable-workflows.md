# Reusable Workflows

Share entire workflows across repositories.

## Creating a Reusable Workflow

```yaml
# .github/workflows/reusable-build.yml
name: Reusable Build

on:
  workflow_call:
    inputs:
      node-version:
        description: 'Node.js version'
        required: false
        type: string
        default: '20'
      environment:
        description: 'Target environment'
        required: true
        type: string
    secrets:
      npm-token:
        description: 'NPM publish token'
        required: true
    outputs:
      artifact-url:
        description: 'URL to build artifact'
        value: ${{ jobs.build.outputs.artifact-url }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-url: ${{ steps.upload.outputs.artifact-url }}
    steps:
      - uses: actions/checkout@v6

      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
          registry-url: 'https://registry.npmjs.org'

      - run: npm ci
        env:
          NODE_AUTH_TOKEN: ${{ secrets.npm-token }}

      - run: npm run build

      - uses: actions/upload-artifact@v4
        id: upload
        with:
          name: build-${{ inputs.environment }}
          path: dist/
```

## Calling a Reusable Workflow

### Same Repository

```yaml
jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      node-version: '22'
      environment: 'production'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

### Different Repository

```yaml
jobs:
  call-build:
    uses: owner/repo/.github/workflows/reusable-build.yml@main
    with:
      environment: 'production'
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

### Pin to Version

```yaml
jobs:
  call-build:
    uses: owner/repo/.github/workflows/reusable-build.yml@v1.0.0
    # or SHA
    uses: owner/repo/.github/workflows/reusable-build.yml@abc123
```

## Input Types

```yaml
on:
  workflow_call:
    inputs:
      string-input:
        type: string
        required: true

      boolean-input:
        type: boolean
        default: false

      number-input:
        type: number
        default: 10

      # No choice type for workflow_call
```

## Passing Secrets

### Individual Secrets

```yaml
jobs:
  call-workflow:
    uses: ./.github/workflows/reusable.yml
    secrets:
      token-a: ${{ secrets.TOKEN_A }}
      token-b: ${{ secrets.TOKEN_B }}
```

### Inherit All Secrets

```yaml
jobs:
  call-workflow:
    uses: ./.github/workflows/reusable.yml
    secrets: inherit
```

## Workflow Outputs

### Define in Reusable

```yaml
on:
  workflow_call:
    outputs:
      version:
        description: 'Built version'
        value: ${{ jobs.build.outputs.version }}

jobs:
  build:
    outputs:
      version: ${{ steps.version.outputs.value }}
    steps:
      - id: version
        run: echo "value=1.0.0" >> $GITHUB_OUTPUT
```

### Use in Caller

```yaml
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Deploying version ${{ needs.build.outputs.version }}"
```

## Permissions

Reusable workflows inherit caller's permissions by default.

```yaml
# Caller explicitly sets permissions
jobs:
  call-build:
    permissions:
      contents: read
      packages: write
    uses: ./.github/workflows/reusable-build.yml
```

## Nested Workflows

Workflows can call other reusable workflows:

```yaml
# reusable-ci.yml calls reusable-build.yml
jobs:
  build:
    uses: ./.github/workflows/reusable-build.yml
    with:
      environment: 'staging'

  test:
    uses: ./.github/workflows/reusable-test.yml
    needs: build
```

Maximum nesting depth: 4 levels.

## Common Patterns

### CI/CD Template

```yaml
# reusable-cicd.yml
on:
  workflow_call:
    inputs:
      environment:
        type: string
        required: true
    secrets:
      deploy-key:
        required: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: npm ci
      - run: npm run build

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - run: ./deploy.sh
        env:
          DEPLOY_KEY: ${{ secrets.deploy-key }}
```

### Release Template

```yaml
# reusable-release.yml
on:
  workflow_call:
    inputs:
      tag:
        type: string
        required: true
    secrets:
      github-token:
        required: true

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - run: npm run build

      - uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ inputs.tag }}
          files: dist/*
        env:
          GITHUB_TOKEN: ${{ secrets.github-token }}
```

### Matrix Wrapper

```yaml
# reusable-matrix-test.yml
on:
  workflow_call:
    inputs:
      matrix:
        type: string
        required: true
        description: 'JSON matrix configuration'

jobs:
  test:
    strategy:
      matrix: ${{ fromJSON(inputs.matrix) }}
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

Caller:
```yaml
jobs:
  call-test:
    uses: ./.github/workflows/reusable-matrix-test.yml
    with:
      matrix: '{"os":["ubuntu-latest"],"node":["18","20","22"]}'
```

## Limitations

- Maximum 20 unique reusable workflows per workflow file
- Maximum 4 levels of nesting
- Cannot use `workflow_dispatch` in reusable workflow
- Environment must be defined in reusable workflow jobs

## Best Practices

### 1. Version Your Workflows

```yaml
# Use tags for stability
uses: owner/repo/.github/workflows/ci.yml@v1

# Or SHA for security
uses: owner/repo/.github/workflows/ci.yml@abc123def
```

### 2. Document Inputs/Outputs

```yaml
on:
  workflow_call:
    inputs:
      environment:
        description: |
          Target deployment environment.
          Valid values: staging, production
        type: string
        required: true
```

### 3. Provide Defaults

```yaml
inputs:
  node-version:
    type: string
    default: '20'
```

### 4. Use `secrets: inherit` Carefully

Only inherit when necessary; prefer explicit secrets for security.

### 5. Keep Workflows Focused

Create small, composable workflows rather than monolithic ones.

## Reusable vs Composite

| Feature | Reusable Workflow | Composite Action |
|---------|-------------------|------------------|
| Scope | Entire jobs | Single step |
| Location | `.github/workflows/` | `action.yml` |
| Matrix | Yes | No |
| Secrets | Native support | Must pass as inputs |
| Outputs | Workflow-level | Action-level |
| Best for | Complex CI/CD | Reusable step logic |
