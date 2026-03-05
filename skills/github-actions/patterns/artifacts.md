# Artifacts

Share files between jobs and persist build outputs.

## Upload Artifact

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-output
    path: dist/
```

### Options

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: my-artifact
    path: |
      dist/
      build/
      !build/**/*.map
    retention-days: 5           # Default: 90
    compression-level: 6        # 0-9, default 6
    overwrite: true             # Replace existing
    if-no-files-found: error    # error, warn, ignore
    include-hidden-files: false
```

## Download Artifact

```yaml
- uses: actions/download-artifact@v4
  with:
    name: build-output
    path: ./downloaded
```

### Download All Artifacts

```yaml
- uses: actions/download-artifact@v4
  # No name = download all
```

### Download by Pattern

```yaml
- uses: actions/download-artifact@v4
  with:
    pattern: build-*
    merge-multiple: true  # Merge into single directory
```

## Cross-Job Sharing

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  test:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - run: npm test

  deploy:
    needs: [build, test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
      - run: ./deploy.sh
```

## Matrix Artifacts

### Unique Names Per Matrix

```yaml
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - run: npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: build-${{ matrix.os }}
          path: dist/
```

### Collect All Matrix Artifacts

```yaml
jobs:
  build:
    # ... matrix job ...

  collect:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: build-*
          path: all-builds/
          merge-multiple: true
```

## Path Patterns

### Multiple Paths

```yaml
path: |
  dist/
  build/
  output.txt
```

### Exclusions

```yaml
path: |
  dist/
  !dist/**/*.map
  !dist/**/*.d.ts
```

### Wildcards

```yaml
path: |
  **/coverage/
  **/*.log
```

## Retention

### Custom Retention

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: logs
    path: logs/
    retention-days: 1  # Minimum 1, maximum 90
```

### Default Retention

- Public repos: 90 days
- Private repos: 400 days (configurable in settings)

## Artifact Size

### Compression Level

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: binaries
    path: target/release/
    compression-level: 9  # Maximum compression
```

Levels:
- 0: No compression (fastest)
- 6: Default
- 9: Maximum compression (smallest)

## Cross-Workflow Artifacts

### Download from Another Run

```yaml
- uses: actions/download-artifact@v4
  with:
    name: build-output
    github-token: ${{ secrets.GITHUB_TOKEN }}
    repository: owner/repo
    run-id: ${{ github.event.workflow_run.id }}
```

### Using dawidd6/action-download-artifact

For more flexibility:

```yaml
- uses: dawidd6/action-download-artifact@v3
  with:
    workflow: build.yml
    name: my-artifact
    branch: main
```

## Common Patterns

### Build Once, Test Many

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - run: npm ci && npm run build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/

  test-unit:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - run: npm run test:unit

  test-e2e:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - run: npm run test:e2e
```

### Release Binaries

```yaml
jobs:
  build:
    strategy:
      matrix:
        include:
          - os: ubuntu-latest
            artifact: linux-x64
          - os: macos-latest
            artifact: darwin-x64
          - os: windows-latest
            artifact: windows-x64.exe
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v6
      - run: cargo build --release
      - uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.artifact }}
          path: target/release/myapp*

  release:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          pattern: '*'
          path: binaries/
          merge-multiple: true
      - uses: softprops/action-gh-release@v2
        with:
          files: binaries/*
```

### Test Reports

```yaml
- uses: actions/upload-artifact@v4
  if: always()  # Upload even on test failure
  with:
    name: test-results
    path: |
      test-results/
      coverage/
    retention-days: 14
```

## Best Practices

### 1. Use Descriptive Names

```yaml
name: build-${{ github.sha }}-${{ matrix.os }}
```

### 2. Set Appropriate Retention

```yaml
retention-days: 1   # CI artifacts
retention-days: 30  # Release artifacts
retention-days: 90  # Compliance artifacts
```

### 3. Compress Large Artifacts

```yaml
compression-level: 9
```

### 4. Fail on Missing Files

```yaml
if-no-files-found: error
```

### 5. Upload on Failure

```yaml
- uses: actions/upload-artifact@v4
  if: failure()
  with:
    name: failure-logs
    path: logs/
```

## Troubleshooting

### "No files found"

- Check path is correct
- Verify files exist after build step
- Check glob patterns

### "Artifact size exceeds limit"

- Use higher compression
- Exclude unnecessary files
- Split into multiple artifacts

### "Artifact not found"

- Verify artifact name matches exactly
- Check job dependencies (`needs`)
- Verify artifact hasn't expired
