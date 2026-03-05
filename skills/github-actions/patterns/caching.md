# Caching

Speed up workflows by caching dependencies and build outputs.

## Built-in Caching

Many setup actions include built-in caching:

### Node.js

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: '20'
    cache: 'npm'           # Also: 'yarn', 'pnpm'
    cache-dependency-path: '**/package-lock.json'
```

### Python

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: '3.12'
    cache: 'pip'           # Also: 'pipenv', 'poetry'
    cache-dependency-path: '**/requirements*.txt'
```

### Go

```yaml
- uses: actions/setup-go@v5
  with:
    go-version: '1.22'
    cache: true            # Caches go mod and build cache
```

### Java

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'         # Also: 'gradle', 'sbt'
```

## actions/cache

For custom caching needs:

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
```

### Required Inputs

| Input | Description |
|-------|-------------|
| `path` | Path(s) to cache |
| `key` | Unique cache key |

### Optional Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `restore-keys` | Fallback keys | None |
| `enableCrossOsArchive` | Cross-OS restore | `false` |
| `fail-on-cache-miss` | Fail if no match | `false` |
| `lookup-only` | Check without restore | `false` |
| `save-always` | Save even on failure | `false` |

## Cache Keys

### Hash Files

```yaml
key: npm-${{ hashFiles('**/package-lock.json') }}
key: pip-${{ hashFiles('**/requirements.txt', '**/setup.py') }}
key: cargo-${{ hashFiles('**/Cargo.lock') }}
key: go-${{ hashFiles('**/go.sum') }}
```

### OS-Specific

```yaml
key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
```

### Branch-Specific

```yaml
key: build-${{ github.ref }}-${{ github.sha }}
```

### Matrix-Specific

```yaml
key: npm-${{ matrix.os }}-${{ matrix.node }}-${{ hashFiles('**/package-lock.json') }}
```

## Restore Keys

Fallback to partial matches:

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
    restore-keys: |
      npm-${{ runner.os }}-
      npm-
```

Keys are tried in order until a prefix match is found.

## Cache Hit Detection

```yaml
- uses: actions/cache@v5
  id: cache
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}

- name: Install dependencies
  if: steps.cache.outputs.cache-hit != 'true'
  run: npm ci
```

## Common Cache Paths

### Node.js

```yaml
path: |
  ~/.npm
  node_modules
```

### pnpm

```yaml
- run: pnpm store path
  id: pnpm-store

- uses: actions/cache@v5
  with:
    path: ${{ steps.pnpm-store.outputs.stdout }}
    key: pnpm-${{ hashFiles('**/pnpm-lock.yaml') }}
```

### Python

```yaml
path: |
  ~/.cache/pip
  .venv
```

### Rust

```yaml
path: |
  ~/.cargo/bin/
  ~/.cargo/registry/index/
  ~/.cargo/registry/cache/
  ~/.cargo/git/db/
  target/
```

Or use Swatinem/rust-cache:

```yaml
- uses: Swatinem/rust-cache@v2
```

### Go

```yaml
path: |
  ~/go/pkg/mod
  ~/.cache/go-build
```

### Gradle

```yaml
path: |
  ~/.gradle/caches
  ~/.gradle/wrapper
```

### Maven

```yaml
path: ~/.m2/repository
```

## Multiple Paths

```yaml
- uses: actions/cache@v5
  with:
    path: |
      ~/.npm
      ~/.cache/Cypress
      node_modules
    key: deps-${{ hashFiles('**/package-lock.json') }}
```

## Save on Failure

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}
    save-always: true  # Save even if job fails
```

## Separate Save/Restore

For more control:

```yaml
- uses: actions/cache/restore@v5
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}

# ... do work ...

- uses: actions/cache/save@v5
  if: always()
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}
```

## Cache Limits

- **Size limit**: 10 GB per repository
- **Retention**: 7 days unused
- **Eviction**: LRU when limit reached

## Cross-OS Caching

Enable for portable caches:

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}
    enableCrossOsArchive: true  # Works across OS
```

## Best Practices

### 1. Use Specific Keys

```yaml
# Good: Specific to content
key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}

# Bad: Too broad
key: npm-cache
```

### 2. Include OS in Key

```yaml
key: build-${{ runner.os }}-${{ hashFiles('**/*.go') }}
```

### 3. Use Restore Keys

```yaml
restore-keys: |
  npm-${{ runner.os }}-
  npm-
```

### 4. Cache After Install

```yaml
- uses: actions/cache@v5
  id: cache
  with:
    path: node_modules
    key: modules-${{ hashFiles('**/package-lock.json') }}

- run: npm ci
  if: steps.cache.outputs.cache-hit != 'true'
```

### 5. Don't Cache Source

Cache dependencies, not source code.

### 6. Consider Build Outputs

```yaml
- uses: actions/cache@v5
  with:
    path: |
      ~/.npm
      dist/       # Cached build output
    key: build-${{ github.sha }}
```

## Troubleshooting

### Cache Not Restoring

- Check key matches exactly
- Verify path exists
- Check for typos in hashFiles pattern

### Cache Size Too Large

- Cache only essential directories
- Exclude unnecessary files:
  ```yaml
  path: |
    node_modules
    !node_modules/.cache
  ```

### Cache Key Conflicts

- Include more context in key (OS, matrix values)
- Use more specific hash patterns
