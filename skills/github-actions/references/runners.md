# Runners Reference

Complete reference for GitHub Actions runners.

## GitHub-Hosted Runners

### Standard Runners

| Label | vCPUs | RAM | Storage | OS |
|-------|-------|-----|---------|---|
| `ubuntu-latest` | 4 | 16 GB | 14 GB | Ubuntu 24.04 |
| `ubuntu-24.04` | 4 | 16 GB | 14 GB | Ubuntu 24.04 |
| `ubuntu-22.04` | 4 | 16 GB | 14 GB | Ubuntu 22.04 |
| `macos-latest` | 3 (M1) | 7 GB | 14 GB | macOS 14 |
| `macos-14` | 3 (M1) | 7 GB | 14 GB | macOS 14 (Sonoma) |
| `macos-13` | 4 | 14 GB | 14 GB | macOS 13 (Ventura) |
| `windows-latest` | 4 | 16 GB | 14 GB | Windows Server 2022 |
| `windows-2022` | 4 | 16 GB | 14 GB | Windows Server 2022 |
| `windows-2019` | 4 | 16 GB | 14 GB | Windows Server 2019 |

### Larger Runners (Teams/Enterprise)

| Label | vCPUs | RAM |
|-------|-------|-----|
| `ubuntu-latest-4-cores` | 4 | 16 GB |
| `ubuntu-latest-8-cores` | 8 | 32 GB |
| `ubuntu-latest-16-cores` | 16 | 64 GB |
| `ubuntu-latest-32-cores` | 32 | 128 GB |
| `ubuntu-latest-64-cores` | 64 | 256 GB |

Similar patterns for `macos-*` and `windows-*`.

### ARM Runners

| Label | Architecture |
|-------|-------------|
| `ubuntu-24.04-arm` | ARM64 |
| `macos-14` | ARM64 (M1) |

### GPU Runners (Enterprise)

Available for ML workloads. Contact GitHub for access.

## Runner Selection

### Basic Selection

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
```

### Specific Version

```yaml
jobs:
  build:
    runs-on: ubuntu-22.04  # Pin to specific version
```

### Matrix of Runners

```yaml
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
```

### Self-Hosted Runner Labels

```yaml
jobs:
  build:
    runs-on: self-hosted

  gpu-job:
    runs-on: [self-hosted, linux, gpu]

  specific:
    runs-on: [self-hosted, linux, x64, production]
```

## Pre-installed Software

### Ubuntu

- Docker, Docker Compose
- Node.js (multiple versions)
- Python (multiple versions)
- Go, Rust, Ruby
- Java (multiple JDKs)
- .NET SDK
- PHP
- Git, Git LFS
- AWS CLI, Azure CLI, Google Cloud SDK
- Terraform, Packer
- kubectl, Helm
- Chrome, Firefox, Edge

### macOS

- Xcode (multiple versions)
- Homebrew
- Node.js, Python, Ruby
- CocoaPods, Carthage
- fastlane
- Chrome, Safari, Firefox

### Windows

- Visual Studio
- .NET Framework and .NET SDK
- Node.js, Python
- PowerShell
- Chocolatey
- Docker Desktop
- Chrome, Edge, Firefox

## Runner Context

Access runner information:

```yaml
steps:
  - run: |
      echo "Runner name: ${{ runner.name }}"
      echo "OS: ${{ runner.os }}"           # Linux, Windows, macOS
      echo "Arch: ${{ runner.arch }}"       # X64, ARM64
      echo "Temp: ${{ runner.temp }}"
      echo "Tool cache: ${{ runner.tool_cache }}"
```

## Self-Hosted Runners

### Setup

1. Navigate to repository Settings → Actions → Runners
2. Click "New self-hosted runner"
3. Follow instructions for your OS

### Runner Application

```bash
# Download
curl -o actions-runner-linux-x64.tar.gz -L \
  https://github.com/actions/runner/releases/download/v2.XXX.X/actions-runner-linux-x64-2.XXX.X.tar.gz

# Extract
tar xzf actions-runner-linux-x64.tar.gz

# Configure
./config.sh --url https://github.com/org/repo --token XXXXXX

# Run
./run.sh

# Or install as service
sudo ./svc.sh install
sudo ./svc.sh start
```

### Labels

Default labels:
- `self-hosted`
- OS: `linux`, `windows`, `macos`
- Arch: `x64`, `arm`, `arm64`

Custom labels:
```yaml
runs-on: [self-hosted, linux, gpu, cuda-11]
```

### Runner Groups (Enterprise)

Organize runners and control access:

```yaml
jobs:
  production:
    runs-on:
      group: production-runners
      labels: [linux, x64]
```

## Container Jobs

Run entire job in a container:

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    container:
      image: node:20-alpine
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
      env:
        NODE_ENV: production
      ports:
        - 80
      volumes:
        - /data:/data
      options: --cpus 2 --memory 4g
```

### Service Containers

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test
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

Access services:
```yaml
steps:
  - run: |
      # In container job, use service name
      psql -h postgres -U postgres

      # In non-container job, use localhost
      psql -h localhost -p 5432 -U postgres
```

## Performance Tips

### Choose Appropriate Runner

```yaml
# Fast builds: use Linux
runs-on: ubuntu-latest

# macOS/iOS builds: use macOS
runs-on: macos-latest

# Windows-specific: use Windows
runs-on: windows-latest
```

### Larger Runners for CI

```yaml
# Heavy builds benefit from more cores
runs-on: ubuntu-latest-8-cores
```

### Cache Dependencies

```yaml
- uses: actions/cache@v5
  with:
    path: ~/.npm
    key: npm-${{ runner.os }}-${{ hashFiles('**/package-lock.json') }}
```

### Parallel Matrix Jobs

```yaml
strategy:
  fail-fast: false
  max-parallel: 4  # Limit concurrent jobs
  matrix:
    os: [ubuntu-latest, macos-latest, windows-latest]
```

## Security

### Self-Hosted Runner Security

- Never use self-hosted runners for public repos
- Use runner groups to isolate by trust level
- Keep runner software updated
- Use ephemeral runners when possible
- Limit network access

### GitHub-Hosted Runner Security

- Fresh VM for each job
- No persistence between jobs
- Automatic security updates
- Isolated network

## Troubleshooting

### Check Available Space

```yaml
- run: df -h
```

### Check Installed Software

```yaml
- run: |
    node --version
    python --version
    docker --version
```

### Debug Runner Issues

```yaml
- run: |
    echo "Runner: ${{ runner.name }}"
    echo "OS: ${{ runner.os }}"
    echo "Arch: ${{ runner.arch }}"
    uname -a
    cat /etc/os-release || true
```

### Runner Logs

For self-hosted runners, check:
- `_diag/` directory for logs
- `svc.sh status` for service status
