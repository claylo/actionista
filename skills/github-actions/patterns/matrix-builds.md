# Matrix Builds

Run jobs across multiple configurations in parallel.

## Basic Matrix

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
        node: [18, 20, 22]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v6
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm test
```

This creates 9 jobs (3 OS × 3 Node versions).

## Matrix Options

### `fail-fast`

Control whether to cancel other jobs on failure:

```yaml
strategy:
  fail-fast: false  # Continue all jobs even if one fails
  matrix:
    os: [ubuntu-latest, macos-latest]
```

Default is `true` (cancel on first failure).

### `max-parallel`

Limit concurrent jobs:

```yaml
strategy:
  max-parallel: 2  # Only 2 jobs at a time
  matrix:
    config: [a, b, c, d, e]
```

Useful for rate-limited APIs or shared resources.

## Include and Exclude

### `include`

Add specific combinations or extra variables:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    node: [18, 20]
    include:
      # Add extra variable to specific combo
      - os: ubuntu-latest
        node: 20
        coverage: true

      # Add entirely new combo
      - os: windows-latest
        node: 22
        experimental: true
```

### `exclude`

Remove specific combinations:

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
    node: [18, 20, 22]
    exclude:
      - os: macos-latest
        node: 18
      - os: windows-latest
        node: 18
```

## Using Matrix Values

### In `runs-on`

```yaml
runs-on: ${{ matrix.os }}
```

### In Step Inputs

```yaml
- uses: actions/setup-node@v4
  with:
    node-version: ${{ matrix.node }}
```

### In Conditionals

```yaml
- name: Upload coverage
  if: matrix.coverage == true
  run: npm run coverage
```

### In Environment Variables

```yaml
env:
  NODE_ENV: ${{ matrix.environment }}
```

## Matrix Context

Access matrix info:

```yaml
steps:
  - run: |
      echo "OS: ${{ matrix.os }}"
      echo "Node: ${{ matrix.node }}"
      echo "Job index: ${{ strategy.job-index }}"
      echo "Job total: ${{ strategy.job-total }}"
```

## Dynamic Matrix

Generate matrix from previous job:

```yaml
jobs:
  generate:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - id: set-matrix
        run: |
          echo 'matrix={"node":[18,20,22],"os":["ubuntu-latest","macos-latest"]}' >> $GITHUB_OUTPUT

  build:
    needs: generate
    strategy:
      matrix: ${{ fromJSON(needs.generate.outputs.matrix) }}
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
```

### From File

```yaml
jobs:
  generate:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - uses: actions/checkout@v6
      - id: set-matrix
        run: |
          echo "matrix=$(cat matrix.json)" >> $GITHUB_OUTPUT

  build:
    needs: generate
    strategy:
      matrix: ${{ fromJSON(needs.generate.outputs.matrix) }}
```

## Complex Matrix Examples

### Language Versions with Features

```yaml
strategy:
  matrix:
    python: ['3.10', '3.11', '3.12']
    django: ['4.2', '5.0']
    include:
      - python: '3.12'
        django: '5.0'
        coverage: true
    exclude:
      - python: '3.10'
        django: '5.0'  # Django 5 requires 3.10+
```

### Platform-Specific Configuration

```yaml
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        target: x86_64-unknown-linux-gnu
        artifact: linux-amd64

      - os: macos-latest
        target: x86_64-apple-darwin
        artifact: darwin-amd64

      - os: macos-latest
        target: aarch64-apple-darwin
        artifact: darwin-arm64

      - os: windows-latest
        target: x86_64-pc-windows-msvc
        artifact: windows-amd64.exe
```

### Browser Testing

```yaml
strategy:
  fail-fast: false
  matrix:
    browser: [chrome, firefox, edge]
    include:
      - browser: chrome
        driver: chromedriver
      - browser: firefox
        driver: geckodriver
      - browser: edge
        driver: msedgedriver
```

## Matrix with Services

```yaml
jobs:
  test:
    strategy:
      matrix:
        database: [postgres, mysql]
        include:
          - database: postgres
            image: postgres:15
            port: 5432
            env:
              POSTGRES_PASSWORD: test
          - database: mysql
            image: mysql:8
            port: 3306
            env:
              MYSQL_ROOT_PASSWORD: test

    runs-on: ubuntu-latest
    services:
      db:
        image: ${{ matrix.image }}
        ports:
          - ${{ matrix.port }}:${{ matrix.port }}
        env: ${{ matrix.env }}
```

## Matrix Outputs

Collect outputs from matrix jobs:

```yaml
jobs:
  build:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest]
    outputs:
      result-${{ matrix.os }}: ${{ steps.build.outputs.result }}
    steps:
      - id: build
        run: echo "result=success" >> $GITHUB_OUTPUT

  summary:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - run: echo "All builds completed"
```

## Performance Tips

### Parallelize Heavy Jobs

```yaml
strategy:
  fail-fast: false
  max-parallel: 4
  matrix:
    shard: [1, 2, 3, 4]
steps:
  - run: npm test -- --shard=${{ matrix.shard }}/4
```

### Cache Per Matrix

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: npm-${{ matrix.os }}-${{ matrix.node }}-${{ hashFiles('**/package-lock.json') }}
```

### Skip Slow Combinations

```yaml
- name: Integration tests
  if: matrix.os == 'ubuntu-latest'  # Only run on one OS
  run: npm run test:integration
```

## Best Practices

1. **Use `fail-fast: false`** when jobs are independent
2. **Cache per-matrix** to avoid conflicts
3. **Use `include` for platform-specific config** rather than conditionals
4. **Limit matrix size** to stay under 256 job limit
5. **Consider dynamic matrices** for complex requirements
