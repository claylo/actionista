# Composite Actions

Create reusable actions composed of multiple steps.

## Basic Structure

```yaml
# action.yml
name: 'My Composite Action'
description: 'Does useful things'

inputs:
  version:
    description: 'Version to install'
    required: true
    default: 'latest'

outputs:
  result:
    description: 'The result'
    value: ${{ steps.main.outputs.result }}

runs:
  using: 'composite'
  steps:
    - name: Setup
      shell: bash
      run: echo "Setting up version ${{ inputs.version }}"

    - name: Main step
      id: main
      shell: bash
      run: |
        echo "Doing work..."
        echo "result=success" >> $GITHUB_OUTPUT
```

## Using Composite Actions

### From Repository

```yaml
steps:
  - uses: owner/action-repo@v1
    with:
      version: '2.0'
```

### Local Action

```yaml
# Place in .github/actions/my-action/action.yml
steps:
  - uses: ./.github/actions/my-action
    with:
      version: '2.0'
```

## Inputs and Outputs

### Inputs

```yaml
inputs:
  required-input:
    description: 'This is required'
    required: true

  optional-input:
    description: 'This has a default'
    required: false
    default: 'default-value'

  boolean-input:
    description: 'A boolean flag'
    required: false
    default: 'false'  # Note: Still a string
```

Access in steps:

```yaml
steps:
  - run: echo "${{ inputs.required-input }}"
    shell: bash
```

### Outputs

```yaml
outputs:
  artifact-path:
    description: 'Path to artifact'
    value: ${{ steps.build.outputs.path }}

  version:
    description: 'Detected version'
    value: ${{ steps.version.outputs.value }}

runs:
  using: 'composite'
  steps:
    - id: build
      shell: bash
      run: echo "path=dist/" >> $GITHUB_OUTPUT

    - id: version
      shell: bash
      run: echo "value=1.0.0" >> $GITHUB_OUTPUT
```

## Shell Requirement

Every `run` step must specify a shell:

```yaml
steps:
  - run: echo "Hello"
    shell: bash

  - run: Write-Output "Hello"
    shell: pwsh

  - run: print("Hello")
    shell: python
```

## Using Other Actions

Composite actions can use other actions:

```yaml
runs:
  using: 'composite'
  steps:
    - uses: actions/checkout@v6

    - uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - run: npm ci
      shell: bash
```

## Working Directory

```yaml
steps:
  - run: npm install
    shell: bash
    working-directory: ${{ inputs.path }}
```

## Environment Variables

### Step-Level

```yaml
steps:
  - run: echo "$MY_VAR"
    shell: bash
    env:
      MY_VAR: ${{ inputs.my-input }}
```

### Set for Subsequent Steps

```yaml
steps:
  - run: echo "MY_VAR=value" >> $GITHUB_ENV
    shell: bash

  - run: echo "$MY_VAR"  # Available
    shell: bash
```

## Conditionals

```yaml
steps:
  - run: echo "Running on Linux"
    if: runner.os == 'Linux'
    shell: bash

  - run: echo "Running on Windows"
    if: runner.os == 'Windows'
    shell: pwsh
```

## Error Handling

### Continue on Error

```yaml
steps:
  - run: ./might-fail.sh
    shell: bash
    continue-on-error: true

  - run: echo "This still runs"
    shell: bash
```

### Check Outcome

```yaml
steps:
  - id: check
    run: ./check.sh
    shell: bash
    continue-on-error: true

  - if: steps.check.outcome == 'failure'
    run: echo "Check failed"
    shell: bash
```

## Complete Example

```yaml
# .github/actions/setup-project/action.yml
name: 'Setup Project'
description: 'Install dependencies and build'

inputs:
  node-version:
    description: 'Node.js version'
    required: false
    default: '20'
  install-command:
    description: 'Install command'
    required: false
    default: 'npm ci'
  build:
    description: 'Run build'
    required: false
    default: 'true'

outputs:
  cache-hit:
    description: 'Whether cache was hit'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: 'composite'
  steps:
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ inputs.node-version }}

    - name: Get npm cache directory
      id: npm-cache
      shell: bash
      run: echo "dir=$(npm config get cache)" >> $GITHUB_OUTPUT

    - name: Cache dependencies
      id: cache
      uses: actions/cache@v4
      with:
        path: ${{ steps.npm-cache.outputs.dir }}
        key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
        restore-keys: npm-${{ runner.os }}-

    - name: Install dependencies
      shell: bash
      run: ${{ inputs.install-command }}

    - name: Build
      if: inputs.build == 'true'
      shell: bash
      run: npm run build
```

Usage:

```yaml
steps:
  - uses: actions/checkout@v6
  - uses: ./.github/actions/setup-project
    with:
      node-version: '22'
```

## Testing Composite Actions

### Local Testing

```yaml
# .github/workflows/test-action.yml
name: Test Action

on:
  push:
    paths:
      - '.github/actions/my-action/**'

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6

      - uses: ./.github/actions/my-action
        id: test
        with:
          input: 'test-value'

      - name: Verify output
        run: |
          if [ "${{ steps.test.outputs.result }}" != "expected" ]; then
            echo "Unexpected output"
            exit 1
          fi
```

## Best Practices

### 1. Always Specify Shell

```yaml
steps:
  - run: echo "Hello"
    shell: bash  # Required!
```

### 2. Use Descriptive Names

```yaml
name: 'Setup and Build Project'
description: 'Checks out code, installs dependencies, and builds the project'
```

### 3. Provide Defaults

```yaml
inputs:
  version:
    default: 'latest'
```

### 4. Document Outputs

```yaml
outputs:
  build-path:
    description: 'Path to the build output directory'
```

### 5. Handle Multiple OS

```yaml
steps:
  - run: ./script.sh
    if: runner.os != 'Windows'
    shell: bash

  - run: .\script.ps1
    if: runner.os == 'Windows'
    shell: pwsh
```

### 6. Version Your Actions

Tag releases for stable usage:

```bash
git tag -a v1.0.0 -m "Release v1.0.0"
git push origin v1.0.0
```

## Composite vs Reusable Workflows

| Feature | Composite Action | Reusable Workflow |
|---------|------------------|-------------------|
| Scope | Single step | Entire jobs |
| Location | `action.yml` | `.github/workflows/` |
| Matrix | No | Yes |
| Services | No | Yes |
| Secrets | Pass as inputs | Native support |
| Best for | Reusable step logic | Complex CI/CD |
